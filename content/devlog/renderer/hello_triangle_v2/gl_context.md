+++
title = "The GL Context"
description = "The first abstraction is for the context itself. It is going to be the core of our crate."
date = 2022-12-07
weight = 1
draft = false
template="book.html"

[taxonomies]
categories = ["Devlog"]
tags = ["OpenGL", "Context", "Renderer"]
[extra]
toc = true
keywords = "Triangle, Context"
+++

The first thing we have to think about is our context itself.

Do we want the ability to select it during runtime, or is hard-wiring it during
compilation fine?

Is there a closed set of contexts, or should there be the
ability to potentially create specialized contexts from the outside? 

Then there is the whole mess with the global state machine and binding 
resources, which are more or less unique to OpenGL.

There are dozens of questions, some more relevant than others. Many of them
could be answered with `YAGNI`("You Aren't Gonna Need It"), at least right now.
We start simple, then try to figure out if/how it works out with some more
advanced examples.

Also, just because we go with one approach doesn't mean we can't adjust our
architecture down the road. In fact, it is almost guaranteed.
 
After a bit of back and forth and sketching different approaches in our
playground, we decide that it is fine to have the context being defined during
compile time. Rust's amazing enums will allow us to use static dispatch later as
well, so it will be no problem to define fallback backends. 
The compile time approach has the benefit that we can have per-implementation
configurations. Our OpenGL context for example can require anything with a
proc-address to load the function pointers, while `wgpu` is fine with something
that uses `raw-window-handle`.

So, let's start with the simplest thing first. We define an OpenGL
context.

So we add the file opengl.rs to the src of our context, to create out "OpenGL
Context"

{% code(name="cac_graphics/context/src/opengl.rs") %}
```rust 
pub struct Context {
}
```
{% end %}

My ability to name things is laughable, so feel free to use whatever
name you prefer. To follow Rust's
[API-Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html), we avoid the "OpenGL" prefix for this struct, since it's already the module name.

Let's not forget to add this module to our lib.rs:

{% code(name="cac_graphics/context/src/lib.rs") %}
```rust
pub mod opengl;
```
{% end %}

The creation of a window and the actual OpenGL-context are closely related, and
on certain platforms, the window has to be created with specific GL attributes
in mind.

For that reason, we only require the bare necessities to create our graphics
context and let the user pick whatever window / OpenGL context creation
combination they want. We used `winit` and `raw-gl-context` in our playground, but anything that allows
us to load OpenGL functions should work, for example `glutin`, `glfw`, or `SDL2`.
This has the huge benefit that the OpenGL abstraction is completely platform
independent. We can always add abstraction layers for the GLContext later, or
just integrate the common ones via feature flags.

There is an argument to be made to make our graphics Context take ownership
of the GLContext, to make sure our own context will not outlive the GLContext.
This is less about the function pointers, but more about the OpenGL objects
having their lifetimes being tied to the context itself. Taking ownership of the
GLContext means that our own Context also takes ownership over the GL objects!

There are other ways to archive this, but we are probably never going to do anything else but calling swap buffers on the
gl_context. Maybe making it (not) current later, but making the changes for that
are trivial.

So with this in mind, we abstract the GLContext via a trait and store it in our
graphics Context via a generic:

{% code(name="cac_graphics/contest/src/opengl.rs") %}
```rust 
use crate::gl43_core as gl;

pub trait GLContext {
    fn swap_buffers(&mut self);
    fn get_proc_address(&mut self, name: &'static str) -> *const std::ffi::c_void;
}

pub struct Context<C: GLContext> {
    gl_context: C,
}

impl<C: GLContext> Context<C> {
    pub fn new(mut context: C) -> Self {
        gl::load_with(|name| context.get_proc_address(name));
        Self {
            gl_context: context,
        }
    }

    pub fn update(&mut self) {
        self.gl_context.swap_buffers();
    }
}
```
Technically, our trait function and also the `update()` function don't have to take a mutable self
right now. However, we use the power of foresight to know that we will
definitely fiddle with the context state here at some point, and we also assume
that some context implementations treat swap_buffers as a mutable operation.

Now we need to adjust our playground.rs.
Our playground sees our own crate as an external one, which is the point of examples.
They should show how you use the crate from the point of view of a user.
Since we are making the abstraction via a trait, and we want the external
`raw-gl-context` to implement that trait, we are violating the orphan rule. 
 
```rust 
impl opengl::GLContext for GlContext {  
// Error: only traits defined in the current crate can be
// implemented for types defined outside of the crate
// define and implement a trait or new type instead
} 
```
This rule prevents us implementing external traits on external structs. The
reason for that is that two crates could implement the same trait on the same
struct with different implementations, and the compiler wouldn't know which one to pick.

Rust's error messages are useful as always and tell us the solution without a
spoiler warning. We need to use the [New Type Idiom](https://doc.rust-lang.org/rust-by-example/generics/new_types.html).
It's just a tuple struct wrapping our intended struct. It's now our own type,
containing the external one.

```rust 
struct GLContext(GlContext);
```

Now we can implement the trait for our new type: 

```rust 
impl context::opengl::GLContext for GLContext {
    fn swap_buffers(&mut self) {
        self.0.swap_buffers();
    }

    fn get_proc_address(&mut self, name: &'static str) -> *const std::ffi::c_void {
        self.0.get_proc_address(name)
    }
}
```

And adjust our `main`: 

{% code(name="cac_graphics/context/examples/playground.rs") %}
```rust 
use context::{gl43_core as gl, opengl};

//..

//create a context from the existing winit window
let gl_context = GlContext::create(
    &window,
    raw_gl_context::GlConfig {
        alpha_bits: 0,
        version: (4, 3),
        profile: raw_gl_context::Profile::Core,
        ..Default::default()
    },
)
.unwrap();
gl_context.make_current();
let gl_context = GLContext(gl_context);

let mut context = opengl::Context::new(gl_context);

//..

// replace gl_context.swap_buffers() in the event_loop with 
context.update();
```
{% end %}

Running it shows that everything is dandy.

Our solution has the advantage that our context can control when
`swap_buffers()` should happen, so that it can do calls before and after it(important
because of the implicit `glFinish` in swap_buffers). Granted, it is a bit verbose, but we can
just implement the trait for specific context crates later, avoid the New Type
and put them behind a feature flag.

However, certain context implementations treat the context just as a global, and
there is not a lot we can do about that, except praying that the GLContext will
always outlive ours(spoiler, it won't. Changing the window mode is
usually enough to destroy the context). We could fiddle with some check that
probes whether an object still exists and then invalidate all "references" to the
objects, but for now, let us just avoid this stuff.

Some other context implementations tie the GLContext to a window(e.g. `glfw-rs`),
and we really don't want to deal with Window nonsense in our context.
Some things might make sense, a window is just like a canvas after all, but taking control of the main loop or dealing with
input and device events are definitely out of the scope of a small render
context. We are going to try glfw soon, to check if our abstraction works.

So far, we just created the context without any validations, so let's change that. 
Now is also a great time to start with proper Error-types.

At this point, adding all the source files at the bottom of
every page, is becoming a bit too cumbersome. So I will just add the link to the
repository with the code from now on.

--- 

[Link to the
repository](https://github.com/KevinThielen/cac_graphics/tree/9eb2180f154de2bdb275a5515eeee65a128a1796)


