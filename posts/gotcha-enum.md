I ran into a curious little gotcha with Go's `iota` constract this morning and I wanted to share it with you.

Without cheating, what would you expect the following code to output?

``` go
package main

import "fmt"

const (
    one = 1 << iota
    two
)

func main() {
    fmt.Println(one, two)
}
```

If you said `1 2`, you'd be correct!  There's nothing fishy going on here.

Let's now suppose we need to add another constant.  As it's important and we want it to be visible, we've added it to the top of our `const` block:

``` go
package main

import "fmt"

const (
    greeting = "Hello, Enums!"

    one = 1 << iota
    two
)

func main() {
    fmt.Println(one, two)
}
```

What would you expect the program to output now?

If you said `1 2`, you'd now be wrong!  The output of this program is now `2 4`.  `iota` has been incremented, so `one` is left-shifted by 2 instead of 1, throwing all subsequent enum values out.

**Without proper unit tests, this is the kind of subtle bug that could take down a production application, or worse, not take it down.**

When you understand the [ConstSpec](https://golang.org/ref/spec#Iota), it's becomes clearer why this is the case but as our `greeting` isn't part of our enumeration (or even an integer), I would have expected `iota` to behave differently in this case.

The only correct fix in this example is to move the new constant out of the enumeration `const` block and into a _new_ `const` block.  This is because each `const` block resets `iota`, essentially grouping the constants together, regardless of type and the order in which they appear.