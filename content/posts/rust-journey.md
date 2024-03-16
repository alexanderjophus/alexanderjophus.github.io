---
draft: false
date: 2024-03-16T09:01:07Z
title: "My Journey Into Rust (pt 1)"
tags: ['rust']
---

## Intro

Firstly, I'm back. I've been away for a while, I took a career break and got a dog. I rejoined the working world in January.

In that time I also began learning Rust in depth.
I had dabbled with it before, but I wanted to learn it properly.
I didn't/don't expect it to replace my love of Go, maybe I can use both in the future.

So, what have I built, what have I learned, and what's next?

## What I've Built

In order of when I built them.

### A prototype of a 3D game using Bevy

This is possibly my proudest build, I tried a few iterations, a few game concepts.
Originally I started building games in Ebitengine in Go, it was fun, but a few of the limitations of Go began to hit me.

Introducing [Horror](https://github.com/alexanderjophus/horror) (an untitled horror game).

I used Bevy for this, and honestly, the community there is something else - 100% recommended.
They have a discord, and I've never seen a community so willing to help.
The game jams are always so enticing (but never begrudgingly fit my schedule).

My conclusion: Give bevy a go, the docs need some help (if you fancy OSS contributions), however if you find older tutorials, the changelogs are incredible to help you migrate that tutorial to something you need.

### A few small websites using Dioxus

Code wise, this one is possibly the cleanest, though I've still learnt a lot since starting it, so I may go back and polish it.
I'm not a front end dev by any means, the javascript ecosystem scares the hell out of me.
I had a formula one website that I hadn't touched in 6 months written in js, naturally when I went back to it, it was broken beyond repair.

Introducing [pokesandwich](https://github.com/alexanderjophus/pokesandwich) - https://pokedex.alexanderjophus.dev/

Originally this was a companion to shiny hunting in Pokémon Scarlet and Violet.
Me and my partner at the time enjoyed shiny hunting together, and so often needed to hunt the same type.
This allowed us to select a region/type and showed us the original side by side with the shiny.

I've since added another feature for competitive play, where you can search for pokémon.
For example, if you want a pokemon that knows Tailwind and has a specific Ability and/or type, you can search for it.

I took a hiatus from Dioxus for a while as life got busy, however the website still just works (albeit I'm waiting for the 0.5 update to break everything).

My conclusion: Dioxus is a great library, I've had a lot of fun with it, and the community is second to none.

### A library to contribute something back to Dioxus

This is my most _magical_ library. I hadn't written a library crate before, and wanted to tick that off my bucket list.
I was hanging out in the Dioxus discord when someone mentioned about a slides plugin for Dioxus.
I thought, "I could do that", and so I did (probably terribly though).

Introducing [dioxus-slides](https://github.com/alexanderjophus/dioxus-slides) - A presentation tool using Dioxus.

First and foremost, a lot of the macro magic was copied from the Dioxus teams work on their router-macro crate.
This allowed me to focus on making the developer experience as good as possible.
So now, a user defines an enum, derives the Slidable trait, and off they go.
It supports all renderers that Dioxus supports, though most are not tested.

Fun fact, I gave a talk on Go vs Rust at my local Go meetup, and I used this library to make the slides and presented them in the terminal.
It was genuinely pleasant, though the styling left a lot to be desired (that's on me).

My conclusion: Writing a library is fun, and I'd like to do it again. The Dioxus team are incredibly helpful, and if they're not available, someone on their discord will be.

### In Progress: shllchckr and a reimagining of my final year project in Rust

I'm working on two projects simultatneously at the moment, because who cares for weekends?

The first is shllchckr, a linting tool for shell scripts loosely based on shellcheck. Will it see the light of day? God no.

The second is a reimagining of my final year project in Rust.
During my university degree, I did a final year project on NP hard algorithms.
Specifically, I implemented a couple of algorithms to compute tree widths and tree decompositions for given graphs.
It was written in C++ and the performance was awful, the code was awful, and the tests/version control were non existent.

I recently found it on dropbox, I've tried previously to reimplement it in Java/Python/Go, but each attempt failed due to knowledge of how to implement the algorithms (I'd lost the code, and spent a lot of time trying to understand the papers again).
With the code found, and my new passion of Rust... I'm hoping this will be the one.

Introducing [fyp](https://github.com/alexanderjophus/fyp)
Please don't judge the C++ code, it's terrible and there's no tests.
I'm better than that now.

## What I've Learned

Rust exceeded all expectations. They were high, but they were exceeded.

The stereotype of Rustaceans spitting out the mantra "rewrite it in rust", may be true, but it's for good reason.
The language is incredibly expressive, and the community is incredibly helpful.
I firmly want to put more eggs in my Rust basket.

## What's Next

I'm going to continue with shllchckr and fyp.
You can expect a part 2 when that's done.

I'd also like to explore creating APIs more in Rust.
It's how I make my earning in Go, and would love to see how Rust compares.
If it replaces Go, I'd be happy, if it doesn't, I'd still be happy.

Thanks for listening!
