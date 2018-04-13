To save us writing boilerplate code (and everyone having to learn an ORM) during a major rollout of new Go services, a colleague and I wrote a tool to generate Go structs from MySQL tables [modelgen](https://github.com/LUSHDigital/modelgen).

In addition to the generated structs, the tool also provides a number of helper structs, which handle database `NULL` values (including scanning and encoding) for various primative types.  These types (and their tests) are included in the codebase of each service in the following files:

`x_helpers.go`

`x_helpers_test.go`

As the tests in `x_helpers_test.go` are fairly involved, they tend to polute verbose `go test` output, so I wanted to share our solution for removing them.

`go test` (and `go build`) accepts a number of flags and its one of those flags that came to the rescue in our case:  `--tags`.  The `tags` flag looks at the build tags you've specified and uses them to drive what's sent to the compiler for compilation.

In the following example, we'll use the `tags` argument to suppress tests that we don't want to run.  Firstly, let's write a test file to stand in for the verbose `x_helpers_test.go` tests I mentioned earlier:

```go
//+build helpers

package main

import "testing"

func TestFail(t *testing.T) {
	t.Fatal("Oh noes!")
}
```

Notice the inclusion of the build tag `helpers` which translates to "compile this, if the `helpers` tag has been specified".  A test run with the tag specified shows that this test is being executed as expected:

```bash
go test --tags helpers
--- FAIL: TestFail (0.00s)
        blah_test.go:8: Oh noes!
FAIL
exit status 1
FAIL    sandbox 0.039s
```

When we run the tests *without* the `helpers` tag, the compiler won't compile this file and hence won't run its tests:

```bash
go test
PASS
ok      sandbox 0.039s
```

I like this method for supressing test output for a number of reasons:

- It's familiar because the use of build tags is widespread in Go.
- The default behaviour of `go test` in this setup is to run the unit tests we're *expecting* to run and ignore those we're not.
- We can toggle on the tests whenever we want by including the tag.