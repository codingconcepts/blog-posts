Struct tags in Go provide metadata to fields and are used heavily by the encoders in the stdlib's `encoding` package.  Here's a typical use case for a struct tag:

``` go
import "encoding/json"

type ServerConfig struct {
    Port   int    `json:"port"`
    APIKey string `json:"apiKey"`
}

func main() {
    bytes, _ := json.Marshal(ServerConfig{
        Port:   1234,
        APIKey: "something secret",
    })

    fmt.Println(string(bytes))
}
```

The struct tags in this example give Go's JSON encoder an explicit name to use when marshalling/unmarshalling `ServerConfig` structs.  This is simple and declarative, you're providing this information in-line with the struct's field itself, so there's only one source of truth.  Here's the output of the above example:

``` bash
{"port":1234,"apiKey":"something secret"}
```

##### Background

My team recently deployed a [12-factor](https://12factor.net/) application into Production and as per factor III of the methodology, we're storing our config in "the environment" (environment variables).

In vanilla Go, this means writing one of the following:

``` go
config := os.Getenv("ENV_KEY")
if config == "" {
    // fail if required
}

// parse and use environment variable
```

or 

``` go
if config, ok := ok.Lookupenv("ENV_KEY"); !ok {
    // fail if required
} else {
    // parse and use environment variable
}
```

Of the two, my preference is the latter, because **A** an empty string might be valid configuration for a given struct string field, **B** the "ok idiom" is used elsewhere by the stdlib (think map access), so looks and feels natural/familiar and **C** the resulting code is a more syntactically obvious alternative to checking for an empty string.

My issue with this approach is that even when the logic is refactored out into a helper function, the responsibility of finding the value, parsing it, checking for errors and making decisions as to whether the configuration is required or not is that of the application (and the developer).  Not to mention the code distance between declaring the fields on the struct and setting them from environment variables elsewhere.

This all feels very imperative, so I decided to write a library to take the pain out of factor III.

##### Enter `env`

Env is a very lightweight library that allows you to express your environmental configuration declaratively.  It takes care of the boring stuff, which keeps your code slick and easy to reason with.

Simply pass `env.Set` a pointer to the struct you wish to populate and it'll populate every field with an `env` tag value from environment configuration.  For example, an `env` tag value of `MYAPPLICATION_PORT` for an integer field `Port` will find an environment variable called `MYAPPLICATION_PORT` and populate the `Port` field with the integer representation of the environment value.

If the field is mandatory, you can optionally specify a `required` tag value to ensure that an error is returned if the environment variable is not found.  An error will be returned of the environment variable found cannot be parsed to the field's type or if the field itself is not settable.

Here's a complete example to get you going:

``` go
import "github.com/codingconcepts/env"

type ServerConfig struct {
    Port   int    `env:"MYAPPLICATION_PORT" required:"true"`
    APIKey string `env:"MYAPPLICATION_APIKey"`
}

func main() {
    var config ServerConfig
    if err := env.Set(&config); err != nil {
        // something went wrong grabbing config, fail
    }

    // start server with config
}
```

Feel free to get in touch with issues/suggestions/pull requests!