+++
title = "Hello Triangle"
paginate_by = 2
sort_by = "weight"
weight = 3
transparent = true
insert_anchor_links="heading"
template = "book.html"
[extra]
toc = true
+++

What better way to start than the "Hello World" of graphics programming, 
drawing a triangle?

We use our `playground.rs` from the [context chapter](./../the_context/opengl) as
base: 

{% code(name="cac_graphics/context/examples/playground.rs") %}
```rust 
use raw_gl_context::GlContext;
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

    //red and alpha channels as 1.0, rest as 0.0
    unsafe {
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
            // "clear the screen"
                unsafe {
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


{% code(name="cac_graphics/context/Cargo.toml") %}
```toml 
[package]
name = "context"
version = "0.1.0"
edition = "2021"

[dependencies]
#logging
log = "0.*"

[dev-dependencies]
#OpenGL context creation
raw-gl-context = "0.*"
#error handling
anyhow = "1.*"
#logging
env_logger = "0.*"
#window creation
winit = { version = "0.*", default-features = false, features =["x11"] }  
```
{% end %}


## Drawing Stuff in OpenGL

As mentioned before, I won't go into all the details when it comes to OpenGL,
you are better served with the amazing [learnopengl.com](learnopengl.com). This little
series will go more into the "practical" use and the implementation, while I just
give a short TL;DR explanation at best.

There are a few things we need to draw in modern OpenGL:

- Buffer objects
- Vertex Attributes
- Shaders

So let's dive into the buffer objects first.



