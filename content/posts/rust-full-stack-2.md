---
draft: true
date: 2024-04-15T17:20:57+01:00
title: "Creating a full stack web app in Rust"
tags: ['rust']
---

## Intro/Recap

Last post we talked about how to use Pavex and Diesel to create a simple web server.
We stopped at the point of being able to run it locally, in our specific use case we were able to search through a list of Digimon, and even get specific Digimon by name.

In this post, we'll be deploying the application through shuttle. 🚀

As a recap, here's where we are:
- ✅ Server - Pavex
- ✅ Database - Diesel
- 🚧 Server Hosting - Shuttle
- ❌ Frontend - Dioxus

## Shuttle

From the [Shuttle](https://shuttle.rs/) website:
> Build & ship a backend without writing any infrastructure files. Instead get your infrastructure definitions from your code function signatures and annotations.

I've used it for discord bots in the past and it's been helpful.
Provided the framework you're using is supported, it's a great way to deploy your application without having to worry about the infrastructure.

We'll assume you have a shuttle account, and understand the basics of shuttle at this point.
If you don't, check out their docs, play around with it, it's super fun!

## Shuttle <> Pavex

Remember when I said that shuttle needs to support the framework you're using?
Well, Pavex isn't supported by shuttle, so we'll have to do some work to get it to work.

The first thing to note is that shuttle does work in workspaces with multiple crates.
This means that we can have a workspace with our classic 3 Pavex crates, and a separate crate for the shuttle wrapper.

### Shuttle Wrapper

Typically, with Shuttle, you'd have an entry point that looks something like this:

```rust
#[shuttle_runtime::main]
async fn serenity(
    #[shuttle_runtime::Secrets] secrets: SecretStore,
) -> shuttle_serenity::ShuttleSerenity {
    // rust code here
}
```

where `shuttle_serenity` is the shuttle maintained crate for the serenity framework.
So, copilots first thought was to hallucinate a `shuttle_pavex` crate. Helpful.

This file [services/README.md](https://github.com/shuttle-hq/shuttle/blob/3e63caa290c4e1e9fc745fb073ed9c5b39f098a3/services/README.md), was a huge hint as to how crates like `shuttle_serenity` work.
All we need to do is implement `shuttle_runtime::Service` for a given `PavexService` struct.
Our `shuttle_pavex.rs` file will look something like this:

```rust
/// A wrapper type for [pavex::server::Server] so we can implement [shuttle_runtime::Service] for it.
pub struct PavexService(pub pavex::server::Server);

#[shuttle_runtime::async_trait]
impl shuttle_runtime::Service for PavexService {
    async fn bind(mut self, addr: SocketAddr) -> Result<(), Error> {
        let application_state = build_application_state().await;

        let server = self
            .0
            .bind(addr)
            .await
            .expect("Failed to bind the server TCP listener");

        tracing::info!("Starting to listen for incoming requests at {}", addr);

        run(server, application_state).await;

        Ok(())
    }
}
```

There's a lot to digest, so let's go slowly.

First, we define a struct `PavexService` that wraps a `pavex::server::Server`.
This is so we can implement the `shuttle_runtime::Service` trait for it.
The trait requires us to implement a `bind` function, which is where we'll set up our server.

For context, we're taking inspiration (or copying, if you will) from the `server` crate in Pavex.
This uses the `server_sdk` crates `build_application_state` and `run` functions to set up the server.
We're doing the same here, but with our own `PavexService` struct.

The `build_application_state` function is where we set up our application state - this will now be consistent between `cargo px run` and `cargo shuttle run`.
We then bind the address to the server.
Finally, we `run` the server with the application state.

Job's a good'un.

### Shuttle Pavex

Now, let's dive over to our `main.rs` file and see how we can use this.

```rust
use pavex::server::Server;
use shuttle_runtime::SecretStore;

mod shuttle_pavex;

#[shuttle_runtime::main]
async fn pavex(#[shuttle_runtime::Secrets] secrets: SecretStore) -> shuttle_pavex::ShuttlePavex {
    std::env::set_var(
        "APP_PROFILE",
        secrets
            .get("APP_PROFILE")
            .unwrap_or("development".to_string()),
    );

    let _ = dotenvy::dotenv();

    let server = Server::new();

    let shuttle_px = shuttle_pavex::PavexService(server);

    Ok(shuttle_px)
}
```

There's a bit to unpack here, so let's break it down line by line.

```rust
#[shuttle_runtime::main]
async fn pavex(#[shuttle_runtime::Secrets] secrets: SecretStore) -> shuttle_pavex::ShuttlePavex {
    // code here
}
```

The entry point to our application when running through shuttle is the function with the `#[shuttle_runtime::main]` attribute.
This function takes a `SecretStore` as an argument, which is where we can store secrets for our application.
We could pass in database URIs, API keys, etc. For our demo we're going to pass in the APP_PROFILE, which is either `development` or `production`.

Next up we set an environment variable based on the secret.
There's a bit of funkiness here, in the premade `server` crate from Pavex, it reads env variables for the `APP_PROFILE`.
That's why we set the env var here, so that the server knows what profile to run in.

After that we have "the rest of the fucking owl".

```rust
    let server = Server::new();

    let shuttle_px = shuttle_pavex::PavexService(server);

    Ok(shuttle_px)
```

This loosely follows what we have in the original pavex `server` crate (the entrypoint if you were to run `cargo px run`, or the dockerised version of the app).
My advice if you want to dive deeper is to look at the Pavex docs and find Luca's RustNation UK talk on Pavex.

## Part 2 Conclusion

Shuttle is _slick_.
It supports most Rust frameworks, it is a delight to work with, and the discord server is lively and helpful.

I'd knock points off for not supporting Pavex out of the box, but I think with Pavex's non-conventional design it's tricky to do, and it was very straight forward to actually implement.
So, no, no points deducted. 10/10.

You should not only be able to run `cargo px run` but also `cargo shuttle run` too.

You can now also take this further, and deploy to shuttle in CI.
But I'll leave that as an exercise to the reader.

Join me next time where I'll wrangle a front end onto this project (also written in Rust, because why not?).

Until next time, happy coding! 🚀