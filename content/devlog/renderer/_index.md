+++
sort_by = "weight"
transparent = true
template = "book.html"
insert_anchor_links="heading"
title = "Writing the Renderer"
[extra]
toc = true
+++

# Writing the Renderer

Before we are able to write the game, we need the ability to draw something on the
screen. 

The renderer shouldn't do anything fancy.
At the very least it should be able to display 3D models(including animations),
allow some post processing effects via fullscreen shaders, using a deferred
rendering path, do some basic UI stuff, and work for Desktop and WebGL. 

Advanced things like raytracing, clustered/tiles rendering paths or all the ML-assisted
render-features in modern games are beyond the scope of this little renderer and
probably any game I am going to write in the next decade.

Everything else will just be added as we need for the games at hand.

I don't have a strict plan yet, but the goal is to split the Renderer at least
into these packages: 

### Context
  Hardware abstraction layer that deals with the unsafe graphics
  API's like OpenGL, Vulkan, etc. It's basically "dumb" and just exposes
  safe function to create and manipulate the objects that are living on the graphics
  device and the actual draw calls. Things like vertex buffers, shaders and textures.

### Resources 
  Loads and creates different kinds of resources. Loading images(that are turned
  into textures), models of different formats, reloading on change, etc.

### Core
  Data structures shared by the other packages. Things like Color,
  Materials, Camera and common math structs.

### Graphics 
  The main renderer, using the other crates to offer high level functionality. 
  It deals with the render-paths, draw call sorting, etc. It's the only thing
  directly exposed to the game and exports the API of the others where
  necessary.

So yeah, let's start with the [project structure](./project_setup).
