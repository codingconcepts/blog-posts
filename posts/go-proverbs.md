I love (romantic love) the [Go Proverbs](https://go-proverbs.github.io).  I've got a copy of them stuck up on the wall next to me at work and I've stuck one up in the canteen at work.  They're short, sharp and easy to understand, once you understand them.  See [Rob Pike's talk](https://www.youtube.com/watch?v=PAAkCSZUG1cv) for context.

The idea of language proverbs are nothing new.  Most people have at least *heard* of the [Zen of Python](https://www.python.org/dev/peps/pep-0020/#the-zen-of-python), which helps Python developers develop "pythonic" code.  The Go Proverbs aim to help Go developers write "idiomatic" code in exactly the same vein; they're just more technical.

Here are what the Go Proverbs mean to me (with a bit of help from Rob Pike's talk).

### Don't communicate by sharing memory, share memory by communicating.

In many situations, mutexes are perfectly valid (and arguably the correct) construct to use.  They're easy to reason with *up to a point* and *usually* allow for read/write separation (in Go's case the `RLock/Lock` and `RUnlock/Unlock` methods on `sync.RWMutex`).

In many other situations, mutexes are unecessary, hard to reason with and can slow write access to a shared object if read/write separation has not been properly considered.

### Concurrency is not parallelism.

Concurrency is *managing* multiple things at once, parallelism is *doing* multiple things at once.

### Channels orchestrate; mutexes serialize.

Mutexes allows only one thread of execution into a critical section.  Channels allow for the configuration of thread-safe pipelines of execution.

### The bigger the interface, the weaker the abstraction.

An interface with one method has a clear separation of concerns and requires a consumer to implement just one method in order to test/mock/swap it.

### Make the zero value useful.

If you declare a `sync.Mutex` without initialising it (the zero value), you'll have an unlocked mutex that's ready to use.  Minimise your API's footprint by making *your* type's zero values useful too.

### interface{} says nothing.

The empty interface (`interface{}`) does not contain type information meaning:

* Consumers need to perform type assertions to get at the real value.
* Type-safety is out the window.
* Your API interface is weaker and not self-describing.

### Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.

You might not like the compiler complaining about your syntax but because everyone codes to the same standard and style, all Go code looks the same.

### A little copying is better than a little dependency.

If you have a large package with a tiny but useful method, rather than referencing it and making consumer code larger, copy the code and write some tests to make sure the copied code stays in-sync with the original code.  (Go did this with `strconv.IsPrint`.  Rather than adding an additional dependency to `encoding/unicode` (+100KB), they copied the 3 lines they needed and tested that they could never fall out-of-sync).

### Syscall must always be guarded with build tags.

System calls are specific to the OS they're operating against, so make sure that your resulting binaries contain only the relevant system calls for the OS you're targeting by using build tags.

### Cgo must always be guarded with build tags.

Thanks to the Open Source community, C isn't as portable as you would think.  Some C code will work on Windows but not Linux and vice-versa.  Use build tags to ensure the C code you're referencing is targeting the OS you're building for.

### Cgo is not Go.

When using C, all safety features provided by Go (memory management to name just one), go out the window.  Be aware of this and only make the sacrifice if you're prepared to live with the consequences.  Rob Pike states that of all the bugs he sees in Google's in-house Go applications, 90% result from interop with C.

### With the unsafe package there are no guarantees.

Go provides a compatibility guarantee that promises the language will never change enough to break your code.  They don't provide this guarantee for the `unsafe` package, so use it at your peril.

### Clear is better than clever.

The simplest code is often the most beautiful.  No one (including your future self) will thank you for writing selfishly over-clever and infeasably abstracted code.

### Reflection is never clear.

If you have to use relfection, you're probably not doing it right.  Don't look in the mirror, you might not like what you find.

### Errors are values.

Values are programmable and since errors are just values, errors are programmable.  If you're blindly checking `err != nil`, at what point along the call stack does something take responsibility for the error?  Use errors to your advantage and eliminate boilerplate.  This is what the langauge's `error` interface was designed for.

### Don't just check errors, handle them gracefully.

The try/catch exception mechanism is purposefully missing in Go.  By throwing an exception in a function, you're making the failure someone else's problem; someone else whose responsibility might not and in most cases should not have to extend to handling that error.

Use errors to direct program flow and react intelligently to failure events.

### Design the architecture, name the components, document the details.

An architecture's components should fit together in a way that's as self-describing as possible.  The components that make up the architecture should be named to support this and the low-level details that form the components should be well documented.

### Documentation is for users.

Write documentation as if you were consuming your API, not as someone who implicitly understands it.

### Don't panic.

You should never have to use Go's `panic` construct.  Errors provide the information necessary to allow you to elegantly handle failure scenarios without throwing your toys out the pram.