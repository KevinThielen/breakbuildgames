+++
title = "Buffers"
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

First and foremost, we need buffers. 
Buffers are "dumb" blobs of data living in the memory of the graphics
device<sup>1</sup>. 
They can contain vertices, indices(kinda like vertex ID's) but also "shader variables"(uniforms) and
more. In OpenGL, they are called buffer objects. 
The most basic thing we need are so called "vertex buffer objects". Buffer
objects that contain the vertex data.

> <sup>1</sup> That said, the driver is free to use the memory however it wants, and can even allocate 
> on the stack and heap, so there is no guarantee where the memory will be ultimately allocated

So let us create a simple function in our `playground.rs` to create our vertex
buffer. This is our "safe abstraction" over the unsafe FFI calls for now. 

```rust 
//create our vertex buffer object
fn create_vbo() -> gl::types::GLuint {
}
```

In OpenGL, objects are referenced via their "names". It's just an unsigned id.
To create a buffer, we just need to call `gl::GenBuffers`. In OpenGL, most
create-functions are pluralized and accept arrays. Also, because of C(and cross
platform support in general), most
functions don't have a direct return value(especially the ones that can return arrays),
to put the responsibility of memory management to the caller.

So to use `GenBuffers` we need to pass something that can contain the GLuints
for the count we pass to it. 

Generally, generating as many buffers as we need at once is theoretically better
for the performance, but in reality, you are unlikely to measure a difference on
a modern machine even when generating thousands of buffers.
So for now, we just generate a single buffer on demand.

We are also technically required to delete the buffer at some point, by calling
`gl::DeleteBuffers`. However, we feel young and invincible, and blindly
trust our OS to clean up this mess once the context is destroyed. This is just
the playground to explore OpenGL and will be refactored into proper abstractions
once the triangle is there, so we just take a mental note.

```rust 
fn create_vbo() -> gl::types::GLuint {
    let mut buffer = 0;

    unsafe {
        gl::GenBuffers(1, &mut buffer);
    }

    buffer
}
```

A single buffer without any data is frankly quite boring.
So we need to shove some data into it. 
This is now where the biggest difference between the previously mentioned `DSA`
of OpenGL >= 4.5 and the traditional "modern" OpenGL lies: We need to bind our
buffer to some global state machine before we can use it.

If we look at [gl::BindBuffer](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBindBuffer.xhtml), we will notice that we need to specify a target and the buffer. Since our buffer is going to hold our vertex data, we need to use the nonsensical named `GL_ARRAY_BUFFER` target. The other targets are not relevant for now.

```rust
unsafe {
    gl::GenBuffers(1, &mut buffer);
    //bind the buffer to our global context to the ARRAY_BUFFER "slot"
    gl::BindBuffer(gl::ARRAY_BUFFER, &buffer);
}
```

Now, to actually write data for it, we need to call
[glBufferData](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBufferData.xhtml).

Notice how it doesn't require our buffer as parameter, but instead relies in the
"target". This is the global state machine at work. Also, the same page in the
references shows us mockingly the bindless "glNamedBufferData" version, just to
shatter our dreams at the bottom with the Version Support table. 

Anyway, the following two parameters are just dynamic C-Array style arguments.
One is a size of the data as a whole and the other is the pointer to the first
element of our data blob. Keep in mind that the data needs to be either a slice
of a C compatible type or a struct with `repr(C)`.

Finally, we need to pass in some Usage flags. While modern drivers are able to make
the decision based on actual usage, it is still recommended to follow the
spec. There is no restriction or guarantee about the actual usage, it is merely
a hint.

The usage flag is made out of 2 parts, the access and the frequency.

For the access there are the following three values: 
- **DRAW**: The user can only write data to the buffer, but they can't read it.
- **READ**: The user can only read the data of the buffer, but they can't write to it.
- **COPY**: The user can't neither read nor write data to the buffer. Only OpenGL can. 

Some nifty reader might notice that there is no case for "The user can upload
data to the buffer and the user can read it". The reason is that it would result
in horrible performance. The alternative is to create two buffers, one for
reading and one for writing and then let OpenGL copy the content from to the
other.

The frequency is the other half of the flags, which has also three values: 
- **STATIC**: Set the data only once
- **DYNAMIC**: Set/Change the data occasionally
- **STREAM**: Set/Change the data constantly

OpenGL defines constants for all combinations of these flags.
For our buffer, we want to set it but never read it(`DRAW`), and we only want to
set it once(`STATIC`). So our usage argument is the constant `STATIC_DRAW`.  
If we were to have a terrain that changes occasionally based on a texture for
example, we would use `DYNAMIC_COPY` to occasionally let OpenGL take another
OpenGL object provide the data for the buffer. Though, the difference between
DYNAMIC and STREAM is not clear. 

But again, these are only hints, so nothing prevents us from using STATIC_COPY,
while we set the data every frame manually, except potentially horrible
performance.

For our vbo, we set the data, from a generic slice.
Instead of caring about the usage hints, we just focus on the STATIC_DRAW, since
we only want to draw a triangle.

> **Note**: We don't need to care for the lifetime of
> our data, because it is uploaded to our buffer right away. 
> Generally, OpenGL doesn't block, but just queues them up until we 
> call glFlush, which then executes whatever is in the queue asynchronously.
> Our context calls implicitly glFinish on swap_buffers, which is the same as
> glFlush, with the addition that it blocks until the commands are actually
> finished

```rust 
//take in any data
//WARNING: T must conform the C layout
fn create_vbo<T>(data: &[T]) -> gl::types::GLuint {
    let mut buffer = 0;

    //the size of our blob is the size of a single element(T) * the counts of T in our slice
    let data_size = std::mem::size_of::<T>() * data.len();

    unsafe {
        gl::GenBuffers(1, &mut buffer);
        gl::BindBuffer(gl::ARRAY_BUFFER, buffer);

        gl::BufferData(
            gl::ARRAY_BUFFER,
            data_size.try_into().unwrap(), //we need to cast usize to "isize", panicking is fine in our playground
            data.as_ptr().cast(),          //the pointer to our first element in our slice
            gl::STATIC_DRAW,
        )
    }

    buffer
}
```

This function is in theory not safe anymore. Our buffer data requires to conform
the C layout, alignment and all. This is not a problem for the native types,
which are tightly packed, but more for custom struct and similar, which require
`repr(C)`. To make it safe later, we will either use a trait bound or only
accept certain slice(e.g. u8 slices) of data.

Now that we can create out buffer, we call this function with the data of 4
vertices in our main, right before our `event_loop.run`.

```rust 
#[rustfmt::skip]  //skip the default formatting to make it cleaner
const QUAD_VERTICES: [f32; 3 * 4] = [
//     X,    Y,   Z    Position
    -0.9, -0.9, 0.0, // bottom left
    -0.9,  0.9, 0.0, // top left
     0.9,  0.9, 0.0, // top right
     0.9, -0.9, 0.0, // bottom right
];

let vbo = create_vbo(&QUAD_VERTICES);
```
We created our vertex buffer with 4 vertices, each with a single position in 3D
space. 

The OpenGL coordinate system goes from -1.0 to 1.0 on all three axes. These are
the normalized device coordinates, or in short NDC. The origin
is the center of the window, bottom left is -1.0 on the X and Y axes, while top
right is 1.0 for both. Try to grab your screen and your arm points to the
negative Z axis, which means the positive Z axis is going towards us.
The whole space discussion is arguably boring and I am not nearly enough trained
in math to explain this correctly and there are already pretty nifty resources, so look at
[learnopengl.com](https://learnopengl.com/Getting-started/Coordinate-Systems) or DuckDuckGo for the actual details.

The important thing is that our vertices sit at distance of 0.1 on the X and Y axes away from the border of our window.
If we were to set the Z value to 1.0, our vertices would be as close to the
screen as possible before leaving it and a Z below -1.0 would mean they are
"behind" our screen.

Why 4 vertices when we want to draw a triangle?
W
