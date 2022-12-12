+++
title = "Object Storage"
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

Before we figure out a way to replace the OpenGL calls in our `playground.rs`,
we need to think about their storage and ownership.

Do we want the user to essentially store (wrapped) GLObjects, or should they
solely live in our `opengl::Context`? What about their global state? They also
shouldn't be too expensive to retrieve, but also not a pain to pass around.

After sketching around for a while, I came to the conclusion that it would be nice to store them as they
are in the context, and make manipulation only possible via some sort of "dumb" handle. 
With some sort of generational vec, storing the index and some sort of generation in
the handle. Overwriting or deleting an object will automatically
invalidate all handles to that it, because whatever else is stored at the same
index, has an incremented generation. It will be much clearer once we implement
the functions for it, trust me!
The bottom line is that the user doesn't need to know anything about the backend in use, lifetimes, etc, but still has the same(or similar) benefits. 
The drawback is that we have to always pass our context around to do any GL
call.

There is also the possibility to return some sort of fallback(e.g. pink/purple
checkers texture) for missing objects, or some way to set constant handles to
specific values, like a `let speedwagon_handle = Handle::with_index(3)` always
referring to the vao with the "speedwagon" mesh. However, I feel that this
functionality should be provided by the graphics module later, rather than the context
itself, since we just want a thing wrapper around the different graphics API's,
and be essentially dumb otherwise.

So let's start with the storage for our graphics objects.

## GenVec
```admonish note
There is a useful crate that pretty much offers all we need, called [slotmap](https://crates.io/crates/slotmap).
However, as you probably realized by now, I suffer from severe NIH-syndrome and
only add dependencies when absolutely neccessary. In this case, I assume it shouldn't be
more than a few hundred lines, for the ability to change the functionality as we
need. But really, `slotmap` is a perfect fit for our problem.
```
Since this is likely to be a common problem in the future, needing indices with some
sort of generational information, we create a new package that should offer all
sorts of utility and shared data structures.

So inside our workspace root we create a new library
```bash 
#cac_graphics/
cargo new --lib core
```
Then we add it to our workspace,
```toml 
#cac_graphics/Cargo.toml
[workspace]
members = ["context", "core"]
```
And finally, add it as dependency to our context: 
```toml
[dependencies]
# ...
#core utility, data structs, etc.
core = { path = "../core"  }
```

Now we remove the junk in the `lib.rs`, add our trusty clippy lints and import the gen_vec module 
```rust 
//cac_graphics/core/src/lib.rs
#![warn(clippy::nursery)]
#![warn(clippy::perf)]
#![warn(clippy::pedantic)]

pub mod gen_vec;
```

and add the file to the source directory. 

What should the `GenVec` store?
It needs 2 generics, one for the Handle, which acts like a Key, containing the
index and a generation, and one for the Value, so we can store it.

The values are going to be stored in a continuous block, just like a
`Vec`. In fact, the values are just in a `Vec`. We are not going to store the raw
values, because we also want to add generational information to them, so we can
invalidate the existing handles when a value is deleted/another value takes the
index. In fact, the Value stores an Option, so we can delete whatever it stores
without shuffling the values in the `Vec` around.

```rust 
//cac_graphics/core/src/gen_vec.rs
use std::marker::PhantomData;

pub struct Value<V> {
    value: Option<V>,
    generation: u32,
}

pub struct GenVec<K, V> {
    values: Vec<Value<V>>,
    phantom: PhantomData<K>,
}
```
We need the `PhantomData`, because our `GenVec` is not storing any handles.
Without it, the compiler will complain with: 
```rust
E: parameter `K` is never used
   consider removing `K`, referring to it in a field, or using a marker such as `std::marker::PhantomData`
   if you intended `K` to be a const parameter, use `const K: usize` instead
```

`PhantomData` is just a marker without any size so suppress this warning.

This allows use to enforce some sort of type safety, so that it will be impossible
to accidentally try to get the wrong value from the wrong collection.
The compiler will straight out reject any form of `.get()` on a `GenVec<BufferHandle, GLBuffer>` with a `ShaderHandle` for example.

In fact, in the previous chapter, we could have used this approach to make our Context depend on the
lifetime of the GLContext instead of actually taking ownership. Though, I found
taking the ownership to be more elegant(especially in regards to the swap buffer control) in that case.

Now, the final struct that is missing is the actual handle. It stores the index,
the generation and again, PhantomData!

```rust 
//cac_graphics/core/src/gen_vec.rs
pub struct Handle<K> {
    index: usize,
    generation: u32,
    phantom: PhantomData<K>,
}
```

Our handles should be trivial to copy, so wee need to implement `Copy`. However, we
can't derive from it! Copy requires clone, but more importantly, both require
all of our fields to implement these traits. This is not a problem for `usize`
and `u32`, but our struct is generic over `K`. Luckily, we never store K, the
phantom is just ghost data, so the
workaround is to manually implement Copy and Clone for our struct.


```rust
impl<K> Copy for Handle<K> {}
impl<K> Clone for Handle<K> {
    fn clone(&self) -> Self {
        Self {
            phantom: PhantomData,
            index: self.index,
            generation: self.generation,
        }
    }
}
```


Now we can start implementing the common functions we expect from a `vec`-like
collection. Let's start with `get` and `get_mut`.
With the amazing `Option` functions, we can do this pretty neatly.

```rust
    /// Returns an immutable reference to the value associated with the handle, or None if there is
    /// none.
    pub fn get(&self, handle: Handle<K>) -> Option<&V> {
        self.values
            .get(handle.index)                              // first look if the index is occupied 
            .filter(|v| v.generation == handle.generation)  // then check if the generation of the Value matches the Handle 
            .and_then(|v| v.value.as_ref())  // returns the ref to the inner value of our Value<V> wrapper.
                                             // The caller only cares about the value itself after all.
                                             // TL;DR: It turns &Option<Value<V>> into Option<&V>
    }

    /// Returns a mutable reference to the value associated with the handle, or None if there is
    /// none.
    pub fn get_mut(&mut self, handle: Handle<K>) -> Option<&mut V> {
        self.values
            .get_mut(handle.index)
            .filter(|v| v.generation == handle.generation)
            .and_then(|v| v.value.as_mut())
    }
```

I really like all the quality of life functions in Rust!
Just for comparison, this would be the "traditional" approach without any
iterator-fu: 

```rust 
//Not so neat, so don't do this!
    pub fn get(&self, handle: Handle<K>) -> Option<&V> {
        if let Some(val) = self.values.get(handle.index) {
            if val.generation == handle.generation {
                val.value.as_ref()
            } else {
                None
            }
        } else {
            None
        }
    }
    pub fn get_mut(&self, handle: Handle<K>) -> Option<&V> {
        if let Some(val) = self.values.get(handle.index) {
            if val.generation == handle.generation {
                val.value.as_mut()
            } else {
                None
            }
        } else {
            None
        }
    }
```

Now we add another common vec function: `push`.
Push is the main way to insert a new value.

```admonish Note 
Whether to name this operation push or insert is kinda subjective. On one hand
it is technically not a push since it will often add the new values to fill the
"holes" in the vec, but on the other hand, we will pretty much push like 90% of
the time. While it doesn't matter as much with the handles, because they
abstract the indices away, future iterator operations might produce unintuitive
behaviour, so we should probably stick with insert.
```

Inserting a value is what should return the Handle to that value, increase the generation when necessary and fill 
the holes of the values that we deleted(the ones that are set to None in the
`Vec`). 

A naive approach could be this 
```rust
    /// Pushes a value into the `GenerationVec` and returns a handle to it.
    pub fn insert(&mut self, value: V) -> Handle<K> {
        let (index, generation) = self
            .values
            .iter_mut()
            .enumerate()                       //combine the values with their indices
            .find(|(i, v)| v.value.is_none())  //returns the first "free" slot
            .map_or_else(
                || {
                //there is no "free" slot in the vec, so we need to append to it
                    let i = self.values.len();
                    self.values.push(Value {
                        value: Some(value),
                        generation: 0,
                    });
                    (i, 0)
                },
                // found a free slot at index i, so we fill it and increase the
                // generation, so that  previous handles to this index won't
                // return this new value
                |(i, v)| {
                    v.value = Some(value);
                    v.generation += 1;
                    (i, v.generation)
                },
            );

        Handle {
            index,
            generation,
            phantom: PhantomData,
        }
    }
```

Great, right? 
Nah, we run into Rusts (current) limitations:

```rust 
40 |           let (index, generation) = self
   |  ___________________________________-
41 | |             .values
42 | |             .iter_mut()
   | |_______________________- borrow occurs here
...
45 |               .map_or_else(
   |                ----------- first borrow later used by call
46 |                   || {
   |                   ^^ closure construction occurs here
47 |                       let i = self.values.len();
48 |                       self.values.push(Value {
   |                       ----------- second borrow occurs due to use of `self.values` in closure

error[E0382]: use of moved value: `value`
  --> core/src/gen_vec.rs:54:17
   |
39 |     pub fn push(&mut self, value: V) -> Handle<K> {
   |                            ----- move occurs because `value` has type `V`, which does not implement the `Copy` trait
...
46 |                 || {
   |                 -- value moved into closure here
...
49 |                         value: Some(value),
   |                                     ----- variable moved due to use in closure
...
54 |                 |(i, v)| {
   |                 ^^^^^^^^ value used here after move
55 |                     v.value = Some(value);
   |                                    ----- use occurs due to use in closure
```
Two limitations with one stone.

In the first case, `values` is still borrowed when we don't find an empty value, so we can't push to it.

In the second case, we can't use the value in both paths, because it is
moved into one closure, so the second can't access it. Well, according to the borrow checker.

WE know that there won't be any borrow issue. In the first case, it could
in theory drop the previous borrow to the values, because it is not used anymore.
In the second case, we know that only one closure can be called, so it can move the
value only into either or.

However, the borrow checker can't understand this (yet). Maybe in the future with [Polonius](https://github.com/rust-lang/polonius) it might, but
for now, [the borrow checker is not always
right.](../../the_ugly.html#the-borrow-checker-is-not-always-right) 

That said, we don't really want to iterate through the entire vec anyway, just
to look for an empty slot. Imagine checking hundreds if not thousands of values
when we just end up pushing anyway.

So let's introduce another vec, one which stores the indices to the empty slots.

```rust 
pub struct GenVec<K, V> {
    values: Vec<Value<V>>,
    free: Vec<usize>,  //indices to the empty values 
    phantom: PhantomData<K>,
}
```

With that, inserting gets pretty simple:

```rust
// Inserts a value into the GenVec and returns a handle to it.
pub fn insert(&mut self, value: V) -> Handle<K> {
    //take an index out of the "free" vec, or push a new value
    if let Some(index) = self.free.pop() {
        let v = self
            .values
            .get_mut(index)
            .expect("The free list should be unable to store indices that are out of bounds!");

        v.generation += 1;
        v.value = Some(value);

        Handle {
            index,
            generation: v.generation,
            phantom: PhantomData,
        }
    } else {
        let index = self.values.len();
        let generation = 0;
        self.values.push(Value {
            value: Some(value),
            generation,
        });

        Handle {
            index,
            generation,
            phantom: PhantomData,
        }
    }
}
```

Personally, I tend to avoid things that can panic most of the time. But the
things that could potentially panic will be marked with the assumptions 
why a panic should not happen in theory, which is what the expect does here. Alternatively, just call `&mut values[index]`
here if you don't mind that. It's really just a personal choice.

We can `insert`, we can `get`, so we need to `remove` to finish the trifecta of
collection features.

We want to return the old value when it's removed. This allows the caller to
decide whether they want to keep it, unload it or throw it away. It also signals
whether the removal was successful or not. If a value is removed, we want to put
the index into the free list. There is no need to do anything with the
generation, because it will updated when we insert a new value.
```rust 
//removes the value if the index and generation match, returning the old one
pub fn remove(&mut self, handle: Handle<K>) -> Option<V> {
    self.values
        .get_mut(handle.index)
        .filter(|v| v.generation == handle.generation)
        .and_then(|v| {
            self.free.push(handle.index);
            v.value.take()
        })
}
```

Now finish this off by adding a few more useful functions. `new`, `with_capacity`, `with_values`,
and `clear` which are pretty much self-explanatory, but also a nice little
helper to construct the storage with some values.

```rust 
pub fn new() -> Self {
    Self {
        values: Vec::new(),
        free: Vec::new(),
        phantom: PhantomData,
    }
}

pub fn with_capacity(capacity: usize) -> Self {
    Self {
        values: Vec::with_capacity(capacity),
        free: Vec::with_capacity(capacity),
        phantom: PhantomData,
    }
}

pub fn with_values<const N: usize>(values: [V; N]) -> ([Handle<K>; N], Self) {
    let mut storage = Self::with_capacity(N);

    let mut handles: [Handle<_>; N] = [Handle {
        index: 0,
        generation: 0,
        phantom: PhantomData,
    }; N];

    values.into_iter().enumerate().for_each(|(i, v)| {
        handles[i] = storage.insert(v);
    });

    (handles, storage)
}

pub fn clear(&mut self) {
    self.values.clear();
    self.free.clear();
}
```

And while we are at it, we let RustAnalyzer implement the `Default` trait for
us as code action when we hover over the `new`: 

```rust 
impl<K, V> Default for GenVec<K, V> {
    fn default() -> Self {
        Self::new()
    }
}
```

There are obviously way more functions we could implement. Resizing,
inserting/getting slices, iterators, key-value insertion, etc. However, we are
burning to get back to that OpenGL nonsense, so we just add additional functions as we need.

So let's get back to OpenGL!... would someone say who doesn't appreciate
correctness, but we are better than that. The OpenGL stuff is still heavy work in
progress and in some sort of exploration stage, so there was little value in testing, but now we wrote the first "real"
library code. Time for some unit tests!

## Tests 
```admonish Note title="Why not tests first?"
Good question. Sometimes I write them first, sometimes I don't. 
I'm not dogmatic about it, and usually decide based on the complexity of the code
and gut-feeling. In this case, we are just inserting into/reading from vecs and
kinda require this set of basic functions to validate the results anyway. So
there is little value in having tests that fail first, given that our API is as
simple as it gets. For example, it is impossible to test whether `insert` or
`get` works, without implementing the other, unless we go into the
implementation details and check the values of the underlying containers. 
```

We create the test module at the bottom of our `gen_vec.rs`, and while we are at
it, we create a little helper to create some test storage.

```rust 
#[cfg(test)]
pub mod test {
    pub use super::*;

    //create a test storage with N elements
    fn test_storage<const N: usize>() -> ([Handle<u32>; N], GenVec<u32, String>) {
        let values: Vec<_> = (0..N).into_iter().map(|i| format!("Value{i}")).collect();
        GenVec::with_values(values.try_into().unwrap())
    }
}
```
Now we add some basic test cases. Just test whatever you see fit, to verify if
everything is working as intended. 

And boom: 
```rust 
running 8 tests
test gen_vec::test::clear_test ... ok
test gen_vec::test::construct_with_values_test ... ok
test gen_vec::test::free_cycle_test ... ok
test gen_vec::test::get_mutable_test ... ok
test gen_vec::test::clear_invalidates_handles_test ... FAILED
test gen_vec::test::insertion_test ... ok
test gen_vec::test::removal_test ... ok
test gen_vec::test::insert_over_capacity_test ... ok

failures:

---- gen_vec::test::clear_invalidates_handles_test stdout ----
thread 'gen_vec::test::clear_invalidates_handles_test' panicked at 'assertion failed: `(left == right)`
  left: `Some("NEW ENTRY")`,
 right: `None`', core/src/gen_vec.rs:268:27
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    gen_vec::test::clear_invalidates_handles_test

test result: FAILED. 7 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

With the failing test: 
```rust 
#[test]
fn clear_invalidates_handles_test() {
    let mut storage: GenVec<u32, &str> = GenVec::new();
    let old_handle = storage.insert("old value");
    storage.clear();

    let new_handle = storage.insert("NEW ENTRY");
    assert_eq!(storage.get(old_handle), None);
}
```
I would have preferred if I could say that this was an intentional error, for
educational sake, but it is a genuine one that exposed my sloppy `clear` function.

```rust
pub fn clear(&mut self) {
    self.values.clear();
    self.free.clear();
}
```
At first it seems reasonable. Our container uses 2 backing vecs, so when we
clear it, we clear the containers, right? Wrong! It will also clear all the
generational information we have, starting with gen 0 again, making the initial
handles valid again. I can think of four simple workarounds: 

1. Iterate through the value vec, setting all values to None and pushing the indices to the free vec 
2. Turn generations into shared per-instance rather a per-value counter. 
3. Introduce a second generation value, which is global and gets incremented when a new GenVec is created or cleared
4. Treat it as intended behavior, because it is also possible to use a Handle on
    another `GenVec` with the same handle type, just as it is possible to use
    the same key/index in other collections. 

We can choose between performance(iterating the vec doing stuff), reaching a
max generation quicker, or potentially a higher memory cost. 
The fourth solution however triggers an almost philosophical question:
How much (type-)safety is too much? For example, we already encoded the handle
type into the container type, to prevent accidental mishaps, but is there not an
argument to be made to do the same on the handle, storing information about the stored
value? What about encoding some unique id of the storage itself into a handle?

Whoa, let's calm our horses. First and foremost, our intended goal was to
abstract the concrete graphics API-objects. If we keep the type of the object we
want to store anyway, we might as well refer to the object directly. Our specific use-case is the main reason why this module belongs to our graphics workspace in the first place, rather than being a general standalone
collection. That said, there is no reason to not just create this type safety
via the generic `Handle` parameter when we actually need it, because that is its
entire purpose.

So after a bit of back and forth we decide to go with solution 4. It doesn't fix
the error at hand, but we can assume that something that actually clears all
objects should pretty much behave like a new collection instance, something that
we don't deal with either. However, this pretty much removes the reason
for having a clear function, since we can just create a new instance.
So we just remove the function and the tests we wrote for it and move on.

After we are done, and `cargo test` makes all tests pass, we follow the clippy
lints, so they don't start adding up to an unbearable amount.
Mostly annotating `#[must_use]` or making things const where possible.

## Docs 
Finally, we enable the lint `warn(missing_docs)` for the `gen_vec.rs`. 
Technically we could add this to our `lib.rs`, together with the other lints,
but it would make work extremely noisy to a point where we would end up toggling
it on and off. So we just enable it for the module that we consider "finished".
It does what it is supposed to do, and will only get minor changes/quality of
life features. We could do this as a final step once we actually release the
entire crate, but do be completely honest, adding docs for everything at once,
let alone doc tests, will just be peak boredom and has a high chance that I would skip large parts.

The additional benefit is that we will catch accidental public items. Once you
start writing the docs, you will usually notice when there is very little value
to the doc or the user actually needing to know about the item at all. This is
usually a huge hint to not expose the item to the public.

For example, our value-wrapper shouldn't have been public, because it is never
exposed to the user. Whoops!

That said, there is no need to be overly zealous with the docs. 
Sometimes a `/// constructor` is just a constructor. I certainly won't go as far as
the [Vec docs](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.new),
especially since the target audience will likely be just me, but feel free to add as much doc as you want.

Once done, let's create our documentation and open it with `cargo doc --open`.
There is still a lot of unfinished, unused and unreleated nonsense, but
selecting the `core` crate on the left, let's us take a look at the `gen_vec`
module.


This is my final file: 
```rust 
// cac_graphics/core/src/gen_vec.rs

//! Collection for Handles with automatic invalidation
//!
//! Values will return a handle upon insertion. Once the value has been removed or replaced
//! all existing handles to it will be invalidated, returning None when trying to retrieve the
//! value.
//!
//! The first generic argument, `K`, establishes a type safe connection between handles and the
//! collection.
//!
//! The second generic argument, `V`, is the actual Value stored inside the collection.
//!
//!
//! # Examples
//!
//! ```
//!    use core::gen_vec;
//!    // The `Key` argument could be any type. It is recommended to create every
//!    // storage with a unique Key, so that handles can only be used with that specific storage.
//!    // This prevents accidents, like removing a value from the wrong collection.
//!    struct Key;
//!    let mut storage = gen_vec::GenVec::<Key, _>::new();
//!
//!    //insertion returns handles to the value
//!    let handle_0 = storage.insert("foo");
//!    let handle_1 = storage.insert("bar");
//!
//!    assert_eq!(storage.get(handle_0), Some(&"foo"));
//!    assert_eq!(storage.get(handle_1), Some(&"bar"));
//!
//!    //mutating a value doesn't change it's handle.
//!    if let Some(value) = storage.get_mut(handle_1) {
//!         *value = "changed the value";
//!    }
//!    assert_eq!(storage.get(handle_1), Some(&"changed the value"));
//!
//!    //removing a value invalidates existing handles to that value
//!    let old_value = storage.remove(handle_0);
//!    assert_eq!(old_value, Some("foo"));
//!    assert_eq!(storage.get(handle_0), None);
//! ```

#![warn(missing_docs)]
#![warn(clippy::missing_panics_doc)]

use std::marker::PhantomData;

/// Handle to a value inserted into the `GenVec`
#[derive(PartialEq, Eq)]
pub struct Handle<K> {
    index: usize,
    generation: u32,
    phantom: PhantomData<K>,
}

impl<K> Copy for Handle<K> {}
impl<K> Clone for Handle<K> {
    fn clone(&self) -> Self {
        Self {
            phantom: PhantomData,
            index: self.index,
            generation: self.generation,
        }
    }
}

/// Storage for values that invalidates handles to them once the values are removed/replaces
///
/// The argument K allows type safety to make Handles only work with matching
/// `GenVec` with the same K argument.
///
/// The argument V is the actual value being stored in the collection.
pub struct GenVec<K, V> {
    values: Vec<Value<V>>,
    free: Vec<usize>,
    phantom: PhantomData<K>,
}

struct Value<V> {
    value: Option<V>,
    generation: u32,
}

impl<K, V> GenVec<K, V> {
    /// Constructor
    ///
    /// # Examples
    /// ```
    /// use core::gen_vec;
    ///
    /// //creates a `GenVec` storing Strings  
    /// let storage = gen_vec::GenVec::<u32, String>::new();
    /// ```
    #[must_use]
    pub const fn new() -> Self {
        Self {
            values: Vec::new(),
            free: Vec::new(),
            phantom: PhantomData,
        }
    }

    /// Constructs the `GenVec` and preallocates at least to a certain capacity.
    /// The capacity is at least twice the argument, because it also pre-allocates the space for
    /// the empty values.
    ///
    /// # Examples
    /// ```
    /// use core::gen_vec;
    ///
    /// //creates a `GenVec` storing Strings with a default capacity of 10
    /// let storage = gen_vec::GenVec::<String, String>::with_capacity(10);
    /// ```
    #[must_use]
    pub fn with_capacity(capacity: usize) -> Self {
        Self {
            values: Vec::with_capacity(capacity),
            free: Vec::with_capacity(capacity),
            phantom: PhantomData,
        }
    }

    /// Constructs the `GenVec` and inserts the passed values.
    ///
    /// In addition to the `GenVec`, this function will also return the created handles
    /// to the passed values in the same order as they were passed.
    /// ```
    /// use core::gen_vec;
    ///
    /// struct Key;
    ///
    /// let (handles, storage) = gen_vec::GenVec::<Key, _>::with_values(["foo", "bar"]);
    ///
    /// assert_eq!(storage.get(handles[0]), Some(&"foo"));
    /// assert_eq!(storage.get(handles[1]), Some(&"bar"));
    /// ```
    #[must_use]
    pub fn with_values<const N: usize>(values: [V; N]) -> ([Handle<K>; N], Self) {
        let mut storage = Self::with_capacity(N);

        let mut handles: [Handle<_>; N] = [Handle {
            index: 0,
            generation: 0,
            phantom: PhantomData,
        }; N];

        values.into_iter().enumerate().for_each(|(i, v)| {
            handles[i] = storage.insert(v);
        });

        (handles, storage)
    }

    /// Gets the immutable reference to the value associated with the `Handle`.
    ///
    /// If the `Handle` has no valid associated value, the returned value is `None`.
    /// ```
    /// use core::gen_vec;
    ///
    /// struct Key;
    /// let mut storage = gen_vec::GenVec::<Key, _>::new();
    ///
    /// let handle_0 = storage.insert("foo");
    /// let handle_1 = storage.insert("bar");
    ///
    /// assert_eq!(storage.get(handle_0), Some(&"foo"));
    /// assert_eq!(storage.get(handle_1),Some(&"bar"));
    /// ```
    #[must_use]
    pub fn get(&self, handle: Handle<K>) -> Option<&V> {
        self.values
            .get(handle.index)
            .filter(|r| r.generation == handle.generation)
            .and_then(|r| r.value.as_ref())
    }

    /// Gets the mutable reference to the value associated with the `Handle`.
    ///
    /// If the `Handle` has no valid associated value associated, the returned value is `None`.
    /// ```
    /// use core::gen_vec;
    ///
    /// struct Key;
    /// let mut storage = gen_vec::GenVec::<Key, _>::new();
    /// let handle = storage.insert(5);
    ///
    /// if let Some(value) = storage.get_mut(handle) {
    ///     *value *= 3;
    /// }
    ///
    /// assert_eq!(storage.get(handle), Some(&15));
    /// ```
    #[must_use]
    pub fn get_mut(&mut self, handle: Handle<K>) -> Option<&mut V> {
        self.values
            .get_mut(handle.index)
            .filter(|r| r.generation == handle.generation)
            .and_then(|r| r.value.as_mut())
    }

    /// Inserts a new value into the collection and returns a `Handle` to it.
    /// ```
    /// use core::gen_vec;
    ///
    /// struct Key;
    /// let mut storage = gen_vec::GenVec::<Key, _>::new();
    ///
    /// let handle_0 = storage.insert("foo");
    /// let handle_1 = storage.insert("bar");
    ///
    /// assert_eq!(storage.get(handle_0), Some(&"foo"));
    /// assert_eq!(storage.get(handle_1), Some(&"bar"));
    /// ```
    pub fn insert(&mut self, value: V) -> Handle<K> {
        //take an index out of the "free" vec, or push a new value
        if let Some(index) = self.free.pop() {
            let v = self
                .values
                .get_mut(index)
                .expect("The free list should be unable to store indices that are out of bounds!");

            v.generation += 1;
            v.value = Some(value);

            Handle {
                index,
                generation: v.generation,
                phantom: PhantomData,
            }
        } else {
            let index = self.values.len();
            let generation = 0;
            self.values.push(Value {
                value: Some(value),
                generation,
            });

            Handle {
                index,
                generation,
                phantom: PhantomData,
            }
        }
    }

    /// Remove the value associated with the `Handle` from the collection.
    ///
    /// All handles to the removed value will be invalidated.
    /// Returns the removed Value, or `None` if the valid is invalid.
    /// ```
    /// use core::gen_vec;
    ///
    /// struct Key;
    /// let mut storage = gen_vec::GenVec::<Key, _>::new();
    ///
    /// let handle_0 = storage.insert("foo");
    /// let handle_1 = storage.insert("bar");
    ///
    /// let old_value = storage.remove(handle_0);
    ///
    /// assert_eq!(old_value, Some("foo"));
    /// assert_eq!(storage.get(handle_0), None);
    /// assert_eq!(storage.get(handle_1), Some(&"bar"));
    /// ```
    pub fn remove(&mut self, handle: Handle<K>) -> Option<V> {
        self.values
            .get_mut(handle.index)
            .filter(|v| v.generation == handle.generation)
            .and_then(|v| {
                self.free.push(handle.index);
                v.value.take()
            })
    }
}

impl<K, V> Default for GenVec<K, V> {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
pub mod test {
    pub use super::*;

    //create a test storage with N elements
    fn test_storage<const N: usize>() -> ([Handle<u32>; N], GenVec<u32, String>) {
        let values: Vec<_> = (0..N).into_iter().map(|i| format!("Value{i}")).collect();

        GenVec::with_values(values.try_into().unwrap())
    }

    #[test]
    fn construct_with_values_test() {
        let values = [
            "Value0".to_owned(),
            "Value1".to_owned(),
            "Value2".to_owned(),
        ];

        let (handles, storage) = GenVec::<i32, _>::with_values(values);

        for (i, h) in handles.iter().enumerate() {
            assert_eq!(storage.get(*h), Some(&format!("Value{i}")));
        }
    }

    #[test]
    fn insertion_test() {
        let mut storage: GenVec<String, String> = GenVec::new();
        let mut handles = Vec::new();

        (0..10).for_each(|i| {
            handles.push(storage.insert(format!("Value{i}")));
        });

        for (i, h) in handles.iter().enumerate() {
            assert_eq!(storage.get(*h), Some(&format!("Value{i}")));
        }
    }

    #[test]
    fn free_cycle_test() {
        let (handles, mut storage) = test_storage::<10>();

        storage.remove(handles[0]);
        storage.remove(handles[5]);
        storage.remove(handles[7]);

        let h0 = storage.insert("4".to_owned());
        let h1 = storage.insert("2".to_owned());
        let h2 = storage.insert("42".to_owned());

        //check for proper insertions
        assert_eq!(storage.get(h0), Some(&"4".to_owned()));
        assert_eq!(storage.get(h1), Some(&"2".to_owned()));
        assert_eq!(storage.get(h2), Some(&"42".to_owned()));
    }
    #[test]
    fn removal_test() {
        let (handles, mut storage) = test_storage::<3>();

        let old_val = storage.remove(handles[2]);
        assert_eq!(old_val, Some(format!("Value{}", 2)));

        handles.iter().enumerate().for_each(|(i, h)| {
            if i == 2 {
                assert_eq!(storage.get(*h), None);
            } else {
                assert_eq!(storage.get(*h), Some(&format!("Value{i}")));
            }
        });
    }

    #[test]
    fn get_mutable_test() {
        let mut storage: GenVec<String, String> = GenVec::new();

        let handle = storage.insert("some_string".to_owned());
        let handle_copy = handle;

        if let Some(s) = storage.get_mut(handle) {
            *s = "ANOTHER STRING".to_owned();
        }

        //value has changed
        assert_eq!(storage.get(handle), Some(&"ANOTHER STRING".to_owned()));

        //both should point to the same value
        assert_eq!(storage.get(handle), storage.get(handle_copy));
    }

    #[test]
    fn insert_over_capacity_test() {
        let mut storage: GenVec<String, String> = GenVec::with_capacity(3);

        for i in 0..100 {
            storage.insert(format!("{i}"));
        }
    }
}
```
