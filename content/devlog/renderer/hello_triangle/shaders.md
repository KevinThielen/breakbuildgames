+++
title = "Shaders"
description = "Which OpenGL version to target and how"
date = 2022-12-10
weight = 3
draft = false
template="book.html"

[taxonomies]
categories = ["Devlog"]
tags = ["GameDev", "OpenGL", "Context", "Renderer"]
[extra]
toc = true
keywords = "Game Development, GameDev, OpenGL, Graphics Programming"
+++

Finally, we need shaders. Shaders are programs that run on the GPU, written in
a shader language(OpenGL Shading Language, or in short GLSL, in OpenGL).
They are the main difference between legacy and modern OpenGL. They make the
whole pipeline programmable.

There are multiple kinds of shaders, but we only care about vertex and fragment shaders for now. 

Simply put, vertex shaders are executed for every single vertex and the purpose
is to transform them from 3D space into "Clip Space", often by applying all
kind of math operations to emulate cameras, play animations and simulate
movement. 

The input-data, the vertex attributes, are what we mapped to the different input
locations with the VAO in the previous chapter, so the VAO needs to match the shader(at least provide
the data for the locations that the shader requires). Attributes that are not
enabled by the VAO or not used by the shader are ignored.

After the vertices went through the vertex shader, they go through some
automatic post-processing. In short, they are bundled into primitives, usually triangles, 
and are discarded if the primitive is not at least partially inside the clip space. If a
primitive is partially "visible", it is transformed into one or multiple primitive(s) that
magically fit into the space, discarding the parts that are outside. OpenGL also
does some coordinate conversion into screen space, some other tests.

Then the `Rasterizer` takes the surviving primitives and generates "fragments" for the "screen pixels that are covered by our
primitive". Fragments and pixels are basically the same for all intents and
purposes, and Direct3D uses the term pixels for fragments, but more accurately,
a fragment a potential pixel on the screen. So in OpenGL fragments are the input
and pixels are the output.

Then these fragments are processed by a fragment shader, which calculates one or more color and the depth value of the fragments, so that OpenGL then can use this information to do some additional tests(depth,
alpha, stencil<sup>1</sup> and other masks), blends them, etc., and finally
write the fragment to the output, like a pixel on the screen. 

This overview is by no means exhaustive, there more shaders and stages, but this
is just the TL;DR.

> <sup>1</sup> Stencil values are "custom values" used for some screen effects

## Shader Sources
So let's start with a simple vertex shader, to process the vertices that we
specified with our VAO.

As mentioned, OpenGL uses the OpenGL Shading Language for shaders, which
looks a lot like C-based languages. GLSL is out of the scope of this series, so
I cover the basics we need to write for our triangle and just refer to the amazing [Book of Shader](https://thebookofshaders.com/).

There are 3 parts in a shader.

First, there is the input.
Our vertex attributes, that we specified via our VAO.
The mapping happens either via a location attribute in the shader, or
manually via [glBindAttribLocation](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBindAttribLocation.xhtml).
If we do neither, OpenGL does it automatically and completely yolo, entirely
arbitrary. 
No matter whether we used a location attribute or not, we need to add the type qualifier `in` to signal that our variable is
an input variable, give it a type matching our attribute to some
extent<sup>1</sup>, and giving it any name we want.

Our position is a float made out of 3 components, which maps to a vec3 in GLSL.

So if we go with hard-coded locations, the first line of our shader is: 

```c 
layout(location = 0) in vec3 pos;
```
 

> <sup>1</sup> Providing data for a vec4, but defining it as a vec3 or even float is perfectly valid. The other way around works too, but we would get us default values for the additional components.


Then, we got outputs, which are defined as our inputs, just with a `out`
qualifier and without the location nonsense.

```c 
out vec4 fragment_color;
```

So our vertex shader will calculate or just pass a color to our fragment shader
later.

Finally, we have a C-like main function: 
```C 
void main()
{
    gl_Position = vec4(pos, 1.0);
}
```

gl_Position is a built-in output variable we can use, that passes our vertex
position to the next stage without doing anything fancy. It's weirdly a vec4,
because of math. Our 3D position has no perspective at all, but to make the math
simpler when we are going to use a "Camera" later, OpenGL does a
"perspective divide" as part of it's pipeline. It just divides the X, Y and Z
components separately by the fourth value, the W, to turn the "Clip Space" into the true NDC.
The higher the W, the closer are the values to the center of the screen, which allows the perspective distortion.
This explanation is overly simplistic, but a value of 1.0 just means the
position will be taken as is.


Let's talk about interpolation while we are doing this math dump.
Since or vertices are just "points", and the
vertex shader is executed only for vertices, the whole area of our
primitives(e.g. the triangles) needs to calculate their input values via
interpolating our vertex output. This sounds more complicated than it is. It
just means the closer a fragment is to a vertex, the closer it's own value is to
that vertex. The further away it goes, the closer it gets to another nearer
vertex. If Vertex A is Black and Vertex B is White, all fragments between them
would be grayish, getting darker closer to Vertex A and lighter closer to Vertex
B, like a happy little gradient.
Which is why we need to pass our attributes through the vertex shader, if we
want to make them available for our fragment shader later, which is common for
texture coordinates.

So let's calculate our fragment_color variable in our main:

```C 
    fragment_color = vec4(pos.x, pos.y, pos.x + pos.y, 1.0);
```

The calculation is arbitrary for now.

Also, because GLSL used to be updated which each OpenGL, we need to specify a
version at the top of our shader source code. Since OpenGL 3.3, it is pretty straight
forward. `#version {GL_MAJOR}.{GLMINOR}0`. so four our 4.3 version, it is
`#version 4.30` at the top of the shader, before anything else. 

We could write the whole source code for the shader in a separate file and then
load it during runtime as a file, include it via the `include_string!` macro, or we
could just define these 5 lines as a const.

For our playground project, I opted for the simple const: 
```rust 
const VS_SOURCE: &str = r##"
#version 430
layout(location = 0) in vec3 pos;

out vec4 fragment_color;

void main()
{
    fragment_color = vec4(pos.x, pos.y, pos.x + pos.y, 1.0);
    gl_Position = vec4(pos, 1.0);
}"##;
```

Now we need to do the same for our fragment shader. 
Instead of vertex attributes, we need to use the previously defined output
variables, just with the `in` this time. It is just our fragment_color in this
case. Also, we declare yet another output var, which is actually going to be the
color of our "pixel". We just pass it through. The layout is used for the output this time. Technically
it is not necessary, since a single output variable is assumed to be at
location 0, which is where the OpenGL spec assumes the color of our render target: 

```rust
const FS_SOURCE: &str = r##"
layout(location=0) out vec4 result;

in vec4 fragment_color;
void main()
{
    result = fragment_color;
}"##;
```

## Finally Creating The Bloody Shader Objects

The last step is actually compile the shaders and use them in our playground.

The creation is pretty straight forward, we call `glCreateShader` with the type
of shader we want to create, upload the shader code with `glShaderSource` and
finally compile them with `glCompileShader`.

So let us just create our shaders first.


```rust 
let vs = gl::CreateShader(gl::VERTEX_SHADER);
let fs = gl::CreateShader(gl::FRAGMENT_SHADER);
```

Now we just upload our source code to them - 
I "just", but turns out that the
[glShaderSource](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glShaderSource.xhtml) wants pointers of pointers..
What a lunatic.

It takes in a bunch of source code fragments and stitches them together. Which
is also while the byte length is also a pointer, since this is how arrays work
in C.

Luckily, we can skip the length when we are using null-terminated string. 
We could turn our shader code into raw byte literals with `b"SOURCE CODE"`,
make sure we add a \0 at the end and use the raw pointer to that if we like danger, and then call it a day.
However, I am a cowards and just bite the bullet of an additional allocation into a `CString` which fails
if our str can't be converted into a valid C-String.

```rust
//convert Rust Str to CString
let vert_source = CString::new(VS_SOURCE).unwrap();
let frag_source = CString::new(FS_SOURCE).unwrap();

gl::ShaderSource(vs, 1, [vert_source.as_ptr()].as_ptr(), std::ptr::null());
gl::ShaderSource(fs, 1, [frag_source.as_ptr()].as_ptr(), std::ptr::null());
```

However, now we really only need to compile them.

```rust
gl::CompileShader(vs);
gl::CompileShader(fs);
```

Keep in mind that here is where would check for compile errors. I defer this for
our refactoring, because I am just longing for this dang triangle at this point.

## The ShaderProgram

We got our shader objects, but there is a final object that we need to face, the
`ShaderProgram`. We see that our shaders are not really linked together. We
defined some input and output variables, but they are just two different
objects. The ShaderProgram is the one doing this link. Because of it, we can
create all kinds of different pipelines for our vertices, with all kind of
different shader effects, while reusing existing shader objects. A fragment
shader that outputs red pixels and a shader that outputs blue ones might
use the same vertex shader for example.

So all we need to do is create yet another object, attach our shaders to it with
the conveniently named `glAttachShader` function, link it and et voila, we are
done.

```rust
let program = gl::CreateProgram();
gl::AttachShader(program, vs);
gl::AttachShader(program, fs);
gl::LinkProgram(program);
```

Before the linking, we could manually set up attribute input locations in our
vertex shader, or the output locations of the fragment shader.

After linking we should check for errors yet again, but I will use the same
excuse as before.

However, after the linking, it is good practice to detach the shaders from the
program, so that they could properly deleted by someone who cleans up their own
mess. So the final result looks like this: 

```rust
fn create_shader() -> gl::types::GLuint {
    let program;
    unsafe {
        let vs = gl::CreateShader(gl::VERTEX_SHADER);
        let fs = gl::CreateShader(gl::FRAGMENT_SHADER);

        //convert Rust Str to CString
        let vert_source = CString::new(VS_SOURCE).unwrap();
        let frag_source = CString::new(FS_SOURCE).unwrap();

        gl::ShaderSource(vs, 1, [vert_source.as_ptr()].as_ptr(), std::ptr::null());
        gl::ShaderSource(fs, 1, [frag_source.as_ptr()].as_ptr(), std::ptr::null());

        gl::CompileShader(vs);
        gl::CompileShader(fs);

        program = gl::CreateProgram();
        gl::AttachShader(program, vs);
        gl::AttachShader(program, fs);
        gl::LinkProgram(program);
        gl::DetachShader(program, vs);
        gl::DetachShader(program, fs);

        //we are free to delete shaders,
        // since our program only contains the linked program
        gl::DeleteShader(vs);
        gl::DeleteShader(fs);
    }

    program
}
```

Don't forget to call our new function.


```rust
let shader_program = create_shader();
```

And that's it, now we can draw the triangle.. Finally..

## Hello Triangle 

All that is left is to bind the vao and the shader. We technically never unbound
the vao, but it is good practice to bind it in case some hoodlum touched
OpenGL's global state machine.


We do this right after the clear.

```rust 
unsafe {
    gl::UseProgram(shader_program);
    gl::BindVertexArray(vao);
}
```

And in our loop, right after the clear, we call [glDrawArrays](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawArrays.xhtml).
As the name suggests, it draws what we specified with our VAO.

The first parameter is the mode, the kind of primitive we want to use.
There are multiple kinda of primitives, but we are mostly interested in the
Triangle version. `GL_TRIANGLE` will take our first 3 vertices to make a
triangle out of them, then the next 3 and so on. `GL_TRIANGLE_STRIP` will take
the first 3 vertices as well, but each subsequent vertex will re-use the
previous two vertices to create a new triangle. 

The next two parameters are for the `first` and the `count`. 
As mentioned before, a VAO can be used for an arbitrary amount of meshes. 
With the `first` parameter we specify the index of the first index in our VAO,
that belongs to the mesh we wan't to draw. The `count` specifies how many
vertices belong to our mesh.

Long story short, we want: 

```rust
gl::DrawArrays(gl::TRIANGLES, 0, 3);
```
right before swapping the buffers.

There it is! Our magnificent triangle, our first geometry before we start
conquering the graphics world!

Change the `first` argument from a 0 to a 1, for a different triangle!
Instead of using the first 3 vertices in our buffer that stored 4, we are
drawing the last 3 now.

Change the `mode` from `GL_TRIANGLES` to `GL_TRIANGLE_STRIP`, for no visible
change! Because it only works differently starting with the 4th vertex. So let's set
`first` back to a 0, and the `count` to 4 and we are no longer bound to mere
triangles! I mean sure, we are drawing two overlapping triangles,
everything we will ever draw will be made of triangles, because that is the only way to draw things in OpenGL,
but still! (well, we will take a look at points and lines one day)

![hello_triangle](../img/context/hello_triangle.png)

---

Beautiful!.. is something else.. Sure, we are programmers, not artists, but even
we should have been able to come up with more aesthetically pleasing colors.. At least
something that doesn't repel us.

Anyway, this concludes our playground project. We know a bit more about OpenGL to
a point where we could try to make some proper abstractions, clean up our mess
and start with proper error checking and handling. 

Final playground.rs:
```rust 
//cac_graphics/context/examples/playground.rs
use std::ffi::CString;
use context::{gl43_core as gl, opengl};
use raw_gl_context::GlContext;
use winit::{
    dpi::LogicalSize,
    event::{Event, WindowEvent},
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
};

const VS_SOURCE: &str = r##"
#version 430
layout(location = 0) in vec3 pos;

out vec4 fragment_color;

void main()
{
    fragment_color = vec4(pos.x, pos.y, pos.x + pos.y, 1.0);
    gl_Position = vec4(pos, 1.0);
}"##;

const FS_SOURCE: &str = r##"
#version 430
layout(location=0) out vec4 result;

in vec4 fragment_color;
void main()
{
    result = fragment_color;
}"##;

fn create_shader() -> gl::types::GLuint {
    let program;
    unsafe {
        let vs = gl::CreateShader(gl::VERTEX_SHADER);
        let fs = gl::CreateShader(gl::FRAGMENT_SHADER);

        //convert Rust Str to CString
        let vert_source = CString::new(VS_SOURCE).unwrap();
        let frag_source = CString::new(FS_SOURCE).unwrap();

        gl::ShaderSource(vs, 1, [vert_source.as_ptr()].as_ptr(), std::ptr::null());
        gl::ShaderSource(fs, 1, [frag_source.as_ptr()].as_ptr(), std::ptr::null());

        gl::CompileShader(vs);
        gl::CompileShader(fs);

        program = gl::CreateProgram();
        gl::AttachShader(program, vs);
        gl::AttachShader(program, fs);
        gl::LinkProgram(program);
        gl::DetachShader(program, vs);
        gl::DetachShader(program, fs);
        gl::UseProgram(program);

        gl::DeleteShader(vs);
        gl::DeleteShader(fs);
    }

    program
}
fn create_vao(vbo: gl::types::GLuint) -> gl::types::GLuint {
    let mut vao = 0;

    unsafe {
        gl::GenVertexArrays(1, &mut vao);
        gl::BindVertexArray(vao);

        gl::EnableVertexAttribArray(0);
        gl::VertexAttribFormat(0, 3, gl::FLOAT, gl::FALSE, 0);
        gl::VertexAttribBinding(0, 0); //attribute 0 uses buffer binding 0

        gl::BindVertexBuffer(0, vbo, 0, std::mem::size_of::<f32>() as i32 * 3);
    }

    vao
}

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

fn main() {
    let event_loop = EventLoop::new();
    let window = WindowBuilder::new().build(&event_loop).unwrap();

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

    context.make_current();

    gl::load_with(|symbol| context.get_proc_address(symbol));

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

    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Poll;

        match event {
            Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                window_id,
            } if window_id == window.id() => *control_flow = ControlFlow::Exit,
            Event::MainEventsCleared => unsafe {
                gl::Clear(gl::COLOR_BUFFER_BIT);
                gl::DrawArrays(gl::TRIANGLE_STRIP, 0, 4);

                context.swap_buffers();
            },
            _ => {}
        }
    });
}
```
