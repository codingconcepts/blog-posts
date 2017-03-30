I'm a big fan of optimising (?:as early as I possibly can|only when absolutely necessary).

There's a great case for not wasting time optimising things which may never need optimisation.  Especially if that affects readability.  Happily, I've enjoyed *both* in a recent project and I'd like to share the experience.  My last project involved speech synthesis by way of Amazon's Polly service.  You ask it to say something in a given voice and it returns you a big blob of raw PCM data.

In order for this data to be useful downstream, a WAV header needs to be slapped on the front of it (as per [the Wave spec](http://soundfile.sapp.org/doc/WaveFormat/)).  Unfortunately, the underlying `io.ReadCloser` implementation doesn't provide a way to get at the data length, without buffering it into memory first (as the PCM stream could be arbitrarily large).

###### Original implementation

**Step 1** Read the data into a `bytes.Buffer`:

``` go
buf := new(bytes.Buffer)
if _, err = io.Copy(buf, output.AudioStream); err != nil {
    return
}
```

**Step 2** Write the WAV header:

``` go
// ... other header fields
header.Subchunk2Size = buf.Len()
// ... other header fields
```

**Step 3** Stream the WAV result to the caller:

Let's assume `header` is an `io.Reader` for the following pseudo code:

``` go
io.MultiReader(header, buf)
```

If you'd assumed I was newing up a `bytes.Buffer` for every request, you'd be absolutely correct.  Go is a garbage-collected language, so while I'm spinning up lots of potentially large objects, they're all getting dutifully cleaned up after me.

Still, this felt lazy and rude.  I span up a `pprof` endpoint to monitor the heap and all looked fine.  The GC was doing its job and there were no nasty surprises.  I added some `StatsD` metric points and fired up Grafana, which told a different story...

``` go
func heartbeatHeap() {
	stats := runtime.MemStats{}
	for range time.NewTicker(time.Second * 10).C {
		runtime.ReadMemStats(&stats)
		statsd.Gauge(1, "heapalloc", fmt.Sprintf("%d", stats.HeapAlloc))
	}
}
```

The thing was like a city scape.  That poor garbage collector :'(

###### Enter `sync.Pool`

Go's standard library provides a useful thread-safe construct called `sync.Pool`, which will attempt to reuse objects that reside in it, if they're yet to be collected.  If you come from a C# background, this will feel similar to the `WeakReference`.

The `sync.Pool` operates on the empty interface `interface{}`, meaning you'll need to keep an eye on your type assertions.  As I was dealing with byte buffers (which can be `Reset`), I decided to wrap it with a struct of my own:

``` go
type BufferPool struct {
    sync.Pool
}

func NewBufferPool() (pool *BufferPool) {
    return &BufferPool{
        Pool: sync.Pool{
            New: func() interface{} {
                return new(bytes.Buffer)
            },
        },
    }
}
```

The `New` method on `sync.Pool` allows you to tell it how to create new objects, should they be required (and not reclaimed from the pool itself).  Having a wrapper over `sync.Pool` allows us to wrap the `Get` and `Put` behaviour as well; encapsulating the the type assertions:

``` go
func (pool *BufferPool) Get() (buffer *bytes.Buffer) {
    return pool.Pool.Get().(*bytes.Buffer)
}
```

When writing *back* to the pool, we can perform any necessary teardown operations on the buffers, readying them for later use, should they be spared garbage collection.  In our case, resetting the buffer (zeroing its data but keeping its memory footprint) is perfect:

``` go
func (pool *BufferPool) Put(buffer *bytes.Buffer) {
    buffer.Reset()
    pool.Pool.Put(buffer)
}
```

###### Food for thought

In time, we can identify a sensible average buffer size from the stats we're publishing.  This would allow us to prevent huge buffers from being added back into the pool and hogging unnecessarily large chunks of memory:

``` go
func (pool *BufferPool) Put(buffer *bytes.Buffer) {
    if buffer.Len() > pool.MaxBufferSize {
        return
    }
    
    buffer.Reset()
    pool.Pool.Put(buffer)
}
```

We could also take the hit of preparing the byte buffer's underlying array by having a sensibly sized one ready for the pool's `New` call.  Early optimisations eh?