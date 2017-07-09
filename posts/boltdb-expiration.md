I like a good index.  I especially like the handy [MongoDB TTL indexes](https://docs.mongodb.com/manual/core/index-ttl/) you can apply to date columns to take care of old data automatically.

My latest personal project manages secrets and is backed by [BoltDB](https://github.com/boltdb/bolt), a charming embedded key/value store, written in Go.

Being as simple as it is, BoltDB doesn't have the concept of item expiry.  I spent a while Googling for elegant solutions but ended up (as we all often do), rolling my own.  For this post, I'm using the same `Store` semantics as the [BoltDB documentation](https://github.com/boltdb/bolt) for consistency.

##### tl;dr

The trick is to create two buckets for everything you want to store, one for the items themselves and another to keep track of when they were created.

##### Background

To give a bit of context to the rest of the post, here's how I'm modelling the data.  The following variables are used to identify the buckets:

``` go
var (
    bucketSecret = []byte("secret")
    bucketTTL    = []byte("ttl")
)
```

**secret bucket**

The main bucket my application depends on is the `secret` bucket.  A `secret` is simply a key/value pair comprising a cryptographically secure key that point to some [soon-to-be] encrypted data.  A secret value will be posted to the server and the key will be returned, giving access to the value.  This concept is nothing special and certainly nothing new.

`secret` items are represented as follows:

``` go
type Secret struct {
    Key       string        `json:"key"`
    Value     string        `json:"value"`
}
```

**ttl bucket**

The TTL bucket acts as a sort of audit for secrets.  When a secret is posted to the database, a TTL record that contains the key and the secret's creation time is also written.

Storing the creation time in a separate key/value map allows us to scan the TTL bucket (*whose keys are stored in order and so can be scanned*) in a separate read-only transaction, minimising impact on the `secret` bucket.

##### Writing data

In this snippet, we're storing the secret *and* the TTL records in the same transaction.  The key for the `secret` bucket becomes the *value* for the `TTL` bucket.  The *key* for the `TTL` bucket is a timestamp:

``` go
func (s *Store) Post(secret Secret) (err error) {
    key := []byte(secret.Key)

    return s.db.Update(func(tx *bolt.Tx) (err error) {
        bSecret := tx.Bucket(bucketSecret)
        if err = bSecret.Put(key, []byte(secret.Value)); err != nil {
            return
        }

        bTTL := tx.Bucket(bucketTTL)
        return bTTL.Put([]byte(time.Now().UTC().Format(time.RFC3339Nano)), key)
    })
}
```

##### Care-taking

As BoltDB doesn't have a process for pruning old data, I'm simply kicking off a goroutine to manage that for me.

The end-to-end "sweep" process boils down to this:

1. Find all `TTL` items whose key (a timestamp) is before a given point in time (e.g. 10 minutes old).  This requires a read-only view that doesn't lock the `TTL` bucket.  If we *just* miss a record during a sweep, we'll pick it up on the next one.

1. Store the *values* of each `TTL` item found in-memory.  These are the *keys* of items in the `secret` bucket.

1. Perform a batch delete of all `secret` items using the in-memory keys picked up during the previous step.  This requires a read-write transaction but as it's a batch operation, we're not locking the `secret` bucket for very long.

``` go
func (s *Store) Sweep(maxAge time.Duration) (err error) {
    keys, err := s.GetExpired(maxAge)
    if err != nil || len(keys) == 0 {
        return
    }

    return s.db.Update(func(tx *bolt.Tx) (err error) {
         bSecret := tx.Bucket(bucketSecret)

        for _, key := range keys {
            if err = bSecret.Delete(key); err != nil {
                return
            }
        }
        return
    })
}
```

``` go
func (s *Store) GetExpired(maxAge time.Duration) (keys [][]byte, err error) {
    keys = [][]byte{}
    ttlKeys := [][]byte{}

    err = s.db.View(func(tx *bolt.Tx) error {
        c := tx.Bucket(bucketTTL).Cursor()

        max := []byte(time.Now().UTC().Add(-maxAge).Format(time.RFC3339Nano))
        for k, v := c.First(); k != nil && bytes.Compare(k, max) <= 0; k, v = c.Next() {
            keys = append(keys, v)
            ttlKeys = append(ttlKeys, k)
        }
        return nil
    })

    err = s.db.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket(bucketTTL)
        for _, key := range ttlKeys {
            if err = b.Delete(key); err != nil {
                return err
            }
        }
        return nil
    })

    return
}
```

This solution is by no means perfect.  In the `GetExpired` function, I delete keys to prevent them showing up in subsequent sweeps but this should be wrapped in a transaction so that if the `secret` deletion fails for any reason, we try to delete them again.