---
title: "3/? CREATING A ML CLASSIFIER GRPC SERVICE WITH GOLANG (creating a model)"
date: 2021-11-07T10:37:11+01:00
draft: false
---

DISCLAIMER: This is intending to be a learning exercise and may not be the most efficient way to do things. This is intended to be a multi-part blog post describing how to create a recommender gRPC service in Go.

For the full source code, visit [trelore/iris-classification](https://github.com/trelore/iris-classification).

## Intro

In this post, we'll learn how to create a model using [gorgonia](https://gorgonia.org/).
We will be following the [iris tutorial](https://gorgonia.org/tutorials/iris/) mostly, just changing a few bits here and there for our use case.
We will create the model, and store it in alongside our codebase.
Lastly, we'll create a small cli tool to ensure we get the results we expected.

## Boilerplate stuff

We'll create our cli application the same way we made our service in part 2 of this post.
Our run function will differ, but we'll get to that later.

## Loading our datasets

For this we will use one of my favourite features from [Go 1.16](https://go.dev/blog/go1.16), go:embed.
We will embed a csv into into our tool, so once we build the binary, we can move it around and the code is guaranteed to work.

Grab the csv you want and drop it into a `models` directory.
For a project this size it's best to keep it in the same package until patterns start to form, but to make the tutorial/blog as clear as possible I've separated it.
In this directory, we'll also want to have a go file that embeds the csv, allowing other packages to read data from our datasets.
Let's take a look how this works.

```go
package datasets

import (
	"embed"
)

//go:embed *.csv
var Data embed.FS
```

Done.
That's it.
We define our package as you'd expect, after that we import `embed`, it's important to import `embed` even if you're embedding strings or []byte.
The directive is then used above the variable we want to use.
We wild card the things we want to embed just so we can import multiple datasets if we want to.
You can and should read more about [go:embed](https://blog.carlmjohnson.net/post/2021/how-to-use-go-embed/), it is incredibly useful.

With this package, all we need users to do to read a csv file is call `datasets.Data.ReadFile("iris.csv")`, which is pretty neat.

## Creating a model

We're going to copy all the code from [gorgonias iris example](https://github.com/gorgonia/gorgonia/blob/v0.9.17/examples/iris/main.go)
We need to make a few ammendments, first changing `func main()` to `func run(cmd *cobra.Command, args []string)`.
As well as using our new datasets package to read the file.
The start of the `getXYMat()` func should look something like this;

```go
b, err := datasets.Data.ReadFile("iris.csv")
if err != nil {
    log.Fatal(err)
}
df := dataframe.ReadCSV(bytes.NewReader(b))
```

For further reading of this, [gorgonia released a blog post talking about this](https://gorgonia.org/tutorials/iris/) in a lot more detail than I can.
The main take aways for this blog/tutorial is the `save` func, where we get a `theta.bin`, which is our model.
Awesome.

Our implementation of this section can be found in our repo at [cmd/train](https://github.com/trelore/iris-classification/tree/main/cmd/train).

You should be able to run `go run cmd/train/main.go`, which should print out its progress and leave a model in the root of our repository.
This should only take a second or so.

## Testing our model

As always, we create use cobra to make a simple cli.
We also embed our model, using a similar practice earlier.

Our code looks something like this;

```go
func run(cmd *cobra.Command, args []string) {
	b, err := models.Data.ReadFile("theta.bin")
	if err != nil {
		log.Fatal(err)
	}
	dec := gob.NewDecoder(bytes.NewReader(b))
	var thetaT *tensor.Dense
	err = dec.Decode(&thetaT)
	if err != nil {
		log.Fatal(err)
	}
	g := gorgonia.NewGraph()
	theta := gorgonia.NodeFromAny(g, thetaT, gorgonia.WithName("theta"))
	values := make([]float64, 5)
	xT := tensor.New(tensor.WithBacking(values))
	x := gorgonia.NodeFromAny(g, xT, gorgonia.WithName("x"))
	y, err := gorgonia.Mul(x, theta)
	if err != nil {
		log.Fatal(err)
	}
	machine := gorgonia.NewTapeMachine(g)
	defer machine.Close()
	values[4] = 1.0
	values[0] = 5.1
	values[1] = 3.5
	values[2] = 1.4
	values[3] = 0.2

	if err = machine.RunAll(); err != nil {
		log.Fatal(err)
	}
	switch math.Round(y.Value().Data().(float64)) {
	case 1:
		fmt.Println("setosa")
	case 2:
		fmt.Println("virginica")
	case 3:
		fmt.Println("versicolor")
	default:
		fmt.Println("unknown iris")
	}
	machine.Reset()
}
```

What we're interested in mostly are `values` 0-3, they are what represents the petal/sepal width/length.
Play about with those numbers and run the code with, you should see the application predicting from the 3 classes of iris.

Our implementation of this section can be found in our repo at [cmd/predict](https://github.com/trelore/iris-classification/tree/main/cmd/predict).

# In Summary
We have;

- Created a model
- Tested the model returns reasonable results
- Saved the model

Next we will;

- Embed the model in our service
