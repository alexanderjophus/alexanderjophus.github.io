---
title: "MAKING THE JUMP TO GRAPHQL (Formula 1 API)"
date: 2021-11-28T17:31:17Z
draft: false
---

To view the final repo browse my [github](https://github.com/trelore/formulagraphql)

## Intro

Say you have a rest API and for argument sake, it's a very RESTful implementation of a formula 1 API.
You can get a list of current constructors in the standings, it provides all the information about all the constructors (or bonus points you can specify the constructor you want and retrieve just that one).
Similarly you can get drivers standings, a list of circuits, and the schedule for a given season.

Your front ends may want to mix and match this data as they see fit.
The problem here is that they may underfetch or overfetch data.
By this we mean they use the API and retrieve less information than they need, or more information than they need.
That doesn't sound too bad, until you consider mobile data, sketchy internet connections, or just the sheer amount of data available.

How do we stop someone underfetching or overfetching data?
Have them declaritively state what information they need.

If they just need the top three of the drivers standings, and of those drivers just the driver code, have them state that and return just that.

## Getting Started with GraphQL in Go

There are a few different ways to get started with graphql in Go, for a fuller picture see this [comparison chart](https://gqlgen.com/feature-comparison/).
Personally I'd recommend [gqlgen](https://github.com/99designs/gqlgen), it's feature complete and works like a charm.

The first thing we want to do is follow the [quickstart guide](https://github.com/99designs/gqlgen#quick-start).
Following this, we import gqlgen into our project.

```sh
printf '// +build tools\npackage tools\nimport _ "github.com/99designs/gqlgen"' | gofmt > tools.go
go mod tidy
```

What the above snippet does is include an import to gqlgen in a `tools.go` file.
We can then run a command to initialise a 'hello world' style project.

```sh
go run github.com/99designs/gqlgen init
```

Lastly we can run our API and visit `http://localhost:8080/`.

```sh
go run server.go
```

## Expanding our GraphQL API

The first thing we're going to do is throw out the generated `schema.graphqls` file and rewrite our own.
We're going to create a query like below, which is to say we're going to create a `Query` called `ConstructorStandings`, which takes in a `StandingsFilter` which has default values for `year` and `top`, and it will return a `ConstructorStandingsReport`.

```graphql
type Query {
  ConstructorStandings(filter: StandingsFilter = {year: "current", top: -1}): ConstructorStandingsReport
}
```

That was a lot to unpack, we also need to define a couple of other things too!
Next we'll define our `StandingsFilter` and `ConstructorStandingsReport`.

```graphql
input StandingsFilter {
  year: String = current
  round: String
  top: Int = -1
}

type ConstructorStandingsReport {
  season: String
  round: String
  teams: [TeamStanding]
}

type TeamStanding {
  position: String
  points: String
  wins: String
  team: Constructor
}

type Constructor {
  id: String
  name: String
  url: String
  nationality: String
}
```

Hopefully this all makes a lot of sense, we'll briefly talk over what each bit means, but I recommend looking at the [graphql docs](https://graphql.org/learn/).
First, we have an `input`, this is a special type that means the user will be inputting these values.
We define default values here too, this allows a user to explicitly state a filter, but not have to worry about each value within the filter.
After that we have `types`, the only non-trivial syntax is the `teams` value where `[TeamStanding]`, means an array (or slice if you will) of `TeamStanding`.
Almost everything else is a `String`.

To regenerate that code we simply invoke the `gqlgen` library again.

```sh
go run github.com/99designs/gqlgen generate
```

This will create a bunch of files, most importantly is a `schema.resolvers.go` file, which will have a function defined like below.

```go
func (r *queryResolver) ConstructorStandings(ctx context.Context, filter *model.StandingsFilter) (*model.ConstructorStandingsReport, error) {...}
```

All we need to do is implement the filter, and make the request to the REST API, and the graphQL library will take care of the rest for us - such as which fields to be returned.

## Implementing the Resolvers

For brevity, I'm just going to link to the [code](https://github.com/trelore/formulagraphql/blob/4197607d66bfeda6b5dad235237d06bf56eb6a8b/graph/schema.resolvers.go#L21-L54).

If you've ever written a tool that calls a RESTful API this code should look familiar to you.
It's a bit scrappy - I will admit - however it gets the job done.
We make a request to the endpoint based on the filters we have, we check the response and deserialise the result.

That's it.
We're done.
Short section.
