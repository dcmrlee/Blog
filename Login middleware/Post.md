# Practical Golang: Writing a simple login middleware

## Introduction

In this part we'll be creating a simple ***middleware*** you can easily apply to your handlers to get *authentication/authorization*. Middleware like this is an awesome way to add additional functionality to your Go server. Here we will only do *authorization* as we will only ask for a password, not a login. Although if you want, then you can easily extend this system to any authentication/authorization you'd like.

## Implementation

We will mainly use the *stdlib*, and will use *cookies* to remember who's already logged in. To generate the cookie values we will use the go.uuid library. So remember to
```
	go get github.com/satori/go.uuid
```

 We will start with a basic structure of our system with an already existing "Hello World!!!" handler:

```go
package main

import (
	"net/http"
	"fmt"
	"github.com/satori/go.uuid"
	"sync"
)

const loginPage = "<html><head><title>Login</title></head><body><form action=\"login\" method=\"post\"> <input type=\"password\" name=\"password\" /> <input type=\"submit\" value=\"login\" /> </form> </body> </html>"

func main() {

	http.Handle("/hello", helloWorldHandler{})
	http.HandleFunc("/login", handleLogin)

	http.ListenAndServe(":3000", nil)
}

type helloWorldHandler struct {
}

func (h helloWorldHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello World!!!")
}

type authenticationMiddleware struct {
	wrappedHandler http.Handler
}

func (h authenticationMiddleware) ServeHTTP(w http.ResponseWriter,r *http.Request) {
}

func authenticate(h http.Handler) authenticationMiddleware {
	return authenticationMiddleware{h}
}

func handleLogin(w http.ResponseWriter, r *http.Request) {
}
```

Ok, lets go over this code.

```go
package main

import (
	"net/http"
	"fmt"
	"github.com/satori/go.uuid"
	"sync"
)

const loginPage = "<html><head><title>Login</title></head><body><form action=\"login\" method=\"post\"> <input type=\"password\" name=\"password\" /> <input type=\"submit\" value=\"login\" /> </form> </body> </html>"

func main() {

	http.Handle("/hello", helloWorldHandler{})
	http.HandleFunc("/login", handleLogin)

	http.ListenAndServe(":3000", nil)
}

type helloWorldHandler struct {
}

func (h helloWorldHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello World!!!")
}
```

First we declare the package, imports and the html code of the login page.

We also declare the basic hello world handler.

Now to get to the interesting part.

```go
type authenticationMiddleware struct {
	wrappedHandler http.Handler
}

func (h authenticationMiddleware) ServeHTTP(w http.ResponseWriter,r *http.Request) {
}

func authenticate(h http.Handler) authenticationMiddleware {
	return authenticationMiddleware{h}
}

func handleLogin(w http.ResponseWriter, r *http.Request) {
}
```

we create a handler which will supply authorization, and if authorized successfully, will let the user through to the underlying handler. We also define the *authenticate* method, a simple function wrapper over creating a struct, and a function to handle the login.

That means we can also define the last route, the secured hello world route.

```go
func main() {
	http.Handle("/hello", helloWorldHandler{})
	http.Handle("/secureHello", authenticate(helloWorldHandler{}))
	http.HandleFunc("/login", handleLogin)

	http.ListenAndServe(":3000", nil)
}
```

Ok, we will also need a simple client struct, which will just save if the target client session is authorized, and a map containing our Clients with cookie values being the keys.

```go
var sessionStore map[string]Client
var storageMutex sync.RWMutex

type Client struct {
	loggedIn bool
}
```

We also need the mutex for concurrent map access. In the main function we initialize the map:

```go
func main() {
	sessionStore = make(map[string]Client)
	http.Handle("/hello", helloWorldHandler{})
```

Now we can go to the authenticationMiddleware's ServeHTTP function:

We'll begin with checking if the cookie is present, if it isn't there, we'll continue and create a new. If the error is nonstandard then we just return.

```go
func (h authenticationMiddleware) ServeHTTP(w http.ResponseWriter,r *http.Request) {
	cookie, err := r.Cookie("session")
	if err != nil {
		if err != http.ErrNoCookie {
			fmt.Fprint(w, err)
			return
		} else {
			err = nil
		}
	}
```

We later check, unless the cookie exists, if it's saved in our map. If it's not, then we will later generate a new one.

```go
var present bool
var client Client
if cookie != nil {
	storageMutex.RLock()
	client, present = sessionStore[cookie.Value]
	storageMutex.RUnlock()
} else {
	present = false
}
```

Now, if the cookie wasn't present, then we can generate a new one!:

```go
if present == false {
	cookie = &http.Cookie{
		Name: "session",
		Value: uuid.NewV4().String(),
	}
	client = Client{false}
	storageMutex.Lock()
	sessionStore[cookie.Value] = client
	storageMutex.Unlock()
}
```

We can then set the cookie to our response writer, and if the client isn't logged in, send him the login page, however, if he is logged in, then we can send him what he wanted:

```go
	http.SetCookie(w, cookie)
	if client.loggedIn == false {
		fmt.Fprint(w, loginPage)
		return
	}
	if client.loggedIn == true {
		h.wrappedHandler.ServeHTTP(w, r)
		return
	}
}
```

So far so good, that's actually already about the main part of the middleware, now we'll write a function just for handling the login logic. The *handleLogin* function:

```go
func handleLogin(w http.ResponseWriter, r *http.Request) {
	cookie, err := r.Cookie("session")
	if err != nil {
		if err != http.ErrNoCookie {
			fmt.Fprint(w, err)
			return
		} else {
			err = nil
		}
	}
	var present bool
	var client Client
	if cookie != nil {
		storageMutex.RLock()
		client, present = sessionStore[cookie.Value]
		storageMutex.RUnlock()
	} else {
		present = false
	}

	if present == false {
		cookie = &http.Cookie{
			Name: "session",
			Value: uuid.NewV4().String(),
		}
		client = Client{false}
		storageMutex.Lock()
		sessionStore[cookie.Value] = client
		storageMutex.Unlock()
	}
	http.SetCookie(w, cookie)}
}
```

First we created the part which is accountable for the cookie handling, as in the recent function. Now we get to the form parsing and the actual login part.

```go
http.SetCookie(w, cookie)
err = r.ParseForm()
if err != nil {
	fmt.Fprint(w, err)
	return
}

if subtle.ConstantTimeCompare([]byte(r.FormValue("password")), []byte("password123")) == 1 {
	//login user
} else {
	fmt.Fprintln(w, "Wrong password.")
}
```

We parse the login form and check if the login conditions are met. Here we only need the password to be correct. If it is we log the client in:

```go
if subtle.ConstantTimeCompare([]byte(r.FormValue("password")), []byte("password123")) == 1 {
	client.loggedIn = true
	fmt.Fprintln(w, "Thank you for logging in.")
	storageMutex.Lock()
	sessionStore[cookie.Value] = client
	storageMutex.Unlock()
}
```

We use *subtle.ConstantTimeCompare* as it protects us from time-based attacks. (Thanks for the tip in the reddit comment.)

That's basically all we need. Now you can secure the routes you want easily.

## Conclusion

Remember, that for a secure implementation you need encrypt the network traffic using **SSL/TLS**, otherwise, somebody can just read the cookie and impersonate your user.

Another thing to consider is redirecting the user to the page he wanted to get to after logging in.

Have fun with creating other interesting middleware!
