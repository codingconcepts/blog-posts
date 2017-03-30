I was one of many developers lured to Go by its promise of obscenely high concurrency.  I estimate that in my first 37 minutes of playing with Go, I'd spun up in excess of 78 trillion goroutines.

I'm also one of just as many developers who are now in love with Go because of its interfaces.

###### Recap

In Go, interfaces are implemented *implicitly*, meaning there's none of this:

``` c#
public interface IAnimal
{
    void Move();
}

public class Dog : IAnimal
{
    public void Move()
    {
    }
}
```

Only this:

``` go
type Mover interface {
    Move()
}

type Dog struct {
}

func (dog *Dog) Move() {
}
```

###### Example

As an example, let's assume you've imported Amazon's Polly client and wish to pass around an implementation of it in a `Server` object:

``` go
type Server struct {
    synthesiser *polly.Polly
}
```

When it comes to testing the `Server` (if you have time for such frivolity), you're completely at the mercy of the `polly.Polly` struct.  To test the happy paths, you'd need a fully-fledged instance of the `polly.Polly` stuct, complete with a valid AWS session.  If a session can't be established, you'll get errors, so bang goes your happy path.

###### Then the penny dropped

It wasn't until I needed to mock a third-party object, that I truly appreciated the value of interfaces and realised it wasn't just my structs that could implement my interfaces, but *other people's* structs...

There was much weeping and jubiliation.  Here's how I did it:

**Step 1** Create an interface whose only method is the one you're wanting to test/mock:

``` go
type Synthesiser interface {
    // this matches the method receiver on *polly.Polly
    SynthesizeSpeech(input *polly.SynthesizeSpeechInput) (*polly.SynthesizeSpeechOutput, error)
}
```

**Step 2** Update the `Server` that depend on `*polly.Polly` so that it only cares about the new interface:

``` go
type Server struct {
    synthesiser Synthesiser
}
```

**Step 3** Spin up the `Server` just as you would have done before, passing in exactly the same instance of `*polly.Polly`:

``` go
&Server{
    synthesiser: polly,
}
```

###### Testing your interface

Now, you can pass any old object that implements your new `Synthesiser` interface to your `Server`.

Here's an example of a happy path mock:

``` go
type MockSynthesiser struct {
    AudioStream         io.ReadCloser
    ContentType         *string
    RequestedCharacters *int64
}

func (mock *MockSynthesiser) SynthesizeSpeech(input *polly.SynthesizeSpeechInput) (out *polly.SynthesizeSpeechOutput, err error) {
    out = &polly.SynthesizeSpeechOutput{
        AudioStream:       mock.AudioStream,
        ContentType:       mock.ContentType,
        RequestCharacters: mock.RequestedCharacters,
	}
    return
}
```

Here's an example of an error mock:

``` go
type MockErrorSynthesiser struct {
    Error error
}

func (mock *MockErrorSynthesiser) SynthesizeSpeech(input *polly.SynthesizeSpeechInput) (out *polly.SynthesizeSpeechOutput, err error) {
    err = mock.Error
    return
}
```

Here's an example of a mock that goes mental and panics, simulating a worst-case scenario:

``` go
type MockPanicSynthesiser struct {
    Error error
}

func (mock *MockPanicSynthesiser) SynthesizeSpeech(input *polly.SynthesizeSpeechInput) (out *polly.SynthesizeSpeechOutput, err error) {
    panic(mock.Error)
}
```

The properties in all of the above mocks allows us to perform expectations on how the `Server` deals with each scenario.