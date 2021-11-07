---
title: "4/4 CREATING A ML CLASSIFIER GRPC SERVICE WITH GOLANG (finishing the project)"
date: 2021-11-07T18:37:11+01:00
draft: true
tags: ['iris']
---

DISCLAIMER: This is intending to be a learning exercise and may not be the most efficient way to do things. This is intended to be a multi-part blog post describing how to create a recommender gRPC service in Go.

For the full source code, visit [trelore/iris-classification](https://github.com/trelore/iris-classification).

## Intro

We'll update our server to load the `theta.bin` model we created in post 3.
We'll update the `Predict` function to use the model in the server.

## Load the model

There's no point reading in the model every time we make a request, it'll cause the system to be slower than it needs to.
The question becomes, how much can we do in the server initialisation and how much do we need in the function.
The answer to that is that we need a tensor Dense matrix representation of the model, and the rest we can do in the function.

```go
// S Implements the IrisClassificationService service
type S struct {
	thetaT *tensor.Dense
	pb.UnimplementedIrisClassificationServiceServer
}
```

Our server should look something like that.
If you followed part 3, it should be clear what we're about to chuck into the server, if you need to read more refer to that post and its references.

```go
// New returns a new S
func New() S {
	b, err := models.Data.ReadFile("theta.bin")
	if err != nil {
		log.Fatal(err)
	}
	var thetaT *tensor.Dense
	err = gob.NewDecoder(bytes.NewReader(b)).Decode(&thetaT)
	if err != nil {
		log.Fatal(err)
	}
	return S{thetaT: thetaT}
}
```

As always, we read in the file through our models package which embeds the model.
After it's read, we decode the bytes into a tensor Dense matrix.

## Combining user input with our model

Lastly, let's get our user input, the model and combine the two.
This should allow us to predict what type of iris the user input.

```go
// Predict implements proto
func (s *S) Predict(ctx context.Context, req *pb.PredictRequest) (*pb.PredictResponse, error) {
	g := gorgonia.NewGraph()
	theta := gorgonia.NodeFromAny(g, s.thetaT, gorgonia.WithName("theta"))

	values := []float64{
		req.GetSepalLength(),
		req.GetSepalWidth(),
		req.GetPetalLength(),
		req.GetPetalWidth(),
		1.0,
	}
	xT := tensor.New(tensor.WithBacking(values))
	x := gorgonia.NodeFromAny(g, xT, gorgonia.WithName("x"))
	y, err := gorgonia.Mul(x, theta)
	if err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}
	machine := gorgonia.NewTapeMachine(g)
	defer machine.Close()

	if err = machine.RunAll(); err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}

	var class string
	switch math.Round(y.Value().Data().(float64)) {
	case 1:
		class = "setosa"
	case 2:
		class = "virginica"
	case 3:
		class = "versicolor"
	default:
		return nil, status.Error(codes.Internal, "unknown iris")
	}
	machine.Reset()
	return &pb.PredictResponse{
		Predicition: class,
	}, nil
}
```

This should look almost identical to the test we ran in post 3, but with a few key changes.
We now get the values from user input, and we assign the classification to a variable which we then use on the response.
We can also test this using evans (as we saw in post 2), and you should see something like this:

```
iris_classification.v1.IrisClassificationService@127.0.0.1:32400> call Predict
sepal_length (TYPE_DOUBLE) => 5.1
sepal_width (TYPE_DOUBLE) => 3.5
petal_length (TYPE_DOUBLE) => 1.4
petal_width (TYPE_DOUBLE) => 0.2
{
  "predicition": "setosa"
}
```

## Closing remarks and future steps

Overall this project was very fun, I stopped and started more times than I'd like to count.
Like everything each iteration became better and better and I'm really happy with the end result.

Future steps:
- 'Productionise' the ML service with tracing, logs, metrics etc.
- Building the ML model through something like Kubeflow for auditing and question answering of _why_ we get certain answers.
- Pick a better ML project, we covered the fundamentals with this project, it'd be fun to create a movie recommender given user input.
