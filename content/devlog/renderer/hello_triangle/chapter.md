+++
title = "Hello Triangle"
description = "We are finally going to start with OpenGL, to draw a simple triangle on the screen"
date = 2022-12-03
draft = false
weight = 0
render = false

[taxonomies]
categories = ["Devlog"]
tags = ["OpenGL", "Context", "Renderer"]

[extra]
chapter = true
toc = true
keywords = "Triangle, OpenGL"
+++

What better way to start than the "Hello World" of graphics programming, 
drawing a triangle?

We are going to use our [playground](https://github.com/KevinThielen/cac_graphics/tree/1f206abc09f326569f3fcf21feddde8fca70c8d0) from the [context chapter](./../the_context/opengl) as
base.


## Drawing Stuff in OpenGL

As mentioned before, I won't go into all the details when it comes to OpenGL,
you are better served with the amazing [learnopengl.com](https://learnopengl.com/). This little
series will go more into the "practical" use and the implementation, while I will just
give a short TL;DR explanation at best.

There are a few things we need to draw in modern OpenGL:

- Buffer objects
- Vertex Attributes
- Shaders

So let's dive into the buffer objects first.



