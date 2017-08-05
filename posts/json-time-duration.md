From time to time, you'll want to serialise/deserialise a struct whose default serialisation seems counter-intuitive.  Take for example, the JSON serlisation of `time.Duration`:

``` go
json.NewEncoder(os.Stdout).Encode(time.Hour)

// result:  3600000000000
```

I don't event think *science* knows what this number is.

Asking people to configure 1 hour as "3600000000000" is not only cruel, it's *asking* for trouble; miss a zero and you've had it.

A much friendlier and more natural alternative is to allow for the confguration of `time.Duration` as you're *used* to seeing them appear, `1h`, `1m30s` etc.  To acheive this, you'll need to perform some customer JSON marshaling and unmarshaling.

Here's a copy-paste-friendly example:

**Step 1** - create a struct to house the `time.Duration` and receive the custom marshal operations.

``` go
type ConfigDuration struct {
	time.Duration
}

func (d *ConfigDuration) UnmarshalJSON(b []byte) (err error) {
	d.Duration, err = time.ParseDuration(strings.Trim(string(b), `"`))
	return
}

func (d ConfigDuration) MarshalJSON() (b []byte, err error) {
	return []byte(fmt.Sprintf(`"%s"`, d.String())), nil
}
```

**Step 2** - create a struct that consumes the new `ConfigDuration` struct:

``` go
type ServerConfig struct {
	ReadTimeout  ConfigDuration `json:"readTimeout"`
}
```

**Step 3** - Unmarshal some JSON and access the `time.Duration`:

``` go
config := ServerConfig{}
json.Unmarshal([]byte(`{"readTimeout":"10s"}`), &config)

fmt.Println(config.ReadTimeout.Duration)

// result: 10s
```