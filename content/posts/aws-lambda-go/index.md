---
title: "AWS Lambda Functions"
date: 2023-05-25T11:09:26-03:00
draft: false
---

If you have **experience** writing and deploying AWS Lambda functions in a
**non-strongly-typed language** or if you have **no experience** with it at all,
**this article is for you.**

I'll be explaining AWS Lambda from the basics up to developing using
test-first methodologies such as _TDD_ and deploying it.

_**Note:** I'll be referring to AWS Lambda functions as just Lambda._

## What's Lambda?

Lambda is an event-based system that allows you to configure compute and
networking resources to run code in some pre-defined runtimes. It operates on
the principle of functions (or HTTP handlers), where you can write and deploy
functions that handles a trigger, such as changes in data, user actions, or
scheduled tasks. When an event occurs, Lambda automatically executes the
corresponding function, allowing developers to respond quickly to real-time
events.

## Writing a Web program in Go

We usually start by running
[ListenAndServe](https://pkg.go.dev/net/http#ListenAndServe) from the
[net/http](https://pkg.go.dev/net/http#pkg-overview) package in the standard
library. It takes two arguments: a variable `addr` of type `string` and a variable
`handler` of type [Handler](https://pkg.go.dev/net/http#Handler), which is an
`interface` that declares the following method:

```go
ServeHTTP(ResponseWriter, *Request)
```

Therefore, any function that takes a `ResponseWriter` and a pointer to a `Request`
can be used as a handler in a Go program using the default HTTP server.

## Writing a Lambda in Go

Everything starts by running
[Start](https://pkg.go.dev/github.com/aws/aws-lambda-go/lambda#Start) from the
[aws-lambda-go](github.com/aws/aws-lambda-go/lambda) package. The
signature of the Start function is as follows:

```go
func Start(handler interface{})
```

It takes just one argument: a variable `handler` of type `interface{}` (empty interface).

At this point, it is important to read the entire documentation of the Start
function. _I won't duplicate it here_, but I'll assume that you have read it.

You should have noticed that the only similarity between a handler using Go
standard library and a handler for Lambda is that they both block after
starting. But don't worry, **we will cover all the details.**

Now, let's delve into what `TIn` and `TOut` are, but basically, they represent **events**.

## Events

The package [events](https://pkg.go.dev/github.com/aws/aws-lambda-go/events)
define all event types possibles to receive and return within a Lambda. Some are
event types to be used as input (TIn) and some are event types to be used as
output (TOut).

## Context

By employing context, developers can pass along important signals or
cancellation requests from lower-level components to the higher-level handler.
This mechanism enables the handler to be aware of any impending server shutdown
and perform necessary actions or cleanup tasks accordingly. The context serves
as a conduit for sharing critical information and facilitates the graceful
handling of server termination scenarios.

Lambda implement the following types:

```go
type LambdaFunctionURLRequestContext struct {
    AccountID    string
    RequestID    string
    Authorizer   *LambdaFunctionURLRequestContextAuthorizerDescription
    APIID        string
    DomainName   string
    DomainPrefix string
    Time         string
    TimeEpoch    int64
    HTTP         LambdaFunctionURLRequestContextHTTPDescription
}
```

LambdaFunctionURLRequestContext contains the information to identify the AWS
account and resources invoking the Lambda function.

```go
type LambdaFunctionURLRequestContextHTTPDescription struct {
    Method    string
    Path      string
    Protocol  string
    SourceIP  string
    UserAgent string
}
```

LambdaFunctionURLRequestContextHTTPDescription contains HTTP information for the request context.

## Scenario

Imagine that we are writing a Lambda that will receive an HTTP POST
request from an HTML form, send a message to a queue, and return HTML back to
the browser. With that in mind, let's consider the code we need to write:

1. Parse the POST data.
2. Call the AWS API to send the message to a given queue.
3. Write HTML back to caller.

## Code

### Handler function signature

Before starting any code, we need to understand and define the handler function
signature. Do you remember the rules, don't you? If not, please refer to this
[link](https://pkg.go.dev/github.com/aws/aws-lambda-go/lambda#Start).

1. First, it needs to be a function, obviously.
2. Does it need to be context-aware?

   Since we are going to perform a write (send the message to queue) and want to
   ensure that the server waits until this call finishes before shutting down if
   needed, our handler should receive a context.

3. Does it need any input?

   As we are receiving a POST request from a form on the internet, the handler
   must accept an event.

4. Should we return an error?

   Yes, there can be various issues in our code.

5. Should we return any data?

   Absolutely, we want to return HTML back to the browser.

After answering these questions, we can come up with the following function
signature:

```go
func Handler(ctx context.Contenxt, input events.SomeType) (events.SomeType, error)
```

Now, we need to figure out the types for these two events.

### Input type

For the event input type or TIn, we will use
[LambdaFunctionURLRequest](https://pkg.go.dev/github.com/aws/aws-lambda-go/events#LambdaFunctionURLRequest).

```go
type LambdaFunctionURLRequest struct {
    Version               string
    RawPath               string
    RawQueryString        string
    Cookies               []string
    Headers               map[string]string
    QueryStringParameters map[string]string
    RequestContext        LambdaFunctionURLRequestContext
    Body                  string
    IsBase64Encoded       bool
}
```

LambdaFunctionURLRequest contains data coming from the HTTP request to a Lambda
Function URL.

### Output type

For the event out type or TOut, we will use
[LambdaFunctionURLResponse](https://pkg.go.dev/github.com/aws/aws-lambda-go/events#LambdaFunctionURLResponse).

```go
type LambdaFunctionURLResponse struct {
    StatusCode      int
    Headers         map[string]string
    Body            string
    IsBase64Encoded bool
    Cookies         []string
}
```

LambdaFunctionURLResponse configures the HTTP response to be returned by Lambda
Function URL for the request.

### Prototyping

If you are not sure what to set on each of these fields, whether it is the first
time you are coding a Lambda in Go or whether you are working with a different
scenario that involves a type you have never worked with before, first, keep in
mind that all events are JSON.

Also, It's completely fine to do some prototyping, but
don't forget that our goal is to work with a test-first mindset.

### Returning error

Before we can start to code, I need to cover one more thing that I consider very
helpful from the Lambda server, it is the ability of returning errors from the
Handler.

Returning error from inside of the Lambda means the following:

1. The response status code is set to int `http.StatusInternalServerError` (500).
2. The response body is set to string `Internal Server Error`.
3. The error message is stored in the logs.

### Unit testing

Alright, let's finally start coding.

Download the following [zip file](files/mypackage.zip) containing the basic
package structure. After extracting the zip, you should have the following
files:

```text
- mypackage -> Go module root directory
- mypackage/go.mod -> Go mod file
- mypackage/go.sum -> Go checksum file
- mypacakge/mypackage_test.go -> Empty file with mypackage_test package
- mypacakge/mypackage.go -> Empty file with mypackage package
```

Our first step is to parse the POST data sent via a form. In order to test
whether we receive the expected data, let's write a test step by step. Update
the `mypackage_test.go` file.

```go
func TestHandleMyFunction_ReturnsStatusAcceptedGivenPOSTWithData(t *testing.T) {
  t.Parallel()
}
```

It's very pleasant to read a test name and understand what's going to happen.
Also, it's a good idea to run your tests in parallel. If you want more
information about it, read the nice article from John Arundel on [Test names
should be sentences](https://bitfieldconsulting.com/golang/test-names).

```go
want := http.StatusAccepted
```

If everything goes smoothly we want to return Status Accepted. Hence, this is
our want value for the test.

```go
lambdaReq := events.LambdaFunctionURLRequest{
  RequestContext: events.LambdaFunctionURLRequestContext{
    HTTP: events.LambdaFunctionURLRequestContextHTTPDescription{
      Method: http.MethodPost,
      Path:   "/",
    },
  },
  Headers: map[string]string{
    "content-type": "application/x-www-form-urlencoded",
  },
  Body: "firstName=Thiago&lastName=Carvalho",
}
```

Next, we need to define the `input event` or `TIn` of the handler. This is done
via instantiation of `lambdaReq` variable.

```go
lambdaResp, err := mypackage.HandleMyFunction(context.Background(), lambdaReq)
```

To get the result, we call the handler, which returns a response and an error as
defined before.

```go
if err != nil {
   t.Fatal(err)
}
```

We should fail the test if an error is found.

```go
got := lambdaResp.StatusCode
```

We set our `got` value to be the status code of the Lambda response.

```go
if want != got {
   t.Fatalf("want response status code %d, got %d", want, got)
}
```

Finally, we compare whether what we want is different from what we got, and fail
the test with enough information for debugging.

If we try to run tests we should get the following message:

```bash
$ go test
# mypackage_test [mypackage.test]
./mypackage_test.go:24:31: undefined: mypackage.HandleMyFunction
FAIL    mypackage [build failed]
```

If we are following TDD, that's exactly what we want, call the handler and make
sure it fails because the function does not exist yet. So now, we need to write
the minimum code to run our code and see the test failing.

Let's update `mypackage.go` and add the following content.

```go
func HandleMyFunction(ctx context.Context, input events.LambdaFunctionURLRequest) (events.LambdaFunctionURLResponse, error) {
  return events.LambdaFunctionURLResponse{}, nil
}
```

If we run the test, we get:

```bash
$ go test
--- FAIL: TestHandleMyFunction_ReturnsStatusBadRequestGivenPOSTMissing (0.00s)
    mypackage_test.go:30: want response status code 202, got 0
FAIL
```

Cool, that's what I was expecting to see. Let's write the minimum code to get
the test passing.

```go
return events.LambdaFunctionURLResponse{
  StatusCode: http.StatusAccepted,
}, nil
```

That's all we should do if we are following TDD. Run the test and see if it's passing.

Now, let's test the unhappy path. In order to parse the data, we need to know
the format in which the data is coming. Therefore, we should test whether we
receive the expected HTTP header for the content-type.

```go
func TestHandleMyFunction_ReturnsStatusBadRequestGivenPOSTWithUnexpectedContentType(t *testing.T) {
  t.Parallel()
  want := http.StatusBadRequest
  lambdaReq := events.LambdaFunctionURLRequest{
    RequestContext: events.LambdaFunctionURLRequestContext{
      HTTP: events.LambdaFunctionURLRequestContextHTTPDescription{
        Method: http.MethodPost,
        Path:   "/",
      },
    },
    Headers: map[string]string{
      "content-type": "bogus",
    },
    Body: "firstName=Thiago&lastName=Carvalho",
  }
  lambdaResp, err := mypackage.HandleMyFunction(context.Background(), lambdaReq)
  if err != nil {
    t.Fatal(err)
  }
  got := lambdaResp.StatusCode
  if want != got {
    t.Fatalf("want response status code %d, got %d", want, got)
  }
}
```

Running the test we should get:

```go
$ go test
--- FAIL: TestHandleMyFunction_ReturnsStatusBadRequestGivenPOSTWithUnexpectedContentType (0.00s)
    mypackage_test.go:59: want response status code 400, got 202
FAIL
```

I'm sure you can write enough code to get the test passing and keep going by yourself.

Before finishing, I want to cover one more step that we should definitely do. In
order to render content in the browser from a Lambda, we must set the header
content-type in the response. This is how I would test it:

```go
func TestHandleMyFunction_ReturnsExpectedContentTypeGivenPOSTWithBodyData(t *testing.T) {
  t.Parallel()
  want := "text/html"
  lambdaReq := events.LambdaFunctionURLRequest{
    RequestContext: events.LambdaFunctionURLRequestContext{
      HTTP: events.LambdaFunctionURLRequestContextHTTPDescription{
        Method: http.MethodPost,
        Path:   "/",
      },
    },
    Headers: map[string]string{
      "content-type": "application/x-www-form-urlencoded",
    },
    Body: "firstName=Thiago&lastName=Carvalho",
  }
  lambdaResp, err := mypackage.HandleMyFunction(context.Background(), lambdaReq)
  if err != nil {
    t.Fatal(err)
  }
  got := lambdaResp.Headers["content-type"]
  if want != got {
    t.Fatalf("want content-type %q, got %q", want, got)
  }
}
```

## Deploying

Deploying is very similar to other languages, but when working with compiled
languages, you need to upload the binary for the architecture you're running,
instead of just uploading the source code. In reality, you can upload a zip file
with just the binary and nothing else.

Note: You can run Lambda on AMD64 and ARM architectures.

## Conclusion

I hope this information gives you enough knowledge to develop your own Lambda
functions in Go.

Don't hesitate to contact me via LinkedIn or Twitter if you want to discuss
anything or need any help.
