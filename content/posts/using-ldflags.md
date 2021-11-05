---
title: "ADDING COMPILE TIME INFORMATION TO YOUR GO BINARIES"
date: 2021-01-31T15:51:31+01:00
draft: false
---

From time to time it’s important to be able to tell certain metadata about exactly what is running. A couple important use cases may be; injecting the commit SHA to know exactly what code is running; injecting the build time to know how old the build is; and many more!

Introducing ldflags! It allows us to inject compile time information into our binaries. Let’s take a quick look at a small example.

```go
package main

import "fmt"

var version = "develop"

func main() {
	fmt.Printf("build version: %s", version)
}
```

As you can see, this is a relatively simple program that just prints the build version, which is hardcoded to be “develop”.

If we compile the code (go build -o app) and run it (./app) we get the output:

```
build version: develop
```
This looks good, when building and running the app while in development, we can see clearly it is a development version!

This is cool, how do I use ldflags to set the version to the commit SHA though?

```sh
export GIT_COMMIT=$(git rev-list HEAD) && \
  go build -o app -ldflags "-X main.version=$GIT_COMMIT"
```
Now if we run ./app we should see something like

```
build version: 3a9bd79b8185ba3738cee7279157deb8ea5101fe
```