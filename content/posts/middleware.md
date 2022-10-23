---
title: "ADDING MIDDLEWARE TO A gRPC GO SERVICE"
date: 2021-11-14T11:17:51Z
draft: false
tags: ['iris']
---

WARNING: This space is moving very fast, I'll try to keep this project/blog up to date.

## Intro

Let's start by asking some questions about our service;
- What was the error that caused a failure?
- What percentage of calls are successful?
- Where did our service spend most of its time/what can we optimise?

Deploying a service without any logging/metrics/traces is going to cause frustration when answering these questions.

What's the best way to answer these questions?

Middleware.
With middleware we can essentially slot a piece of code _between_ when the call comes in to the service and when the function handling the call actually runs.
You could say, it runs in the middle.

## The middleware ecosystem

Introducing the [middleware ecosystem](https://github.com/grpc-ecosystem), in particular for our usecase the [go libraries](https://github.com/grpc-ecosystem/go-grpc-middleware).
This library comes with an incredible bit of functionality.
The first thing to note is the [chaining](https://pkg.go.dev/github.com/grpc-ecosystem/go-grpc-middleware#hdr-Chaining) pattern.
This allows the user to plug together multiple gRPC calls, in any order they wish (here be dragons though, we'll get to that).

With these building blocks, we can create a series of middleware that will first log our request, then add to our metrics provider, then our tracing provider and lastly ensure that we recover from panics correctly so our service stays alive.

Let's do it!

## Instrumenting our service

I'm going to pick the [iris classifier service](https://github.com/trelore/iris-classification).
For a basic run down, it's a service with an embedded ML model, a request comes in with iris sepal/petal width/length and the service predicts what type of iris it is.
If that sounds interesting, check the [blog](https://alexanderjophus.github.io/tags/iris/).

### Logging

Let's take a look at the options of [logging frameworks](https://pkg.go.dev/github.com/grpc-ecosystem/go-grpc-middleware#readme-logging) we can use.
Our options are essentially [zap](https://github.com/uber-go/zap) or [logrus](https://github.com/sirupsen/logrus).
We're going to arbitrarily pick zap for now, mainly because I more familiarity with it.
One day we may explore logging packages in the Go ecosystem.

With that in mind, we need to make a zap logger and pass it into the middleware func.
Let's take a look at how we can do that.

```go
zapLogger, err := zap.NewProduction()
if err != nil {
    log.Fatal(err)
}
defer zapLogger.Sync()
```

Zap heavily uses the [options pattern](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis). 
They also take it further by creating some sensible default loggers that you can then build on top of.
For this we just need to use the `NewProduction()`, but I would heavily suggest reading the [docs](https://pkg.go.dev/go.uber.org/zap) before taking it to production.

### Metrics

This is where the space gets moving fast, it used to be the case that you should just use a prometheus exporter and be done with it.
Recently [OpenMetrics](https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md) has become a foundation that aims to standardise metrics.
For now, we'll just use the prometheus exporter, there's no need to initialise anything for this.

## Tracing

I'd recommend this blog where Stephen Watts explains [the differences between opentracing projects](https://www.bmc.com/blogs/opentracing-opencensus-openmetrics/).
From what I understand opentracing and opencensus were two foundations aiming at standardising tracing, thankfully they combined into one standard `OpenTelemetry`.

What does that mean for us?
We're able (in theory) to swap our tracing provider between projects like [zipkin](https://github.com/openzipkin/zipkin-go) and [jaeger](https://github.com/jaegertracing/jaeger-client-go).

We're going to use jaeger, because as with logging that's what I'm more familiar with.
We can set up jaegertracing here like this.

```go
cfg := jaegercfg.Configuration{
    ServiceName: serviceName,
    Sampler: &jaegercfg.SamplerConfig{
        Type:  jaeger.SamplerTypeConst,
        Param: 1,
    },
    Reporter: &jaegercfg.ReporterConfig{
        LogSpans: true,
    },
}

tracer, closer, err := cfg.NewTracer()
if err != nil {
    log.Fatal(err)
}
defer closer.Close()

// Set the singleton opentracing.Tracer with the Jaeger tracer.
opentracing.SetGlobalTracer(tracer)
```

This also allows use to use tracing for more specific parts of our service if needed.

### Putting it together

We'll create a middlewares variable that is a chain of our tracing, metrics, logging, and a recovery middleware like so.

```go
middlewares := grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
    grpc_opentracing.UnaryServerInterceptor(grpc_opentracing.WithTracer(tracer)),
    grpc_prometheus.UnaryServerInterceptor,
    grpc_zap.UnaryServerInterceptor(zapLogger),
    grpc_recovery.UnaryServerInterceptor(),
))

grpcS := grpc.NewServer(middlewares)
defer grpcS.GracefulStop()
```

There is the consideration of the order of the middlewares, does it matter? Yes. Why?
Let's take a look at a middleware func as an example to see how it works and why the order is important.

This is the zap logger middleware

```go
// UnaryServerInterceptor returns a new unary server interceptors that adds zap.Logger to the context.
func UnaryServerInterceptor(logger *zap.Logger, opts ...Option) grpc.UnaryServerInterceptor {
	o := evaluateServerOpt(opts)
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
		startTime := time.Now()

		newCtx := newLoggerForCall(ctx, logger, info.FullMethod, startTime, o.timestampFormat)

		resp, err := handler(newCtx, req)
		if !o.shouldLog(info.FullMethod, err) {
			return resp, err
		}
		code := o.codeFunc(err)
		level := o.levelFunc(code)
		duration := o.durationFunc(time.Since(startTime))

		o.messageFunc(newCtx, "finished unary call with code "+code.String(), level, code, err, duration)
		return resp, err
	}
}
```

The first thing to notice is that it takes a logger and options, and returns a function.
For more information on [Go closures](https://gobyexample.com/closures), try the gobyexamples resource.
We're mainly looking at the function that is being returned though and how it works.

In the middle of the function we call `handler`, this is our call to the next middleware in the stack (or the actual server functionality itself).
The function also returns a response and err object to the parent middleware (or caller if it's the last in the stack).

The other bits of information are around the log itself, such as figuring out timing of the call, response code, and log level.

### Running our application

First lets run our tracing service, then we'll run our Go service.

```sh
docker run -d --name jaeger \
        -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
        -p 5775:5775/udp \
        -p 6831:6831/udp \
        -p 6832:6832/udp \
        -p 5778:5778 \
        -p 16686:16686 \
        -p 14268:14268 \
        -p 14250:14250 \
        -p 9411:9411 \
        jaegertracing/all-in-one:1.28
```

```sh
go run svc/main.go
```

You should be able to navigate to `http://localhost:2112/metrics`, to see your metrics via the Go service, and `http://localhost:16686` to access the jaeger tracing UI.
Make a request through [evans](https://github.com/ktr0731/evans) to `Predict`, and you should see a log statement on your service.
You'll also be able to see the trace in the tracing UI, and the metrics updated (if you refresh the page).

### Future steps

- [ ] Replace jaeger with opentracing
- [ ] Add more context to tracing - i.e. use a trace when the service makes the request to the ML model.