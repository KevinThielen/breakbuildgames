+++
title = "Hello Triangle V2"
description = "We got our quick and dirty triangle, so let' do it again, but properly this time!"
date = 2022-12-07
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

We just yeeted all of our code into our playground, didn't care
about error handling, validating any input, nor care about cleaning up
our resources and we have some potentially unsound holes in our design.
That we also have gl calls in our main loop is just the tip of the iceberg.

However, this is how I generally like to tackle new things in programming: Explore the
problem in some throwaway program, then once I have a better idea of what I
need, I start thinking about proper data organization and abstractions.
I feel like Rust's strict ownership rules really encourage this kind of
thinking, or you end up [rewriting your code later](/blog/my-year-with-rust#rewrite-it-in-rust).

Even when coming up with the API, the playground can serve as a way to get a feel for the ergonomics and usability. What good is an abstraction when we have to store or refer to our context as `RenderContext<Context=OpenGL, Texture=GLTexture, Buffer=GLBuffer, ..>` or store resources as concrete implementation dependent types, like a `OpenGLBuffer`, so that swapping our backend will force us to change the fields in dozens of structs, or we end up with heavy generics all over the place? 

So let's go back to our triangle, and do it properly this time!
