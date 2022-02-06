---
title: "DEV TOOLING I USE A LOT"
date: 2022-02-06T11:22:03Z
draft: false
---

## Intro

I borrow a lot from everyone I work with, so all credit goes out to everyone I've ever worked with.
I'm going to create a few separate sections, such as kubernetes, REST, gRPC, etc.
The list is by no means complete; I'm always on the lookout for new tools to play with so if I missed something, let me know and I'll happily update this list.
It's probably important to date this post, so read the date and with that we'll get on.

## Helpful Tips

If you find yourself using a tool over and over (especially one that's not linked), look for an `awesome-$project` github page.
You should find a few examples linked in this page.

Speak to your colleagues, they'll know some tools you don't.
Attend meetups, watch talks, speak to other professionals - again - they'll know some tools you don't.

## Essentials

These are things I'd recommend over anything else in this post.

- [dotfiles](https://github.com/webpro/awesome-dotfiles). The basic premise of dotfiles is to get your machine to your liking from a fresh install as quickly as possible.
My advice here is to build it over time, everytime you find a new tool you like, add it to your dotfiles (I'm terrible at this).
Explore the linked repo, it has a few examples as a starting off point, fork one you like and modify to your hearts content.
- [fish](https://fishshell.com/) or [zsh](https://ohmyz.sh/), choose a shell that suites you.
I made the jump to fish and if I'm honest I'm not looking back, there are just too many good things you can do with it.
Experimenting with shells costs almost nothing, you can install them and use them at will.
- [starship](https://starship.rs/). So the goal here is to make your terminal prompt work for you, be it starship, or [spaceship](https://github.com/spaceship-prompt/spaceship-prompt), or something else entirely.
Personally I'm a colour by numbers kind of dev, so I like as much colour as possible. Starship provides that.
Whether you're using a common language like [python](https://starship.rs/config/#python) or [Go](https://starship.rs/config/#go), or you're more concerned about things like [terraform](https://starship.rs/config/#terraform), or [docker](https://starship.rs/config/#docker-context), there's something for everyone.
By default starship (and spaceship) get a lot of things right, but you can customize just about everything.

## Other Essentials

This is mostly a set of coreutils replacements, and other terminal commands I consider essential.

- [lsd](https://github.com/Peltoche/lsd), have you ever though `ls` is cool, but where are the colours, the icons, the other missing features?
lsdeluxe remains compatible with `ls`, is frightningly fast, and like most things on this list configurable to your tastes.
- [fd](https://github.com/sharkdp/fd) is `find`s replacement, did I mention colours yet? This tool has them.
While it's not 100% compatible with `find`, it is pretty close and the places it's not compatible it's to make it more user friendly.
Did you ever want to find the all the Go files in your project and sort them by line count? `fd -e go -E vendor --exec wc -l | sort` took 26 milliseconds on my machine for a medium size project.
- [ripgrep](https://github.com/BurntSushi/ripgrep) (or rg as it's known), boy oh boy. This one is probably my favourite.
Again, colours are great, if I'm looking for a specific pattern in my code, it's helpful to know the file, line number, and the matching text - these are all coloured.
Like many of these tools it's also written in rust, so you can be assured it's fast.
- [bat](https://github.com/sharkdp/bat), a self described cat replacement with wings.
Can you guess why I like it? Syntax highlighting, git integration, it's fast, oh and colours.
- [fx](https://github.com/antonmedv/fx)/[jq](https://github.com/stedolan/jq), you ever wanted to parse or manipulate json?
Here's something for you!
`jq` I often use to manipulate json files to check if certain fields are populated to see how I like.
`fx` on the otherhand I'm just getting started with, but I forsee myself using it a lot with tailing k8s logs.

## Git

The goal here is to make your gitflow work for you and your teams.
If that's through git commands and them alone, then great, if you prefer a UI, also great.
Personally I'd opt for as close to the git command line tool as possible, layers on top of it will come and go, but the basics remain the same.
As always check out [awesome-git](https://github.com/dictcp/awesome-git)

- [git aliases](https://git-scm.com/book/en/v2/Git-Basics-Git-Aliases). These allow you to create custom workflows super easily. Here are a couple of my more unique aliases that may help you.
  - `up = !git fetch origin && git rebase origin/main`. This fetches the latest changes from origin, and rebases your branch onto main (you can change main for master to suit your usecase), I also created an `updev` for development branches
  - `clean-branches = !git branch --merged | grep -v main | xargs git branch -d`. This command will remove all local branches that have been merged.
- [git tower](https://www.git-tower.com) is available for mac and windows (I assume there's a linux version people would recommend).
I'm a terminal purist, but it seems to have everything you need and is highly recommended by a buddy of mine.

## Kubernetes

If you're using kubernetes in any capacity, I hope these tools come in handy.
If you have additional tools you'd recommend, I'd love to hear them.
Also, check out this [aweomse-k8s](https://ramitsurana.gitbook.io/awesome-kubernetes/docs) book for more resources!

- [k9s](https://github.com/derailed/k9s) is my goto for anything kubernetes, it allows you to see anything on the cluster and wraps pretty much every kubectl commands.
As always, it's super colourful, and allows you to easily grok what's going on in your cluster.
  - Want to port-forward to specific pods easily, sure.
  - What about benchmarking your applications, k9s integrates `hey` which we'll talk about later.
  - How about sanitizing my cluster, k9s with popeye has got your back.
- [kubectx](https://github.com/ahmetb/kubectx) allows you to switch between kubernetes namespaces and clusters with ease.
- [krew](https://github.com/kubernetes-sigs/krew/) allows you to install kubectl plugins with ease.

## gRPC

If you're working with gRPC at all.
There's an [awesome-gRPC](https://github.com/grpc-ecosystem/awesome-grpc) list too!

- [evans](https://github.com/ktr0731/evans), is a CLI tool to inspect, describe, and call gRPC services.
It can be used in scripts, or used as an exploratory tool to test behaviour of your services.
Oh, it's also very colourful and pretty.
- [ghz](https://github.com/bojand/ghz), ever wanted to see how your service handles under load? ghz has got you covered.
It's all kinds of configurable, such as varying output formats (ever wanted load tests reports as grafana graphs?), custom data through templating etc.
- [bloomRPC](https://github.com/bloomrpc/bloomrpc), if you're more GUI inclined, this ones for you!
bloomRPC allows you to explore a given service just like postman but for gRPC.

## REST

Case study: you're writing RESTful APIs and want to test them, we have you covered.
There's a list for this too: [awesome-rest](https://github.com/marmelab/awesome-rest)

- [postman](https://www.postman.com/) is a tool to help test APIs.
It provides all kinds of benefits; manually testing APIs, exporting tests into version control, a CLI tool to run a suite of tests in CI.
- [hey](https://github.com/rakyll/hey), have you ever wanted to test how your API works under load through a sweet CLI tool?
The output is almost identical as ghz (discussed earlier).

## Bonus list for making it to the end

- [sl](https://github.com/mtoyoda/sl) - A steam locomotive for fixing bad habits.

## Conclusion

This turned out more buzzfeedy than intended, but I'm ok with that.
I think the last serious callout is the parent [awesome](https://github.com/sindresorhus/awesome) list, 
In conclusion I'm sorry for all the open tabs you have now.