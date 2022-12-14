+++
title = "Error Handling"
description = "We weaseled our way out of error handling so far. Now is the time to do it right!"
date = 2022-12-07
weight = 2
draft = false
template="book.html"

[taxonomies]
categories = ["Devlog"]
tags = ["OpenGL", "Context", "Renderer"]
[extra]
toc = true
keywords = "OpenGL, Triangle, Error Handling, Context"
+++

We managed to weasel our way out of error handling so far. However, since we are
now entering the stage of proper abstractions, error handling is part of that.
So let's create an error module.

{% note(title="Why not thiserror?", kind="question") %}
We could use the [thiserror](https://crates.io/crates/thiserror) crate, it's an amazing crate to avoid a lot of boilerplate, especially when it comes to all the conversion functions.
However, adding `proc-macro2` as a transient dependency [seems a bit heavy](../../the_bad.md#dependency-hell). We are not
going to do anything where we need to touch multiple error types or require
nested errors, nor do we need any conversions, so there is not a lot of
boilerplate that `thiserror` would save in our particular case.
{% end %}

## Our Error Module

Writing our own error types is fairly straightforward.
We use an Enum, and implement the Error trait, which also requires us to
implement Display and Debug. That's it.

{% code(name="cac_graphics/context/src/error.rs") %}
```rust 
use std::fmt::Display;

#[derive(Debug)]
pub enum Error {
    //The Context doesn't fit the requirements
    InvalidContext(String),
}

impl std::error::Error for Error {}

impl Display for Error {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::InvalidContext(error) => write!(f, "invalid context, caused by {error}"),
        }
    }
}
```
{% end %}

and add it to our lib.rs 

```rust
pub mod error;
```

That's it. We will just add another variant to our enum and another message in
the Display implementation for new error types. The message itself follows the
[API guidelines](https://rust-lang.github.io/api-guidelines/interoperability.html?#error-types-are-meaningful-and-well-behaved-c-good-err) of using lower case, with no trailing punctuation.

We also spit out an error message, which might depend on the specific context we
are trying to create. It could make sense to split the `InvalidContext` error further, like
"InvalidContextVersion" and such, or create dedicated error types for the
different context implementations(OpenGL, Vulkan, ..), but for now this "catch
all" is enough. It's not like we would handle them differently. If one fails, we
just try another one or we panic.

Now back to our `opengl.rs`, we make the context creation fallible.
Right now there are only 2 things that could fail: The context doesn't support
our OpenGL version(must be >= 4.3), or our function pointers are not
loaded.

So we first check if we loaded the function pointers for a function that existed
since legacy OpenGL(`glGet`), then we use the same function to get the version
of our context.

```rust
//cac_graphics/context/src/opengl.rs
impl<C: GLContext> Context<C> {
    pub fn new(mut context: C) -> Result<Self, Error> {
        //load all the OpenGL 4.3 function pointers
        gl::load_with(|name| context.get_proc_address(name));


        if !gl::GetIntegerv::is_loaded() {
            return Err(Error::InvalidContext(String::from(
                "failed to load OpenGL fn pointers",
            )));
        }

        let mut version = (0, 0);
        unsafe {
            gl::GetIntegerv(gl::MAJOR_VERSION, &mut version.0);
            gl::GetIntegerv(gl::MINOR_VERSION, &mut version.1);
        }

        if version.0 < 4 || (version.0 == 4 && version.1 < 3) {
            return Err(Error::InvalidContext(format!(
                "version 4.3 required, received {}.{}",
                version.0, version.1
            )));
        }

        Ok(Self {
            gl_context: context,
        })
    }
    //..
}
```
Since OpenGL is forward compatible, we only have to check if the version is
lower than 4.3. A context with the version 4.6 works perfectly fine, so would a
possibly future 5.1. 

We are just checking a single function pointer instead of all we need. That
means our program will panic if a not-loaded function is called, but at that point there
is something severely wrong with our bindings or the OpenGL library itself. 

To be sure, we could hijack the loader function and check if the pointer is valid
before returning it in the `gl::load_with` closure, but this will probably produce
unsatisfactory results, since it does not seem to be guaranteed to return null
pointers on failure. For example, manually calling
`context.get_proc_address("EvenSpeedwagonIsAfraid")` will return a non-null
pointer on my Linux machine. It is possible that it might behave differently on other devices/with
other drivers. Maybe I am missing something here(really encourage some input on
that to edit this part!), but it doesn't seem reliable for our purpose.

Thanks to our clippy lints, the function looks noisy in our editor: 

![clippy noise](../clippy_doc.png)

We could mute this noise, but we might as well just document our assumptions
about the error cases. On one hand, we will often need to change/rewrite the
docs, but on the other hand, nothing is more soul crushing than writing all the
docs at the end, which just increases the odds that we procrastinate on them.For now, we just
bear it until we reach a point with our abstractions where we are happy, that is
when removed all stray gl calls and unwraps() from our playground. We embrace the noise as a
feature, not a problem, like a little warning light we should deal with in the
near future, while we still have the failure cases in our head.
If you want to mute them for now, you can just add `#[allow(clippy::missing_errors_doc)]` before the function/struct, enable it for the entire module, or do it for the entire crate.

Don't forget to adjust the `playground.rs` to make the error check. We change
the return value of our main to anyhow's Kirby-Result, and we use the `?`
operator on our construction. We ignore the unwraps() in the other parts for
now, because we will nuke our playground anyway in the near future, and we just
want to make sure that our error handling works.

```rust 
fn main() -> anyhow::Result<()> {
//...
    let mut context = opengl::Context::new(gl_context)?;
//...
```

We can test it by changing the requested version in the GlConfig, to make our context creation
fail with:

```bash
Error: invalid context, caused by version 4.3 required, received 3.3
```
It's not the best message, but it does its job.

Finally, there is one more thing we want to do:
Validating our OpenGL calls.

## OpenGL Error Callback
As mentioned before, OpenGL relies on global state and constants. We don't have
the same type safety and guarantees as with Rust when we invoke gl calls.

However, OpenGL offers tools to validate its state and the calls we make.
The old way was to poll the value of
[glGetError](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glGetError.xhtml) and check the status code.
Because polling the state after every single call is tedious, there are 2 common
traps used in practice. One is to poll the state at the end of the frame(since
it's a queue it will not clear the previous errors) to check for errors in
general, and then in the debug step to manually insert it in the code to actually find the culprit.

The other trap is the use of macros to wrap all gl calls to automatic
poll after the call, like `GL_CALL(glClear(GL_COLOR_BUFFER_BIT))`, which adds
needless verbosity and noise.

Generally, any OpenGL function binding generator worth its salt, allows the option for debug variants, which does the polling after invoking the actual function, so that the call site doesn't have to deal with it. They usually also come with debug callbacks. Our used `gl_generator` is not different.  

Nowadays, even an OpenGL 3.3 context wants to generally check for the `GL_KHR_debug`
or `ARB_debug_output` output extensions, which are supported on pretty much all
hardware that came out in the past decade.

With 4.3 the functionality has been made core, so that we can just use
`glDebugMessageCallback`, which will be automatically called once there is a GL
error. This might require a specific debug context, depending on the implementation. 

So all we have to do is create the callback, which just logs the debug messages,
and then enable it in our context creation. We will just use the log crate and
let the user deal with how they want to display them.

{% code(name="cac_graphics/context/src/opengl.rs") %}
```rust 
extern "system" fn debug_callback(
    source: u32,
    kind: u32,
    id: u32,
    severity: u32,
    _length: i32,
    message: *const i8,
    _user_param: *mut std::ffi::c_void,
) {
    let source = match source {
        gl::DEBUG_SOURCE_API => "API",
        gl::DEBUG_SOURCE_SHADER_COMPILER => "SHADER COMPILER",
        gl::DEBUG_SOURCE_WINDOW_SYSTEM => "WINDOW SYSTEM",
        gl::DEBUG_SOURCE_OTHER => "OTHER",
        gl::DEBUG_SOURCE_APPLICATION => "APPLICATION",
        gl::DEBUG_SOURCE_THIRD_PARTY => "THIRD PARTY",
        _ => "UNKNOWN",
    };

    let kind = match kind {
        gl::DEBUG_TYPE_ERROR => "ERROR",
        gl::DEBUG_TYPE_DEPRECATED_BEHAVIOR => "DEPRECATED",
        gl::DEBUG_TYPE_UNDEFINED_BEHAVIOR => "UNDEFINED BEHAVIOUR",
        gl::DEBUG_TYPE_PORTABILITY => "PORTABILITY",
        gl::DEBUG_TYPE_PERFORMANCE => "PERFORMANCE",
        _ => "UNKNOWN",
    };

    let error_message = unsafe {
        std::ffi::CStr::from_ptr(message)
            .to_str()
            .unwrap_or("[FAILED TO READ GL ERROR MESSAGE]")
    };

    match severity {
        gl::DEBUG_SEVERITY_HIGH => log::error!("{id}: {kind} from {source}: {error_message}"),
        gl::DEBUG_SEVERITY_MEDIUM => log::warn!("{id}: {kind} from {source}: {error_message}"),
        gl::DEBUG_SEVERITY_LOW => log::info!("{id}: {kind} from {source}: {error_message}"),
        gl::DEBUG_SEVERITY_NOTIFICATION => {
            log::trace!("{id}: {kind} from {source}: {error_message}");
        }
        _ => log::trace!("{id}: {kind} from {source}: {error_message}"),
    };
}
```
{% end %}

It looks kinda verbose, but we just banish it at the bottom of our module and
forget about it.

All we do is turning all the weird constants(which
are just numbers), into readable &str's. The CStr::from_ptr is unfortunate
though, because we only get an i8 pointer for the message.. It should be
null-terminated by default, so we can ignore the message length and in the worst
case just have some message that tells us that the conversion failed.

Finally, we use the severity for the different log levels.

And in our new-function we enable it with: 
```rust 
//cac_graphics/context/src/opengl.rs
pub fn new(mut context: C) -> Result<Self, Error> {
    //...
    unsafe {
        gl::Enable(gl::DEBUG_OUTPUT);
        gl::DebugMessageCallback(Some(debug_callback), std::ptr::null());
    }
    Ok(Self {
        gl_context: context,
    })
}
```
Later, we can use environment variables, feature flags or profiles to
conditionally enable it.

So, let's run our program and we see... nothing.
The reason is that we still need to use a logging implementation to actually log
the messages. In the context creation chapter, we added the `env_logger`
crate for this, but any other logger, like `simple_logger`, `fern`, `log4rs`,
etc. work. `env_logger` is simple and it's easy to modify with environment
variables.

```rust
//cac_graphics/context/examples/playground.rs
fn main() -> anyhow::Result<()> {
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("warn"))
        .format_timestamp(None)
        .init();

//...
}
```

We use a default level to log everything above "warn", which means either
"warnings" or "error". But we could change this by using the environment
variable `RUST_LOG`. For example, to get more information, we could just run
```bash
RUST_LOG="context=trace" cargo run --example playground
```
to print all log levels for the crate "context"(our crate).

To verify whether it is working, we can change to just trace and see some output
from winit: 

![winit logs](../winit_log.png)

Depending on your driver, you might see some log like:

```
 UNKNOWN from API: Buffer detailed info: Buffer object 1 (bound to GL_ARRAY_BUFFER_ARB, usage 
hint is GL_STATIC_DRAW) will use VIDEO memory as the source for buffer object operations.
```

Worry not, it's not an error, just some info.


Now, let's add an actual error in our playground to test the message callback.
We make a reasonable error. When we want to clear the color of
our screen, we accidentally use the wrong constant, because there is no
type-safety. 

{% code(name="cac_graphics/context/examples/playground.rs") %}
```rust 
//...
    unsafe {
        //Spoiler: This should have been COLOR_BUFFER_BIT instead, what a silly error
        gl::Clear(gl::COLOR);
    }
```
{% end %}

Now we run it again, and boom goes the dynamite: 

![boom goes the dynamite](../boom.png)

"Invalid clear bits", there we go.
It tells us what went wrong, it is kinda pretty, albeit spammy(because it's in a loop), it is everything we want.
Now, let's get back to that GLFW-thingy.

{% note(title="Warning", kind="warn") %}
Since our current context dependency doesn't allow us to create a specific
OpenGL debug context, it is possible that there are different or no messages at
all, depending on your platform. We will take care of this in the next chapter.
{% end %}

---

[Link to the repository](https://github.com/KevinThielen/cac_graphics/tree/b3a36b598b596c210656bb177b7dcb189db5724f)
