From time to time, you'll want to serialise/deserialise a struct whose default serialisation seems clunky.  Take for example, the JSON serlisation of `time.Duration`:

``` go
json.NewEncoder(os.Stdout).Encode(time.Hour)

// result:  3600000000000
```

I don't event think *science* knows what this number is, it's so big.

Asking people to configure 1 hour as "3600000000000" is asking for trouble.  Miss a zero and you've missed your mark.

A much friendlier alternative is to allow for the confguration of `time.Duration` as you're used to seeing it appear, `1h`.  To acheive this, you'll need to perform some customer JSON marshaling and unmarshaling.

Here's an example I've nabbed from a chaos monkey I'm writing:

**Step 1** C

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

``` go
type ServerConfig struct {
	ReadTimeout  ConfigDuration `json:"readTimeout"`
	WriteTimeout ConfigDuration `json:"writeTimeout"`
}
```

``` go
serverConfig.ConfigDuration.Duration
```