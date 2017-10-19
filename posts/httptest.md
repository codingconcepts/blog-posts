Go's `httptest` package provides a really simple, elegant way to test your HTTP services.  It allows you to create requests to and capture the responses from anything that implements the `http.Handler` interface:

``` go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

There are many ways to acheive this and each is tested slightly different.  A colleague of mine recently asked for some help testing his HTTP server and I'm hoping that this post might help others test theirs, regardless of how they've implemented it.

I'll provide a super simple example of each server flavour, along with an example of how to test it using the `httptest` package.

#### Background

Before we dive in, I'll explain the basics of the `httptest` package, to highlight why it's so useful.

An HTTP handler in Go takes two arguments, an `http.ResponseWriter`, which is an interface that describes how to provide a response to the caller and a pointer to an `http.Request` which contains the request URI, the request body and various other properties.  Therefore, in order to make a request (at runtime or in tests), you'll need to provide these.

Happily, the `httptest` package provides functions that simplify the creation of both.  Namely, `httptest.NewRequest` and `httptest.NewRecorder`.

``` go
// Everything required by a ResponseWriter.
resp := httptest.NewRecorder()
```

The call to `httptest.NewRecorder` returns an `httptest.ResponseRecorder` which implements the `http.ResponseWriter`.  This satisfies the first parameter of `ServeHTTP`.

``` go
// Everything required to make a GET request with no body.
req := httptest.NewRequest(http.MethodGet, "/", nil)
```

The call to `httptest.NewRequest` returns a bare-bones instance of an `*http.Request` containing an HTTP verb (in this case "GET"), a target URL (in this case "/") which becomes the `RequestURI` and a body (in this case `nil`).  This satisfies the second parameter of `ServeHTTP`.

Remembering that everything in Go's world of HTTP boils down to a call to `ServeHTTP` with two parameters, you now have everything you'll need to test an HTTP server.

To keep things simple and still copy/paste friendly, I've removed all package names and imports from the following examples.  In the 3rd-party handler sections, I'm using the Echo server mux, which can be installed as follows:

``` bash
$ go get -u github.com/labstack/echo/...
```

``` go
import (
    "github.com/labstack/echo"
)
```

#### Flavours

* [Built-in `http.HandleFunc` handlers](#builtin)
* [Built-in `http.ServeMux` handlers (direct)](#builtin-servemux-direct)
* [Built-in `http.ServeMux` handlers (routing)](#builtin-servemux-routed)
* [Wrapped `http.HandleFunc` handlers](#builtin-wrapped)
* [3rd-party mux handlers (direct)](#mux-direct)
* [3rd-party mux handlers (routing)](#mux-routing)
* [3rd-party mux handlers (embedded)](#mux-embedded)

##### <a name="builtin"></a>Built-in handlers

Go's built-in handlers are vanilla HTTP handlers that satisfy the `http.Handler` interface when a call to `http.HandleFunc` is used in conjunction with a function (anonymous or otherwise) with the signature `func(w http.ResponseWriter, r *http.Request)`:

###### Server code

``` go
func main() {
    http.HandleFunc("/", hello)
    log.Fatal(http.ListenAndServe(":1234", nil))
}

func hello(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusTeapot)
    w.Write([]byte("hello"))
}
```

This is the quintessential Hello, World! HTTP server example.  It exposes a single endpoint that returns the word "hello" and a status code of 418 (I'm a teapot).

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

In this example, we only want to test our `hello` function, as that contains our business logic.  We therefore make a call directly to it and bypass the HTTP routing to "/" provided by the `*http.Server` (which is configured in the call to `http.ListenAnServe`).

By now, the only line that might seem unfamiliar to you is the direct call to `hello` with our `httptest.ResponseRecorder` and `*http.Request` parameters.

As the call to `NewRecorder` returns a pointer to an `http.ResponseRecorder`, it'll be mutated inside the call to `hello`.  This is what allows us to interrogate the response after making the request.

The tests pass because we've configured the `hello` function to return a status code of "418" and a body of "hello"; all of which is captured by the `httptest.ResponseRecorder`.

##### <a name="builtin-servemux-direct"></a>Built-in `http.ServerMux` direct handlers

Go's `http.ServeMux` is a multiplexer that you can configure routes and handlers against.  It's a step between calling `http.HandleFunc`, which abstracts you away from the `http.Handler` interface and a 3rd-party multiplexer that adds additional layers of abstraction.

###### Server code

``` go
func main() {
    mux := http.NewServeMux()
    mux.Handle("/", &handler{})
    log.Fatal(http.ListenAndServe(":1234", mux))
}

type handler struct {
}

func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusTeapot)
    w.Write([]byte("hello"))
}
```

In this example, we've created a `handler` struct which implements the `http.Handler` interface because of its `ServeHTTP` method receiver.  With this configuration, we could add additional handlers of varying complexity, to handle different routes.

It's worth noting that `mux.Handle("/", &handler{})` is equivalent to `http.Handle("/", &handler{})`, so the test will work against either, without modification.

###### Test code

``` go
func TestHello(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/", nil)
    resp := httptest.NewRecorder()

    h := &handler{}
    h.ServeHTTP(resp, req)

    if resp.Code != http.StatusTeapot {
        t.Fatalf("exp %d but got %d", http.StatusTeapot, resp.Code)
    }
    if resp.Body.String() != "hello" {
        t.Fatalf("exp %s but got %s", "hello", resp.Body.String())
    }
}
```

This time round we're invoking a method receiver on `*handler`, rather than calling the top-level function.  Everything else remains exactly the same.

Once again, the routing logic to "/" isn't being tested, we're just testing the function that'll get called *after* the requset has been routed.  If you'd like routing to form part of your unit tests, you might want to consider using `http.ServeMux` as follows.

##### <a name="builtin-servemux-routed"></a>Built-in `http.ServerMux` routed handlers

###### Server code

``` go
func main() {
    mux := newMux()
    http.ListenAndServe(":1234", mux)
}

type mux struct {
    m *http.ServeMux
}

func newMux() (m *mux) {
    m = &mux{
        m: http.NewServeMux(),
    }

    m.m.HandleFunc("/a", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("hello"))
    })
    m.m.HandleFunc("/b", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("goodbye"))
    })
    return
}

func (m *mux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    m.m.ServeHTTP(w, r)
}
```

In this example, we attach all of our handlers to the `http.ServeMux` using `HandleFunc` and not `Handle`.  This hides the `ServeHTTP` methods for the handlers themselves, allowing the test code to call the `ServeHTTP` method on the `mux` struct; testing our routing logic.

###### Test code

``` go
func TestHello(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/a", nil)
    resp := httptest.NewRecorder()

    m := newMux()
    m.ServeHTTP(resp, req)

    if resp.Body.String() != "hello" {
        t.Fatalf("exp %s but got %s", "hello", resp.Body.String())
    }
}
```

This time, I've bound two handlers, one for route "/a" and another for "/b".  The test will pass, as the handler configured for route "/a" returns hello but change the request's target URI to "/b" and you'll receive the following error:

``` bash
--- FAIL: TestHello (0.00s)
    main_test.go:19: exp hello but got goodbye
```

Our routing is being tested!

##### <a name="builtin-wrapped"></a>Wrapped built-in handlers

The `http.HandlerFunc` type reduces a web request to a simple function call.  Functions are first-class citizens in Go, meaning you can call one `http.HandlerFunc` from another.  The preceding `http.HandlerFunc` is referred to as "middleware".

Go's support for closures gives you the added benefit of closing over variables and keeping them within the scope of your final handler.  This is useful for passing around loggers and database sessions etc.

In the following example, I create a logging handler and chain it onto my `hello` handler:

###### Server code

``` go
func main() {
    l := log.New(os.Stdout, "", 0)
    http.HandleFunc("/", withLogging(hello, l))
    http.ListenAndServe(":1234", nil)
}

func hello(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("hello"))
}

func withLogging(h http.HandlerFunc, l *log.Logger) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        l.Println("before")
        defer l.Println("after")
        h(w, r)
    }
}
```

When the endpoint is hit, we'll execute our logging middleware first, then our `hello` handler.

###### Test code

``` go
func TestHello(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/", nil)
    resp := httptest.NewRecorder()

    buf := &bytes.Buffer{}
    logger := log.New(buf, "", 0)
    middleware := withLogging(hello, logger)
    middleware(resp, req)

    if resp.Body.String() != "hello" {
        t.Fatalf("exp %s but got %s", "hello", resp.Body.String())
    }

    output := buf.String()
    if output != "before\nafter\n" {
        t.Fatalf("exp %s got %s", "before\nafter\n", output)
    }
}
```

In the test, we create a `*log.Logger` and set its `io.Writer` to be a `*bytes.Buffer`, meaning we can interrogate whatever's logged bye our middleware.  We wrap the `hello` handler in the logging middleware and make the request via the logging middleware, just like we would with our `hello` handler directly.

The end-user sees "hello", just as they did before and we're able to call the `String()` method on the logger's writer to assert that our logging middleware wrote the expected log lines.

##### <a name="mux-direct"></a>3rd-party server mux (direct)

Every server mux is different and will require a different approach to testing.  Under the covers however, everything is still just `ServeHTTP`, so the tests will look familiar.

I'm a big fan of the [Echo](https://echo.labstack.com) framework.  Its API made sense to me from the get go and it has become my go-to server mux for web services.  In the case of Echo, handlers are similar to those expected by the stdlib's `http.HandleFunc` function except for the fact that they wrap the `*http.Request` and `http.ResponseWriter` into an `echo.Context`.  This behaviour is shared by most 3rd party multiplexers.

###### Server code

``` go
func main() {
    e := echo.New()
    e.GET("/", hello)
    e.Start(":1234")
}

func hello(c echo.Context) (err error) {
    return c.String(http.StatusTeapot, "hello")
}
```

###### Test code

``` go
func TestHello(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/", nil)
    resp := httptest.NewRecorder()
    
    e := echo.New()
    c := e.NewContext(req, resp)

    if err := hello(c); err != nil {
        t.Fatalf("exp not error not %v", err)
    }
    if resp.Code != http.StatusTeapot {
        t.Fatalf("exp %d but got %d", http.StatusTeapot, resp.Code)
    }
    if resp.Body.String() != "hello" {
        t.Fatalf("exp %s but got %s", "hello", resp.Body.String())
    }
}
```

In this example, the only deviation from standard handler testing is the creation of the wrapping context.  Contexts in the Echo framework are pooled in a `sync.Pool` and created with the very same `NewContext` method that we're using in the test, so it's a great way to test Echo's runtime context creation beaviour too.

Like the majority of tests in this post, this test bypasses routing and shows how to test just the handler we care about.  The following example tests Echo's routing behaviour.

##### <a name="mux-routing"></a>3rd-party server mux (routing)

In order to test routing with the Echo mux, we'll need to be able to call `ServeHTTP` on `*echo.Echo` in our test.  To do this, I've moved its creation into a method called `newMux`, which performs all of the route configuration and provides something for us to call `ServeHTTP` on.

###### Server code

``` go
func main() {
    e := newMux()
    e.Start(":1234")
}

func newMux() (e *echo.Echo) {
    e = echo.New()
    e.GET("/a", hello)
    e.GET("/b", goodbye)
    return
}

func hello(c echo.Context) (err error) {
    return c.String(http.StatusTeapot, "hello")
}

func goodbye(c echo.Context) (err error) {
    return c.String(http.StatusTeapot, "goodbye")
}
```

###### Test code

``` go
func TestHello(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/a", nil)
    resp := httptest.NewRecorder()
    
    mux := newMux()
    mux.ServeHTTP(resp, req)

    if resp.Code != http.StatusTeapot {
        t.Fatalf("exp %d but got %d", http.StatusTeapot, resp.Code)
    }
    if resp.Body.String() != "hello" {
        t.Fatalf("exp %s but got %s", "hello", resp.Body.String())
    }
}
```

##### <a name="mux-embedded"></a>3rd-party server mux (embedded)

###### Server code

In your production code, you're more likely to want to hide the 3rd-party server mux along with other dependencies in a struct of your own.

Testing an embedded Echo mux is just as easy as testing a naked one, you just have to get at its `ServeHTTP` method.  I achieve this by exposing it via a method receiver on my `server` struct:

``` go
func main() {
    s := newServer()
    s.start(":1234")
}

type server struct {
    router *echo.Echo
}

func newServer() (s *server) {
    s = &server{}
    s.router = echo.New()
    s.router.GET("/a", s.hello)
    s.router.GET("/b", s.goodbye)
    return
}

func (s *server) start(addr string) (err error) {
    return s.router.Start(addr)
}

func (s *server) hello(c echo.Context) (err error) {
    return c.String(http.StatusTeapot, "hello")
}

func (s *server) goodbye(c echo.Context) (err error) {
    return c.String(http.StatusTeapot, "goodbye")
}

func (s *server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    s.router.ServeHTTP(w, r)
}
```

It's worth noting that I could have called my `ServeHTTP` method anything, as it's not essential for the tests.  However, it does allows us to use `server` directly in a `http.Handler` and semantically, it's clear to others what's happening so there are definitely benefits.

###### Test code

``` go
func TestHello(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/a", nil)
    resp := httptest.NewRecorder()
    
    s := newServer()
    s.ServeHTTP(resp, req)

    if resp.Code != http.StatusTeapot {
        t.Fatalf("exp %d but got %d", http.StatusTeapot, resp.Code)
    }
    if resp.Body.String() != "hello" {
        t.Fatalf("exp %s but got %s", "hello", resp.Body.String())
    }
}
```

As with the [Built-in `http.ServeMux` routed handlers](#builtin-servemux-routed) tests, I've created two endpoints, one that returns "hello" and one that returns "goodbye" to highlight that we're testing the routing logic as well.  Make a change to target "/b" instead of "/a" and you'll get an error.