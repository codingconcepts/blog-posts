Go's `httptest` package provides a really simple, elegant way to test your HTTP services.  It allows you to create requests to and capture the responses from any server that implements the `http.Handler` interface:

``` go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

There are many ways to acheive this and each is tested slightly differently.  A colleague of mine recently asked for some help getting off the ground with `httptest`, which prompted me to write this.

I'll provide a super simple example of each server flavour, along with an example of how to test it using the `httptest` package.

#### Background

Before we dive in, I'll explain the basics of the `httptest` package, to highlight why it's so useful.

An HTTP handler in Go takes two arguments, an `http.ResponseWriter`, which is an interface and a pointer to an `http.Request`.  Therefore, in order to make a request (at runtime or in tests), you'll need to provide these two as parameters.

Happily, the `httptest` package provides functions that simplify the creation of both for you.  Namely, `httptest.NewRequest` and `httptest.NewRecorder`:

``` go
// Everything required by a ResponseWriter.
resp := httptest.NewRecorder()
```

The call to `httptest.NewRecorder` returns an `httptest.ResponseRecorder` which implements the `http.ResponseWriter`.  This satisfies the first parameter of `ServeHTTP`.

``` go
// Everything required to make a GET request with no body.
req := httptest.NewRequest(http.MethodGet, "/", nil)
```

The call to `httptest.NewRequest` returns a bare-bones instance of an `*http.Request` containing an HTTP verb (in this case "GET"), a target URL (in this case "/") which eventually becomes the `RequestURI` and a body (in this case `nil`).  This satisfies the second parameter of `ServeHTTP`.

Remembering that everything in Go's world of HTTP boils down to a call to `ServeHTTP` with two parameters, you might be thinking that you've now got everything you need to get going and you'd be right.

To keep things simple and still copy/paste friendly, I've removed all package names and stdlib imports from the following examples.

#### Flavours

* [Built-in `http.HandleFunc` handlers](#builtin)
* [Built-in `http.ServeMux` handlers](#builtin-servemux)
* [Wrapped `http.HandleFunc` handlers](#builtin-wrapped)
* [3rd-party mux handlers](#builtin-wrapped)
* [Embedded 3rd-party mux handlers](#builtin-wrapped)

##### <a name="builtin"></a>Built-in handlers

Go's built-in handlers are vanilla HTTP handlers that satisfy the `http.Handler` interface when a call to `http.HandleFunc` is used in conjunction with a function (anonymous or otherwise) with the signature `func(w http.ResponseWriter, r *http.Request)`:

###### Server code

``` go
func main() {
    http.HandleFunc("/", hello)
    http.ListenAndServe(":1234", nil)
}

func hello(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusTeapot)
    w.Write([]byte("hello"))
}
```

###### Test code

``` go
func TestHello(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/", nil)
    resp := httptest.NewRecorder()
    hello(resp, req)

    if resp.Code != http.StatusTeapot {
        t.Fatalf("exp %d but got %d", http.StatusTeapot, resp.Code)
    }
    if resp.Body.String() != "hello" {
        t.Fatalf("exp %s but got %s", "hello", resp.Body.String())
    }
}
```

As this test is only meant to test our `hello` function, we're making a direct call to it and bypassing the HTTP routing to "/" provided by the `*http.Server` that's created in the call to `http.ListenAnServe`.

By now, the only line that might seem unfamiliar to you is the direct call to `hello` with our `httptest.ResponseRecorder` and `*http.Request` parameters.

As the call to `NewRecorder` returns a pointer to an `http.ResponseRecorder`, it'll be mutated inside the call to `hello`.  This is what allows us to interrogate the response after making the request.

The tests pass because we've configured the `hello` function to return a status code of "418" and a body of "hello"; all of which is captured by the `httptest.ResponseRecorder`.

###### <a name="builtin-servemux"></a>Built-in `http.ServerMux` handlers

Go's `http.ServeMux` allows for the configuration of basic 

``` go
func main() {
    mux := http.NewServeMux()
    mux.Handle("/", &handler{})
    http.ListenAndServe(":1234", mux)
}

type handler struct {
}

func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusTeapot)
    w.Write([]byte("hello"))
}

```

###### <a name="builtin-wrapped"></a>Wrapped built-in handlers
###### <a name="mux"></a>Server mux
###### <a name="mux-embedded"></a>Embedded server mux