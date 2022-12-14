+++
title = "RenderTargets"
description = "We are going to replace the calls that update our render target. Also colors."
date = 2022-12-10
weight = 4
draft = false
template="book.html"

[taxonomies]
categories = ["Devlog"]
tags = ["OpenGL", "Context", "Renderer"]
[extra]
toc = true
keywords = "RenderTarget"
+++

We can finally start moving the raw GL calls into proper abstractions, being ideally backend
agnostic. 

Before we start with the actual GL calls, we can put our current `update()`
function into an abstraction first. We are certain that all backends will use
some sort of update function. So we might as well start with that.

Right now, our update function, which just swaps buffers on our context for now,
is tied to our `opengl::Context`. Let's create a trait that
will be shared by all the different implementations.

As I mentioned before, my naming skills are as great as my art skills - just
dreadful. So it's no surprise that this shared trait, that is the namesake for our
crate, will just be boringly called `Context`.

Because it is such an integral part of this crate, we might as well write it
into our `lib.rs`, since there is very little reason in creating a dedicated 
module, except for being a laughingstock for adding yet another thing called context into this madness.
Hey, at least we can just use RustAnalyzer's code actions to rename them
automatically, once we hit a creative spark.


{% code(name="cac_graphics/context/src/lib.rs") %}
```rust 
//cac_graphics/context/src/lib.rs
pub trait Context {
    fn update(&mut self);
}
```
{% end %}

*(Geez, at least call this ContextTrait or something..)*

Now we go to our `opengl::Context`

And implement this trait by extracting the already existing update function into
it.

```rust 
impl<C: GLContext> crate::Context for Context<C> {
    fn update(&mut self) {
        self.gl_context.swap_buffers();
    }
}
```

Don't forget to import it in our `playground.rs`
```rust 
use context::{gl43_core as gl, opengl, Context as _};
```

The reason we use the `as _` is that we don't need to refer to the trait via
its name, but since it is a trait, we need to bring it into the scope to use it.
The advantage is that we avoid potential name clashes, like the already existing 
`use glfw::Context`. Naming things is hard.

And that's it, everything should still work.

## RenderTarget 

The next functions on the agenda are `glClear` and `glClearColor`.
On the first glance, they look straight forward, we just add the functions to
the trait in our `lib.rs` and pass the required arguments to them. After giving it a bit of
thought, we come to the conclusion that things are not quite that simple.

We don't want to tie our context to a specific API, the whole point of this abstraction is to be
API-agnostic. We want to leave the global state machine of OpenGL!
Not just that, but what if we don't want to clear the screen, but another
target, like a texture we are rendering to? Or not even the whole screen, but
only a small part of it? 

So our first abstraction should be some sort of `RenderTarget`. Our canvas if
you will. We will pass it to the draw call later, together with the shader and
VAO and what not. The caller should not care *how* the context does it, all that
counts is the result; drawing to whatever is passed as target.
We also don't want to have to constantly clear the screen on every single draw call either, but rather let the user decide when to and how to clear.


So the best way to deal with that, would be a dedicated struct that represents
our render target. We get rid of the global scope, the user can copy and store
it, and use it to clear whenever they want.

Let's just add a `render_target.rs` to our source files, with some naive
definition.

{% code(name="cac_graphics/context/src/render_target.rs") %}
```rust
pub struct RenderTarget {
    pub clear_color: Option<(f32, f32, f32, f32)>, 
}

impl RenderTarget {
    pub fn clear(&self) {
        //TODO: clear
    }
}
```
{% end %}

Don't forget to add it to our `lib.rs`.

```rust
pub mod render_target;
```

{% note() %}
Instead of always creating the file and then adding the module to the `lib.rs`, I usually just
add the module first and use the amazing RustAnalyzer to automatically create the file.
{% end %}

However, how are we actually going to clear?
In fact, how can we make the connection to the actual context?
Sounds like a job for some generics!

Actually, I used the current playground to sketch around with different
solutions. From having actual associated types to get the backend-specific
types or wrappers, to plain enums or shoving every single function into a single context
trait, or a dozen different traits on the context. None of that felt really
good, some even were tedious. Ultimately, I ended up with "passing around the
context" as the most ergonomic solution, so let's take a look at that. Chances are that we might need to refactor, but
let's first give it a try.

First we need some sort of trait bound, so we can call the actual "context
specific functions". Our root context trait might be the first
to consider, but using the power of foresight again, we know that we would be
working towards some mega-trait, if we shove everything into that one.
It's not just the RenderTarget, but probably a dozen functions for every single GL object, all kinds of draw
functions, etc. This is fine in theory, but incredibly unwieldy when we actually
implement it.

Another issue is the naming of the functions. For example, a simple `create()` could
apply to every single GL objects, and we also probably want to have different create
functions with different parameters, so we might end up with humorously long
names, like `create_buffer_with_f32()`, `create_vertex_buffer_from_slice`, etc.
There is no easy way to group the functions belonging to that buffer, unless we
look at the parameters and/or return value, or we use some weird naming
conventions, like starting all functions with a prefix, like
`buffer_create_vertex_buffer_from_slice`.  

On the other hand, having too many smaller traits is annoying when
it comes to actually using the functions, because traits need to be in scope.
Having a single Context-trait is fine, but always importing a dozen is a drag.
The issue is even enhanced when there is a naming clash. 

We are trying to find a middle ground, by using smaller traits, but not force
them to be exposed to the user. In fact, our abstractions will use trait bounds
only on the small subset of traits they need.

Let's add a little trait to our `render_target.rs`: 
```rust
//It's a different Context, I swear!
pub trait Context {
    fn clear(&mut self, target: &RenderTarget);
}
```

And make the Context in the `lib.rs` require implementing that Context.

```rust 
pub trait Context: render_target::Context {
    fn update(&mut self);
}
```

Now we can adjust our `clear` function in the `RenderTarget` impl: 

```rust 
impl RenderTarget {
    pub fn clear<C: Context>(&self, ctx: &mut C) {
        ctx.clear(self);
    }
}
```

This might look confusing and tedious to write.
However, actual implementation per backend will be done just once at this point.

The alternative would be to use some sort of PhantomData, to make the entire struct contain the type information of the
Context-implementation, but then we would "infect" all code trying to store or reference that type.


For example, our future game wants to store a RenderTarget in its struct of some sub-menu, like for some sort of a minimap.
Just a little part of the main window.

We would need to make the sub-menu, the main menu owning that sub-menu, the game class owning that main-menu 
and so on, all be generic over that Context we created at the start of the main.  
I mean, sure, there are far worse things to do, but there is no reason to
not contain this virality to just the code base that needs to know.
And this ignores potential technical roadblocks, like being able to switch the
backends during runtime rather than compile time. 

This function just passes the responsibility to the Context implementation.
So all we need to do is to call `my_target.clear(&mut ctx)` later.
The caller doesn't need to know how the `clear` works. The struct might contain
a handle to a native GL resource, just call an associated function with some
arguments or whatever else. 

{% note(title="Why not put the swap_buffers() into the RenderTarget as well?", kind="question") %}
Since our RenderTarget is an abstraction for our "canvas", there might be an
argument to add the swap_buffers() to "update" the screen to the
render_target::Context. However, this is an implementation detail, depending on the graphics backend. Some
require us to do it, others don't. There is very little benefit in exposing this
boring implementation detail to the user, especially because it comes with a lot
of hidden implications, like stalling the thread to wait for the glFinish(). For
now, it works fine in the update function since it's a global operation.
{% end %}

Now we only need to implement this Context for our OpenGL-Context. We put it
into our opengl module for now, but we will definitely put this into a dedicated
file once we add more functions to it.

{% code(name="cac_graphics/context/src/opengl.rs") %}
```rust
impl<C: GLContext> render_target::Context for Context<C> {
    fn clear(&mut self, target: &render_target::RenderTarget) {
        if let Some(color) = target.clear_color {
            unsafe {
                gl::ClearColor(color.0, color.1, color.2, color.3);
                gl::Clear(gl::COLOR_BUFFER_BIT);
            }
        }
    }
}
```
{% end %}

This implementation does two things. It sets the clear color AND does the
clearing of the screen. There is very little reason in doing it the OpenGL way
with clear flags, when we can just check whether there is a color to clear or
not, via some sort of Option. Once we have other clear flags, like for the depth
values, we can use `Option` for them as well. I don't think there will ever be a
case where we want a clear color, but not clear the target with it.

This might cause some performance penalty, given that we
are always adding another GL-call when we clear, because we always set the color, even if it didn't change.
On the plus side we also have the opportunity to just cache the current global clear color and compare it with the
new one or use some sort of dirty flag on our `RenderTarget`, to make this
conditional, if it ever becomes an actual performance issue.
In reality, this performance impact is negligible. The important thing is we got a lot of flexibility with the implementation that is hidden away from the caller. As mentioned before, this abstraction should work
with textures and whatnot later, so we can just add a handle to the
`RenderTarget` struct to enable that when we are at that point.

All that is now left is the actual construction of a RenderTarget.
Since the RenderTarget is not something that lives on the GPU, at least when it
comes to the "Screen", we can just have basic constructors. 

```rust 
impl RenderTarget {
    pub fn new() -> Self {
        Self { clear_color: None }
    }

    pub fn with_clear_color(clear_color: (f32, f32, f32, f32)) -> Self {
        Self {
            clear_color: Some(clear_color),
        }
    }
}
```
No need for generics or trait bounds, at least for the simple case where we just
want to target the screen. The constructors seem a bit overkill given that we
just have a single field, but it will be way more useful once we add more to this
struct.

Back to our playground, we remove the raw `glClearColor()` and `glClear` calls
with our new RenderTarget. We also rename our context object, because we are
going to pass it around a lot. So we will just shorten its name to `ctx`. 


```rust
//when we create the context
let mut ctx = opengl::Context::new(GLFWContext(window))?;
let screen = RenderTarget::with_clear_color((0.0, 1.0, 0.0, 1.0));

//right before our DrawArrays inside the loop
screen.clear(&mut ctx);
```

Now, our background should be green. 
However, look at this hideous tuple for the color and its pretentious double parentheses.

At this point we might add some proper Colors struct.

## Colors
It's only a tiny detour, I swear, we will go into the color space nonsense later,
but for now, having some simple abstractions for the colors instead of using
tuples would be nice.

Since Colors seems to be quite a commonly shared type, we put it into a new
package into our workspace, the one that should contain all sorts of utility and
common behavior, the `core` package.

So inside our workspace root we create a new library
```bash 
#cac_graphics/
cargo new --lib core
```
Then we add it to our workspace, next to our context:

{% code(name="cac_graphics/Cargo.toml") %}
```toml 
[workspace]
members = ["context", "core"]
```
{% end %}

And finally, add it as dependency to our context: 

{% code(name="cac_graphics/context/Cargo.toml") %}
```toml
[dependencies]
# ...
#core utility, data structs, etc.
core = { path = "../core"  }
```
{% end %}

Now we remove the junk in the `lib.rs`, add our trusty clippy lints and import the color module.

While we are here, we will also re-export the upcoming Color
struct, because it is such a fundamental and commonly used type later, that it
might live in the root of our utility.

One thing first, there are different ways to represent colors.
OpenGL requires floating points in the range of 0.0 - 1.0, but almost everyone will run into the
u8 based colors in the 0-255 range at some point, so we might as well be clear now and give us room to add other types later.
In fact, it is almost guaranteed we will add them at some point(spoiler alert, when we are writing the automated tests).
So we call it color32, to signal we store 32 bits per channel.

So, what this all means, is:

{% code(name="cac_graphics/core/src/lib.rs") %}
```rust 
#![warn(clippy::nursery)]
#![warn(clippy::perf)]
#![warn(clippy::pedantic)]

pub mod color32;
pub use color32::Color32;
```
{% end %}


{% note(title="Why not just a Color enum?", kind="question") %}
True, differentiating the colors by their bits is a great use for enums.
However, enums always assume the size of the biggest variant, which means our
future Color8 will take as much size as the Color32 variant, making it 4x as big
as it needs to be. This is especially painful, because the 8 bit variant is
commonly used when dealing with textures or pixel data, so the scale actually
matters, since we are in the tens of thousands if not more.
{% end %}


Now, let's add a simple struct to the color module: 

```rust 
#[derive(Debug, PartialEq, Copy, Clone)]
#[repr(C)]
pub struct Color32 {
    r: f32,
    g: f32,
    b: f32,
    a: f32,
}
```
We just added some overall useful derives and made sure that we can send it over
FFI(rept(C)), since we plan on using it with OpenGL later.

Now a few functions to actually use and create the colors, following Clippy's
advice while we are at it, to make them `const` and `[must_use]`. Because they are unlikely to change, there is no need to postpone this.

```rust 
impl Color32 {
    #[must_use]
    pub const fn from_rgb(r: f32, g: f32, b: f32) -> Self {
        Self { r, g, b, a: 1.0 }
    }

    #[must_use]
    pub const fn from_rgba(r: f32, g: f32, b: f32, a: f32) -> Self {
        Self { r, g, b, a }
    }

    #[must_use]
    pub const fn as_rgb(&self) -> (f32, f32, f32) {
        (self.r, self.g, self.b)
    }

    #[must_use]
    pub const fn as_rgba(&self) -> (f32, f32, f32, f32) {
        (self.r, self.g, self.b, self.a)
    }
}
```

The "weird" naming for the constructors and hiding the channels is intentional
and will make more sense later. The short story is that colors are actually
in color spaces, and we want to follow Rust's philosophy of being explicit where it
matters. There should be no question about the used space, which is why they
will use different sets of "getters" and "setters" and just convert on demand.

It would be nice to have some constants for our colors, to avoid 
fiddling with the channels and be aesthetically more pleasing than bright red or
green. So we just add some at the top of our impl: 

```rust 
impl Color32 {
    /// Almost black with a touch of green
    pub const DARK_JUNGLE_GREEN: Self = Self::from_rgb(0.102, 0.141, 0.129);
    /// Grape like purple
    pub const PERSIAN_INDIGO: Self = Self::from_rgb(0.20, 0.0, 0.30);
    /// Dirty White
    pub const GAINSBORO: Self = Self::from_rgb(0.79, 0.92, 0.87);
    /// It's really nice to look at
    pub const UNITY_YELLOW: Self = Self::from_rgb(1.0, 0.92, 0.016);

    /// The Color Black
    pub const BLACK: Self = Self::from_rgb(0.0, 0.0, 0.0);
    /// The Color Red
    pub const RED: Self = Self::from_rgb(1.0, 0.0, 0.0);
    /// The Color Blue
    pub const BLUE: Self = Self::from_rgb(0.0, 0.0, 1.0);
    /// The Color Green
    pub const GREEN: Self = Self::from_rgb(0.0, 1.0, 0.0);
    /// The Color Yellow
    pub const YELLOW: Self = Self::from_rgb(1.0, 1.0, 0.0);
    /// The Color White 
    pub const WHITE: Self = Self::from_rgb(1.0, 1.0, 1.0);
}
```

 I used some [random online tool](https://encycolorpedia.com/1a2421) to find the
 names of some colors I want. It provides the percentage of the channels, so the
 conversion is simple(just move the decimal to the front). Since the hamster
 begins to hobble for me when it comes to art, I also add some comment
 describing the colors with my muggle eyes. Later we might add actual sample colors
 to the docs, but for now it should only make sense for us.

 This all might seem to be better put into the user side of things, letting them
 implement the colors they prefer, but I feel like having some common set of
 colors inside the renderer itself is nice for some quick prototyping and
 testing.

 Now we run back to our OpenGL stuff and update the render target to accept
 actual colors, to hide the hideous tuples:

{% code(name="cac_graphics/context/examples/playground.rs") %}
 ```rust
let screen = RenderTarget::with_clear_color(Color32::DARK_JUNGLE_GREEN);
```
{% end %}

{% code(name="cac_graphics/context/src/render_target.rs") %}
```rust
#[derive(Copy, Clone)]
use core::Color32;
//..

pub struct RenderTarget {
    pub clear_color: Option<Color32>,
}

impl RenderTarget {
    pub fn with_clear_color(clear_color: Color32) -> Self {
        Self {
            clear_color: Some(clear_color),
        }
    }
    //..
}
```
{% end %}

{% code(name="cac_graphics/context/src/opengl.rs") %}
```rust
impl<C: GLContext> render_target::Context for Context<C> {
    fn clear(&mut self, target: &render_target::RenderTarget) {
        if let Some(color) = target.clear_color {
            let (r, g, b, a) = color.as_rgba();
            unsafe {
                gl::ClearColor(r, g, b, a);
                gl::Clear(gl::COLOR_BUFFER_BIT);
            }
        }
    }
}
```
{% end %}

More than 2000 words in and we basically only cleared the screen.. Guess we will look
at the GL objects in the next chapter.

---

[Link to the repository](https://github.com/KevinThielen/cac_graphics/tree/7e520448dcda91f989393312119e418f4327333f)

