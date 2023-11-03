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
- **Flexible**: code that can be used in a variety of contexts
- **Composable**: code that can interface with itself???
- **Safe**: code that’s guaranteed not to have undefined behavior and other errors
- **Performant**: these aren’t at the expense of performance

## Preview

Start knowing nothing about Rust.

End with enough understanding of the type system to interpret this.

```rust
impl<T, B> AddAssign<&B> for Buffer<T> 
    where:
        T: Add<Output = T> + Copy + B: Block<Item = T>> {

    fn add_assign(&mut self, rhs: &B) {
        for (a, b) in self
            .as_slice_mut()
            .iter_mut()
            .zip(rhs.as_slice()) {

            *a = *a + *b;
        }
    }
}
```

See how these features can be used to build DSP elements in a functional way like this.

To make sure the rest of this talk is broadly accessible to non rustaceans, I'm going to introduce new aspects of the Rust types system as they become relevant to the feature we're implementing. We're going to start with the simple, naive implementation of vaious library features like buffers and processors, and then gradually iterate on them to take advantage of more advanced features of the Rust type system.

---

## Rust Basics

### Basic Types

Primitives, slices, etc. Remember the slices because they're going to come up again in about 30 minutes.

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
        // TODO
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
    fn process(&mut self, input: f32) -> f32 {
        input * db_to_linear(self.gain)
    }
}

fn db_to_linear(f: f32) -> f32 {
    // something here
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

    fn process(&mut self, input: Self::Input) -> Self::Output;
}
```

As you can see, our processor has been defined with two associated types, `Input` and `Output`. As you can see, this version captures much better the *essense* of what a processor is: a device that transforms some input into some output. We can implement it on our `Distortion` type as seen below.

```rust
impl Processor for Distortion {
    type Input = f32;
    type Output = f32;

    fn process(&mut self, input: Self::Input) -> Self::Output {
        input * db_to_linear(self.gain)
    }
}
```

We can implement this trait on our distortion multiple times if the assocated types are different. For example, we might also implement it for the 64 bit float.

```rust
impl Processor for Distortion {
    type Input = f64;
    type Output = f64;

    fn prepare(&mut self, rate: f32) {
        // ignore this
    }

    fn process(&mut self, input: Self::Input) -> Self::Output {
        input * db_to_linear(self.gain)
    }
}
```

### Trait Bounds

But wait! It's starting to look like we have some unecessary code duplication! Let's eliminate it using what we've previously learned about generics. Now for this example, I'm going to introduce two things.

- First, I'm going to be using the `Float` crate (or library), which has a trait called `Float` that's implemented on various floating point numbers, including `f32` and `f64`;
- Second, I'm going to be using a feature known as a trait bounds, to constrain our generic parameter to implement this trait.

```rust
use float::Float;

pub struct Distortion<F: Float> {
    gain: F
}

impl<F: Float> Distortion<F> {
    pub fn new() -> Self {
        Self {
            gain: F::ZERO
        }
    }

    pub fn set_gain(&mut self: gain: F) {
        self.gain = gain;
    }
}

impl<F: Float> Processor for Distortion<F> {
    type Input = F;
    type Output = F;

    fn process(&mut self, input: Self::Input) -> Self::Output {
        input * db_to_linear(self.gain)
    }
}

fn db_to_linear<F: Float>(f: F) -> F {
    // something here
}

pub fn main() {
    let distortion = Distortion::new();
    distortion.set_gain(10.0);
    let output: f32 = distortion.process(10.0);
}
```

As you can see, in each of our implementations we've defined a generic parameter `F`, which is constrained to be a type that implements the `Float` trait. We've assigned `F` to be the input and output of our processor, and since `Float` types are guaranteed to support adding and multiplication, we can perform the necessary arithmetic in our process method.

Importantly, if in our `Processor` implementation, the `F` wasn't constrained to be a `Float`, the compiler wouldn't let us multiply the input with the gain value since the generic parameter wouldn't be guaranteed to be a type that can be multiplied.

### Buffer Improvements

Let's implement a few useful methods to our buffer type that take advantage of some useful trait bounds. 

```rust
pub struct Buffer<T> {
    items: Vec<T>,
}

impl<F: Float> Buffer<F> {

}
```

We can also implement some methods for our buffer type using built-in traits.

```rust
impl<T: Copy> Buffer<T> {
    pub fn init(value: T, size: usize) -> Self {
        let mut items = Vec::with_capacity(size);

        for _ in 0..size {
            items.push(value);
        }

        Self { items }
    }
}

impl<T: Copy + Default> Buffer<T> {
    pub fn new(size: usize) -> Self {
        let mut items = Vec::with_capacity(size);

        for _ in 0..size {
            items.push(T::default());
        }

        Self { items }
    }
}

pub fn main() {
    let buffer_1 = Buffer::init(0.0, 512);
    let buffer_2 = Buffer::new(512);
}
```

The first example...
The second example...

---

## Implementing our Library

At this point we've been introduced to the basic language features we're going to need to develop our library. So we're going to take a moment to step back and think about some of the what we want this library to accomplish.

As mentioned earlier, we want users of our library to be able to write DSP code that's expressive, flexible, and composable, while being safe and performant. Here's some specific features we'll look at that together will help accomplish this goal.

- **Generic over multiple sample types**: We've already seen how to do this with floats specifically. We're also going to make our library generic over multiple *sample* types, which may have multiple channels.
- **Generic over multiple buffer types**: We don't want to restrict the user to using a single buffer type. Real audio code may need to interact with other libraries and languages, so all the buffer processing functions in our library should support our own `Buffer` type, the built-in rust buffers like `&mut [f32]` and raw pointers like `*mut f32` that you might get from a C/C++ library.
- **Auto implementation of useful methods**: We want useful methods to be auto implemented for our types
- **Simple element initialization and composition**: We want to be able to easily initialize and compose our elements in to processing graphs. The syntax of the faust language is a good reference for this.

Note that our goals of expressivity, flexibility, and composability are for *users* of our library. Implementing the library itself is going to require some fairly abstract uses of the Rust type system. So prepare to look at some gnarly code, knowing at the end that using our library will be much more friendly and flexible.

### Sample Trait

We've already seen how to write code that is generic over multiple float types, and our next task will be to make our library generic over multiple sample types as well. By sample here I mean a single snapshot of some continuous-time signal that may have multiple channels. So the sample might be in mono, stereo, or even spatial formats. By making our code generic over these sample types we'll be able to re-use the same DSP elements in all these scenarios.

The first step to making our library generic over sample types is to create a few of these types. Our basic, single channel types will just be float values, but we can create another one to represent a stereo sample.

```rust
pub struct Stereo<F: Float> {
    left: F,
    right: F
}

impl<F: Float> Stereo<F> {
    fn pan(self, db: F) -> Self {
        // Implement pan here
    }
}
```

The next step to making our library generic over sample types is to create a sample trait. 

```rust
pub trait Sample: Copy + Clone + Add<Self, Output = Self> {
    type Float: Float;

    const CHANNELS: usize;
    const EQUILIBRIUM: Self;

    fn mono(self) -> Self::Float;
    fn sin(self) -> Self;
    fn cos(self) -> Self;
    fn tan(self) -> Self;
    fn powf(self, e: Self) -> Self;
    fn min(self, rhs: Self) -> Self;
    fn max(self, rhs: Self) -> Self;

    fn gain(&self, db: Self::Float) -> Self;
}
```

As you can see here, we have declared a simple sample trait, which has an associated `Float` type, which must implement trait `Float`. It also has two associated constant values, `CHANNELS` and `EQUILIBRIUM`, which define the number of channels associated with that sample, and the "zero" value of it. It also has a bunch of functions that may be used on that sample type.

You may also have noticed at the top, that the sample trait is further constrained so it must implement the `Copy`, `Clone`, and `Add` traits (where `Add` takes a `Self` value and outputs another `Self`). If you implement a library like this for yourself you'll want to add a bunch of other constraints for operations like `Sub`, `Mul`, `Div`, and so on.

The next step would be to implement this sample trait for every sample type we want to support. I'm not going to do that now but the process would look something like this.

```rust
impl Sample for f32 {
    type Float = f32;

    const CHANNELS: usize = 1;
    const EQUILIBRIUM: Self = 0.0;

    fn mono(self) -> Self::Float {
        self
    }

    // And so on below...
}

impl<F: Float> Sample for Stereo<F> {
    type Float = F;

    const CHANNELS: usize = 2;
    const EQUILIBRIUM: Self = Sample {
        left: F::EQUILIBRIUM,
        right: F::EQUILIBRIUM
    };

    fn mono(self) -> Self::Float {
        self.left + self.right
    }

    // And so on below...
}
```

And you could continue expanding this list to include support for spatial audio formats or any othr kind of sample you like.

### Block Trait

Now that we've got the basic building blocks for making our library generic over different sample types, we're going to use this while implementing the trait that will make our library generic over different *buffer* types. To do this we're going to implement a `Block` trait, which will represent a non-owned frame of samples or some other data. Here's a first implementation.

```rust
pub trait Block {
    type Item;

    fn as_slice(&self) -> &[Self::Item];
    fn as_slice_mut(&mut self) -> &mut [Self::Item];
}
```

Most generically, a block is a list of *items*, so we have the corresponding `Item` associated type. For our block to be useful we must be able to access these items somehow, so we're going to retrieve them as a slice (or list) of these items.

We can implement other useful default methods on our block using type constraints. For example, we'll first implement an `rms` method when the block contains samples. First consider the signature of this method. What we want to do is constrain our function so that it is only available when the block `Item` implements `Sample`.

```rust
fn rms(&self) -> Self::Item where Self::Item: Sample;
```

Here's the appropriate constraint. Now we just need to provide a default implementation in our block trait.

First 

```rust
pub trait Block {
    type Item;

    fn as_slice(&self) -> &[Self::Item];
    fn as_slice_mut(&mut self) -> &mut [Self::Item];

    fn rms(&self) -> Self::Item where Self::Item: Sample {
        let count = Self::Item::from(self.len());
        let mut acc = Self::Item::EQUILIBRIUM;

        for s in self.as_slice() {
            acc += *s / count;
        }

        return acc;
    }
}
```

As you can see, this method is a bit more complicated than ones you might have seen implemented on concrete types. The advantage of this approach is that this method will now be available in our library for any combination of types that implement `Float` and `Block`.

As an example that will mutate the block, we'll also add a `copy_from` method that will copy the values from some source block to our current one.

```rust
    fn copy_from<B>(&mut self, src: &B)
        where
            Self::Item: Copy 
            B: Block<Item = Self::Item>> {

        self.as_slice_mut()
            .copy_from_slice(src.as_slice());
    }
```

The signature of the `copy_from` method might look a bit complicated, but if you walk through it you can see that it's actually fairly simple. The method first declares a generic parameter `B`, and as arguments takes a mutable reference to self and an immutable reference to some source block. The first constraint ensures that `Item` is copyable, so we can copy it from one block to the other. The second constraint ensures that the items in the source and destination block are the same type.

Our last step before being able to use these methods is to implement our `Block` trait on all the types we want to support it. Firs we'll implement it on the buffer type we started working on earlier.

```rust
impl<T> Block for Buffer<T> {
    type Item = T;

    fn as_slice(&self) -> &[T] {
        self.items.as_slice()
    }

    fn as_slice_mut(&mut self) -> &mut [T] {
        self.items.as_mut_slice()
    }
}
```

Here's implementing our trait for `[S]` the built-in rust array type.

```rust
impl<S> Block for [S] {
    type Item = S;

    fn as_slice(&self) -> &[Self::Item] {
        self
    }

    fn as_slice_mut(&mut self) -> &mut [Self::Item] {
        self
    }
}
```

And here's implementing our trait for `(*mut S, usize)`, a tuple containing a raw pointer to a sample and a size.

```rust
impl<S> Block for (*mut S, usize) {
    type Item = S;

    fn as_slice(&self) -> &[Self::Item] {
        unsafe {
            std::slice::from_raw_parts(self.0, self.1)
        }
    }

    fn as_slice_mut(&mut self) -> &mut [Self::Item] {
        unsafe {
            std::slice::from_raw_parts_mut(self.0, self.1)
        }
    }
}


```

Now that we've got a couple methods implemented on our traits, we can get a sense of the payoff to these more abstract implementations when using the library. Take a look at the following example.

```rust
pub fn test_mono(
        buffer_1: (*mut f32, usize),
        buffer_2: &[f32]
        buffer_3: &Buffer<f32>
    ) -> Buffer<f32> {

    let mut sum = Buffer::init(0.0, count);

    sum.copy_from(&buffer_1);
    sum.copy_from(&buffer_2);
    sum.copy_from(&buffer_3);

    buffer_3.copy_from(&buffer_1);
    buffer_2.copy_from(&buffer_3);

    let rms = buffer_2.rms();
}
```

As you can see, the first method copies data from a raw pointer of floats to our custom buffer, and also from this buffer to the built-in slice type. All of these calls use the same `copy_from` implementation. The first method uses the `f32` as the float type, but you could use any sample type, or even make the function generic over every sample type. Here's an example of that.

```rust
pub fn test_generic<S: Sample>(
        buffer_1: (*mut S, usize),
        buffer_2; &mut [S]
    ) {

    let mut buffer_3 = Buffer::init(S::EQUILIBRIUM, count);

    buffer_3.copy_from(&buffer_1);
    buffer_2.copy_from(&buffer_3);

    let rms = buffer_2.rms().mono();
}
```

The next thing you'd want to do is implement a laundry list of useful functions on our block and float traits. These could include methods for adding blocks, equilibrating a block, applying a function across a block, and more. These implementation will automatically become supported on all the different block types, and will be also supported *between* all the different combinations of block types.

For the sake of time, that will be left as an exercise to the reader. Next, we're going to take another look at the DSP elements that use these processors.
