+++
title = "[Excursion] Handwritten GL Bindings"
description = "How to write the bindings from scratch"
date = 2022-12-10
weight = 2
draft = false
template="book.html"

[taxonomies]
categories = ["Devlog"]
tags = ["GameDev", "OpenGL", "Context", "Renderer"]
[extra]
toc = true
keywords = "Game Development, GameDev, OpenGL, Graphics Programming"
+++

{% note(title="Note") %}
This is a simple excursion about writing the previously generated OpenGL
bindings from scratch. Feel free to just skim through this chapter or skip it
altogether.
{% end %}


In the previous chapter we have setup a simple playground for us.

Before going further into the context creation, let's start with a simple
excurse about the OpenGL bindings. 

Our playground is currently using automatically generated gl bindings.

However, the OpenGL spec is huge and many of the functions/types are legacy
luggage, or at least not needed for our purpose. For example, there are around
800 lines of constant, while we only need a few dozens of them at best.
The generated file is a humongous 900kb chunkster.
So, how much "bloat" are we generating by using a bindings generator over just
writing what we need? 

Well, let's write some bindings and compare the result :)

### Writing Bindings From Hand
The [gl.xml](https://github.com/KhronosGroup/OpenGL-Registry/blob/main/xml/gl.xml) on the official OpenGL-Registry repository defines everything that is needed.

Generally, the bindings require 4 parts: 
- defining the GL types 
- setting the GL constants 
- defining the functions
- loading the function pointers

The easiest thing to start with are the actual function definitions. 
Our playground from the [previous chapter](./the_context.md) example requires calls to glClear and glClearColor.
They are defined as this, in the XML-file:

{% code(name="gl.xml") %}
```XML 
<command>
     <proto>void <name>glClear</name></proto>
     <param group="ClearBufferMask"><ptype>GLbitfield</ptype> <name>mask</name></param>
     <glx type="render" opcode="127"/>
</command>
<command>
    <proto>void <name>glClearColor</name></proto>
    <param group="ColorF"><ptype>GLfloat</ptype> <name>red</name></param>
    <param group="ColorF"><ptype>GLfloat</ptype> <name>green</name></param>
    <param group="ColorF"><ptype>GLfloat</ptype> <name>blue</name></param>
    <param group="ColorF"><ptype>GLfloat</ptype> <name>alpha</name></param>
    <glx type="render" opcode="130"/>
</command>
```
{% end %}

This is a lot of noise. All we need are the return values, the name of the
function, and types and names of the parameters. Instead of crawling through
the XML, we could also just be searching for the functions in the
[refpages](https://registry.khronos.org/OpenGL-Refpages/gl4/) or on
[docs.gl](docs.gl).

```c 
//C Specification
void glClear(GLbitfield mask);
void glClearColor(GLfloat red, GLfloat green, GLfloat blue, GLfloat alpha);
```

Both are void functions, which means they have no return values. glClear
takes a mask with the type GLbitfield and glClearColor takes in the 4 color
channels as GLfloat. Just as the XML specifies, but more concise! Great!

More than that, they also describe the accepted values.
For example, the parameters for glClear specify the mask argument with: 

> **mask** \
> Bitwise OR of masks that indicate the buffers to be cleared. The three masks are GL_COLOR_BUFFER_BIT, GL_DEPTH_BUFFER_BIT, and GL_STENCIL_BUFFER_BIT

So we also know that we require the 3 constants for the buffer bits as well.
This relation is specified in the XML via the group, but makes finding everything
that is needed less convenient than just using the docs.

Now that we know the types we are going to need, the second step is to define them.

First we need the "platform" definitions. Things that map OpenGL-types to FFI-compatible Rust types.
Thing like GLint, GLEnum, etc being defined with
compatible "c types" that will be used for the FFI calls.

Luckily, there is an official table with a description and the number of bits for each
type on the [khronos wiki](https://www.khronos.org/opengl/wiki/OpenGL_Type). The
alternative would be reading the official specs.

| C type  | Bitdepth  | Description  |
|---|---|---|
| GLboolean | 1+  | A value, either GL_TRUE or GL_FALSE  |
| GLbyte | 8  | Signed, 2's complement binary integer  |
| GLubyte | 8  | Unsigned binary integer  |
| GLshort | 16  | Signed, 2's complement binary integer  |
| GLint | 32  | Unsigned binary integer  |
| GLuint | 32  | Signed, 2's complement binary integer  |
| GLenum | 32 | An OpenGL enumarator value  |
| GLsizeptr | pointer size  | Non-negative binary integer size for memory offsets and ranges   |
| GLfloat | 32  | An IEEE-754 floating-point value |
| ..      | ..  | .. |

Which makes it pretty straight forward.

```rust 
pub mod handmade_gl {
    pub mod types {
        use core::ffi;

        pub type GLbitfield = ffi::c_uint;
        pub type GLfloat = ffi::c_float;
    }
}
```

This is by no means exhaustive. As mentioned, the point of this exercise is to
define just as little as needed and keep adding as we go. `GLbitfield` and `GLfloat`
are all we need for `glClear` and `glClearColor`.

Now, we can also define the constants.
As mentioned, glClear also defines 3 constants for the clear bits.
This is where the
[XML](https://raw.githubusercontent.com/KhronosGroup/OpenGL-Registry/main/xml/gl.xml) is being handy. We can just search for the constant definition and copy the values: 

```XML
<enums namespace="GL" group="AttribMask" type="bitmask">
    ..
    <enum value="0x00000100" name="GL_DEPTH_BUFFER_BIT" group="ClearBufferMask,AttribMask"/>
    ..
    <enum value="0x00000400" name="GL_STENCIL_BUFFER_BIT" group="ClearBufferMask,AttribMask"/>
    ..
    <enum value="0x00004000" name="GL_COLOR_BUFFER_BIT" group="ClearBufferMask,AttribMask"/>
    ..
</enums>
```
Which translates into this in Rust-land:
```rust 
mod handmade_gl {
    //constants
    const GL_COLOR_BUFFER_BIT: GLbitfield   = 0x00004000;
    const GL_DEPTH_BUFFER_BIT: GLbitfield   = 0x00000100;
    const GL_STENCIL_BUFFER_BIT: GLbitfield = 0x00000400;
}
```
Now that this is done, let's define all the gl functions and
their signatures as types for convenience.

```rust 
mod handmade_gl {
    type Clear = unsafe extern "system" fn(mask: types::GLbitfield);
    type ClearColor = unsafe extern "system" fn(
        red: types::GLfloat,
        green: types::GLfloat,
        blue: types::GLfloat,
        alpha: types::GLfloat,
    );
```

The `extern "system"` specifies that Rust calls external function through a foreign function
interface(FFI). They are in an external library. 

All that is left is to create the function pointers.

Let's start by defining a struct for the function pointers, that loads the
functions via some sort of loader function, that uses the "name" to return a
pointer.

```rust
mod handmade_gl {
    struct FnPtr {
        //the actual function to call
        ptr: *const core::ffi::c_void,
        //symbol of the function
        name: &'static str,
    }

    impl FnPtr {
        pub const fn new(name: &'static str) -> Self {
            Self {
                ptr: core::ptr::null(),
                name,
            }
        }

        // TODO: error handling :P
        pub unsafe fn load(
            &mut self,
            loader: impl Fn(&str) -> *const core::ffi::c_void,
        ) {
            self.ptr = loader(self.name);
        }
    }
}
```

Now let's define all the functions. They are static mutables, but it's also
possible to put them into a struct or organize them differently. 

Also, we will define a function that will load all the function pointers we need.

```rust
mod handmade_gl {
    mod fn_ptrs {
        pub(super) static mut CLEAR: super::FnPtr = super::FnPtr::new("glClear");
        pub(super) static mut CLEAR_COLOR: super::FnPtr = super::FnPtr::new("glClearColor");
    }


    pub fn load_with(loader: impl Fn(&str) -> *const core::ffi::c_void) {
        unsafe {
            fn_ptrs::CLEAR.load(&loader);
            fn_ptrs::CLEAR_COLOR.load(&loader);
        }
    }
}
```
Now it should be clear what that weird proc_address from before was all about:
The proc_address is just a function that looks up the symbol that we pass to it. 
It's pretty much (like) taking a loaded dynamic library(.dll) and then getting the addresses of the functions by looking them
up via their name.

Finally, we need to "map" the pointers with to the concrete function types.
What good is a pointer if we can't pass arguments to it?

To do this, we just define the gl functions one last time, "cast" the pointers
into the concrete function type with a transmute, and then invoke them with arguments.

```rust
    pub unsafe fn Clear(bits: GLbitfield) {
        //"casts" the function pointer inside the CLEAR instance into the
        //actual Clear type and invokes it
        unsafe {
            core::mem::transmute::<_, ClearType>(fn_ptrs::CLEAR.ptr)(bits);
        }
    }

    pub unsafe fn ClearColor(
        red: types::GLfloat,
        green: types::GLfloat,
        blue: types::GLfloat,
        alpha: types::GLfloat,
    ) {
        //"casts" the function pointer inside the CLEAR_COLOR instance into the
        //actual ClearColor type and invokes it
        unsafe {
            core::mem::transmute::<_, ClearColorType>(fn_ptrs::CLEAR_COLOR.ptr)(
                red, green, blue, alpha,
            );
        }
    }
```

There is clearly an opportunity to write a `macro_rules!` that deals with all
the boilerplate. However, before going further into this madness, let's compare
the results with the initial approach at the top.

Let's not forget to replace the `gl::` in our `main fn` with our `handmade_gl`
module.

### Results

These are the results after re-building and running the example multiple times
and taking the averages, running --release with:

```toml
[profile.release]
strip = "symbols"
lto = true 
codegen-units = 1 
opt-level = "s"
```
(and bin size running once without any of these flags)

| Approach | Build time (clean / incremental) | Binary Size with/without stripping | load_with() time |
|---|---|---|---| 
| gl generator |  16.47s / 0.86s | 5.2MB / 755kb | 1.6ms |
| "handmade" | 14.69s / 0.86s | 4.9MB / 639kb | 7.6µs |

So, the huge caveat is that we only loaded a handful of symbols. 
Meanwhile, `gl_generator` loaded "everything". There is barely any difference and adding more and more functions will reduce
the gains further. 

Given that the differences are largely negligible in the grand scheme of things, especially when we are going to load
game assets later anyway, which makes the few 100kb a drop in the bucket, the whole endeavor nothing more than a
learning experience. There are also even more things to consider, like fallback
functions and extensions.

Granted, having to define the function pointers manually like this allows us to
add logs/traces add additional validation layers, ideally behind some feature flag. It is
possible to validate the values in many cases, like glClear accepting a
GLbitfield, but only accepting the bitwise OR of any of the three clear constants
we defined above, would make this an ideal candidate for manual validation.
Also, before the debug callback function in OpenGL was widely
supported, this used to be a great opportunity to insert glError polling
after each call(which is also possible with `gl_generator` settings). 



Final code:
```rust 
use raw_gl_context::GlContext;

use winit::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
};

pub mod handmade_gl {
    pub mod types {
        use core::ffi;

        pub type GLbitfield = ffi::c_uint;
        pub type GLfloat = ffi::c_float;
    }

    pub const COLOR_BUFFER_BIT: GLbitfield = 0x00004000;
    pub const DEPTH_BUFFER_BIT: GLbitfield = 0x00000100;
    pub const STENCIL_BUFFER_BIT: GLbitfield = 0x00000400;

    type Clear = unsafe extern "system" fn(mask: types::GLbitfield);
    type ClearColor = unsafe extern "system" fn(
        red: types::GLfloat,
        green: types::GLfloat,
        blue: types::GLfloat,
        alpha: types::GLfloat,
    );

    struct FnPtr {
        ptr: *const core::ffi::c_void,
        name: &'static str,
    }

    impl FnPtr {
        pub const fn new(name: &'static str) -> Self {
            Self {
                ptr: core::ptr::null(),
                name,
            }
        }

        pub unsafe fn load(
            &mut self,
            loader: impl Fn(&str) -> *const core::ffi::c_void,
        ) -> Option<()> {
            self.ptr = loader(self.name);
            self.ptr.is_null().then_some(())
        }
    }

    mod fn_ptrs {
        pub(super) static mut CLEAR: super::FnPtr = super::FnPtr::new("glClear");
        pub(super) static mut CLEAR_COLOR: super::FnPtr = super::FnPtr::new("glClearColor");
    }

    pub unsafe fn Clear(bits: GLbitfield) {
        unsafe {
            core::mem::transmute::<_, Clear>(fn_ptrs::CLEAR.ptr)(bits);
        }
    }

    pub unsafe fn ClearColor(
        red: types::GLfloat,
        green: types::GLfloat,
        blue: types::GLfloat,
        alpha: types::GLfloat,
    ) {
        unsafe {
            core::mem::transmute::<_, ClearColor>(fn_ptrs::CLEAR_COLOR.ptr)(red, green, blue, alpha);
        }
    }

    pub fn load_with(proc_address: impl Fn(&str) -> *const core::ffi::c_void) {
        unsafe {
            fn_ptrs::CLEAR.load(&proc_address);
            fn_ptrs::CLEAR_COLOR.load(&proc_address);
        }
    }
}

fn main() {
    let event_loop = EventLoop::new();
    let window = WindowBuilder::new().build(&event_loop).unwrap();

    //create a context from the existing winit window
    let context = GlContext::create(
        &window,
        raw_gl_context::GlConfig {
            alpha_bits: 0,
            ..Default::default()
        },
    )
    .unwrap();
    context.make_current();

    handmade_gl::load_with(|symbol| context.get_proc_address(symbol));

    unsafe {
       handmade_gl::ClearColor(1.0, 0.0, 0.0, 1.0);
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
                   handmade_gl::Clear(gl::COLOR_BUFFER_BIT);
                }
                context.swap_buffers();
            }
            _ => {}
        }
    });
}
```

