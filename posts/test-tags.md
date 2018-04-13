One of the most useful go build flags I've used recently is tags.  Its purpose is to look at the build tags you've specified and use them to drive what's sent to the compiler for compilation.

The tags flag is also available to the go test command, which means you can toggle swathes of tests on and off, without having to resort to code changes such as using flag in your TestMain function or - heaven forbid - writing logic into your tests themselves.

As part of a massive Go microservice rollout and to save everyone having to write boilerplate code or learn an ORM, a colleague and I wrote a tool to generate Go structs from MySQL tables modelgen.  In addition to outputing generated structs, we also provide some helper structs to handle primative types going into and out of nullable database fields.

These helper structs are bundled in with packr into a file called x_helpers.go and their tests are also bundled for completeness, into a file - unsurprisingly - calld x_helpers_test.go.  These structs are thoroughly tested, which means when the project that references them calls go test -v, there's a lot of output.

The following example demonstrates how we suppresed these tests, without resorting to code changes.

To keep things tidy, the following file is a stand-in for x_helpers_test.go.  It contains one test, which fails to make the fact that it has run more obvious:

    //+build helpers
    
    package main
    
    import "testing"
    
    func TestFail(t *testing.T) {
    	t.Fatal("Oh noes!")
    }

At the top of the file, you'll notice the inclusion of the build tag "helpers".  This translates to "compile this, if the helpers tag has been specified and don't if it hasn't".  A test run with the tag specified shows that this test is being executed as expected:

    go test --tags helpers
    --- FAIL: TestFail (0.00s)
            blah_test.go:8: Oh noes!
    FAIL
    exit status 1
    FAIL    sandbox 0.039s

When we run the tests without the "helpers" tag, the compiler won't compile this file and hence won't run its tests:

    go test
    PASS
    ok      sandbox 0.039s

I like this method for supressing test output for a number of reasons:

- It's familiar because the use of build tags is widespread in Go.
- The default behaviour of go test in this setup is to run the unit tests we're expecting to run and ignore those we're not.
- We can toggle on the tests whenever we want by including the tag.
- A build tag is less invasive than a code change.