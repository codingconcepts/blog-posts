If your code is particularly busy, eking out 

##### Use gcflags

The `-gcflags` command line argument to `go build` instructs the compiler to output the results of its escape analysis.  This allows you to determine which of your program's variables "escape" to the heap.

Let's build some code with `-gcflags` and inspect the results:

``` go

```

``` bash
$ go build -gcflags '-m'
```