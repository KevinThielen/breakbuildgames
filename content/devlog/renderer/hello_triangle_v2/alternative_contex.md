+++
title = "GLFW"
description = "We are going to try our current context abstraction with GLFW, which is a non-Rusty alternative."
date = 2022-12-09
weight = 3
draft = false
template="book.html"

[taxonomies]
categories = ["Devlog"]
tags = ["OpenGL", "Context", "Renderer"]
[extra]
toc = true
keywords = "OpenGL, GLFW, Error Handling, Context"
+++

So far we only used `winit` and `raw-gl-context` for our window and context
creations. However, to be certain that our current abstraction works with
different context providers, we will try `glfw`.

GLFW stands for Graphics Library Framework and is a lightweight solution to
create a window and OpenGL/Vulkan context. 
In C++ it's one of the most common solutions in addition to SDL2.

Well, let's see how our abstraction works with this C-based library.

First of all, we need to follow the prerequisites as described on the [crates.io
page](https://crates.io/crates/glfw) and actually install GLFW
on our machine. I use Arch by the way, so I only had to call:

```bash
pacman -S glfw
```

but your mileage may vary.

Then we add it to our cargo.toml as another dev-dependency, disabling the
default-features again(unless you need wayland support): 

{% code(name="cac_graphics/context/Cargo.toml") %}
```toml 
#alternative window and context creation
glfw = { version = "0.*", default-features = false }
```
{% end %}

Now, we can pretty much yoink the sample code, just how we did with winit.
However, I added some window hints to make sure we get suitable context, and I
also made it a debug context, to make sure we get (more) debug messages with our
callback.
Also, we don't need event handling, so I just checked for ESC to close the window and removed the event handler

{% note() %} 
Because I am using a tiling window manager, I make the window not resizeable.
This has the effect that the window "pops out" of the tiles and is floating on
top.
{% end %}

Instead of putting it into main, I rename our existing one to `winit_main` and
this new one `glfw_main`, so that we can toggle between them in our actual main.

```rust 
use glfw::{Action, Context, Key};

fn glfw_main() {
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
        .create_window(300, 300, "Hello this is GLFW", glfw::WindowMode::Windowed)
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

Now we implement our GLContext trait over a newtype again. `GLFW` is one of the
aforementioned dependencies that actually require mutability for `swap_buffers`
and `get_proc_address`, so our foresight paid off!

{% code(name="cac_graphics/context/examples/playground.rs") %}
```rust 
struct GLFWContext(glfw::Window);
impl opengl::GLContext for GLFWContext {
    fn swap_buffers(&mut self) {
        self.0.swap_buffers()
    }

    fn get_proc_address(&mut self, name: &'static str) -> *const std::ffi::c_void {
        self.0.get_proc_address(name)
    }
}
```
{% end %}

Now we try to do the same with did before: We try to create our context after
making the GLContext current.

```rust
let mut context = opengl::Context::new(GLFWContext(window))?;
```


However, now we run into an issue: glfw gives us a window, which also acts as
context. If we were to transfer ownership, we wouldn't be able to use the window
anymore.

Rust throws at us: 

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

Let's take a moment to appreciate the amazing error messages that Rust provides.

Well, it makes sense. We can't move ownership when we are using the object
afterwards. There are two possible solutions.

We either do what Rust-programmers hate and use shared mutable ownership, also known as good ol' `Rc<RefCell<T>>`, so our
window and context can co-exist, or we add another function to expose the
"inner context" somehow.

{% note() %}
Actually, GLFW allows us to create a `RenderContext` from the Window, which we
could pass to our Context with no problems. Its intended use is for sharing the
context between threads, so it would be kinda a misuse. We would also need to
expose something else to load the function pointers. Exposing the "inner"
field seems to be the better alternative.
{% end %}

While this is probably one of the few cases where `Rc<RefCell<T>>` is not that
bad, since the overhead is negligible(just a few calls per frame) and since we
are limited to the main thread anyway, and there should never be a borrow issue,
exposing the inner field seems to be valuable in general.

So let's just add this function to our Context: 

{% code(name="cac_graphics/context/src/opengl.rs") %}
```rust 
pub fn raw_context(&mut self) -> &mut C {
    &mut self.gl_context
}
```
{% end %}

and then adjust the calls in our main. While we are at it, let's add our OpenGL
calls as well: 

```rust 
fn glfw_main() -> anyhow::Result<(), anyhow::Error> {

    //..

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
changing the GLContext trait, but it's literally just 2 calls, so we can marginally bear it without making anyone angry.

Now, let's get rid of dem pesky GL calls in our playground.

---

[Link to the repository](https://github.com/KevinThielen/cac_graphics/tree/c3ec355bebb78e0d778bd2aff3fb41436749b992)
