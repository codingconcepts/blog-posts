The benefits of thorough testing can't be understated, especially when it comes to anything that's in any way magical.

For a language that avoids magic, I'd consider Go's named return values and defer pretty magical, *when used together*.  A big assumption and a very subtle syntax oversight and this magic would have lead to a **serious** production issue had it not been for a test that saved the day.

In my latest project, I'm performing a lot of database reads/writes, so require database atomicity.  Working in a RDBMS/NewSQL environment, that naturally means reaching for transactions.  Rather than rolling a new transaction manually every time I need one, I'm using a helper function that wraps the creation and commit/rollback of all transactions.  Here's a severely cut-down, version of my final implementation, which contains a **critical** bug.

Without cheating, or wishing to sound like a disgraced children's TV presenter, can you see what it is yet?

```go
func executeTx(db *sql.DB, actions func(*sql.Tx) error) error {
	tx, err := db.Begin()
	if err != nil {
		return err
	}

	defer func() {
		if r := recover(); r != nil {
			tx.Rollback()
			panic(r)
		} else if err != nil {
			tx.Rollback()
		} else {
			tx.Commit()
		}
	}()

	return actions(tx)
}
```

Let's count the number of rows in `some_table` before we start to play with it:

```bash
+----------+
| count(*) |
+----------+
|      100 |
+----------+
```

In the following example, we delete everything from `some_table`, then return an error, to force our transaction rollback code to kick in and prevent the deletion from being committed:

```Go
err := executeTx(db, func(tx *sql.Tx) error {
	_, err := tx.Exec("DELETE FROM `some_table`")
	if err != nil {
		return err
	}

	return errors.New("oh noes")
})
```

Nothing spooky going going on here.  Let's count the number of rows in `some_table` again, to make sure that our transaction code is doing what it's meant to be doing:

```bash
+----------+
| count(*) |
+----------+
|        0 |
+----------+
```

#### OH MY SHIT!

We've just ensured that the exact opposite of what we want to happen, will happen, every time an error occurs.  Now imagine this behaviour had gone unchecked and we'd released it into the wild.

Happily, this code went nowhere near a production database.  Here's the unit test that saved the day (error handling omitted for your viewing pleasure):

```go
package main

import (
	"database/sql"
	"errors"
	"log"
	"testing"

	"gopkg.in/DATA-DOG/go-sqlmock.v1"
)

func TestDBRollback(t *testing.T) {
	db, mock, _ := sqlmock.New()

	mock.ExpectBegin()
	mock.ExpectExec("^DELETE FROM (.+)")
	mock.ExpectRollback()

	err := executeTx(db, func(tx *sql.Tx) error {
		tx.Exec("DELETE FROM `some_table`")
		return errors.New("oh noes")
	})

	if err = mock.ExpectationsWereMet(); err != nil {
		log.Fatal(err)
	}
}
```

In this test, I'm asserting that a database transaction is started, has a `DELETE` statement performed against it and is rolled back.  All using the wonderful [sqlmock](gopkg.in/DATA-DOG/go-sqlmock.v1) package from [DataDog](https://www.datadoghq.com/).


```bash
go test -v
=== RUN   TestDBRollback
2018/03/30 17:56:36 there is a remaining expectation which was not matched: ExpectedRollback => expecting transaction Rollback
exit status 1
FAIL    pocs/db/mysql   0.033s
```

##### The problem/solution

If you follow best practice guildlines, or regularly read code online, you might be in the habit of making named return values naked or returning the result of function calls if the return values match your function's return values.

In my case, it was a bit of both that ended up shafting my function, as the original code (see [references](#references)) didn't contain the bug.

The issue was two-fold:  I had converted a named return value to naked return value and made the assumption that the deferred function would be able to close over the error created during `db.Begin()`.

To allow the error be closed over as expected, either of the following will work:

**Named return value**

```go
func executeTx(db *sql.DB, actions func(*sql.Tx) error) (err error)
```

**Capture the error before attempting to return it**

```go
err = actions(tx)
return err
```

If we run the test again with either of these changes in place, we'll see that the rollback is now happening as expected and our data is safe(r):

``` bash
go test -v
=== RUN   TestErrorRollback
--- PASS: TestErrorRollback (0.00s)
PASS
ok      pocs/db/mysql   0.037s
```

##### <a name="references"></a> References

[Helper function inspiration](https://stackoverflow.com/a/23502629/304957)