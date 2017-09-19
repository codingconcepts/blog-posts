Let's Encrypt is rocking the SSL boat and the water's warm.

If you need that swanky EV banner, you're happy to pay $€£¥‎ for the privilege and you want to manually renew/replace expired certificates, this post is probably not for you.

##### tl;dr

Running services in containers ([Docker](https://www.docker.com) in particular) is becoming more and more popular, as is securing stuff for free with [Let's Encrypt](https://letsencrypt.org).  This post shows you how to combine the two.

##### What you'll need

* A domain name ([namecheap](http://namecheap.com) provide very affordable domain names)
* A server that can run in your domain (I like to use [Digital Ocean](http://digitalocean.com) droplets and One-Click Apps)

##### Code

The are a number of ways to spin up an HTTPS service in Go and while I could have used the built-in `http.Server` and gained loads of configurability, I'll keep thing simple and use the one provided by the [Echo](https://echo.labstack.com) server mux.

The code below sets up a single-endpoint API that returns "Hello, TLS!" to the caller.  I'll go through each non-obvious lines in more detail:

``` go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/labstack/echo"
	"golang.org/x/crypto/acme/autocert"
)

func main() {
	r := echo.New()
	r.GET("/", func(c echo.Context) (err error) {
		c.String(http.StatusOK, "Hello, TLS!")
		return
	})

	r.AutoTLSManager.HostPolicy = autocert.HostWhitelist("example.com")
	r.AutoTLSManager.Cache = autocert.DirCache("/certs")
	r.AutoTLSManager.Prompt = autocert.AcceptTOS

	if err := r.StartAutoTLS(fmt.Sprintf(":443")); err != nil {
		log.Fatal(err)
	}
}
```

Echo's `AutoTLSManager` simply wraps the `http.Server`, so if you're familiar with the `http.Server` TLS options, the code will feel familiar to you.  If not, [here's](https://blog.cloudflare.com/exposing-go-on-the-internet/) a great post about configuring an SSL `http.Server` to help you understand what's going on.

First, we let Let's Encrypt know what domain we're running in.  When the first request arrives, `autocert` will talk to Let's Encrypt, which will issue your server with some SSL-related challenges designed to prove domain ownership.

``` go
router.AutoTLSManager.HostPolicy = autocert.HostWhitelist("example.com")
```

Let's Encrypt rate-limit requests in their production environment to 10 certificate issues per domain per week.  This means you'll want to cache your certificates, to prevent asking for new ones every time your server starts up:

``` go
router.AutoTLSManager.Cache = autocert.DirCache("/certs")
```

In order to use Let's Encrypt, you'll need to accept their [Terms Of Service](https://community.letsencrypt.org/tos).  This can be done as part of the `autocert` negotiation and can be enabled as follows:

``` go
router.AutoTLSManager.Prompt = autocert.AcceptTOS
```

##### Dockerfile

I really like small containers, they're quick to build, quick to push from your build machine and quick to pull onto your production machines.

[Alpine](https://alpinelinux.org) is a *really* small Linux distribution that adds just few megabytes to the size of your container.  As the Alpine base image is so small, it doesn't contain the ca-certificates necessary to secure your server via Let's Encrypt by default, so we'll need to harness our Dockerfile to make sure these are in place.

The following Dockerfile builds on top of the `alpine` base image, installs the missing ca-certificates and mounts a volume called "/certs", which we'll map to a directory on the host machine to allow for caching between container restarts:

``` Dockerfile
FROM alpine
RUN apk update && apk add ca-certificates && rm -rf /var/cache/apk/*
COPY . /
CMD ["./hello"]
EXPOSE 443
VOLUME ["/certs"]
```

##### Running

Once pulled, the Docker container can be started with the following arguments.  The important one for TLS is the volume mapping **-v /certs:/certs**.  It creates and maps a "/certs" directory on the host machine to a directory that will be created by `autocert` on the container, allowing certificates to be persisted between container restarts:

``` bash
$ docker run -d -p 443:443 -v /certs:/certs hello
```

##### Resources

[Alpine ca-certificates](https://github.com/octoblu/docker-alpine-ca-certificates/blob/master/Dockerfile)

[Ubuntu 14.04.1 ca-certificates](http://blog.cloud66.com/x509-error-when-using-https-inside-a-docker-container/)