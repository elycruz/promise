# PROMISE
[![Go Report Card](https://goreportcard.com/badge/github.com/chebyrash/promise)](https://goreportcard.com/report/github.com/chebyrash/promise)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat)](https://github.com/chebyrash/promise)
[![Build Status](https://travis-ci.org/chebyrash/promise.svg?branch=master)](https://travis-ci.org/chebyrash/promise)

## About
Promises library for Golang. Inspired by [JS Promises.](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

Supports automatic panic recovery and nested promise flattening.

## Installation

    $ go get -u github.com/chebyrash/promise

## Quick Start
```go
var p = promise.New(func(resolve func(interface{}), reject func(error)) {
  // Do something asynchronously.
  const sum = 2 + 2

  // If your work was successful call resolve() passing the result.
  if sum == 4 {
    resolve(sum)
    return
  }

  // If you encountered an error call reject() passing the error.
  if sum != 4 {
    reject(errors.New("2 + 2 doesnt't equal 4"))
    return
  }

  // If you forgot to check for errors and your function panics the promise will
  // automatically reject.
  // panic() == reject()
})

// A promise is a returned object to which you attach callbacks.
p.Then(func(data interface{}) interface{} {
  fmt.Println("The result is:", data)
  return data.(int) + 1
})

// Callbacks can be added even after the success or failure of the asynchronous operation.
// Multiple callbacks may be added by calling .Then or .Catch several times,
// to be executed independently in insertion order.
p.
  Then(func(data interface{}) interface{} {
    fmt.Println("The new result is:", data)
    return nil
  }).
  Catch(func(error error) error {
    fmt.Println("Error during execution:", error.Error())
    return nil
  })

// Since callbacks are executed asynchronously you can wait for them.
p.Await()
```

## Examples

### [HTTP Request](https://github.com/Chebyrash/promise/blob/master/examples/http_request/main.go)
```go
func parseJSON(data []byte) *promise.Promise {
  return promise.New(func(resolve func(interface{}), reject func(error)) {
    var body = make(map[string]string)
    
    err := json.Unmarshal(data, &body)
    if err != nil {
      reject(err)
      return
    }

    resolve(body)
  })
}

func main() {
  var requestPromise = promise.New(func(resolve func(interface{}), reject func(error)) {
    var url = "https://httpbin.org/ip"
		
    resp, err := http.Get(url)
    defer resp.Body.Close()

    if err != nil {
      reject(err)
      return
    }

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
      reject(err)
      return
    }
    
    resolve(body)
  })

  requestPromise.
    // Parse JSON body
    Then(func(data interface{}) interface{} {
      // This can be a promise, it will automatically flatten
      return parseJSON(data.([]byte))
    }).
    // Work with parsed body
    Then(func(data interface{}) interface{} {
      var body = data.(map[string]string)

      fmt.Println("[Inside Promise] Origin:", body["origin"])

      return body
    }).
    Catch(func(error error) error {
      fmt.Println(error.Error())
      return nil
    })
  
    // Your resolved values can be extracted from the Promise 
    // But you are encouraged to handle them in .Then and .Catch 
    value, err := requestPromise.Await()
    
    if err != nil {
      fmt.Println("Error: " + err.Error())
      return
    }
    
    origin := value.(map[string]string)["origin"]
    fmt.Println("[Outside Promise] Origin: " + origin)
}
```

### [Finding Factorial](https://github.com/Chebyrash/promise/blob/master/examples/factorial/main.go)

```go
func findFactorial(n int) int {
  if n == 1 {
    return 1
  }
  return n * findFactorial(n-1)
}

func findFactorialPromise(n int) *promise.Promise {
  return promise.Resolve(findFactorial(n))
}

func main() {
  var factorial1 = findFactorialPromise(5)
  var factorial2 = findFactorialPromise(10)
  var factorial3 = findFactorialPromise(15)
    
  factorial1.Then(func(data interface{}) interface{} {
    fmt.Println("Result of 5! is", data)
    return nil
  })

  factorial2.Then(func(data interface{}) interface{} {
    fmt.Println("Result of 10! is", data)
    return nil
  })
  
  factorial3.Then(func(data interface{}) interface{} {
    fmt.Println("Result of 15! is", data)
    return nil
  })

  promise.AwaitAll(factorial1, factorial2, factorial3)
}
```

### [Chaining](https://github.com/Chebyrash/promise/blob/master/examples/http_request/main.go)
```go
var p = promise.Resolve(0)

p.
  Then(func(data interface{}) interface{} {
    fmt.Println("I will execute first")
    return nil
  }).
  Then(func(data interface{}) interface{} {
    fmt.Println("And I will execute second!")
    return nil
  }).
  Then(func(data interface{}) interface{} {
    fmt.Println("Oh I'm last :(")
    return nil
  })

p.Await()
```
