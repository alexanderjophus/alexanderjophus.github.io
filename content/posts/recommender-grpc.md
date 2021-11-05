---
title: "CREATING A RECOMMENDER GRPC SERVICE WITH MACHINE LEARNING AND GOLANG (THE GRPC BIT)"
date: 2021-02-07T15:37:11+01:00
draft: false
---

DISCLAIMER: This is intending to be a learning exercise and may not be the most efficient way to do things. This is intended to be a multi-part blog post describing how to create a recommender gRPC service in Go.

In this post we’re going to cover how to define a contract between a client and a server using gRPC. gRPC is an open source Remote Procedure Call framework, it uses protocol buffers as the description language. For further reading on gRPC, the [official gRPC docs](https://grpc.io/docs/) are fantastic. For further reading on protocol buffers, check out [googles documentation](https://developers.google.com/protocol-buffers).

A quick comparison for those familiar with REST/json, protocol buffers are essentially your JSON, and REST is essentially gRPC. Rather than reiterate why gRPC/proto, [as it has already been answered](https://cloud.google.com/blog/products/api-management/understanding-grpc-openapi-and-rest-and-when-to-use-them), we will explore how to use gRPC/proto.

## Defining the contracts

Let’s explore the proto defined below to understand what is going on.

```proto
syntax = "proto3";

package irisclassification;

option go_package = "github.com/trelore/iris-classification/proto/gen/go/iris-classification;irisclassificationpb";

// IrisClassificationService is a service
service IrisClassificationService {
  // Predict the Iris Classification
  //
  // The prediction will be one of;
  // - Iris-setosa
  // - Iris-versicolor
  // - Iris-virginica
  rpc Predict(PredictRequest) returns (PredictResponse);
}

// PredictRequest contains all info needed to
// classify the iris type
message PredictRequest {
  float petal_length = 1;
  float petal_width = 2;
  float sepal_length = 3;
  float sepal_width = 4;
}

// PredictResponse a string containing the iris type
message PredictResponse {
  string predicition = 1;
}
```

On line 1, we declare what syntax we’re using, we’re using proto3. Proto2 is also available and there are [pros/cons of using proto2](https://trelore.github.io/posts/recommender-grpc/#:~:text=pros/cons%20of%20using%20proto2). On line 3 declared a package of irisclassification, this allows better naming and prevents clashes between different packages. Lastly on line 5 we declare what go package this should be in. We still need to explicitly generate the go code, but this line declares where the module should be **github.com/trelore/iris-classification/proto/gen/go/iris-classification**, followed by what it should be named **irisclassificationpb**.

In the next chunk of our proto we define the predictor service itself. We give it a name **IrisClassificationService** on line 8, and a little description on line 7 (you can definitely get a little more descriptive). Then we document and define a function on line 15, what’s interesting to note is that our request and response messages and both named after the rpc, with the appropriate suffix. This is a convention in proto, itt allows for quick glanceability of what is a request message for what, and what’s a response.

Messages in proto define the contents of the contracts between the client and the server. Messages can contain other messages, primitives, maps, as well as repeated elements. For this example we’ve kept our protos fairly simple, the request contains a few floats containing details about the iris flower, and the response is a singular string containing the type of iris our service believes this to be.

## Generating the code
Proto looks great, but how can I use that in my Go/Java/C#/Python/ project? Below is a makefile for how to generate the Go code for our proto definitions. To find out how to apply this to your language, follow [one of the grpc tutorials](https://grpc.io/docs/tutorials/).

```makefile
## Locations to store google .proto files
google_api_base_location = vendor/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis/
location = $(google_api_base_location)google/api/
http_out = $(location)http.proto
annotations_out = $(location)annotations.proto

PROTOS := $(shell cd proto/idl; find . -path -prune -o -name '*.proto')

.PHONY: proto_cleanup
proto_cleanup: ## Delete generated proto files
	rm -rf proto/gen

.PHONY: proto
proto: proto_gen_go ## Generate the proto files

.PHONY: proto_gen_go
proto_gen_go: proto_imports ## generate the go files
	@mkdir -p proto/gen/go
	@cd proto/idl && protoc -I=. -I../../$(google_api_base_location) \
	--go_opt=paths=source_relative,plugins=grpc \
	--go_out=../gen/go ${PROTOS}

.PHONY: proto_imports
proto_imports: $(location)annotations.proto $(location)http.proto
```
We have a few helper recipies here, however day to day we would use make proto and make proto_clean. The former generates our go code, while the latter removes all of our generated code. There’s many resources to find out [how to use makefiles](https://www.tutorialspoint.com/makefile/index.htm).

We end up with the following code (and so much more!).

```go
// IrisClassificationServiceClient is the client API for IrisClassificationService service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type IrisClassificationServiceClient interface {
	// Predict the Iris Classification
	//
	// The prediction will be one of;
	// - Iris-setosa
	// - Iris-versicolor
	// - Iris-virginica
	Predict(ctx context.Context, in *PredictRequest, opts ...grpc.CallOption) (*PredictResponse, error)
}

// IrisClassificationServiceServer is the server API for IrisClassificationService service.
type IrisClassificationServiceServer interface {
	// Predict the Iris Classification
	//
	// The prediction will be one of;
	// - Iris-setosa
	// - Iris-versicolor
	// - Iris-virginica
	Predict(context.Context, *PredictRequest) (*PredictResponse, error)
}

// UnimplementedIrisClassificationServiceServer can be embedded to have forward compatible implementations.
type UnimplementedIrisClassificationServiceServer struct {
}

func (*UnimplementedIrisClassificationServiceServer) Predict(context.Context, *PredictRequest) (*PredictResponse, error) {
	return nil, status.Errorf(codes.Unimplemented, "method Predict not implemented")
}
Parsing this, we see the client is an interface which is great because this means we can mock it super easily. It defines exactly what we expect and nothing more. We see the same is also true for the server, there’s an interface we can fulfil. What’s also interesting is there is an UnimplementedIrisClassificationServiceServer, we can point to this in our server code when developing. This allows us to always be compliant with the interface, however we simply return Unimplemented for the functions we haven’t implemented (readers call on if this is good or not).
```

# In Summary
We have;

- Defined our service and endpoint
- Generated the Go code that we can use to implement our service

Next we will;

- Implement a service that returns a random result
