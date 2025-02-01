---
title: "Bevy Safari"
date: 2024-10-14T11:18:01+01:00
draft: true
---

# Bevy Safari

N.B. This post is on Bevy 0.14, follow their amazing upgrade guides to keep up to date with the latest changes.

## Intro

This is a blog post to accompany a talk I did at JUXT's internal safari (knowledge sharing sessions).

Quickly on Bevy; it's a game engine written in Rust.
Bevy uses the Entity-Component-System (ECS) pattern to manage game state, and it's designed to be data-oriented and very fast.

Some fun stats for bevy (as of 14/10/2024):
- 35.7k stars on GitHub
- 1099 contributors
- 6 game jams
- `0.14.2` latest version (not a 1.x release in sight yet)

In this post we'll be talking about how to create a game using ECS - specifically a tower defense game.

## What's ECS?

ECS is a design pattern that's used in game development to manage game state.
It stands for Entity-Component-System, we'll dig into what each of these means as we progress.

### Entity

An entity is a unique identifier for a game object.
That's it, it's just an ID, nothing less nothing more.
In this game, I like to think of an entity as something like an instance of a particular enemy type.
i.e. Not the class of Orc enemies, but a specific orc enemy (remember though, it's just the ID for the game object).

### Component

A component is a piece of data that describes an entity.
In this orc example, a component could be the health of the orc, or the speed of the orc.

### System

This is where the magic happens.
A system is a function that operates on entities that have a specific set of components.
In our game, it could be moving all entities that have a `Position` and `Velocity` component (orcs, or projectiles from towers, etc).

## Getting Started

To start a new project there are a few different options; while I have not tried it yet, I'd recommend getting started through a template.
Templates are a great way to get started quickly, they often provide CI setup, wasm support, and more.

We'll start from the basics though, so let's create a new project with cargo:

```bash
cargo new tower-defense
cd tower-defense
```

Now we need to add Bevy to our project.
Add the following to your `Cargo.toml`:

```toml
[dependencies]
bevy = "0.14"
```

And now we can start writing some code!
