# Gin JWT Custom Claims
The [appleboy gin-jwt](http://gopkg.in/appleboy/gin-jwt.v2) JWT middleware for Gin allows you to plug [JWT](https://jwt.io) into your Gin router with little to no fuss.

Recently, I've used it to store basic user information to allow my router code to intelligently handle different user roles etc.

### Step 1 - Grab Gin and gin-jwt
```bash
$ go get github.com/gin-gonic/gin
$ go get gopkg.in/appleboy/gin-jwt.v2
```

###### Step 2 - Write some code
Rather than stepping through line-by-line, I'll describe what we're doing using in-line comments (in a format you can just copy and paste into your own editor to play around with).

```go
package main

import (
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	jwt "gopkg.in/appleboy/gin-jwt.v2"
)

func main() {
	// create a gin router
	router := gin.Default()

	// spin up a JWT middleware, there are two methods omitted here,
	// you'll want to check the docs to see what they do but for this
	// example, the default behaviour will suffice.
	jwtMiddleware := &jwt.GinJWTMiddleware{
		Realm:         "robreid.io",
		// store this somewhere, if your server restarts and you're
		// generating random passwords, any valid JWTs will be invalid
		Key:           []byte("something super secret"),
		Timeout:       time.Hour,
		MaxRefresh:    time.Hour * 24,
		Authenticator: authenticate,
		// this method allows you to jump in and set user information
		// JWTs aren't encrypted, so don't store any sensitive info
		PayloadFunc:   payload,
	}

	// expose a login method and hook up the JWT login handler
	router.POST("/login", jwtMiddleware.LoginHandler)

	// create a group to secure, you can secure any number of
	// groups but for this example, we'll only secure "v1"
	v1 := router.Group("/v1")

	// wrap v1's methods in the JWT middleware, anything inside
	// the v1 group will be protected with the JWT.
	v1.Use(jwtMiddleware.MiddlewareFunc())
	{
		v1.GET("/hello", hello)
		v1.GET("/refreshToken", jwtMiddleware.RefreshHandler)
	}

	router.Run(":1234")
}

func hello(c *gin.Context) {
	// the JWT middleware provides a useful method to extract
	// custom claims, it's basically the reverse of what's being
	// done in the payload function below
	claims := jwt.ExtractClaims(c)

	// for this example, we'll just dump out our custom claims
	// but in reality you could create your own middleware
	// handler to intercept this information and provide an
	// additional level of role-based security
	c.String(http.StatusOK, "id: %s\nrole: %s", claims["id"], claims["role"])
}

func authenticate(email string, password string, c *gin.Context) (string, bool) {
	// it goes without saying that you'd be going to some form
	// of persisted storage, rather than doing this
	if email == "example@robreid.io" && password == "fred123" {
		return email, true
	}
	
	return "", false
}

func payload(email string) map[string]interface{} {
	// in this method, you'd want to fetch some user info
	// based on their email address (which is provided once
	// they've successfully logged in).  the information
	// you set here will be available the lifetime of the
	// user's sesion
	return map[string]interface{}{
		"id":   "1349",
		"role": "admin",
	}
}

```

###### Step 3 - Test

First, login to the web server using the hard-coded email address and password you can see in the code:

``` bash
curl -X POST -H "Content-Type: application/json" -d '{
	"username": "example@robreid.io",
	"password": "fred123"
}' "http://localhost:1234/login"
```

The server will return a new `Bearer` token, with a `Timeout` value of 1 hour.  This means you'll need to refresh it again using the /v1/refreshToken endpoint within the hour, or lose your session:

``` bash
{
	"expire":"2017-03-28T19:10:57+01:00",
	"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE0OTA3MjQ2NTcsImlkIjoiZXhhbXBsZUByb2JyZWlkLmlvIiwib3JpZ19pYXQiOjE0OTA3MjEwNTcsInJvbGUiOiJhZG1pbiJ9.til0AFO-aJyCf64s4lbdWEL0_gZ0ZEId1F1Ii5YQWo0"
}
```

Thirdly, whack that into a new /v1/hello GET request, using the Bearer token we received:

``` bash
curl -X GET -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE0OTA3MjQ2NTcsImlkIjoiZXhhbXBsZUByb2JyZWlkLmlvIiwib3JpZ19pYXQiOjE0OTA3MjEwNTcsInJvbGUiOiJhZG1pbiJ9.til0AFO-aJyCf64s4lbdWEL0_gZ0ZEId1F1Ii5YQWo0" "http://localhost:1234/v1/hello"
```

The server will return our email address and role, as instructed, indicating we've successfully logged in and our role has been read from the database (to be used in subsequent requests):

``` bash
id: example@robreid.io
role: admin
```

The values you place into these custom claims are **not** encrypted, so make sure you don't store any sensitive information, in case the token is misplaced.

Lastly, refresh the token.  You can do this right up to the `MaxRefresh` time:

``` bash
curl -X GET -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE0OTA3MjQ2NTcsImlkIjoiZXhhbXBsZUByb2JyZWlkLmlvIiwib3JpZ19pYXQiOjE0OTA3MjEwNTcsInJvbGUiOiJhZG1pbiJ9.til0AFO-aJyCf64s4lbdWEL0_gZ0ZEId1F1Ii5YQWo0" "http://localhost:1234/v1/refreshToken"
```

The server will provide you with a fresh token, which can be used for as long as the `Timeout` allows (which in our case, is 1 hour):

``` bash
{
	"expire":"2017-03-28T19:19:04+01:00",
	"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE0OTA3MjUxNDQsImlkIjoiZXhhbXBsZUByb2JyZWlkLmlvIiwib3JpZ19pYXQiOjE0OTA3MjEwNTcsInJvbGUiOiJhZG1pbiJ9.Wuh062eImcJseoOG8XUBpcFgi09CZvND59Aclx2C3PE"
}
```