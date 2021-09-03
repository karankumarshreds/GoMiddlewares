Today we will learn how to implement middlewares in Go and also we will learn how to chain the middlewares in an efficient way without using any third party package.

## Middleware 
**Middleware** is an entity that intercepts the server's request/response life cycle. In simple words, it is a piece of code that runs before/after the server caters to a request with a response.
A middleware can do the following:
- Process the request before running business logic (authentication)
- Modify the request to the next handler function (attaching payload)
- Modify the response for the client 
- Logging.... and much more 

--- 

## Basic Middleware

Let's write a basic route handler

```Go
func main() {
    http.Handle("/", handler)
    http.HandleAndServe(":8000", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Prinln("Executing the handler")
    w.Write([]byte("OK"))
}
```

This piece of code simply returns "OK" response to the `GET` request at `http://localhost:8000/`.

Now let us create a simple middleware that will be a function that intercepts the request -> response lifecycle.

```Go
func middleware(originalHandler http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Println("Running before handler")
        w.Write([]byte("Hijacking Request "))
        originalHandler.Serve(w, r)
        fmt.Println("Running after handler")
    })
}
```

**Explanation**: This middleware function expects a function (our handler in this case) of type `handler` and will return a `handler` after tweaking the request response flow.

So what is does is:
1. Runs Println function
2. Writes to the response `"Hijacking Request"`
3. Serves the original handler which was passed as an argument 
4. Runs a Println function

Let us hook this up with the main function:

```Go
func main() {
    // converting our handler function to handler 
    // type to make use of our middleware 
    myHandler := http.Handlerfunc(handler)
    http.Handle("/", middleware(myHandler)) // 👈
    http.HandleAndServe(":8000", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Prinln("Executing the handler")
    w.Write([]byte("OK"))
}

func middleware(originalHandler http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Println("Running before handler")
        w.Write([]byte("Hijacking Request "))
        originalHandler.Serve(w, r)
        fmt.Println("Running after handler")
  })
}
```

Now if we hit it with a `GET` request at `http://localhost:8000` we will get 

> Client response:

```
Hijacking Request OK
```

👆 This shows the middleware added the response "Hijacking Request" to the original response by the handler, that is "OK".

> Server logs:

```
Running before handler
Executing the handler 
Running after handler 
```

---

## Real world example 

Now let us take what we have learned and build upon it. This middleware example has two goals:

1. Write a middleware that makes sure request has `Header` `"Content-Type"` `application/json`
2. Write a middleware that adds `current server time` to the reponse `cookie`

### Middleware #1 

```Go
func filterContentType(handler http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
    if r.Header.Get("Content-Type") != "application/json" {
      w.WriteHeader(http.StatusUnsupportedMediaType)
      w.Write([]byte("405 - Header Content-Type incorrect"))
      return
    }
    handler.ServeHTTP(w, r)
  })
}
```

### Middleware #2

```Go
func setTimeCookie(handler http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
    // Cookie here is a struct that represents an HTTP
    // cookie as sent in the Set-Cookie header of HTTP request
    cookie := http.Cookie{
      Name: "Server Time (UTC)" // can be anything
      Value: strconv.Itoa(int(time.Now().Unix()))
      // 👆 converted time to string
    }
    // now set the cookie to response 
    http.SetCookie(w, &cookie)
    handler.ServeHTTP(w, r)
  })
}
```

Now let us create a handler to handle the POST request, and then we will use the middlewares:

### Handler 

```Go
type Person struct {
	Firstname string
	Lastname string
}

// main handler 
func postHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != "POST" {
		w.WriteHeader(http.StatusMethodNotAllowed)
		w.Write([]byte("405 - Method Not Allowed"))
		return
	}

	var p Person
	decoder := json.NewDecoder(r.Body)
	err := decoder.Decode(&p)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		w.Write([]byte("500 - Internal Server Error"))
		return
	}
	defer r.Body.Close()

	fmt.Printf("Got firstName and lastName as %s, %s",
	p.Firstname, p.Lastname)
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("201 - Created"))
}
```

Now let we will (in the main function):
- Take the middlewares 
- Chain them together 
- Wrap them around the handler 

```Go
func main() {
	myHandler := http.HandlerFunc(postHandler)
	chain := filterContentType(setTimeCookie(myHandler)) // 👈
	http.Handle("/", chain)
	http.ListenAndServe(":8000", nil)
}
```

`filterContentType(setTimeCookie(myHandler))`
This will return the handler and run the middlewares in the order:

1. `filterContentType`
2. `setTimeCookie`
3. ... and then our handler 


Let us test this now:

`curl -H "Content-Type: application/json" -X POST
http://localhost:8000/city -d '{"firstname": "John", "lastname": "Doe"}'`

The server will respond with

```
201 - Created
```

Now let us test the Header middleware 

`curl -i -X POST http://localhost:8000/city -d '{"firstname": "John", "lastname": "Doe"}'`

The server will respond with 

```
415 - Unsupported Media Type
```

That is exactly what we wanted. Also you can use `postman` or `curl` command to check if the response has the attached cookie as well. And it will because of the other middleware.

But chaining these middlewares is not a neat process, imagine we had 5 middlewares, the chaining would look like: 

`chain := m1(m2(m3(m4(m5(handler)))))`

👆 This is not what we want, so let us create our own chaining middleware handling logic.

---

## Chaining Middlewares 



**Goal** is to use `chain := m1(m2(m3(m4(m5(handler)))))` as `chain := CreateChain(m1, m2, m3, m4, m5).Then(handler)`

Let us write a pseudo code for that we want to do:

1. We need a `CreateChain` function that takes an Slice/List of middlewares

2. We need a Then function that would be used as a method on `Chain` type object that will do run : `m1(m2(m3(m4(m5(handler)))))`

Let us create a struct for a `Middleware` and `Chain` (which is slice/list of middleware):

```Go
type Middleware func(http.Handler) http.Handler
type Chain []Middleware
```

Now let us write function `CreateChain`

```Go
// returns a Slice of middlewares 
func CreateChain(middlewares ...Middleware) Chain {
	var slice Chain
	return append(slice, middlewares...)
}
```

Now let us write function

```Go
func (c Chain) Then(originalHandler http.Handler) http.Handler {
	if originalHandler == nil {
		originalHandler = http.DefaultServeMux
	}

	for i := range c {
		// Same as to m1(m2(m3(originalHandler)))
		originalHandler = c[len(c) -1 -i](originalHandler)
	}
	return originalHandler
}
```

So here, the for loop will loop through `each` middleware from in the Slice/List of middlewares and keep it wrap itself around the originalHandler passed into the function.


Now we can use this in the main function as: 

```Go
func main() {
	myHandler := http.HandlerFunc(postHandler)
	chain := CreateChain(filterContentType, setTimeCookie).Then(myHandler) // 👈
	http.Handle("/", chain)
	http.ListenAndServe(":8000", nil)
}
```
