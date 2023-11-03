---
title: 'Writing Elegant DSP Code in Rust'
date: 2023-11-01T00:00:02-04:00
draft: true
showToc: true
showReadingTime: true
---

## Introduction

In this post I’m going to attempt to show some of the advantages the rust type system brings to writing more elegant DSP code. We're going to start with a crash course on the relevant parts of the rust type system, and then we'll see how this can be applied to constructing the types and traits that will be the foundation of a Rust audio processing library. We'll then see how to apply some more abstract features of the Rust type system to make these elements more composable and modular. 

But first, as a point of clarification, what exactly do I mean by elegant?
- **Expressive**: code that’s concise and readable
- **Safe**: code that’s guaranteed not to have undefined behavior and other errors
- **Performant**: these aren’t at the expense of performance

## Preview

Start knowing nothing about Rust.

End with enough understanding of the type system to interpret this.

See how these features can be used to build DSP elements in a functional way like this.

## Rust Type System Basics

To make sure the rest of this talk is broadly accessible to non rustaceans, I'm going to introduce new aspects of the Rust types system as they become relevant to the feature we're implementing. We're going to start with the simple, naive implementation of vaious library features like buffers and processors, and then gradually iterate on them to take advantage of more advanced features of the Rust type system.

### Structs and Methods

Like many languages, Rust supports composing types into a `struct` as well as implementing methods on that struct. A `struct` may have multiple members and they will be accessible to the methods implemented on it. As an example we can make a first pass at creating one for our audio buffer.

```rust
struct AudioBuffer {
    data: Vec<f32>
}
```

As you can see this buffer is essentially a wrapper around a vector of 32 bit floats. We can implement various methods on this struct.

```rust
impl AudioBuffer {
    pub fn new() -> Self {
        Self {
            data: Vec::new()
        }
    }

    pub fn len(&self) -> usize {
        self.data.len()
    }

    pub fn zero(&mut self) {
        for s in &mut self.data {
            *s = 0.0;
        }
    }
}
```

As you can see, the first method we implemented doesn't take a `self` type as an argument, so it should be called using the `::` syntax instead of the `.` syntax. It's a standard convention for the `new` function to create a new instance of that type.

```rust
let buffer = AudioBuffer::new();
```

Our other methods both take a reference to `self`. You can see that the first one just returns the length of the buffer, and the second one zeros the contents of data. Importantly, you might have noticed that the second method specifically takes a *mutable* reference to `self.` Remember that in Rust, variables are immutable by default. So in order to modify the contents of `data`, we need to use the `mut` keyword when getting a reference to it.

### Generics

Like other languages, Rust also has support for generics, which can be used both in structs and functions. We'll generalize our `AudioBuffer` type into a `Buffer` type, and then we'll change our implementation to only apply to a specific generic value.

```rust
pub struct Buffer<T> {
    data: Vec<T>
}

impl<T> Buffer<T> {
    pub fn new() -> Self {
        Self {
            data: Vec::new()
        }
    }

    pub fn len(&self) -> Self {
        self.data.len()
    }
}
```

As you can see above, our buffer type is now generic over any kind of contents. We can't implement our zero method on it generically, though, since the contents won't always be a `f32`, but we *can* implement it on a specific value of the generic, as seen below.

```rust
impl Buffer<f32> {
    pub fn zero(&mut self) {
        for s in &mut self.data {
            *s = 0.0;
        }
    }

    pub fn rms(&self) -> f32 {
        self.data.map(|s| s / self.len())
    }
}
```

We can also alias a `Buffer<f32>` to an `AudioBuffer`, so we can use our new more generic type the same as our previous one.

```rust
type AudioBuffer = Buffer<f32>;

pub fn main() {
    let buffer = AudioBuffer::new();
    let rms = buffer.rms();
    buffer.zero();
}
```

### Traits

Okay. With the basics out of the way we're going to take a brief detour to introduce traits. We'll return to our buffer struct once we've covered traits and trait bounds.

Superficially you can think of traits as like Rust's version of a class or interface, and they can be used to define a set of implemented or unimplemented methods associated with that trait. For our library, we'll use the example of an `AudioProcessor`. Here's a naive implementation of our trait.

```rust
pub trait AudioProcessor {
    fn prepare(&mut self, rate: f32);
    fn process(&mut self, input: f32) -> f32;
}
```

As you can see, our processor method has unimplemented prepare and process methods. We can implement this trait on any type. We'll use a simple distortion as an example.

```rust
pub struct Distortion {
    gain: f32
}

impl Distortion {
    pub fn new() -> Self {
        Self {
            gain: 0.0
        }
    }

    pub fn set_gain(&mut self: gain: f32) {
        self.gain = gain;
    }
}

impl AudioProcessor for Distortion {
    fn prepare(&mut self, rate: f32) {
        // ignore this
    }

    fn process(&mut self, input: f32) -> f32 {
        input * db_to_linear(self.gain)
    }
}

pub fn main() {
    let mut distortion = Distortion::new();
    distortion.set_gain(10.0);
    let output = distortion.process(5.0);
}

```

As you can see, we've declared a distortion with a `new` and `set_gain` function, then we've implemented our `AudioProcessor` trait on it. Note that user defined traits can be implemented on almost any type you want. For example, I'm not sure why you'd want to do this, but we could even implement it on a float type.

```rust
impl AudioProcessor for f32 {
    fn prepare(&mut self, rate: f32) {
        // ignore this
    }

    fn process(&mut self, input: f32) -> f32 {
        input * db_to_linear(self.gain)
    }
}

pub fn main() {
    let output = 10.0.process(5.0);
}
```

Again, you wouldn't want to implement this trait on the float type. But I'm just showing you this to emphasize how widely traits can be applied.

### Associated Types

The next feature we're going to introduce is called the associated type, which we'll use to generalize our `AudioProcessor` into a `Processor`. Associated types are types that are part of the trait definition, but are left undefined until the trait is implemented. Here's an example.

```rust
pub trait Processor {
    type Input;
    type Output;

    fn prepare(&mut self, rate: f32);
    fn process(&mut self, input: Self::Input) -> Self::Output;
}
```


But
Base types
Traits 
Associated type constraints
Basic Traits
`

### Type Inference

The rust compiler makes heavy use of `let` expressions and type inferencing. This will be important later when we'll construct stack-allocated types whose types are too complex to be witten out, but the basic syntax of these declarations is below.

```rust
fn main() {
    let a = 5;
    let b = 4;
}
```