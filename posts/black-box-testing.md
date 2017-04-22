Go's tooling continues to delight and this one's a real hidden gem...  I've just discovered, with the help of [exago.io](https://www.exago.io), the `testing/quick` package in Go's standard library.  It's a very simple blackbox testing package, which repeatedly executes a given block of your code with **values you wouldn't think to try yourself**.

Within minutes of discovering it, I'm already starting to think differently about how I write exported functions and here's why.

##### The scenario

Consider the following innocuous little function.  It simply returns a random number between a minimum and maximum value.

``` go
func Between(min int, max int) int {
	return rand.Int()%(max-min) + min
}
```

The hardcore among you might even think to add a litle bit of manual testing to make sure it's working correctly:

``` go
func init() {
	rand.Seed(time.Now().UnixNano())
}

func main() {
	for i := 0; i < 10; i++ {
		fmt.Println(Between(0, 10))
	}
}
```

``` bash
$ go run main.go
3
8
1
3
4
8
7
4
0
5
```

Looks great!  You commit and deploy to production.  Users *love* having the ability to generate a weak random number between a minimum and maximum value.  Who wouldn't!?

Then I checked out my exago.io [score](https://www.exago.io/project/github.com/codingconcepts/albert) and ran through the checklist (their gamification slant on improving code really makes sense to me):

`Blackbox Tests: In addition to standard tests, does the project have blackbox tests?	
`

After a bit of digging, I found that they were referring to Go's blackbox testing tool `testing/quick`.  So I had a play...  Read the following code carefully.

``` go
func TestBlackBoxCheckBetween(t *testing.T) {
	f := func(min int, max int) bool {
		result := Between(min, max)

		return result >= min && result <= max
	}
	if err := quick.Check(f, nil); err != nil {
		t.Error(err)
	}
}
```

If the basic gist of `TestBlackBoxCheckBetween` makes sense to you, your assumptions about how the function works and, more importantly, your assumptions about how *users will treat the function* are pretty much the same as mine.

`testing/quick` detects your function's parameter types and repeated invokes it with generated values (a minimum and maximum in the case of `Between`).  After the invocation, you perform an assertion and return `true` or `false`; `true` if the output of your function contains the expected result and `false` if it doesn't.

I gave it a run:

``` bash
$ go test
--- FAIL: TestBlackBoxCheckBetween (0.00s)
        main_test.go:15: #1: failed on input 4106209714314777601, -2352281900722994752
FAIL
exit status 1
```

The output of this test has just made me acutely aware of how fragile my `Between` method was (not to mention most of the other methods I've *ever* written).  With very little effort, `testing/quick` has just surfaced the following assumptions in my code:

* Users will know what values to pass in
* I know what value users will pass in
* Minimum will be reasonable
* Maximum will be reasonable
* Minimum will be less than maximum

Horrified at my ignorance, I added a little check in the `Between` method and ran the test again:

``` go
func Between(min int, max int) int {
	// swap min and max if max is less than min
	if max < min {
		min, max = max, min
	}

	return rand.Int()%(max-min) + min
}
```

Running the test again at this point will result in the same failure but for a different reason.  Your *production* code is now better able to cope with weird input but your *test* code needs a small tweak.

Our assumption that result would be between the minimum and maximum values is now wrong because it's possible that the minimum value could be greater than the maximum value.

```go
func TestBlackBoxCheckBetween(t *testing.T) {
	f := func(min int, max int) bool {
		result := Between(min, max)

		// Between ensures min is less than or equal to max,
		// so perform the same switch here
		if min > max {
			min, max = max, min
		}
		return result >= min && result <= max
	}
	if err := quick.Check(f, nil); err != nil {
		t.Error(err)
	}
}
```

``` bash
$ go test -v
=== RUN   TestBlackBoxCheckBetween
--- PASS: TestBlackBoxCheckBetween (0.00s)
```

I'm now going to create black box tests for everything I've ever written, ever.