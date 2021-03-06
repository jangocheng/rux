# rux router

[![GoDoc](https://godoc.org/github.com/gookit/rux?status.svg)](https://godoc.org/github.com/gookit/rux)
[![Build Status](https://travis-ci.org/gookit/rux.svg?branch=master)](https://travis-ci.org/gookit/rux)
[![Coverage Status](https://coveralls.io/repos/github/gookit/rux/badge.svg?branch=master)](https://coveralls.io/github/gookit/rux?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/gookit/rux)](https://goreportcard.com/report/github.com/gookit/rux)

Simple and fast request router for golang HTTP applications.

> **[中文说明](README_cn.md)**

- support route group
- support route path params
- support cache recently accessed dynamic routes
- support route middleware, group middleware, global middleware
- support generic `http.Handler` interface middleware
- support add handlers for handle `NotFound` and `NotAllowed`

## Godoc

- [godoc for github](https://godoc.org/github.com/gookit/rux)

## Quick start

```go
package main

import (
	"github.com/gookit/rux"
)

func main() {
	r := rux.New()
	
	// Static Assets
	// one file
	r.StaticFile("/site.js", "testdata/site.js")
	// allow any files in the dir.
	r.StaticDir("/static", "testdata")
	// file type limit
	r.StaticFiles("/assets", "testdata", "css|js")

	// Add Routes:
	
	r.GET("/", func(c *rux.Context) {
		c.Text(200, "hello")
	})
	r.GET("/hello/{name}", func(c *rux.Context) {
		c.Text(200, "hello " + c.Param("name"))
	})
	r.POST("/post", func(c *rux.Context) {
		c.Text(200, "hello")
	})
	r.Group("/articles", func() {
		r.GET("", func(c *rux.Context) {
			c.Text(200, "view list")
		})
		r.POST("", func(c *rux.Context) {
			c.Text(200, "create ok")
		})
		r.GET(`/{id:\d+}`, func(c *rux.Context) {
			c.Text(200, "view detail, id: " + c.Param("id"))
		})
	})

	// quick start
	r.Listen(":8080")
	// can also
	// http.ListenAndServe(":8080", r)
}
```

## Use Middleware

rux support use middleware, allow:

- global middleware
- group middleware
- route middleware

**Call priority**: `global middleware -> group middleware -> route middleware`

### Example

```go
package main

import (
	"fmt"
	"github.com/gookit/rux"
)

func main() {
	r := rux.New()
	
	// add global middleware
	r.Use(func(c *rux.Context) {
	    // do something ...
	})
	
	// add middleware for the route
	route := r.GET("/middle", func(c *rux.Context) { // main handler
		c.WriteString("-O-")
	}, func(c *rux.Context) { // middle 1
        c.WriteString("a")
        c.Next() // Notice: call Next()
        c.WriteString("A")
        // if call Abort(), will abort at the end of this middleware run
        // c.Abort() 
    })
	
	// add more by Use()
	route.Use(func(c *rux.Context) { // middle 2
		c.WriteString("b")
		c.Next()
		c.WriteString("B")
	})

	// now, access the URI /middle
	// will output: ab-O-BA
}
```

- **Call sequence**: `middle 1 -> middle 2 -> main handler -> middle 2 -> middle 1`
- **Flow chart**:

```text
        +-----------------------------+
        | middle 1                    |
        |  +----------------------+   |
        |  | middle 2             |   |
 start  |  |  +----------------+  |   | end
------->|  |  |  main handler  |  |   |--->----
        |  |  |________________|  |   |    
        |  |______________________|   |  
        |_____________________________|
```

> more please see [middleware_test.go](middleware_test.go) middleware tests

## Use http.Handler

rux is support generic `http.Handler` interface middleware

> You can use `rux.WrapHTTPHandler()` convert `http.Handler` as `rux.HandlerFunc`

```go
package main

import (
	"net/http"
	
	"github.com/gookit/rux"
	// here we use gorilla/handlers, it provides some generic handlers.
	"github.com/gorilla/handlers"
)

func main() {
	r := rux.New()
	
	// create a simple generic http.Handler
	h0 := http.HandlerFunc(func (w http.ResponseWriter, r *http.Request) {
		w.Header().Set("new-key", "val")
	})
	
	r.Use(rux.WrapHTTPHandler(h0), rux.WrapHTTPHandler(handlers.ProxyHeaders()))
	
	r.GET("/", func(c *rux.Context) {
		c.Text(200, "hello")
	})
	// add routes ...
	
    // Wrap our server with our gzip handler to gzip compress all responses.
    http.ListenAndServe(":8000", handlers.CompressHandler(r))
}
```

## Multi Domains

> code is ref from `julienschmidt/httprouter`

```go
package main

import (
	"github.com/gookit/rux"
	"log"
	"net/http"
)

type HostSwitch map[string]http.Handler

// Implement the ServeHTTP method on our new type
func (hs HostSwitch) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Check if a http.Handler is registered for the given host.
	// If yes, use it to handle the request.
	if router := hs[r.Host]; router != nil {
		router.ServeHTTP(w, r)
	} else {
		// Handle host names for which no handler is registered
		http.Error(w, "Forbidden", 403) // Or Redirect?
	}
}

func main() {
	// Initialize a router as usual
	router := rux.New()
	router.GET("/", Index)
	router.GET("/hello/{name}", func(c *rux.Context) {})

	// Make a new HostSwitch and insert the router (our http handler)
	// for example.com and port 12345
	hs := make(HostSwitch)
	hs["example.com:12345"] = router

	// Use the HostSwitch to listen and serve on port 12345
	log.Fatal(http.ListenAndServe(":12345", hs))
}
```

## Refer

- https://github.com/gin-gonic/gin
- https://github.com/gorilla/mux
- https://github.com/julienschmidt/httprouter
- https://github.com/xialeistudio/go-dispatcher

## License

**[MIT](LICENSE)**
