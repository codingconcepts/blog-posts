If you've ever needed to kick off multiple goroutines from `func main`, you'd have hopefully noticed that the main goroutine isn't likely to hang around long enough for the other goroutines to finish:

``` go
package main

import (
	"fmt"
	"time"
)

func main() {
	go run(1, "A")
	run(5, "B")
}

func run(iter int, name string) {
	for i := 0; i < iter; i++ {
		time.Sleep(time.Second)
		fmt.Println(name)
	}
}
```

It'll come as no surprise that this program outputs nothing and exits with an exit code of 0.  The nature of goroutines is to be asynchrounous, so while the "A" and "B" goroutines are being scheduled, the main goroutine is running to completion and hence closing our application.

There are many ways to run both the "A" and "B" goroutines to completion, some more involved than others.  Here are a few:

###### <a name="synchronous"></a>Run a goroutine synchronsly

If you're confident that one of your goroutines will run for longer than the other, you could simply call one of the routines synchronously and hope for the best:

``` go
package main

import (
	"fmt"
	"time"
)

func main() {
	go run(1, "A")
	run(5, "B")
}

func run(iter int, name string) {
	for i := 0; i < iter; i++ {
		time.Sleep(time.Second)
		fmt.Println(name)
	}
}
```

``` bash
$ go run main.go
B
A
B
B
B
B
<EXIT 0>
```

This of course falls down if the goroutine you're waiting on takes less time than the other, as the only thing keeping you're application running is the goroutine you're running synchronously:

``` go
go run(5, "A")
run(1, "B")
```

``` bash
$ go run main.go
B
<EXIT 0>
```

...so not a workable solution unless you're running things like long-running web servers.

###### <a name="waitgroup"></a>sync.WaitGroup

A more elegant solution would be to use `sync.WaitGroup` configured with a delta equal to the number of goroutines you're spawning.  Your application will run to completion after each of the goroutines exit.

In the following example, I'm assuming that we don't have access to the `run` function and so am dealing with the `sync.WaitGroup` internally to the `main` function.

``` go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		defer wg.Done()
		run(1, "A")
	}()
	go func() {
		defer wg.Done()
		run(5, "B")
	}()
	wg.Wait()
}

func run(iter int, name string) {
	for i := 0; i < iter; i++ {
		time.Sleep(time.Second)
		fmt.Println(name)
	}
}
```

``` bash
$ go run main.go
B
A
B
B
B
B
<EXIT 0>
```

This is a more elegant solution to the [hit-and-hope](#synchronous) solution as it leaves nothing to chance.  As with the above example, you'll likely need to keep the wait group code within the context of your `main` function, so provided you don't mind poluting it with wait code, you're all good.

If you need to add/remove a goroutine, don't forget to increment the delta, or your application won't behave as expected!

###### <a name="channels"></a>Channels

It's also possible to use channels to acheive this behaviour, by creating a buffered channel with the same size as the delta you initialised the `sync.WaitGroup` with.

In the below example, I once again assume no access to the `run` function and keep all synchronisation logic in the `main` function.

``` go
package main

import (
	"fmt"
	"time"
)

func main() {
	done := make(chan struct{}, 2)

	go func() {
		defer func() { done <- struct{}{} }()
		run(1, "A")
	}()

	go func() {
		defer func() { done <- struct{}{} }()
		run(5, "B")
	}()

	for i := 0; i < 2; i++ {
		<-done
	}
}

func run(iter int, name string) {
	for i := 0; i < iter; i++ {
		time.Sleep(time.Second)
		fmt.Println(name)
	}
}
```

``` bash
$ go run main.go
B
A
B
B
B
B
```

The obvious added complexity and the fact that the synchronisation code needs to be updated if a goroutine needs to be added/removed detract from the elegance of this approach.

###### <a name="goexit"></a>runtime.Goexit()

Another solution is to use the runtime package's `Goexit` function.  This function executes all deferred statements and then stops the calling goroutine, leaving all other goroutines running.

Exit wise, once the `Goexit` call is in place, your application can only fail.  If your application is running in an orchestrated environment like Kubernetes, this might be absolutely fine but something to be aware of.

There are two ways your application can now exit (both resulting in an exit code of 2):

* If all of the other goroutines run to completion, there'll be no more goroutines to schedule and so the runtime scheduler will panic with a deadlock informing you that `Goexit` was called and that there are no more goroutines.
* If any of the other goroutines panic, the application will crash as if any other unrecovered panic had occurred.

With all the doom and gloom out the way, let's take a look at the code:

``` go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	go run(1, "A")
	go run(5, "B")

	runtime.Goexit()
}

func run(iter int, name string) {
	for i := 0; i < iter; i++ {
		time.Sleep(time.Second)
		fmt.Println(name)
	}
}
```

``` bash
$ go run main.go
B
A
B
B
B
B
fatal error: no goroutines (main called runtime.Goexit) - deadlock!
<STACK OMITTED>
<EXIT 2>
```

This solution understandably won't be for everyone, especially if you're working with inexperienced gophers (for reasons of sheer confusion, "my application keeps failing" and "nice, I'll use this everywhere) but it's nevertheless an interesting one, if only from an academic perspective.

###### Summary

`Goexit` keeps your code slim but will never allow it to exit cleanly.  Treat with respect and use with caution.