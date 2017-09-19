Channels are Go's implementation of Tony Hoare's [CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes) concurrency model.  Rather than reiterate the [basics](https://tour.golang.org/concurrency/2), I'll dive straight into some silly channel tricks that I've found to be useful.  Note that these examples are designed to be easy-to-follow, rather than easy-to-copy-straight-into-production.

##### Prevent sender from blocking

Sending to a full channel blocks by design.  This prevents fast senders from saturating a channel (and your available memory) but it'll penalise them by forcing them to block, instead of penalising slow readers for not keeping up.

A good analogy for this would be a stock ticker, where only the latest stock prices can be considered relevant.  We don't want to prevent the delivery of the latest prices just to allow a slow reader to keep up with the oldest because that's silly.  Our bank wants to be too fast to fail.

###### Penalising fast senders

In this example, the price publisher in our stock ticker analogy blocks to allow a slow reader to chug through every price at its own pace:

``` go
c := make(chan int, 10)

// Fast sender
go func() {
    for i := 0; i < 20; i++ {
        c <- i
    }
}()

// Slow reader
go func() {
    for o := range c {
        time.Sleep(time.Millisecond * 500)
        fmt.Println("SUB", o)
    }
}()
```

The above code outputs the following and completes in 10s:

``` bash
$ go run main.go
SUB 0
SUB 1
SUB 2
SUB 3
SUB 4
SUB 5
SUB 6
SUB 7
SUB 8
SUB 9
SUB 10
SUB 11
SUB 12
SUB 13
SUB 14
SUB 15
SUB 16
SUB 17
SUB 18
SUB 19
```

###### Penalising slow readers

The following ensures the sender never blocks in the case of a slow reader.  With the use of a `select` statement, we can decide what happens if the channel we're sending to is full.  In this case, our sender reads off and throws away an item from the channel to make space for a newer item.  This means we only keep the most recent items, essentially penalising the slow reader.

In this example, the price publisher in our stock ticker analogy publishes all of its prices onto the channel, removing the oldest if they haven't been read to allow the latest to be published:

``` go
c := make(chan int, 10)

// Fast sender
go func() {
    for i := 0; i < 20; i++ {
        select {
        case c <- i:
        
        // If we hit the default case, the channel was full.  Remove an
        // item, then resend.  The select statement serialises access to
        // the channels in our case statements, so we're protected from
        // race conditions.
        default:
            <-c
            c <- i
        }
    }
}()

// Slow subscriber
go func() {
    for o := range c {
        time.Sleep(time.Millisecond * 500)
        fmt.Println("SUB", o)
    }
}()
```

The above code outputs the following and completes in 5s:

``` bash
$ go run main.go
SUB #0
SUB #10
SUB #11
SUB #12
SUB #13
SUB #14
SUB #15
SUB #16
SUB #17
SUB #18
SUB #19
```