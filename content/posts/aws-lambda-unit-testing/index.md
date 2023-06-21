---
title: "Unit Testing - Mocking AWS API"
date: 2023-06-21T00:09:26-03:00
draft: false
---

Unit testing is an essential part of software development that allows us to
ensure the quality and reliability of our code. When working with services like
AWS, it's crucial to be able to fake the AWS API for efficient and reliable
testing. In this blog post, we will explore the importance of unit testing and
how to effectively mock the AWS API.

## Introduction

It is not uncommon to see Lambda code with few or even zero unit tests. While
there are many strategies available on the internet, I couldn't find an easy way
that satisfied me. Most people resort to using mocking services like localstack,
which is ideal for integration tests. However, I believe it is excessive for
unit testing. Unit tests should not depend on external services, especially
since we often run multiple tests frequently, especially when following TDD
principles.

I came up with what I consider a decent approach that will be described here.

## Go and AWS SDK FTW

The design of the Go language makes it easier, and AWS has implemented the APIs
correctly.

### First-Class Functions

n Go, you can set a type to be a function signature. Hence, you can write the
following code:

```go
package main

import "fmt"

func sayHelloTo(name string) string {
    return fmt.Sprintf("Hello, %s", name)
}

type greeting func(string) string

func main() {
    var greeting greeting
    greeting = sayHelloTo
    fmt.Println(greeting("Thiago"))
}
```

It should produce the output `Hello, Thiago`. Run this code in the [playground](https://go.dev/play/p/XdF63jvZydI).

### Each service has it own types

You'll notice that the AWS SDK provides a dedicated type for each service. For
example, when working with S3, you can use the following function signature to
list buckets:

```go
ListBuckets(ctx context.Context, params *ListBucketsInput, optFns ...func(*Options)) (*ListBucketsOutput, error)
```

If you have read my previous blog post about [Coding Lambda
Functions](../aws-lambda-go/index.md), you are already familiar with this
function signature pattern. If not, I will briefly explain it to you.

First of all, you need to pass a `context` and the input type
`ListBucketsInput`, which returns a `ListBucketsOutput` and an `error`. So now
it should be clear that each service has its own types. You are required to
provide a pre-defined input and you will receive a pre-defined output.

## The gotcha

In my unit tests, I can **inject a function that validates my input and returns
the output without having to communicate with the AWS API**.

### Examples

The most fundamental aspect that I need to test is whether the API is being
called.

```go
func TestHandleLandingPage_CallsResolveCustomerWithContext(t *testing.T) {
```

```go
    called := false
```

The very first thing I do is instantiate a new variable called `called` with
value `false` meaning that the API has not been called.

```go
    body := base64.StdEncoding.EncodeToString([]byte("inputName=bogus&inputEmail=bogus"))
    lambdaReq := events.LambdaFunctionURLRequest{
        RequestContext: events.LambdaFunctionURLRequestContext{
            HTTP: events.LambdaFunctionURLRequestContextHTTPDescription{
                Method: http.MethodPost,
                Path:   "/",
            },
        },
        Headers: landingpage.ContentTypeFormURLEncoded,
        QueryStringParameters: map[string]string{
            "x-amzn-marketplace-token": "bogus",
        },
        Body:            body,
        IsBase64Encoded: true,
    }
```

Here, we only have some paperwork to call the lambda function. If you want to
understand it, please read my [previous blog post](../aws-lambda-go/index.md).

```go
    l := landingpage.LandingPage{
        ResolveCustomerWithContext: func(ctx context.Context, input *marketplacemetering.ResolveCustomerInput, opts ...request.Option) (*marketplacemetering.ResolveCustomerOutput, error) {
            called = true
            return &marketplacemetering.ResolveCustomerOutput{
                CustomerIdentifier: new(string),
                ProductCode:        new(string),
            }, nil
        },
    }
```

This is the injection bit which does the magic for the unit test. During the
unit test the function is going to do two things:

1. Set `called` to `true`.
2. Return an empty output - as it does not matter for this specific test - and a
   nil error.

```go
    _, err := l.HandleLandingPage(context.Background(), lambdaReq)
    if err != nil {
        t.Fatal(err)
    }
    if !called {
        t.Fatal("function ResolveCustomerWithContext not called")
    }

}
```

Let's deep dive into another example.

```go
func TestHandleLandingPage_SetsProperRegistrationTokenInResolveCustomerWithContextAPICall(t *testing.T) {
```

So, I want to test that when I call my function `HandleLandingPage` it will invoke
the AWS API `ResolveCustomer` with the correct `Registration Token`.

If you stop to think about it, I don't need to test whether AWS does the right
thing when I call the API. What I really need to test is whether I'm sending the
right input.

```go
    want := "CallsResolveCustomerWithCorrectRegistrationToken"
```

This is my fake registration token that I want to check that was passed
correctly to the AWS API.

```go
    body := base64.StdEncoding.EncodeToString([]byte("inputName=bogus&inputEmail=bogus"))
    lambdaReq := events.LambdaFunctionURLRequest{
        RequestContext: events.LambdaFunctionURLRequestContext{
            HTTP: events.LambdaFunctionURLRequestContextHTTPDescription{
                Method: http.MethodPost,
                Path:   "/",
            },
        },
        Headers: landingpage.ContentTypeFormURLEncoded,
        QueryStringParameters: map[string]string{
            "x-amzn-marketplace-token": "CallsResolveCustomerWithCorrectRegistrationToken",
        },
        Body:            body,
        IsBase64Encoded: true,
    }
```

Similar to before, this is the necessary setup to call a lambda function from
the test. If you want to understand it, please refer to my [previous blog
post](../aws-lambda-go/index.md).

```go
    l := landingpage.LandingPage{
        ResolveCustomerWithContext: func(ctx context.Context, input *marketplacemetering.ResolveCustomerInput, opts ...request.Option) (*marketplacemetering.ResolveCustomerOutput, error) {
            got := *input.RegistrationToken
            if want != got {
                t.Fatalf("want registration token %q, got %q", want, got)
            }
            return &marketplacemetering.ResolveCustomerOutput{
                CustomerIdentifier: new(string),
                ProductCode:        new(string),
            }, nil
        },
    }
```

Here, we are injecting the function `ResolveCustomerWithContext`. We validate
that the received registration token is what we expect and fail the test if it
is not equal. Finally, we return the output with an `empty customeridentifier
and productcode`, along with a `nil error`.

```go
    lambdaResp, err := l.HandleLandingPage(context.Background(), lambdaReq)
    if err != nil {
        t.Fatal(err)
    }
    if lambdaResp.StatusCode != http.StatusAccepted {
        t.Fatalf("unexpected response status code %d", lambdaResp.StatusCode)
    }
}
```

We assert that error is nil which we know will be and that the status code from
lambda response is status accepted.

## Conclusion

I hope that after reading this blog post, your development of Lambda functions
becomes faster and more efficient. By understanding the concepts of unit testing
and mocking the AWS API, you can streamline your development process, ensuring
the reliability and correctness of your code. Embracing these techniques will
enable you to write comprehensive tests, validate inputs and outputs, and
iterate on your code with confidence. Happy coding!
