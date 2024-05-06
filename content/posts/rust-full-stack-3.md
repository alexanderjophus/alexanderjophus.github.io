---
draft: true
date: 2024-04-21T10:14:53+01:00
title: "Creating a full stack web app in Rust 3/3"
tags: ['rust']
---

[Source Code](https://github.com/alexanderjophus/cyber-sleuth)

## Intro/Recap

In this series we have/will;
- ✅ Written a simple web server using Pavex
- ✅ Used Diesel to interact with an in mem database
- ✅ Deployed the application using Shuttle
- 🚧 Write a front end using Dioxus
- 🚧 Host the frontend on github pages

Out API exposes 2 endpoints currently:
- `GET /digimon` - Returns a list of all Digimon (filterable)
- `GET /digimon/:name` - Returns a specific Digimon by name

## Dioxus

From the [Dioxus](https://dioxuslabs.com/) website:

Dioxus is a Rust library for building apps that run on desktop, web, mobile, and more.

For those that remember my [rust-journey]({{< ref "posts/rust-journey.md#a-few-small-websites-using-dioxus" >}}) post, I built a couple of small websites using it. I also contributed back bits and pieces to the project.

This was all done on their 0.4.x release, and I'm excited to see what they've done with the 0.5.x release.
Which we'll dive into in this post.

## The Plan

One of the things that always confused me coming from Pokémon to Digimon is the complex evolution trees.
So, I thought it would be cool to build a simple web app that shows the evolution tree of a given Digimon, from Baby to Mega.

