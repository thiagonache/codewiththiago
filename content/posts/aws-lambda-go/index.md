---
title: "AWS Lambda Functions"
date: 2023-05-24T11:09:26-03:00
draft: true
---

If you have **experience** writing and deploying AWS Lambda functions in a
**non-strongly-typed language** or if you have **no experience** with it at all,
**this article is for you.**

I'll be explaining AWS Lambda from the basics up to developing using
test-first methodologies such as _TDD_ and deploying it.

_**Note:** I'll be referring to AWS Lambda functions as just Lambda._

## What's Lambda?

Lambda is an event-based system that allows you to configure compute and
networking resources to run code in some pre-defined runtiems. It operates on
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
    Method    string `json:"method"`
    Path      string `json:"path"`
    Protocol  string `json:"protocol"`
    SourceIP  string `json:"sourceIp"`
    UserAgent string `json:"userAgent"`
}
```

LambdaFunctionURLRequestContextHTTPDescription contains HTTP information for the request context.

## Scenario

Imagine that we are writing a Lambda function that will receive an HTTP POST
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
   Since we are going to send a message to a queue and want to ensure that the
   server waits until this call finishes before shutting down if needed, our
   handler should receive a context.
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

If you are not sure what to set on each of these fields, whether it is the
first time you are coding a Lambda in Go whether you are working
with a different scenario that involves a type that you never worked before.
First, keep in mind that all events are JSON and mainly, **it's completely fine to do
some prototype** but don't forget that our goal is to work in a test first
mindset.

### Unit testing
