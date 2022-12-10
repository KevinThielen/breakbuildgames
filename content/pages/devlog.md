+++
title = "Devlog"
path = "devlogs"
template = "pages.html"

[extra]
toc = true
+++

# Devlog 

If you want the full background, read the [motivation](./motivation/) behind all
of this. The short story is that I want to make my own games in my own engine,
using Rust, because it's fun.

The following chapters are me logging my process, thoughts and decisions during
this journey.

It's basically just me trying to solve my problems at hand, voicing my thoughts, and probably
me refactoring multiple times. Writing these down after the fact, with
the "perfect code" would kinda go against my intentions of showing a
transparent and genuine progress. The earlier chapters might feel closer to
"tutorials" though, because I am not starting at 0 with my knowledge, so I try
to give a TL;DR explanation where possible.

Ideally, there should be an update every few days.

## Chapters 


### Writing the renderer
1. Intro(writing 
1. [Setting up the project](project-setup)
1. The context
    - \[EXCURSION\] Handwritten GL Bindings
    - Hello Triangle
        - Buffers 
        - Vertex Attributes
        - Shaders
    - Time for Abstractions
        - Object Storage: GenVec
        - GL Abstractions 
        - GL Objects
        - Hello Triangle V2


