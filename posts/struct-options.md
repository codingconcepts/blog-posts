It's important to be explicit about the dependencies in your application.  If your `Server` struct requires access to a database, it makes sense to force consumers to provide it with the means to connect to that database during creation.  Peter Bourgon's brilliant [Go best practices, six years in](https://peter.bourgon.org/go-best-practices-2016/#dependency-management) post makes a brilliant case for other mandatory explicit dependencies and I urge you to read it.

What about optional dependencies?  When starting a new application, I'm always faced with the decision of how best to manage optional dependencies.  There are loads of possible ways but I've settled on the pattern adopted by many open-source projects including the [NATS messaging system](https://nats.io).

Here's an example from the NATS client package, it separates the mandatory dependencies (URL in this case) from the optional ones:

``` go
func Connect(url string, options ...Option) (*Conn, error) {
```

This allows the package consumer to spin up a basic NATS client with zero knowledge of the `Connect` function other than that it takes a URL (which is pretty self-explanatory):

``` go
nats.Connect("nats://localhost:4222")
```

Need to provide root certificates for a TLS client?  No problem, just use one of the variadic `Option` parameters:

``` go
nats.Connect("tls://localhost:4443", nats.RootCAs("./configs/certs/ca.pem"))
```

To make this possible, the `Option` parameters are defined as follows:

``` go
type Option func(*Options) error
```

This is simply a function that takes a pointer to the NATS client's `Options` struct and returns an error if anything went wrong.  The beauty of this approach?  The consumer never actually deals directly with the `Options` struct, they just declaratively build up the constructor with their own overrides for default properties.

Let's have a look at the `Connect` function in its entirety to see how these properties are getting applied:

``` go
func Connect(url string, options ...Option) (*Conn, error) {
    opts := GetDefaultOptions()
    opts.Servers = processUrlString(url)
    for _, opt := range options {
        if err := opt(&opts); err != nil {
            return nil, err
        }
    }
    return opts.Connect()
}
```

The first thing you might notice is how clean the function is; there's nothing spooky going on:

1. Create an `opts` variable to hold our default configuration.  If the consumer hasn't provided any `Option` parameters, we'll have a perfectly sensible NATS default client.
1. NATS clients can learn about new servers via INFO messages from the server but can also connect to any number of servers at start-up.  This line just breaks up a comma-separated list of hosts into individual servers.
1. Foreach of the options the consumer provides, apply them over the default values.
1. Use the `Options` struct to connect to the server and return the connection.

The `GetDefaultOptions` function is similarly non-spooky and does exactly what you'd expect it to do:

``` go
func GetDefaultOptions() Options {
    return Options{
        AllowReconnect:   true,
        MaxReconnect:     DefaultMaxReconnect,
        ReconnectWait:    DefaultReconnectWait,
        Timeout:          DefaultTimeout,
        PingInterval:     DefaultPingInterval,
        MaxPingsOut:      DefaultMaxPingOut,
        SubChanLen:       DefaultMaxChanLen,
        ReconnectBufSize: DefaultReconnectBufSize,
    }
}
```

The eagle-eyed among you may notice the omission of anything TLS related from the `Option` example I gave above...  I'd guess that this is because the default options returned by this function are important to the running of the server.  The server can run without TLS configuration, so its default is null (and hence omitted) but it'd struggle to run without a sensible timeout configured etc.

Here's a very contrived and easily copy/pasted example I've thrown together to allow for some tinkering with the idea (and a [playground](https://play.golang.org/p/aUOjl_PFx_) link):

``` go
package main

import (
    "fmt"
    "log"
)

func main() {
    s, err := newServer(1234, certs("certs"), logs("logs"))
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println(s)
}

type server struct {
    certs   string
    logs    string
    port    int
}

type option func(s *server) error

func certs(value string) option {
    return func(o *server) error {
        o.certs = value
        return nil
    }
}

func logs(value string) option {
    return func(o *server) error {
        o.logs = value
        return nil
    }
}

func newServer(port int, options ...option) (*server, error) {
    s := server{
        port:    port,
        certs: "/etc/certs",
    }

    for _, opt := range options {
        if err := opt(&s); err != nil {
            return nil, err
        }
    }

    return &s, nil
}
```