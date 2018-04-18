

When developing a service, you'll want to ensure that you have decent test coverage of its endpoints.  I've seen endpoint coverage achieved in a number of ways, some interesting, some *interesting* and I'd like to share my method.

If you've arrived here, you're probably no stranger to validating input to endpoints and it's this use case that I'll be demonstrating here.

First and foremost, I'm using table-drive tests.  In the context of service testing, they allow me to make multiple, requests to a service, using similar request bodies, without changing the test code I'm using to invoke the service, or polluting my test cases with huge, largely identical code.

To keep things simple, I've written a very contrived example of a service, which satisfies the following contract:

* Takes an `application/json` body which contains a first and last name.
* It asserts that both the first and second name have been provided and are of sensible lengths.
* It returns a full name, which is the concatenation of first and last name.

##### Code

I'm using [govalidator](https://github.com/asaskevich/govalidator) to validate the input and in order to make the various error states obvious, I'm using custom error message.  So you'll have to excuse the verbosity:

``` go
type fullNameRequest struct {
	First string `json:"first" valid:"runelength(1|100)~first name should contain 1-100 characters,required~first name required"`
	Last  string `json:"last" valid:"runelength(1|100)~last name should contain 1-100 characters,required~last name required"`
}
```

The endpoint is simple, it binds the request body to a `fullNameRequest` struct and validates it.  If the request isn't valid, it'll bail with a `422` and if it's valid, the full name will be returned:

``` go
func getFullName(w http.ResponseWriter, r *http.Request) {
	var request fullNameRequest
	if err := bindAndValidateJSON(r.Body, &request); err != nil {
		http.Error(w, err.Error(), http.StatusUnprocessableEntity)
		return
	}

	w.WriteHeader(http.StatusOK)
	w.Write([]byte(fmt.Sprintf("%s %s\n", request.FirstName, request.LastName)))
}
```

While not essential to this example, the helper methods I'm using for binding and validating are copy/paste-friendly, so I've included them in case you want them:

``` go
func bindAndValidateJSON(rc io.ReadCloser, target interface{}) error {
	if err := bindJSON(rc, target); err != nil {
		return err
	}

	_, err := govalidator.ValidateStruct(target)
	return err
}

func bindJSON(rc io.ReadCloser, target interface{}) error {
	defer rc.Close()

	body, err := ioutil.ReadAll(rc)
	if err != nil {
		return err
	}

	return json.Unmarshal(body, target)
}
```

To run the server, it's the standard two-liner:

``` go
func main() {
	http.HandleFunc("/", getFullName)
	http.ListenAndServe(":1234", nil)
}
```

Let's give it a whirl, to see if we get what we're expecting from the various requests we could throw at it:

``` bash
curl localhost:1234/ -d '{"first": "Rob", "last": "Reid"}'
Rob Reid

curl localhost:1234/ -d '{"last": "Reid"}'
first name required

curl localhost:1234/ -d '{"first": "Rob"}'
last name required
```

##### Test Code

Rather than paste one massive test method, I've separate each section out into it's component parts.  If you're familiar with [table-drive tests](https://blog.golang.org/subtests), the following will still read fairly naturally to you.  Here's a high-level workflow of what the test is doing:

* For each of the test cases:
  * Create a valid request for the endpoint using a struct.
  * Modify it to make it invalid.
  * Marshal the stuct to something we can throw at the endpoint.
  * Perform some expectations.

Like in the service code, I've included any helpers I've used to A) keep the code clean and B) give you something to copy/paste if you think it'd be useful:

``` go
// ToJSONBody turns a struct into an io.Reader request body.
func ToJSONBody(tb testing.TB, i interface{}) io.Reader {
	j, err := json.Marshal(i)
	if err != nil {
		tb.Fatalf("error stringifying: %v", err)
	}

	return strings.NewReader(string(j))
}

// Equals performs a deep equal comparison against two
// values and fails if they are not the same.
func Equals(tb testing.TB, exp, act interface{}) {
	tb.Helper()
	if !reflect.DeepEqual(expected, actual) {
		tb.Fatalf("\n\texp: %#[1]v (%[1]T)\n\tgot: %#[2]v (%[2]T)\n", exp, act)
	}
}
```

At the top of the test, the test cases are defined as a struct that has a name, a function and some expectations.  The interesting bit here is the function, `mod`.  This function allows me to pass a pointer of a `fullNameRequest` and modify it:

``` go
cases := []struct {
    name    string
    mod     func(r *fullNameRequest)
    expCode int
    expBody string
}
```

Next, are the test definitions themselves.  I've included just enough of them to demonstrate what's happening.  Notice that I'm not creating a new `fullNameRequest` for every test case, so the usefulness of this pattern comes into its own as struct size increases. 

``` go
{
    name: "missing first name",
    mod: func(r *fullNameRequest) {
        r.FirstName = ""
    },
    expCode: http.StatusUnprocessableEntity,
    expBody: "first name required\n",
},
{
    name: "first name too long",
    mod: func(r *fullNameRequest) {
        r.FirstName = strings.Repeat("a", 101)
    },
    expCode: http.StatusUnprocessableEntity,
    expBody: "first name should contain 1-100 characters\n",
},
{
    name:    "Testy McTestface",
    expCode: http.StatusOK,
    expBody: "Testy McTestface\n",
}
```

In the main body of the function, we're creating a decent request and passing it to the `mod` function if one has been provided.  We're then passing the request through the `ToJSONBody` function to derive an `io.Reader` to chuck at the endpoint.  Finally, we're asserting that the expected status code and response body have been returned.

``` go
t.Run(c.name, func(t *testing.T) {
    body := fullNameRequest{
        FirstName: "Rob",
        LastName:  "McSomething",
    }

    if c.mod != nil {
        c.mod(&body)
    }

    w := httptest.NewRecorder()
    r := httptest.NewRequest("GET", "/", ToJSONBody(t, body))
    getFullName(w, r)

    Equals(t, c.expCode, w.Code)
    Equals(t, c.expBody, w.Body.String())
})
```