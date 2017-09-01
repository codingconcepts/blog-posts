Semaphores are like mutexes that allow N threads to access a shared resource instead of just 1.  Go's buffered channels make creating semaphore-like behaviour a doddle.  To create a semaphore that allows 10 threads to concurrently access a shared resource, its simply `make(chan struct{}, 10)` etc.  The only thing our threads need to lock/own is an item of the buffered channel, so it makes sense to use an empty struct, as it uses zero memory.

To keep the interface to our semaphore clean, we'll start by hiding the channel in a struct with a sensible name:

``` go
type semaphore struct {
    channel chan struct{}
}
```

At creation time, we'll want to initialise the semaphore with an underlying channel size, so let's create a `new` method next:

``` go
func newSemaphore(concurrency int) (s *semaphore) {
    return &semaphore{
        channel: make(chan struct{}, concurrency),
    }
}
```

Next, we'll want to ask the semaphore to execute something for us.  The worker doesn't need to hold onto a lock object, so this can be abstracted away too:

``` go
func (s *semaphore) execute(f func()) {
    // Attempt to send an empty struct to the underlying channel. This will
    // block if the channel is full (i.e. there are N concurrent operations
    // already happening).
    s.channel <- struct{}{}

    // By now, we've obtained a 'lock' and can begin execution.  Execute
    // function f and ensure that the empty struct is removed from the
    // channel when we're finished to allow other workers to execute.
    go func() {
        defer func() {
            <-s.channel
        }()
        
        f()
    }()
}
```

Feel free to submit pull requests with any channel tricks you've used, otherwise I'll keep this post updated as I encounter more.