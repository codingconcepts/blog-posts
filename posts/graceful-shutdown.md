Go's `os`, `os/signal` and `syscall` intercept various [OS signals](https://en.wikipedia.org/wiki/Signal_(IPC)), allowing you to gracefully tear-down your application before it exits.

##### Use cases

* Notifying clients of an impending shutdown
* Drain client connections before shutting down
* Stop long-running goroutines

Here's an example I frequently use in my applications.  It captures the `os.Interrupt` (issued by ctrl+c on the command line) and `syscall.SIGTERM` (equivalent of `kill` but can be handled):

``` go
type server struct{}

func (s *server) start() {
    ...
}

func (s *server) stop() {
    ...
}

func main() {
    s := &server{}

    go teardown(func() {
        s.stop()
    }, os.Interrupt, syscall.SIGTERM)

    s.start()
}

func teardown(f func(), sig ...os.Signal) {
    c := make(chan os.Signal, len(sig))
    signal.Notify(c, sig...)

    <-c
    f()
    os.Exit(0)
}
```