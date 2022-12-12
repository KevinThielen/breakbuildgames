+++
title = "Time For Abstractions"
paginate_by = 2
sort_by = "weight"
weight = 4
transparent = true
insert_anchor_links="heading"
template = "book.html"
[extra]
toc = true
+++

We just yeeted all of our code into our playground.rs, didn't care
about error handling, didn't validate any input, didn't care about cleaning up
our resources and we have some potentially unsound holes in our design.
That we also have gl calls in our main loop is just the tip of the iceberg.

However, this is how I generally like to tackle new things in programming: Explore the
problem in some throwaway program, then once I have a better idea of what I
need, I start thinking about proper data organization and abstractions.
I feel like Rust's strict ownership rules really encourage this kind of
thinking, or you end up [rewriting your
code later](../../the_ugly.md#rewrite-it-in-rust). Even when coming up with the API, the playground can serve as a way to get a feel for the ergonomics and usability. What good is an abstraction when we have to store or refer to our context as `RenderContext<Context=OpenGL, Texture=GLTexture, Buffer=GLBuffer, ..>` or store resources as concrete `OpenGLBuffer`, so that swapping our backend will force us to change the fields in dozens of structs, or have some heavy generics all over the place? 

## Creating the Context
The first thing have to think about is our context itself.
Do we want the ability to select it during runtime, or is hard-wiring it during
compilation fine? Is there a closed set of contexts, or should there be the
ability to potentially add specialized contexts from the outside? 
Then there is the whole mess with the global state machine and binding the
resources we want to use, which are more or less unique to OpenGL.

There are dozens of questions, some more relevant than others. Many of them
could be answered with `YAGNI`("You Aren't Gonna Need It"), at least right now.
Also, just because we go with one approach doesn't mean we can't adjust our
architecture down the road. In fact, it is almost guaranteed we will make
changes down the line.

So, let's start with the simplest thing first. We to define an OpenGL
context. So I add the file opengl.rs to the src of our context.

```rust 
//cac_graphics/context/src/opengl.rs
pub struct Context {
}
```
My ability to name things is laughable, so feel free to use whatever
name you prefer. To follow Rust's
[API-Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html), we avoid the "OpenGL" prefix for this structs, since it's already the module name.

Don't forget to add this module to our `lib.rs`
```rust
//cac_graphics/context/src/lib.rs
pub mod opengl;
```

The creation of a window and the actual OpenGL-context are closely related, and
on certain platforms, the Window has to be created with specific GL attributes
in mind.
For that reason, we only require the bare necessities to create our graphics
context and let the user pick whatever window / OpenGL context creation they
want. We used `winit` and `raw-gl-context` in our playground, but anything that allows
us to load OpenGL functions should work, for example `glutin`, `glfw`, or `SDL2`.
This has the huge benefit that the OpenGL abstraction is completely platform
independent. We can always add abstraction layers for the GLContext later, or
just integrate the common ones via feature flags.

There is an argument to be made to make our graphics Context take ownership
of the GLContext, to make sure our own context will not outlive the GLContext.
This is less about the function pointers, but more about the OpenGL objects
lifetimes being tied to the context itself. 

There are other ways to archive this, but we are probably never going to do anything else but calling swap buffers on the
gl_context. Maybe making it (not) current later, but making the changes for that
are trivial, so taking ownership seems to be a reasonable approach.


So with this in mind, we abstract the GLContext via a trait and store it in our
graphics Context:
```rust 
//cac_graphics/contest/src/opengl.rs
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
Technically,  our `update()` doesn't have to be mutable
right now. However, we use the power of foresight to know that we will
definitely fiddle with the context state here at some point.

Now we need to adjust our playground.rs.
Since we are making the abstraction via a trait, and we want the external
`raw-gl-context` implement that trait, we are violating the orphan rule. 
 
```rust 
impl opengl::GLContext for GlContext {  
// Error: only traits defined in the current crate can be
// implemented for types defined outside of the crate
// define and implement a trait or new type instead
} 
```
This rule prevents us implementing external traits on external structs. The
reason for that is that two crates could implement the same trait on the same
struct with different implementations, and the complainer wouldn't know which one to pick.

Rust's error messages are useful as always and tell us the solution without a
spoiler warning. We need to use the [New Type Idiom](https://doc.rust-lang.org/rust-by-example/generics/new_types.html).
It's just a tuple struct wrapping our intended struct. It's our own type,
sitting containing the external one.

```rust 
struct GLContext(GlContext);
```
Now we can implement the trait: 
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
```rust 
//cac_graphics/context/examples/playground.rs 
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
Let's run it and see how everything is working dandy. Our solution has the advantage that our context can control when
swap buffers happens, so that it can do call before and after it(important
because of the implicit `glFinish`). Granted, it is a bit verbose, but we can
just implement the trait for specific context crates later, avoid the New Type
and put them behind a feature flag.

However, certain context implementations treat the context just as a global, and
there is not a lot we can do about that, except praying that the GLContext will
always outlive ours(spoiler, it won't. Changing the resolution/window mode is
usually enough to destroy the context). We could fiddle with some check that
probes whether a object still exists and then invalidate all "references" to the
objects, but for now, let us just avoid this stuff.

Some other context implementations tie the GLContext to a window(e.g. `glfw-rs`),
and we really don't want to deal with Window nonsense in our context.
Some things might make sense, a window is just like a canvas after all, but taking control of the main loop or dealing with
input and device events are definitely out of the scope of a small render
context. We are going to try glfw at the end of this chapter to explore how we
could do this with out abstraction.

So far, just created the context without any validation, so let's change that. 
At this point it is also great to start with proper Error-types.

## Error handling
So let's create an error module.
We could use the `thiserror` crate, but adding `proc-macro2` as transient
dependency [seems a bit heavy](../../the_bad.md#dependency-hell). So we just write our own error types: 


```rust 
//cac_graphics/context/src/error.rs
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

That's it. We will just add another variant to our enum and another message in
the Display implementation for new errors.

It's reasonable to split the `InvalidContext` error further, like
"InvalidContextVersion" and such, or create dedicated error types for the
different context implementations(OpenGL, Vulkan, ..), but for now this "catch
all" is enough.

Now back to our `opengl.rs`, we make the context creation fallible.
Right now there are only 2 things that could fail: The context doesn't support
our OpenGL version(must be >= 4.3), or our function pointers are not
loaded(missing system library, ..).

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
is something severely wrong with our bindings or the OpenGL library itself. To
be sure, we could hijack the loader function and check if the pointer is valid
before return it in the `gl::load_with` closure, but this will probably produce
unsatisfactory results, because it does not seem to be guaranteed to return null
pointers on failure. For example, manually calling
`context.get_proc_address("EvenSpeedwagonIsAfraid")` will return a non-null
pointer on my Linux machine. It is possible that it might behave differently on other devices/with
other drivers. Maybe I am missing something here(really encourage some input on
that to edit this part!), but it doesn't seem reliable for our purpose.

Thanks to our clippy lints, the function looks noisy in our text
editor: 
![clippy noise](../img/context/clippy_doc.png)

We could mute this noise, but we might as well just document our assumptions
about the error cases. On one hand, we will often need to change/rewrite the
docs, but on the other hand, nothing is more soul crushing than writing all the
docs at the end and just increase the odds that we procrastinate on them. For now, we just
bear it until we reach a point with our abstractions where we are happy, that is
when removed all stray gl calls and unwraps() from our playground. We embrace the noise as a
feature, not a problem, like a little warning light we should deal with in the
near future, while we still have the failure cases in our head.
If you want to mute them for now, you can just add `#[allow(clippy::missing_errors_doc)]` before the function/struct, enable it for the entire module, or do it for the entire crate.

Don't forget to adjust the `playground.rs` to make the error check. We change
the return value of our main, finally using anyhow, and we use the `?` operator
in stead of that pesky unwrap.

```rust 
fn main() -> anyhow::Result<(), anyhow::Error> {
//...
    let mut context = opengl::Context::new(gl_context)?;
//...
```

Finally, there is one more thing we want to do in our context creation:
Validating our OpenGL calls.

## OpenGL Error Callback
As mentioned before, OpenGL relies on global state and constants. We don't have
the same type safety and guarantees as Rust when we invoke gl calls.

However, OpenGL offers tools to validate its state and the calls we make.
The old way was to poll the value of
[glGetError](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glGetError.xhtml) and check the status code.
Because polling the state after every single call is tedious, there are 2 common
traps used in practice. One is is poll the state at the end of the frame(since
it's a queue it will not clear the previous errors) to check for errors in
general, and then in the debug step to manually insert it in the code to actually find the culprit.

The other trap is the use macros to wrap all gl calls into an automatic
polling after the call, like `GL_CALL(glClear(GL_COLOR_BUFFER_BIT))`, which adds
needless verbosity and noise.

Generally, any OpenGL function binding generator worth its salt, allows the option for debug variants, which does the polling after invoking the actual function, so that the call site doesn't have to deal with it. They usually also come with debug callbacks.  

Nowadays, even OpenGL 3.3 context want to generally check for the `GL_KHR_debug`
or `ARB_debug_output` output extensions, which are supported on pretty much all
somewhat modern hardware.

With 4.3 the functionality has been made core, so that we can just use
`glDebugMessageCallback`, which will be automatically called once there is a GL
error. This might require a specific debug context, depending on the implementation. 

So all we have to do is create the callback, which just logs the debug messages
and then enable it in our context creation. We will just use the log crate and
let the user deal with how they want to display the logs.


```rust 
//cac_graphics/context/src/opengl.rs
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

It looks kinda verbose, but all we do is turning all the weird constants(which
are just numbers), into readable &str's. The CStr::from_ptr is unfortunate

And in our new-function: 
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
etc. work. `env_logger` is simple, easy to modify with environment
variables.

```rust
//cac_graphics/context/examples/playground.rs
fn main() -> anyhow::Result<(), anyhow::Error> {
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

Now, let's add an actual error in our playground to test the message callback.
We make a reasonable error. Before our event_loop, we want to clear the color of
our screen. So we add:
```rust 
//cac_graphics/context/examples/playground.rs
//...
    unsafe {
        //Spoiler: This should have been COLOR_BUFFER_BIT instead, what a silly error
        gl::Clear(gl::COLOR);
    }
    
    event_loop.run(move |event, _, control_flow| {
    //...
```

Now we run it again, and boom goes the dynamite: 

![boom goes the dynamite](../img/context/error_log.png)

"Invalid clear bits", there we go.
It tells us what went wrong, it is kinda pretty, it is everything we want.
Now, let's get back to that GLFW-thingy.




## Alternative GLContext/Window: GLFW

GLFW stands for Graphics Library Framework and is a lightweight solution to
create a window and OpenGL/Vulkan context. 
In C++ it's the most common solution in addition to SDL2.

Well, let's see how our abstraction works with this C-based library.

First of all, we need to follow the prerequisites as describe on the [crates.io
page](https://crates.io/crates/glfw) and actually to install GLFW
on our machine. I use Arch by the way, so I only had to call:
```Shell
pacman -S glfw
```
but your mileage may vary.

Now, we can pretty much yoink the sample code, just we did with winit.
However, I added some window hints to make sure we get suitable context, and I
also made it a debug context. Also, we don't need event handlings, so I just
checked for ESC to close the window and removed the event handler

```rust 
use glfw::{Action, Context, Key};

fn main() {
    let mut glfw = glfw::init(glfw::FAIL_ON_ERRORS).unwrap();

    // Create a windowed mode window and its OpenGL context
    // window hints have to be set before the window is created
    glfw.window_hint(glfw::WindowHint::ContextVersion(4, 3));
    glfw.window_hint(glfw::WindowHint::OpenGlProfile(
        glfw::OpenGlProfileHint::Core,
    ));
    glfw.window_hint(glfw::WindowHint::OpenGlDebugContext(true));
    glfw.window_hint(glfw::WindowHint::Resizable(false));

    let (mut window, events) = glfw
        .create_window(300, 300, "Hello this is window", glfw::WindowMode::Windowed)
        .expect("Failed to create GLFW window.");

    window.set_key_polling(true);
    window.make_current();

    while !window.should_close() {
        glfw.poll_events();
        for (_, event) in glfw::flush_messages(&events) {
            if let glfw::WindowEvent::Key(Key::Escape, _, Action::Press, _) = event {
                window.set_should_close(true)
            }
        }
    }
}
```
If all went well, running it should open a little window.

However, now we see an issue: glfw gives us a window, which also acts as
context. If we were to transfer ownership, Rust would throw us: 

```rust 
error[E0382]: borrow of moved value: `window`
   --> context/examples/playground.rs:204:12
    |
195 |     let (mut window, events) = glfw
    |          ---------- move occurs because `window` has type `glfw::Window`, which does not implement the `Copy` trait
...
202 |     let context = opengl::Context::new(GLFWContext(window))?;
    |                                                    ------ value moved here
203 |
204 |     while !window.should_close() {
                 ^^^^^^^^^^^^^^^^^^^^^ value borrowed here after move
```

Well, it makes sense. We can't move ownership when we are using the object
afterwards. There are two possible solutions, we either do what Rust-programmers
hate and use shared mutability, also known as good ol\` `Rc<RefCell<T>>`, so our
window and context can co-exist, or we add another function to expose the
"inner context".
```admonish note
Actually, GLFW allows us to create a `RenderContext` from the Window, which we
could pass to our Context with no problems. It's intended use is for sharing the
context between threads, so it would be kinda a misuse. We would also need to
expose something to load the function pointers anyway. Exposing the "inner"
field seems to be a reasonable alternative.
```
While this is probably one of the few cases where `Rc<RefCell<T>>` is not that
bad, since the overhead is negligible(just a few calls per frame) and since we
are limited to the main thread anyway, exposing the inner field seems to be
valuable in general.

So let's just add this function to our context: 
```rust 
pub fn raw_context(&mut self) -> &mut C {
    &mut self.gl_context
}
```
and then adjust the calls in our main. While we are at it, let's add our OpenGL
calls as well: 

```rust 
//cac_graphics/context/examples/playground.rs
use glfw::{Action, Context, Key};

struct GLFWContext(glfw::Window);

impl opengl::GLContext for GLFWContext {
    fn get_proc_address(&mut self, name: &'static str) -> *const std::ffi::c_void {
        self.0.get_proc_address(name)
    }
    fn swap_buffers(&mut self) {
        self.0.swap_buffers()
    }
}

fn main() -> anyhow::Result<(), anyhow::Error> {
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("warn"))
        .format_timestamp(None)
        .init();

    let mut glfw = glfw::init(glfw::FAIL_ON_ERRORS)?;

    // Create a windowed mode window and its OpenGL context
    // window hints have to be set before the window is created
    glfw.window_hint(glfw::WindowHint::ContextVersion(4, 3));
    glfw.window_hint(glfw::WindowHint::OpenGlProfile(
        glfw::OpenGlProfileHint::Core,
    ));
    glfw.window_hint(glfw::WindowHint::OpenGlDebugContext(true));
    glfw.window_hint(glfw::WindowHint::Resizable(false));

    let (mut window, events) = glfw
        .create_window(300, 300, "Hello this is window", glfw::WindowMode::Windowed)
        .unwrap();

    window.set_key_polling(true);
    window.make_current();

    let mut context = opengl::Context::new(GLFWContext(window))?;

    unsafe {
        gl::ClearColor(1.0, 0.0, 0.0, 1.0);
    }

    #[rustfmt::skip]  //skip the default formatting to make it cleaner
    const QUAD_VERTICES: [f32; 3 * 4] = [
        //     X,    Y,   Z    Position
        -0.9, -0.9, 0.0, // bottom left
        -0.9,  0.9, 0.0, // top left
        0.9,  0.9, 0.0, // top right
        0.9, -0.9, 0.0, // bottom right
    ];

    let vbo = create_vbo(&QUAD_VERTICES);
    let vao = create_vao(vbo);
    let shader_program = create_shader();

    unsafe {
        gl::UseProgram(shader_program);
        gl::BindVertexArray(vao);
    }


    while !context.raw_context().0.should_close() {
        unsafe {
            gl::Clear(gl::COLOR_BUFFER_BIT);
            gl::DrawArrays(gl::TRIANGLE_STRIP, 0, 4);
        }
        context.update();

        glfw.poll_events();
        for (_, event) in glfw::flush_messages(&events) {
            if let glfw::WindowEvent::Key(Key::Escape, _, Action::Press, _) = event {
                context.raw_context().0.set_should_close(true)
            }
        }
    }
    Ok(())
}
```

There are some ways to make it a bit more ergonomic, like abusing `Deref` or
changing the GLContext trait, but
it's literally just 2 calls, so we can marginally bear it without making anyone angry.

N
