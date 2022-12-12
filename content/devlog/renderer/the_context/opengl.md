+++
title = "OpenGL"
description = "Which OpenGL version to target and how"
date = 2022-12-10
weight = 1
draft = false
template="book.html"

[taxonomies]
categories = ["Devlog"]
tags = ["GameDev", "OpenGL", "Context", "Renderer"]
[extra]
toc = true
keywords = "Game Development, GameDev, OpenGL, Graphics Programming"
+++

## Why OpenGL
OpenGL development has been mostly abandoned for Vulkan nowadays, so why would
you still want to target OpenGL? 

For starters, it is widely supported and the drivers are matured. It's also way
simpler than Vulkan, but also makes cross platform support way easier. It is
possible to target Desktops, Webbrowsers and mobile devices with almost the same
subset of OpenGL.

However, the goal of this context abstraction is to be graphics API agnostic
anyway, so it is always possible to add another one and will even be required
for console support at some point. However, I also want to write a
[wgpu](https://wgpu.rs/) backend in the near future, just to test the
"flexibility" of the context and compare the performance differences.
Adding Vulkan later is also not off the table either.

So in short, OpenGL is a great entry-level API which runs almost everywhere,
which makes it an ideal candidate for our first game.

## OpenGL Versions
Since we are going to use OpenGL, the first thing we should think about, is about the OpenGL version we want to
target. While it is possible to target a older version and spread `if's` in our
code base to pick the better alternative when supported, it adds an unnecessary
overhead and bloat. Instead, I want to target a specific version, poll some
extensions when needed(e.g. debug callback is a great extension in 3.3 to poll
for), and when needed, just implement a different backend altogether for
another version. So for example, have a dedicated 3.3 backend and a 4.6 one.

Which version we want to target depends first and foremost on the "OpenGL flavor"
we are aiming for. There are basically 3.

- **Legacy OpenGL**      
  Also known as "fixed function pipeline" or "immediate mode OpenGL", is a straight forward
  way to draw things, but comes with a worse performance. You will end up with
  far more API calls, need to send more Data around and is not nearly as flexible as the programmable pipeline.
  It's the OpenGL-style that uses `glBegin`, `glEnd`, `glMatrixMode`, etc, and
  was deprecated with the release of OpenGL 3.0         

<!--->

- **"Modern" OpenGL**   
  This is the "programmable pipeline", where you mostly deal with shaders,
  programs that run on the GPU in various stages of the process, and uses vertex
  array objects to store the format of vertex data. This is pretty much OpenGL
  3.0 - 4.4.

- **Bindless OpenGL**    
  Bindless is about "Direct State Access"(DSA). This changes the previous way of globally binding objects to the context to
  "named" access. Functions take in the object that they are going to manipulate
  instead if relying on global state, where the latter often lead to
  accidentally querying or changing the state of the wrong object. This leads
  not only to fewer errors, but also to more efficiency(no need to constantly
  bind things or keep track of the current bindings).
  This is pretty much OpenGL >= 4.5.

There is are also the core and the compatibility profiles, where the latter just enabled
support for the fixed function pipeline in the newer versions, though they don't
work with programmable pipeline ones(so it's not really possible to mix both
ways). OpenGL features are generally additive, so this profile split was a
way to separate the "new" OpenGL from the legacy luggage.

Which versions of these flavors we want to use depends on the functionality that
we need from the API and the available support on the target platform. 3.3 is a solid choice for a wide range of hardware and
driver maturity, but handy functions like compute shaders came with 4.3. Apple
devices also stopped support for OpenGL versions above 4.1.

Extensions can make muddle this choice further, because it is possible to create
an older context, but get the "newer" functionality via extensions.
So frankly speaking, it is kinda messy.

But which one to pick?
Legacy OpenGL is an easy no for our purposes, because it doesn't make use of any modern hardware. 
The question between "modern" and bindless OpenGL is more difficult.
Bindless makes things easier and more efficient, but "modern" supports a wider
range of hardware. Since the goal is support WebGL, non-bindless makes a lot of
sense. Also, arguably, we would just pick Vulkan if we need the state of the art.

OpenGL 4.3 Core seems to be the best bet for us. It has a great overlap with GLes3, which is used as base for WebGL2,
has compute shaders and debug callbacks. Historically, GLes3.0 was created from
a subset of OpenGL 3.3, but OpenGL 4.3 includes functionality to increase the
compatibility between the other two.

## Creating the context
Before we can do any fancy OpenGL calls, we need to create an OpenGL context.
We are going to use the [raw-gl-context](https://crates.io/crates/raw-gl-context) crate.
It works with `RawWindowHandle`'s, so it's pretty straight forward with `winit`.

```rust 
//create a context from the existing winit window
let context = GlContext::create(
    &window,
    raw_gl_context::GlConfig {
        alpha_bits: 0,
        version: (4, 3),
        profile: raw_gl_context::Profile::Core,
        ..Default::default()
    },
)
.unwrap();

//actually use the context
context.make_current();
```

Let's not forget to add it to our `Cargo.toml`.

```toml 
[dev-dependencies]
raw-gl-context = "0.*"
```

{% note(title="Note") %}
Note: On my machine, X11 requires an zero alpha bits in the context creation with Nvidia drivers. 
In theory, just using the default should be fine. It's a [known issue](https://github.com/glowcoil/raw-gl-context/issues/2) with some
workarounds, but since I don't need alpha bits for now, it is a problem for
 future me.
{% end %}

That's it.  

Creating an OpenGL-context using the Core profile, requesting the version 4.3,
then "activating" the context. Activating means making the OpenGL-context the
current context of the calling thread. A thread can only have one context being bound,
and a context can only be bound to a single thread at a time. This makes
multi-threading in OpenGL is a major pain, so we will use an alternative to
bypass this issue later.



Finally, in our event_loop, we will call 
```rust 
context.swap_buffers();
```
to swap the front with the back buffer. The details are not important, but to
put it simply, it "updates" the content of the screen. We are generally drawing
to the backbuffer, but the window displays the frontbuffer. So to actually see
what he have drawn, we swap them. We didn't draw anything yet, so the screen is
just black. `swap_buffers` waits for the issued calls to be finished before it
swaps. Keep in mind, this is just a minimalistic explanation and in reality, it
is not uncommon for having more than two buffers(e.g. triple-buffering) or
`swap_buffers` blocking until the monitor refresh before swapping(vsync).

## Generating the Bindings
Now that we got an OpenGL-context, we can load OpenGL functions.
The easiest way is to use any bindings generator, which loads the function
pointers we need for us.

We can use a web-service like [glad](https://gen.glad.sh/) to download the files
with the bindings, write a simple `build.rs` that uses
[gl_generator](https://crates.io/crates/gl_generator) or use any of the already
generated bindings like [gl](https://crates.io/crates/gl)(OpenGL 4.6 Core),
[gl33](https://crates.io/crates/gl33)(OpenGL 3.3).

We will go with `gl_generator` for no reason in particular.
The example code shows creating a `build.rs` in the project and uses the output
directory environment variable to include it into the modules, but since this
should pretty much never change, we can create a simple throwaway project to generate
the bindings just once, and copy them into our project.

```bash 
#in some empty dir 
cargo new --lib gl_bindings
```

add `gl_generator` as build dependency:

{% code(name="gl_bindings/Cargo.toml") %}
```toml 
[build-dependencies]
gl_generator = "*"
```
{% end %}

Now we create a `build.rs` in the root of the project:

{% code(name="gl_bindings/build.rs") %}
```rust 
use std::fs::File;
use gl_generator::{Api, Fallbacks, GlobalGenerator, Profile, Registry};

fn main() {
    let mut file = File::create("gl43_core.rs").unwrap();

    Registry::new(Api::Gl, (4, 3), Profile::Core, Fallbacks::All, [])
        .write_bindings(GlobalGenerator, &mut file)
        .unwrap();
}
```
{% end %}

Building the project with `cargo build` should now create a file with the name `gl43_core.rs` in
the project root.

`gl_generator` also allows us to adjust the generated bindings. The `GlobalGenerator`
allows us to access OpenGL functions from anywhere in the project(that has
access to the gl43_core module), without passing some context around. This makes
thing far simpler for us. We are not going to expose the gl lib to the public
and we will use a different way to tie the lifetime to the actual gl context.

We just copy the generated file into the src dir of our context member.(don't forget to delete the throwaway `gl_bindings` project afterwards, it's just clutter on our hard drive now)

To access the bindings from our playground, we need to modify the existing
`lib.rs` on our src directory. While we are at, we remove the existing content,
and for the sake of our sanity, we will just mute clippy for the gl module,
because we will get 2k warnings otherwise. We do that by adding some `allow`
attributes right before we declare the module, or to avoid noise in our source
files, we add the "module wide" attributes at the top of the gl43_core.rs, which
can take a while depending on how great our text editor deals with huge files.
Also, while we are dealing with lints, let's add the stricter `pedantic` and the
experimental `nursery` lints to our crate, just because Clippy is **that** useful. 

{% code(name="cac_graphics/context/src/lib.rs") %}
```rust 
#![warn(clippy::pedantic)]
#![warn(clippy::nursery)]

pub mod gl43_core;
```
{% end %}

{% code(name="cac_graphics/context/src/gl43_core.rs") %}
```rust 
#![allow(clippy::pedantic)]
#![allow(clippy::nursery)]
#![allow(clippy::all)]
```
{% end %}

Fun fact, `allow(clippy::all)` doesn't disable all lints. Pedantic and nursery
lints, both of which we added to our `lib.rs`, must be disabled additionally.
Alternatively, we could manually enable the warnings for the lints we care
about, like [how EmbarkStudios does
it](https://github.com/EmbarkStudios/rust-ecosystem/blob/main/lints.rs).



Now we need modify our `examples/playground.rs`, to alias the gl module with a
more ergonomic name.

```rust
//cac_graphics/context/examples/playground.rs

use context::gl43_core as gl;
```

Finally, we can load the OpenGL functions right after we made our context
current: 

```rust
context.make_current();

gl::load_with(|symbol| context.get_proc_address(symbol));
```

The bindings generated with `gl_generator` expose a load_with function that will load the function pointers
of our OpenGL functions one by one. 

Now, we can test it by calling the following 2 OpenGL function in our example.

Before the event_loop.run(..) 
```rust 
//set a clear color for later
unsafe {
    gl::ClearColor(1.0, 0.0, 0.0, 1.0);
}
```

and before swap_buffers()
```rust 
// clear the "screen" with the previously set clear color
unsafe {
    gl::Clear(gl::COLOR_BUFFER_BIT);
}
```

Because they are just function pointers, calling functions over FFI(basically calling non-Rust
functions), they need to be in unsafe blocks.
Unsafe sounds dangerous and scary, but it just means that Rust can't guarantee that these
functions follow its safety principles, so the burden is on the programmer. 


The final code should look like this:

{% code(name="cac_graphics/context/examples/playground.rs") %}
```rust 
//use raw_gl_context::GlContext;
use winit::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
};
fn main() {
    let event_loop = EventLoop::new();
    let window = WindowBuilder::new().build(&event_loop).unwrap();

    //create a context from the existing winit window
    let context = GlContext::create(
        &window,
        raw_gl_context::GlConfig {
            alpha_bits: 0,
            version: (4, 3),
            profile: raw_gl_context::Profile::Core,
            ..Default::default()
        },
    )
    .unwrap();

    //actually use the context
    context.make_current();

    //load the OpenGL functions with the context
    gl::load_with(|symbol| context.get_proc_address(symbol));

    unsafe {
        //red and alpha channels as 1.0, rest as 0.0
        gl::ClearColor(1.0, 0.0, 0.0, 1.0);
    }

    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Poll;

        match event {
            Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                window_id,
            } if window_id == window.id() => *control_flow = ControlFlow::Exit,
            Event::MainEventsCleared => {
                unsafe {
                    // clear the "screen"
                    gl::Clear(gl::COLOR_BUFFER_BIT);
                }
            // "update" the screen
                context.swap_buffers();
            }
            _ => {}
        }
    });
}
```
{% end %}

If everything works, there should now be a window with a red background when
running the example. And we can dive into the next steps.

Understanding the gl calls above is not important for now, the main thing is to
just test whether OpenGL works. If it fails, there might be some system
dependencies missing, but simply said `glClear(GL_COLOR_BUFFER_BIT)` clears the "screen", in a color that
we specify with `glClearColor`.

{% note(title="Note") %}
I will use the OpenGL notation when referring to the functions in text,
instead of the Rust notation with the module as namespace. The reason for that
is that it makes finding the docs easier, but also less dependent on the
OpenGL bindings solution(and imports). E.g. glClear(GL_COLOR_BUFFER_BIT) instead of 
gl::Clear(gl::COLOR_BUFFER_BIT)
{% end %}

Now we can get our hands dirty.
