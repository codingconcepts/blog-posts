I develop software with one person in mind.  It's not me (to selfless), it's not my employer (too predictable) and it's not a loved one (you'll see no emotional dedications in my comments).

I develop software exclusively for *Future Me*.  He's just like me in every way, except for the fact that he's just a little bit stupider than I currently am.  Infer from that what you will.

Ergo, the code I tend to find most elegant is code that's almost **Brutalist Obvious**.  That's right, you heard it here first.

##### Cancellation

A Go example that's often batted about is along the following lines.  It executes a given action every 250ms (hard-coded for brevity) and returns a function that closes over the timer and allows the caller to stop the timer (without needing to know anything about the timer):

``` go
func startTicker(action func()) (stop func()) {
    timer := time.NewTicker(time.Millisecond * 250)

    go func() {
        for range timer.C {
            action()
        }
    }()

    return func() {
        timer.Stop()
    }
}
```

Here's the calling code.  It starts the timer, then stops it at some point in the future.

``` go
stop := startTicker(func() {
    fmt.Println("Hello, timer closure thing")
})
defer stop()
```

The jarring proximity of start and stop are a small price to pay for the simplicity you gain in being able to stop the timer without having know know about it.  After all, the stop function might be performing a whole heap of complicated tear-down.  I think the slight weirdness is worth the reward in readability and obviousness.

This pattern is used in the Go stdlib's `Context` package for [cancellations](https://golang.org/pkg/context/#WithCancel).  If you ask for a context `WithCancel`, you'll get a `CancelFunc` function returned:

``` go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

What could be simpler than a named, parameterless function call for cancellation propagation?

##### Synchronisation and scoping
