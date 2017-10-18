

## General

* Don't be clever
* Don't use `panic`
* Be explicit about struct dependencies and use constructor functions
* Make the zero value useful.  E.g. for an `io.Writer`, consider using `ioutil.Discard` instead of nil.
* Don't mix value/pointer semantics for the same type
* Use small interfaces, they're easier to use and easier to test
* Go is Open Source, to find out how something works, goto definition!
* Put $GOPATH/bin in your $PATH (executables from `go get` and `go [build|install]` can be run immediately)
* Use pkg/cmd directory structure for applications
* If writing an application, always vendor your dependencies
* If writing a library, never vendor your dependencies
* Your `main` function should be the only thing aware of command line arguments
* Prefer `go install` over `go build` (same results but `install` caches for faster subsequent builds)

## Syntax

* Always use `defer` unless you've got a very good reason not to
* Use the convention "New" for constructor functions that return pointers
* Always use fully-qualified import paths, never relative
* Add a `String() string` method to enum types
* If using `iota` for enums, start with +1 increment (to allow for an undefined/default value)
* Wrap select/for idiom into a function to allow for loop escapes via `return`
* Name properties in struct initialisers (don't rely on order, which might change)
* Use separate lines for properties in struct initialisers
* Name return parameters (results in smaller and simpler compiled output and good for documentation)
* Use good naming conventions
  * Interface naming should usually end in "er" (e.g. `Reader` reads and `ReadCloser` reads and closes)
  * The closer a variable's declaration is to its usage, the short its name can be
  * Acronyms should use upper case (e.g. `ServeHTTP`, not ServeHttp and `catAPI`, not catApi)
  * Don't duplicate package names (it's `bytes.NewBuffer`, not bytes.NewByteBuffer)
  * Package names should lend meaning to their exported types/functions
  * Avoid upper-case letters for package names

## Flow

* Never start a goroutine that might not finish
* If an operation might block, use `context.Context` to provide timeouts and cancellations
* Maintain line-of-sight (errors indented and returning early, happy path at the end)