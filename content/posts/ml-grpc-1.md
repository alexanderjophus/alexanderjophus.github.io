---
title: "1/4 CREATING A ML CLASSIFIER GRPC SERVICE WITH GOLANG (the gRPC bit)"
date: 2021-02-07T15:37:11+01:00
draft: false
tags: ['iris']
---

Edited 05/11/2021: Uses buf for proto generation/linting

DISCLAIMER: This is intending to be a learning exercise and may not be the most efficient way to do things. This is intended to be a multi-part blog post describing how to create a recommender gRPC service in Go.

For the full source code, visit [alexanderjophus/iris-classification](https://github.com/alexanderjophus/iris-classification).

## Intro

We'll discover what proto is, create and define a service.
We will also generate code through buf.

## An intro to gRPC

In this post we’re going to cover how to define a contract between a client and a server using gRPC. gRPC is an open source Remote Procedure Call framework, it uses protocol buffers as the description language. For further reading on gRPC, the [official gRPC docs](https://grpc.io/docs/) are fantastic. For further reading on protocol buffers, check out [googles documentation](https://developers.google.com/protocol-buffers). If you're looking for advice on how to structure your proto directory, check [bufbuilds style guide](https://docs.buf.build/best-practices/style-guide#files-and-packages).

A quick comparison for those familiar with REST/json, protocol buffers are essentially your JSON, and REST is essentially gRPC. Rather than reiterate why gRPC/proto, [as it has already been answered](https://cloud.google.com/blog/products/api-management/understanding-grpc-openapi-and-rest-and-when-to-use-them), we will explore how to use gRPC/proto.

## Defining the service

Let’s explore the proto defined below to understand what is going on.

```proto
syntax = "proto3";

package iris_classification.v1;

option go_package = "github.com/alexanderjophus/iris-classification/proto/gen/go;irisclassificationpb";

// IrisClassificationService is a service to predict the Iris Classification given input
service IrisClassificationService {
  // Predict the Iris Classification
  rpc Predict(PredictRequest) returns (PredictResponse);
}

// petal length, petal width, sepal length, sepal width
message PredictRequest {
  // length of petal
  float petal_length = 1;
  // width of petal
  float petal_width = 2;
  //length of sepal
  float sepal_length = 3;
  // width of sepal
  float sepal_width = 4;
}

// the predication response
message PredictResponse {
  // prediction of what classification of iris it is
  string predicition = 1;
}

```

On line 1, we declare what syntax we’re using, we’re using proto3. Proto2 is also available and there are [pros/cons of using proto2](https://www.crankuptheamps.com/blog/posts/2017/10/12/protobuf-battle-of-the-syntaxes/). On line 3 declared a package of irisclassification, this allows better naming and prevents clashes between different packages. Lastly on line 5 we declare what go package this should be in. We still need to explicitly generate the go code, but this line declares where the module should be **github.com/alexanderjophus/iris-classification/proto/gen/go**, followed by what it should be named **irisclassificationpb**.

In the next chunk of our proto we define the predictor service itself. We give it a name **IrisClassificationService** on line 8, and a little description on line 7 (you can definitely get a little more descriptive). Then we document and define a function on line 15, what’s interesting to note is that our request and response messages and both named after the rpc, with the appropriate suffix. This is a convention in proto, it allows for quick glanceability of what is a request message for what, and what’s a response.

Messages in proto define the contents of the contracts between the client and the server. Messages can contain other messages, primitives, maps, as well as repeated elements. For this example we’ve kept our protos fairly simple, the request contains a few floats containing details about the iris flower, and the response is a singular string containing the type of iris our service believes this to be.

## Generating the code
Proto looks great, but how can I use that in my Go/Java/C#/Python/ project? Below is the `buf.gen.yaml` for how to generate the Go code for our proto definitions. To find out how to apply this to your language, look at the [reference guide](https://developers.google.com/protocol-buffers/docs/reference/overview).

```yaml
version: v1
plugins:
  - name: go
    out: gen/go
    opt: paths=source_relative
  - name: go-grpc
    out: gen/go
    opt:
      - paths=source_relative
```

We end up with the following code (and so much more!).

```go
// IrisClassificationServiceClient is the client API for IrisClassificationService service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://pkg.go.dev/google.golang.org/grpc/?tab=doc#ClientConn.NewStream.
type IrisClassificationServiceClient interface {
	// Predict the Iris Classification
	Predict(ctx context.Context, in *PredictRequest, opts ...grpc.CallOption) (*PredictResponse, error)
}

// IrisClassificationServiceServer is the server API for IrisClassificationService service.
// All implementations must embed UnimplementedIrisClassificationServiceServer
// for forward compatibility
type IrisClassificationServiceServer interface {
	// Predict the Iris Classification
	Predict(context.Context, *PredictRequest) (*PredictResponse, error)
	mustEmbedUnimplementedIrisClassificationServiceServer()
}

// UnimplementedIrisClassificationServiceServer must be embedded to have forward compatible implementations.
type UnimplementedIrisClassificationServiceServer struct {
}

func (UnimplementedIrisClassificationServiceServer) Predict(context.Context, *PredictRequest) (*PredictResponse, error) {
	return nil, status.Errorf(codes.Unimplemented, "method Predict not implemented")
}
```

Parsing this, we see the client is an interface which is great because this means we can test our code super easily. It defines exactly what we expect and nothing more. We see the same is also true for the server, there’s an interface we can fulfil. What’s also interesting is there is an `UnimplementedIrisClassificationServiceServer`, we can point to this in our server code when developing. This allows us to always be compliant with the interface, however we simply return Unimplemented for the functions we haven’t implemented (readers call on if this is good or not).

## In Summary
We have;

- Defined our service and endpoint
- Generated the Go code that we can use to implement our service

Next we will;

- Implement a service that returns a random result
