---
title: "2/4 CREATING A ML CLASSIFIER GRPC SERVICE WITH GOLANG (implementing a server)"
date: 2021-11-05T15:37:11+01:00
draft: false
tags: ['iris']
---

DISCLAIMER: This is intending to be a learning exercise and may not be the most efficient way to do things. This is intended to be a multi-part blog post describing how to create a recommender gRPC service in Go.

For the full source code, visit [trelore/iris-classification](https://github.com/trelore/iris-classification).

## Intro

In this section we're going to implement a gRPC service and test it via manual external tools [evans](https://github.com/ktr0731/evans).

## Setting up

For this we will ignore all machine learning and just get our server returning a random iris classification.
Simple enough.
For this we'll be using [cobra](github.com/spf13/cobra), a library to help make go executables.
We'll also be adopting some patterns highlighted by Mat Ryers [How I write Go Services](https://www.youtube.com/watch?v=rWBSMsLG8po) talk.
Let's start!

```go
package main

import "github.com/trelore/iris-classification/svc/cmd"

func main() {
	cmd.Execute()
}
```

Personally I try to keep main as small as possible, it's a hard package to test and reuse, so why stick around in it?
Let's write our Execute function.

```go
package cmd

// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
	Use:   "svc",
	Short: "A service to predict iris classifications",
	Run:   run,
}

func run(cmd *cobra.Command, args []string) {}

// Execute adds all child commands to the root command and sets flags appropriately.
// This is called by main.main(). It only needs to happen once to the rootCmd.
func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```

Execute is a minimal function, aiming to just call the cobra commands Execute function.
This gives us a `run()` function we can test more easily, we also get passed args and the cmd itself should we need them (we won't in this case).
Another thing to highlight is the fields we set in `rootCmd`, we set the usage name, a short description of what we're doing, and the function to run.
Let's look into that `run()` function in further detail, we need to set up our server still.

```go
import (
	pb "github.com/trelore/iris-classification/proto/gen/go/iris_classification/v1"
	"github.com/trelore/iris-classification/svc/server"
	"google.golang.org/grpc"
)

func run(cmd *cobra.Command, args []string) {
	s := server.New()
	grpcS := grpc.NewServer()
	defer grpcS.GracefulStop()

	pb.RegisterIrisClassificationServiceServer(grpcS, &s)

	address := ":32400"
	log.Printf("listening to address %s", address)
	listener, err := net.Listen("tcp", address)
	if err != nil {
		log.Fatal(err)
	}
	grpcS.Serve(listener)
}
```

The first thing we do is create a new server struct `s` (we'll get around to how that looks next), this code is in another package for separation of concerns.
The next two lines of code are creating a gRPC server, and telling it to shutdown gracefully once we're done.
After that we must register the gRPC server, at this point the code will complain if our server doesn't implement the functions in part 1.
Next we create a port to list on, a listener, and then run `grpcS.Serve(listener)` this is like `http.ListenAndServe`, but for gRPC.

Now we have our cmd package ready to run, let's implement the server.

## Implementing the server

Let's take a look

```go
package server

import (
	"context"

	pb "github.com/trelore/iris-classification/proto/gen/go/iris_classification/v1"
)

// New returns a new S
func New() S {
	return S{}
}

// S Implements the IrisClassificationService
type S struct {
	pb.UnimplementedIrisClassificationServiceServer
}

// Predict implements proto
func (s *S) Predict(ctx context.Context, req *pb.PredictRequest) (*pb.PredictResponse, error) {
	return nil, status.Errorf(codes.Unimplemented, "method Predict not implemented")
}

```

We have a struct `S`, which is our server, and a `New()` func to instantiate the struct. 
It's important to use the Unimplemented server within our own server, there was a long discussion around pros and cons of implementing it [here](https://github.com/grpc/grpc-go/issues/3669).
It is possible to turn it off, but I'd recommend keeping it anyway.
Lastly we have the `Predict` function, which for the time being we've just copied the Unimplemented Servers function of `Unimplemented`.
This is the function we want to eventually predict iris classifications, in the meantime let's just make it return a random classification.

We can start by defining a var outside our Predict function `var irisFlowers = []string{"iris setosa", "iris versicolor", "iris virginica"}`.
After that we can use the [rand](https://pkg.go.dev/math/rand) package to pick a random element from our slice.

```go
// Predict implements proto
func (s *S) Predict(ctx context.Context, req *pb.PredictRequest) (*pb.PredictResponse, error) {
	rand.Seed(time.Now().Unix()) // initialize global pseudo random generator
	iris := irisFlowers[rand.Intn(len(irisFlowers))]

	return &pb.PredictResponse{
		Predicition: iris,
	}, nil
}
```

Now we should be all done with our server, let's test it.
- `go run svc/main.go`

This gets our service up and running, you should see a log line similar to
- `2021/11/07 11:54:55 listening to address :32400`

## Introducing Evans

[Evans](https://github.com/ktr0731/evans) is a tool to call gRPC services from the command line.
From the root of our repo, run 
- `evans -p 32400 proto/idl/iris_classification/v1/service.proto`

From here we can explore our gRPC service through various commands like `show`, and `desc`. 
After a bit of exploring, we'll run `call Predict`, this allows us to enter the details of the Request message from our first blog post.
Enter in some random data (it doesn't matter what so long as it's the right type), as we return a random iris classification anyway.
You should see something like this;

```sh
iris_classification.v1.IrisClassificationService@127.0.0.1:32400> call Predict
petal_length (TYPE_FLOAT) => 1.3
petal_width (TYPE_FLOAT) => 2.1
sepal_length (TYPE_FLOAT) => 5.6
sepal_width (TYPE_FLOAT) => 3.4
{
  "predicition": "iris setosa"
}
```

## In Summary
We have;

- Created a gRPC service
- Tested it runs correctly

Next we will;
- Create a model for our service to consume
