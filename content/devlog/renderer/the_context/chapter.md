+++
title = "The Context"
description = "Adding the first package to our workspace, and creating a little playground for us."
date = 2022-12-01
draft = false
weight = 0
render = false

[taxonomies]
categories = ["Devlog"]
tags = ["Context", "Renderer"]

[extra]
chapter = true
toc = true
keywords = "Playground, Context"
+++

Now that we got our workspace set up, it is time to add the first member, the graphics context. We add a new library package inside our workspace by calling: 

```bash 
cargo new --lib context
```

and adding it into the members field of the `Cargo.toml` in the root.

{% code(name="cac_graphics/Cargo.toml") %}
```toml
[workspace]
members = ["context"]
```
{% end %}

{% note(title="Note") %}
It is also possible to just have a single directory called "crates" and glob
all members automatically with `members = ["crates/*"]`, but I just prefer adding them
explicitly.
{% end %}

The context could be seen as "hardware abstraction layer", "GfxDevice" or similar thing. It is just used as a wrapper around the underlying
graphics API's and allows us using OpenGL, Vulkan and whatever else under a unified API to communicate with the graphics hardware. It's not going to do any fancy ordering, render passes, materials, or other high level
things, those will be part of the core renderer crate. 

The main purposes of this Context are: 
- storing and reading blobs of data on the GPU)
- specify the vertex layout
- compile shaders and set shader specific variables
- draw the vertex layout with a specific shader

Basically everything that OpenGL and friends provide, just in a safe wrapper.

This might sound awfully similar to the problem that [wgpu](https://github.com/gfx-rs/wgpu) solves, and since we are using Rust, we might as well use it. 

However, the reason I decided to go with our own abstraction is the ability to make our own opinionated decisions. I also have more fun writing and understanding the low-level API's :).

That said, I also plan to add wgpu as a possible backend as part of this context
abstraction at some point.

I am already somewhat comfortable with OpenGL, so it will be the main focus for
now. Though, I won't go into all the details of it, but try to give some summaries where necessary. [learnopengl.com](https://learnopengl.com/) does a far better
job than I ever could when it comes to learning materials for OpenGL. 

The attentive reader might have noticed that I didn't say anything about
windows. The reason is that our Context should work with
[winit](https://crates.io/crates/winit), [glfw](https://crates.io/crates/glfw)
and anything else that supports a way to create a graphics context. This gives
us a lot of flexibility later, especially when targeting different platforms. We
can have separate binaries with their own platform specific requirements(some
need to be non-blocking, for example), without making a potentially unergonomic sacrifice in our Renderer. We will basically never take ownership of the main loop.

## Setting Up The Playground
I found that Rust highly encourages an exploratory style of programming, where you reach for quick and dirty solutions, to
understand your problems and your data, before you discard that junk to do it
properly again. Otherwise, we will end up frequently [rewriting it in Rust later](/blog/my-year-with-rust#rewrite-it-in-rust), rather than sooner.

So I often start with a basic "playground.rs" in the examples directory that is not checked into the
version control system, where I just "sketch" different solutions until I am
content. This also helps with figuring out an ergonomic API.

For this, we need some dev-dependencies, which we add to our `context/Cargo.toml`.

[winit](https://crates.io/crates/winit) to create a simple window, [env_logger](https://crates.io/crates/env_logger) to have some basic logging support later and [anyhow](https://crates.io/crates/anyhow) for error bubbling.
If you have any other preferences, feel free to use those instead. None of them
will be directly tied to our library later, and the user is free to pick whatever
they prefer. It's just what we will be using for this playground, our examples,
and tests.

While we are at it, let's also add [log](https://crates.io/crates/log) as a dependency for our crate, because we will definitely end up using it later. It's like part of the "extended standard library", crates that are the de facto standard. Log just allows us to log, without knowing about any specific logging implementation (like the aforementioned env_logger).

I don't need winit's Wayland support on my dev machine for testing, so I disabled it and
only enable X11. This will reduce the total dependencies from 130 to "just" 48,
cold build times from 54.71s to 18.74s, and more importantly incremental build
times from 2.16s to just 0.82s for me. Looking at the features the crates provide and only enabling the
one you actually require, makes a huge difference :) 
The drawback is that we need to pass it as command line argument when we are
trying to run our tests on Wayland later.

The final Cargo.toml like this for now(we will modify the package stuff later):

{% code(name="cac_graphics/context/Cargo.toml") %}
```toml
[package]
name = "context"
version = "0.1.0"
edition = "2021"

[dependencies]
#logging abstraction
log = "0.*"

[dev-dependencies]
#error handling
anyhow = "1.*"
#logging implementation
env_logger = "0.*"
#window creation
winit = { version = "0.*", default-features = false, features =["x11"] }  
```
{% end %}

Now, we yoink winit's example code from their readme, put it into
`cac_graphics/context/examples/playground.rs` and run the example.

```bash
cargo run --example playground
```

It opens an empty window without displaying anything, which makes sense.

{% note(title="Note", kind="warn") %}
It's possible that you won't see anything, because on some window
managers the window only appears when its content is actually updated/drawn to.
{% end %}


The next step would be to create an actual graphics context to load and call some
graphics functions.

But let us talk about our target graphics API first.  

---

[Link to the repository](https://github.com/KevinThielen/cac_graphics/tree/1f206abc09f326569f3fcf21feddde8fca70c8d0)  
*(We only added the playground.rs to the repository to make sharing easier)*
