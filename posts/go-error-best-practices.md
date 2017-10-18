

## General good practices

* Return an error or log it, don't do both

## Good practices

### Assert behaviour

An `error` can be anything that has a method called `Error` that accepts no parameters and returns a string.  This allows for the following, which satisfies the Go Proverb "Don't just check errors, handle them gracefully" by allowing the `HttpError` to decide what types of HTTP errors can be considered recoverable or not.


``` go
type recoverable interface {
    Recoverable() bool
}

type HttpError struct {
    Status int
}

func (err HttpError) Recoverable() {
    return err.Status < http.StatusBadRequest
}

func IsRecoverable(err error) bool {
    rec, ok := err.(recoverable)
    return ok && rec.Recoverable()
}
```

### Opaque errors

Rather than forcing your API consumers to tightly-couple their code to yours, you can return opaque errors (i.e. errors that are just for humans).  By doing this, you gain flexibility and they can remain decoupled but still know something went wrong. 

``` go
func OpenFile(fileName string) error {
    ...
    return fmt.Errorf("error opening file '%s'", fileName)
}
```

Allowing for:

``` go
if err := pkg.OpenFile(); err != nil {
    // log or return to caller
}
```

## Bad practices

### Sentinel errors

A sentinel error is a known error that's exposed by your API.  While this is usually quite useful *internally* to your code, exposing them leads to consumers **A)** tightly-coupling their code to yours and **B)** asserting against error *type* and not error *behaviour* (which should always be preferred).

``` go
var ErrBoom = errors.New("something went boom")
```

Leading to:

``` go
if err == pkg.ErrBoom {
}
```

### Error types

Error types are just structs containing information on the error that occurred.  Their pros and cons are the same as sentinel errors but they can contain more information:

``` go
type ErrBoom struct {
    Stuff     string
    MoreStuff int
}
```

Leading to:

``` go
if boom, ok := err.(ErrBoom); ok {
    fmt.Println(boom.Stuff, boom.MoreStuff)
}
```

### Assert against err.Error()

This is the worst of the bad practices.  Go's errors are just values, so they can be passed around, wrapped or compared easily.  Therefore, under no circumstances should the following seem acceptible to you *in non-test code* because **A)** the error message might change and **B)** magic strings are a known gateway to hell.

``` go
if strings.HasPrefix(err.Error(), "error opening file") {
}
```