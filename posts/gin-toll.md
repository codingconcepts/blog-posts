# Gin Toll
A tollbooth middleware for Gin

I'm what could be described as a heavy Gin user.  I've tried (and failed) to kick the stuff many times but for me, the process of creating a web API has never been easier or quicker than Gin.

For those who've exposed their Go code on the internet before, you'll be keenly aware of the choice of available rate-limiters.  I've done the rounds too and, being lazy, hoped to find something that'd plug straight into Gin.

Nothing.

To keep my API definition as simple as possibly, I created a very simple tollbooth wrapper, which works as a middleware for Gin.  Here's how to limit requests to 1 per second:

Grab it from github:
```bash
$ go get github.com/codingconcepts/gin-toll
```

Import it into your code:
```go
import "github.com/codingconcepts/gin-toll"
```

Use the `LimitMiddleware` method to create a Gin middleware:
```go
router := gin.Default()
router.Use(gintoll.LimitMiddleware(tollbooth.NewLimiter(1, time.Second)))

router.GET("/get", func(c *gin.Context) {
	c.String(http.StatusOK, "Now we know...")
})
router.Run(":1234")
```
