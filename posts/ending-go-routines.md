One of the most enduring phrases I've every picked up as a developer is this:

*What could **possibility** go wrong?*

It's basically Murphey's Law, made into a t-shirt and I recite it like a mantra before beginning every project and committing to every assumption I make.

Recently, I hit an obscure bug which took me well over a day to fix and it all boiled down to the fact that I hadn't paid close enough attention to my one golden rule.  Talking of golden rules, the Go community state that you should never start a goroutine you don't intend to finish.

*What could **possibility** go wrong?*

"Nothing", I thought and developed an infinitely looping channel pipe:

``` go

```