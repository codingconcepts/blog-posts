Mutexes are an important tool for anyone writing concurrent software.  Go's channels remove the need for explicit synchronisation but not every situation lends itself to the use of a channel.

This post discusses the use of the `sync.Mutex` construct and some things you'll want to consider when using them.

##### Sharing

The `sync` package [godocs](https://golang.org/pkg/sync/) come with the following warning:

```
Values containing the types defined in this package should not be copied.
```

The importance of this can't be understated.

##### Multiple locks

``` go
type bad struct {
    mu        *sync.Mutex
    protected map[string]string
}

```

``` go
type bad struct {
    *sync.Mutex
}
```