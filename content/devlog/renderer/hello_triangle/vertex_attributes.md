+++
title = "Vertex Attributes"
description = "We are going to tell OpenGL how to interpret our buffer data."
date = 2022-12-04
weight = 2
draft = false
template="book.html"

[taxonomies]
categories = ["Devlog"]
tags = ["OpenGL", "Context", "Renderer"]
[extra]
toc = true
keywords = "OpenGL, Vertex Array, Vertex Array Object, VAO"
+++

Now that we filled our buffers, we need to tell OpenGL how to
interpret the data. We do this with Vertex Array Objects(VAO's).

VAO's basically explain which vertex attribute [^1] uses data
from what buffer and how the data needs to be interpreted. In short: It makes
the per-vertex input variables in our shader read the data from our buffers.

To illustrate, a VAO tells the graphics device for example:
"Vertex Attribute 0 is in Buffer 1, begins at position BYTE_OFFSET, it has 3 components(x, y, z) of the type F32, and the offset between each vertex is STRIDE_BYTES".

This allows us to store the data in the buffers however we want. We could shove
it all into a single buffer, defining one vertex at a time `[position0, color0,
normal0, position1, color1, normal1, ..]`, which is also known as interleaving, group them
per attribute `[position0, position1, .., color0, color1, .., normal0, normal1, ..]`, or have
the attributes in any other order/position in one or more buffer objects. That
said, it is not possible to take a single attribute(like the positions) and split
it into multiple buffers. Generally, interleaving is better than
non-interleaving, because the GPU can just take all attributes that belong to
the vertex it wants to draw, by visiting a single memory location(since they are next to
each other), rather than going to multiple different ones, taking one at a time.
However, interleaving is often more effort and requires manually processing the
vertex data to store them in a wanted format, which is also why many engines
convert 3D-Assets into a custom format.

It is important to note that there is no direct relationship between models/meshes and VAOs. The VAO just stores the sources of our vertex streams and their layouts.
We could have all vertices from all our models/meshes inside a single buffer and use the same VAO for all of them.
In that case we just need to pass to our future draw call the index of the first vertex of the mesh we want to draw, in addition to the number of vertices that
belong to that mesh, but more about that later.

There can only be one active VAO at a time.
In compatibility profiles, there is a default VAO(with the name "0"), while in core
profiles, binding the VAO 0 will tell our context that we have no VAO currently
bound at all.

> [^1] Vertex Attributes are per-Vertex data. Things like the position, color, texture coordinate and normals of a vertex

So, as before, we start with a function in our playground.rs 

```rust
//creates the vertex array object
fn create_vao() -> gl::types::GLuint {
}
```
Creating the actual object is almost exactly as creating the buffer objects, plural
and all, with the difference that we call `glGenVertexArrays` instead of `glGenBuffers`.
We also bind the vao right away, since we need it shortly after:
```rust 
    let mut vao = 0;
    unsafe {
        gl::GenVertexArrays(1, &mut vao);
        gl::BindVertexArray(vao);
    }
    vao
```

And again, we are technically supposed to delete the objects manually with
`glDeleteVertexArrays`.

Now we only need to describe the data we laid out in our buffer.

There are two ways to do this, the old "modern" way and the new way since we are
using OpenGL 4.3.


## The Old "Modern" Way
Let's start with the old way with [glVertexAttribPointer](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glVertexAttribPointer.xhtml), as it is common since OpenGL 3.3.
There are 3 different functions that convert our data into different types. 
`glVertexAttribPointer` is the one we are mainly going to use, which
produces floating point values, `glVertexAttribIPointer` generates integers and
`glVertexAttribLPointer` creates doubles. All of them can produce the values as
scalars(single value) or vectors[^2]. We will just
look at `glVertexAttribPointer` for now, that is the float variant of these
functions.

> [^2] To clarify, these have nothing to do with the Vec data structure.    
> They are mathematical vectors. A 2 dimensional vector, or in short vec2 has an X and a Y value, while a vec3 also has an additional Z and a vec4 also has an additional W value.

Just to recall, this is our buffer data: 
```rust 
#[rustfmt::skip]  //skip the default formatting to make it cleaner
const QUAD_VERTICES: [f32; 3 * 4] = [
//     X,    Y,   Z    Position
    -0.9, -0.9, 0.0, // bottom left
    -0.9,  0.9, 0.0, // top left
     0.9,  0.9, 0.0, // top right
     0.9, -0.9, 0.0, // bottom right
];
```
So we need to tell OpenGL that our previously created buffer has the data for
the positions, which are interleaved(unintentionally, because we just have positions
for now), start at byte 0, and each position is made out
of 3 F32 components. Additionally, we need to bind the attribute to an index,
so that the shader will later be able to access them.

The first parameter for the `glVertexAttribPointer`, is the index, which has a non-saying description: 
> Specifies the index of the generic vertex attribute to be modified.

Generally when working with new OpenGL functions, I skim through the error-section at
the bottom of the [docs](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glVertexAttribPointer.xhtml)(more about error handling later).

```
GL_INVALID_VALUE is generated if index is greater than or equal to GL_MAX_VERTEX_ATTRIBS.    
GL_INVALID_VALUE is generated if size is not 1, 2, 3, 4 or (for glVertexAttribPointer), GL_BGRA.   
GL_INVALID_ENUM is generated if type is not an accepted value.   
GL_INVALID_VALUE is generated if stride is negative.  
GL_INVALID_OPERATION is generated if size is GL_BGRA and type is not GL_UNSIGNED_BYTE, GL_INT_2_10_10_10_REV or GL_UNSIGNED_INT_2_10_10_10_REV.  
GL_INVALID_OPERATION is generated if type is GL_INT_2_10_10_10_REV or GL_UNSIGNED_INT_2_10_10_10_REV and size is not 4 or GL_BGRA.  
GL_INVALID_OPERATION is generated if type is GL_UNSIGNED_INT_10F_11F_11F_REV and size is not 3.  
GL_INVALID_OPERATION is generated by glVertexAttribPointer if size is GL_BGRA and normalized is GL_FALSE.  
GL_INVALID_OPERATION is generated if zero is bound to the GL_ARRAY_BUFFER buffer object binding point and the pointer argument is not NULL.  
```

We don't need to understand most of them, but just by skimming we get a sense of
the allowed values for the arguments. We will immediately find an important piece
of information:
> GL_INVALID_VALUE is generated if index is greater than or equal to GL_MAX_VERTEX_ATTRIBS.    

In other words, our index must be less than GL_MAX_VERTEX_ATTRIBS.
What is this magic constant? Well, it's hardware dependent. Generally, it is 16
on most machines, but to check for ourselves, we can look at the associated gets
at the bottom of the doc, which leads us to the docs with the different get functions.
Which `Get` we use doesn't matter, but looking at the description for our
constant, we will find: 

> GL_MAX_VERTEX_ATTRIBS  
data returns one value, the maximum number of 4-component generic vertex attributes accessible to a vertex shader.  
The value must be at least 16.

So we know that 16 is a safe assumption. To check for our hardware we can inspect
the value ourself: 

```rust
let mut max_attributes = 0;
gl::GetIntegerv(gl::MAX_VERTEX_ATTRIBS, &mut max_attributes;
println!("max_attributes}") //prints 16
```

Which means that we can only have 16 different vertex attributes, which then
occupy the locations 0 - 15. 

The next parameter is the confusingly named `size`. It has nothing to do with
size, but is just the number of components for a specific attribute. Our position
has an X, Y and Z value, so it has 3 components(it's a vec3) to describe the position of a
single vertex. It could be 1, 2, 3, 4 or a special constant GL_BGRA(only
relevant for Direct3D compatibility).

The type will usually be floating point, which is also the case for our position.
However, there are more types available like basic types like `GL_BYTE`, `GL_INT`, to more esoteric ones like `GL_INT_2_10_10_10_REV`. The latter is relevant for attributes like normals, which don't need the whole bit depth available, but
could store their components in a single 32 bit value(10 bits each for X, Y and Z and 2
bits for a potential W) . However,
`glVertexAttribPointer` will always convert integers to floats! We would need to
use `glVertexAttribIPointer` to get integers.

The normalized parameter is only relevant if our vertex input data is made out of 
integers. If it's false, it will convert them via a c-style cast to floats, taking the values as they are, "putting a .0 behind them".
If it's true, it will map signed values from [-max, max] to [-1.0, 1.0] and
unsigned ones from [0, max] to [0.0, 1.0], with the power of math. The actual
math behind it depends on the OpenGL version. So as an example, it will
turn the byte 128 into either 128.0(GL_FALSE), or 0.5(GL_TRUE).
For floating points(as with our position), it is irrelevant.

The stride value describes the bytes between each subsequent value of the
attribute. So basically the bytes between position_for_vertex_0 and
position_for_vertex_1, .. It is a constant, so the stride must be the same for
all values for an attribute in the vertex stream. Since we only have our position for now, we are
running into a special case where the data is tightly packed, but also
interleaved. So to make things simple, we just use a stride of 0. A stride of 0
means that OpenGL should figure it out(by making the stride essentially the size
of the byte size of a single vertex attribute value).

Finally, we come to the "pointer" attribute, which is an awkward pointer because
this function is being reused. It was used to pass the pointer to vertex data
before vertex buffers were invented, but in "modern" OpenGL it's just a relative byte offset to
the first value of our attribute inside that buffer. Our buffer starts with
our positions, so this is just another 0, albeit an awkwardly cast one, or just
a null pointer.



So in code, it means: 
```rust
gl::VertexAttribPointer(0, 3, gl::FLOAT, gl::FALSE, 0, std::ptr::null());
// alternative with "proper" casting
gl::VertexAttribPointer(0, 3, gl::FLOAT, gl::FALSE, 0, (0 as *const usize).cast());  
```

However, to finally use it, we need to enable it.
```rust
gl::EnableVertexAttribArray(0);
```

This is necessary to make the attribute actually being recognized by the shader
later. Both of these functions are state associated with our bound VAO.

One final step is missing.
How does the VAO know which buffer to use? 

We need to bind a buffer as `GL_ARRAY_BUFFER` target, before we call glVertexAttribPointer!
Afterwards, we can unbind the VBO or bind a completely different one, or keep it
bound, the VAO won't change the binding, unless we also call `glVertexAttribPointer`
again.  

So it makes sense to pass the vbo to our function to bind it(even if it is already
bound since we never unbound it), just to avoid potential issues with the global
state machine(we don't want to accidentally bind the wrong VBO).

So the final function should look like this:

```rust
fn create_vao(vbo: gl::types::GLuint) -> gl::types::GLuint {
    let mut vao = 0;
    unsafe {
        gl::GenVertexArrays(1, &mut vao);
        gl::BindVertexArray(vao);
        gl::BindBuffer(gl::ARRAY_BUFFER, vbo);            
        gl::VertexAttribPointer(0, 3, gl::FLOAT, gl::FALSE, 0, std::ptr::null());
        gl::EnableVertexAttribArray(0);
    }

    vao
}
```

Also, let us not forget to actually call this function with our vbo: 
```rust 
//..
    let vbo = create_vbo(&QUAD_VERTICES);
    let vao = create_vao(vbo);
//..
```

That's it! Also, notice how we never specified the vertex count nor byte size
for the attribute. They only have a start in the buffer. Only when we draw later
will we tell OpenGL how many vertices we want to draw.

## The New Way 
As we noticed, OpenGL relies on the global state machine to make the association
with a buffer and a vertex attribute. Not just that, but the attribute
definition, or the vertex layout as a whole, is directly tied to the storage. 
If we have two different buffers for two different meshes, with the same
layout(just the positions packed together), we would need to do the whole
`glVertexAttribPointer` nonsense again. Surely, there must be a way to just define the layout separately from the actual data, right? 
Well, enter OpenGL 4.3.

OpenGL 4.3 introduced a new set of functions that split the data source from
the attribute layout. Instead of the `glVertexAttribPointer`, we call
[glVertexAttribFormat](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glVertexAttribFormat.xhtml) to describe the attribute. This function comes with the same set of friends, the "normal", the L and the I variants, just as its predecessor.

The parameters are the same as before. We specify the attribute index, the
components, the type and whether it is normalized. The only difference is that
the stride parameter is gone and lo and behold, we don't need to cast for the
relative offset anymore. It takes in a `GLuint` instead of a weird pointer.
Also, it is not a relative offset into the buffer anymore, but a relative offset
to the start of the attribute itself.

Simple example: 
Imagine we have a jumbo-buffer containing the vertex attributes for a "Mr.
Potatohead" at the start of the buffer, followed by lots of nonsense, then the
data for a magnificent "Robert E. O. Speedwagon"-model. Both use the same
layout, that is the first attribute in their layout is the position, then their
normals, then their color, etc.. So the `relativeoffset` for their position is both 0, for their normals is the byte size of the position attribute(s), and their color is the byte size of the positions and the normals. 
Where the actual Potatohead and the Speedwagon start is done later.

```rust 
//old
//gl::VertexAttribPointer(0, 3, gl::FLOAT, gl::FALSE, 0, std::ptr::null());
//new
gl::VertexAttribFormat(0, 3, gl::FLOAT, gl::FALSE, 0);
```

Unfortunately the documentation is a bit wonky, since it describes relativeoffset as: 

> The distance between elements within the buffer.

It is more correct later in the docs: 

> relativeoffset is the offset, measured in basic machine units of the first element relative to the start of the vertex buffer binding this attribute fetches from.

OpenGL is weird.

Now, to specify the Buffer, we need to call [glBindVertexBuffer](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBindVertexBuffer.xhtml).

Important to notice is that `bindingindex` has nothing to do with our attribute
index. Instead, we will bind our buffer to this index.

Then we just specify our buffer(no more need to bind it anymore!).
Then the offset to the first element in the buffer for out layout. 
This is what points to the first attribute of Potatohead or Speedwagon in our
jumbo buffer, and we still don't need to use a pointer, isn't that amazing?

Finally, there is the stride again, keeping its meaning. It tells how many
bytes are between the different vertices. That is the bytes between Position0
and Position1, and so on.
The great thing about the stride being decoupled from the attribute format, is that we
set it on a per-buffer base. Some buffers store them interleaved, others don't,
our format doesn't care how it's stored, it is something that the buffer binding
should take care of.

One bummer is that a stride of 0 doesn't mean packed anymore, but we are now
required to pass it to the function ourselves. One benefit is that a stride of
zero means really zero, so we could provide "default values" for layouts that
don't provide the attributes. They wouldn't "advance" anymore.

So for us it means: 
```rust 
//the distance to the next vertex is always size of a single position(3 * size of f32).
gl::BindVertexBuffer(0, vbo, 0, std::mem::size_of::<f32>() as i32 * 3); 
```

The only step missing is to tell OpenGL which `bindingindex` belongs to which
attribute. So we just call
[glVertexAttribBinding](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glVertexAttribBinding.xhtml).

```rust 
gl::VertexAttribBinding(0, 0); //attribute 0 uses the buffer at bindingindex 0
```

Keep in mind that we still need to enable the attribute, just like before.
So the function using the new way looks like this: 

```rust
fn create_vao(vbo: gl::types::GLuint) -> gl::types::GLuint {
    let mut vao = 0;

    unsafe {
        gl::GenVertexArrays(1, &mut vao);
        gl::BindVertexArray(vao);

        gl::EnableVertexAttribArray(0);
        gl::VertexAttribFormat(0, 3, gl::FLOAT, gl::FALSE, 0);
        gl::VertexAttribBinding(0, 0); 

        gl::BindVertexBuffer(0, vbo, 0, std::mem::size_of::<f32>() as i32 * 3); 
    }

    vao
}
```

I know, this looks boring with just one attribute. It will all unfold once we
add more attributes later, but let us first draw this dang triangle by executing
funny programs on our GPU that are called shaders :)
