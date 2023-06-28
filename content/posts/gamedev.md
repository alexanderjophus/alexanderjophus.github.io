---
title: "2D GAMEDEV IN GO?"
date: 2022-11-12T11:13:19Z
draft: true
---

# Intro

It's time we take a serious look at Go for game development.
If we were to have one of those websites that asks 'Are we gamedev yet?', personally I'd say yes.
Let's have a quick look at Awesome Go's [game development section](https://github.com/avelino/awesome-go#game-development).
There's a wealth of libraries and frameworks to choose from, but I'm going to focus on [ebitengine](https://ebitengine.org/).

Also a quick disclaimer: I'm not a game dev, so this is all one persons experience and exploration.
It could be all wrong.
I'll try to provide as much reading material as possible from people who know what they're talking about.

# Getting up to speed

I think for this blog post to be effective, we need to be on a level playing ground.
We'll assume a basic understanding of Go, but we'll also assume no knowledge of game development.

## How do I make a game?

Honestly, I'm still learning this.
So here's some reading material from people smarter than me:

## What is Ebitengine?

Ebitengine written by Hajime Hoshi is described as 'A dead simple 2D game engine for Go'.
Honestly, I think that's a pretty good description.

What makes a game engine?
It's a grey area so far as I understand it, but imo anything that helps you make a game is a game engine.
There are certain core features that are common to most game engines, such as ways to:
- Render graphics
- Handle input
- Handle audio
- Collision detection
- Create AI
- and more...

Ebiten does a lot of these things, but not all, and that's ok.
There's a lot of room for other libraries to fill in the gaps.
As such, this wouldn't be a great intro to Ebitengine without introducing the [ecosystem of ebitengine](https://github.com/sedyh/awesome-ebitengine).

## I want help, where do I go?

<!-- talk about the slack channel and discord here -->

# Getting started

It put me at ease when I realised that games are just a CLI app.
As such, we can create a file `cmd/main.go`, with the following contents (we'll explore what it does after):

```go
package main

import (
	"errors"
	"fmt"
	_ "image/png"
	"log"
	"os"

	"github.com/alexanderjophus/untitledtd"
	ebiten "github.com/hajimehoshi/ebiten/v2"
)

func main() {
	if len(os.Args) != 1 {
		log.Fatal("Usage: untitledtd")
	}
	ebiten.SetWindowTitle("Tower Defense")
	ebiten.SetFullscreen(true)
	g, err := untitledtd.NewGame()
	if err != nil {
		log.Fatal(fmt.Errorf("failed to create game: %w", err))
	}
	if err := ebiten.RunGame(g); err != nil {
		if errors.Is(err, ErrQuit) {
			return
		}
		log.Fatal(fmt.Errorf("failed to run game: %w", err))
	}
}

var ErrQuit = fmt.Errorf("quit")
```

Let's digest it.
First thing we do in the `main` function is check that we have no arguments.
I created a fakemon game originally, and used to pass in a save file, however I found that it hard to run if running the game in the browser (or via clicking a `.exe` file).
So I removed it.

Next we set the window title and fullscreen mode.
Not much to really say here, there is a bug I'm trying to iron out where fullscreen is not always set.

After that we create a new game.
We'll explore what that is in a moment, but the tl;dr is that the game must implement the [ebiten.Game](https://github.com/hajimehoshi/ebiten/blob/2.4/run.go#L24-L76) interface.
This is the core of the game engine, and is what allows us to run the game.
The main two functions here are `Update` and `Draw`, we'll explore them in more detail when we explore the untitledtd package.

Finally we call [ebiten.RunGame](https://github.com/hajimehoshi/ebiten/blob/2.4/run.go#L170-L211), which will handle pretty much all the game loop stuff for us.
It will call `Update` and `Draw` at the correct times, and handle input and audio.
We just need to plug in the functionality. Simple. You might even say it's dead simple.

# 