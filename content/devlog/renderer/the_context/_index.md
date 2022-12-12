+++
title = "The Context"
paginate_by = 1
sort_by = "weight"
weight = 2
transparent = true
insert_anchor_links="heading"
template = "book.html"
[extra]
toc = true
+++

Now it's time to add the first member, the graphics context, by calling 

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

The context could be seen as "hardware abstraction layer", RenderDevice,
GfxDevice or similar name. It is just used as a wrapper around the underlying
graphics API's and allows us using OpenGL, Vulkan and whatever else under an unified
API. It's not going to do any fancy ordering, render passes, materials or other high level
things, those will be part of the core graphics crate. 

The main purposes are: 
- storing and reading blobs of data on the GPU(buffers of various kinds)
- specify the vertex layout(mapping attributes with buffers)
- compile shaders and set shader specific variables(uniforms)
- draw a range of vertices(start..end), using the vertex layout and a specific shader, with additional per-instance properties

Basically everything that OpenGL and friends provide, just in a safe wrapper.
This might sound awfully similar to the problem that [wgpu](https://github.com/gfx-rs/wgpu) solves.
The reason why I decided to go with this abstraction is for first-class
OpenGL(and WebGL) support, but also the ability support special platforms(e.g. game consoles) without creating a fork,
but also to give me the ability to make some opinionated decisions. I also have more fun writing and understanding the low-level API's :).
That said, I also plan to add wgpu as possible backend.

I am already somewhat comfortable with OpenGL, so it will be the main focus for
now. Though, I won't go into all the details of it. [learnopengl.com](https://learnopengl.com/) does a far better
job than I ever could.

The context itself doesn't care about windows. It should work with [winit](https://crates.io/crates/winit),
[glfw](https://crates.io/crates/glfw) and anything else that supports either
[RawWindowHandle](https://crates.io/raw-window-handle) or offers a proc_address function pointer  
lookup(e.g. `impl FnMut(&'static str) -> *const std::ffi::c_void`).

## Setting Up The Playground
I found that Rust highly encourages an exploratory style of programming, where you reach for quick and dirty solutions, to
understand your problems and your data, before you discard that junk to do it
properly again.

So I often start with a basic "playground.rs" in the examples directory that is not checked into the
version control system, where I just "sketch" different solutions until I am
content. This also helps figuring out an ergonomic API.

For this, we need some dev-dependencies, which we add to our `context/Cargo.toml`.

[winit](https://crates.io/crates/winit) to create a simple window, [env_logger](https://crates.io/crates/env_logger) to have some basic logging support later and [anyhow](https://crates.io/crates/anyhow) for error bubbling.
If you have any other preferences, feel free to use those instead. None of them
will be directly tied to our library later and the user is free to pick whatever
they prefer.

While we are at it, let's also add [log](https://crates.io/crates/log) as a dependency for our crate, because we will definitely end up using later. It's like part of the "extended standard library". Crates that are the de-facto standard. Log just allows us to log, without knowing about any specific logging implementation (like the aforementioned env_logger).

I don't need winit's Wayland support on my dev machine for testing, so I disabled it and
only enabled X11. This will reduce the total dependencies from 130 to "just" 48,
cold build times from 54.71s to 18.74s, and more importantly incremental build
times from from 2.16s to just 0.82s for me. Looking at the features the crates provide and only enabling the
one you actually need, makes a huge difference :)

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
#logging
env_logger = "0.*"
#window creation
winit = { version = "0.*", default-features = false, features =["x11"] }  
```
{% end %}

Now, I yoink winit's example code from their readme, put it into
`cac_graphics/context/examples/playground.rs` and run the example.

```bash
cargo run --example playground
```

It opens and empty window without displaying anything, which makes sense.

The next step would be to create an actual graphics context to load and call some
graphics functions.

But let us talk about our target graphics API first. 

