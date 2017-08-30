Channels are Go's implementation of the Tony Hoare's CSP concurrency model.  Rather than reiterate the basics, I'll dive straight into some silly channel tricks that I've found to be useful.

##### Prevent sender from blocking

Sending to a full channel blocks by design.  This prevents fast senders from saturating the channel (and your available memory) but it'll penalise them by forcing them to block, instead of penalising the slow readers.

A good use case for this would be a stock ticker, where only the latest N stock prices can be considered relevant.  We don't want to prevent the delivery of the latest prices to allow a slow reader to keep up with the oldest.

###### Penalising fast senders (default behaviour)

In this example, the price publisher in our stock ticker use case blocks to allow a slow reader to chug through every price at its own pace:

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

Use the following trick to ensure the sender never blocks in the case of a slow reader.  With the use of a `select` statement, we can decide what happens in the event that the channel we're sending to is full.  In this case, we pop an item from the channel and add a new item, keeping only the most recent items, essentially penalising the slow reader.

In this example, the price publisher in our stock ticker use case publishes all of its prices onto the channel, removing the oldest if they haven't been read to allow the latest to be published:

``` go
c := make(chan int, 10)

// Fast sender
go func() {
    for i := 0; i < 20; i++ {
        select {
        case c <- i:c
        default:
            <-c
            c <- i
        }
    }
}()

// Slow subscriber
go func() {
    for o := range c {
        time.Sleep(time.Millisecond * 100)
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