---
title: 'Writing Elegant DSP Code in Rust'
date: 2023-11-01T00:00:02-04:00
draft: true
showToc: true
showReadingTime: true
showWordCount: true
tags: ["Rust", "DSP"]
---

## Introduction

In this post I’m going to attempt to show some of the advantages the rust type system brings to writing more elegant DSP code. We're going to start with a crash course on the relevant parts of the rust type system, and then we'll see how this can be applied to constructing the types and traits that will be the foundation of a Rust audio processing library. We'll then see how to apply some more abstract features of the Rust type system to make these elements more composable and modular. 

But first, as a point of clarification, what exactly do I mean by elegant? Well, for the purposes of this talk, I'm going to define it with the following list.

- **Expressive**: Concise and readable
- **Flexible**: Usable in a variety of contexts
- **Composable**: Elements can be easily combined into graphs

We want to accomplish this without there being much of a cost in terms of **safety** and **performance**.

## Preview

Just to preview what will be covered in this article. The expectation is that you start knowing something about programming and very little about Rust spcifically. By the middle of this talk you'll have enough understanding of the Rust type system to decipher cryptic looking abstract types like this one.

```rust
#[derive(Copy, Clone)]
pub struct Chain<P1, P2>(pub P1, pub P2);

impl<In, Between, Out, P1, P2> Processor for Chain<P1, P2> 
    where
        P1: Processor<Input = In, Output = Between>,
        P2: Processor<Input = Between, Output = Out> {

    type Input = In;
    type Output = Out;

    fn prepare(&mut self, sample_rate: f64, block_size: usize) {
        self.0.prepare(sample_rate, block_size);
        self.1.prepare(sample_rate, block_size);
    }

    fn process(&mut self, input: Self::Input) -> Self::Output {
        self.1.process(self.0.process(input))
    }
}
```

Which we'll be using in the process of developing the backend of our DSP library. When this process is done we'll have created most of the fundemental data structures needed for DSP programming, and we'll have a simple interface to our library with elements that can be composed using a faust-like syntax. Our goal is to achieve this simple syntax without the need of a domain-specific language. Here's an example.

```rust
pub fn main() {
    let input = Block::init(0.0, 512);
    let output = Block::init(0.0, 512);

    let lfo = lfo(0.5) >> gain(-10.0);
    let graph = distortion(10.0) >> gain(lfo) >> gain(-10.0);

    graph.process_block(&input, &mut output);
}
```



To make sure the rest of this talk is broadly accessible to non rustaceans, I'm going to introduce new aspects of the Rust types system as they become relevant to the feature we're implementing. We're going to start with the simple, naive implementation of vaious library features like buffers and processors, and then gradually iterate on them to take advantage of more advanced features of the Rust type system.

---

## Rust Basics

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

## Floats, Samples, and Buffers

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

You may also have noticed at the top, that the sample trait is further constrained so it must implement the `Copy`, `Clone`, and `Add` traits (where `Add` takes a `Self` value and outputs another `Self`). If you implement a library like this for yourself you'll want to add a bunch of other constraints for operations like `Sub`, `Mul`, `Div`, and so on. Here's how you might implement the add trait for a sample.

```rust
// TODO: Implement add trait
// TODO: Implement add with float trait
```

This will allow us to add these floats as is shown below.

```rust
// Example adding two samples
// Example adding sample with float
```

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

### More Methods

The final thing you'll want to do is implement a laundry list of useful functions on our block and float traits. These could include methods for adding blocks, equilibrating a block, applying a function across a block, and more. These implementation will automatically become supported on all the different block types, and will be also supported *between* all the different combinations of block types. Here's a few examples.

```rust
    /// Apply a function to each sample
    fn apply<F: Fn(Self::Item) -> Self::Item>(&mut self, f: F) where Self::Item: Copy {
        for v in self.as_slice_mut() {
            *v = f(*v);
        }
    }

    /// Apply a gain to the buffer
    fn gain(&mut self, db: <Self::Item as Sample>::Float) where Self::Item: Sample {
        self.apply(| v | v.gain(db))
    }

    /// Fill the buffer with a value
    fn fill(&mut self, value: Self::Item) where Self::Item: Copy {
        self.apply(| _ | { value } );
    }

    /// Equilibrate the buffer
    fn equilibrate(&mut self) where Self::Item: Sample {
        self.fill(Self::Item::EQUILIBRIUM);
    }
```

And these methods can be used as shown below.

```rust
/// Test on a buffer of stereo 32 bit floats
pub fn test_1(buffer: Buffer<Stereo<f32>>) {
    buffer.fill(Stereo::from(0.0));
    buffer.gain(10.0);
    buffer.fill(Stereo::from(0.5));
    buffer.apply(| s | s + 1.0);
}

/// Test on a raw pointer to 32 bit floats
pub fn test_2(buffer: (*mut f32, usize)) {
    buffer.fill(0.0);
    buffer.gain(10.0);
    buffer.fill(0.5);
    buffer.apply(| s | s + 1.0);
}
```

Okay. I think that covers how to implement abstract methods over all the combinations of float, sample, and buffer types. Regardless of if you thought any of the abstract code was complicated, I hope these final examples made it clear how simple and flexible the end result is. Remember, these functions can now be used with any permutation of float type, channel formats, and buffer type.

--- 

## Processors and Generators

Next we're going to take a closer look at DSP processors, and how they can be represented in a way that's abstract and composable. Just a reminder from earlier, we represented processors with the following trait, which consumes an input and yields an output.

```rust
pub trait Processor {
    type Input;
    type Output;

    fn prepare(&mut self, sample_rate: f64, block_size: usize);
    fn process(&mut self, input: Self::Input) -> Self::Output;
}
```

And while we're add it we'll also create a generator trait, which only produces an output.

```rust
pub trait Generator {
    type Output;

    fn prepare(&mut self, sample_rate: f64, block_size: usize);
    fn generate(&mut self) -> Self::Output;
}
```

What we're going to try to do is use the trait system to allow types that implement these traits to be composed into audio graphs with operators, similar to the faust programming language. Here's a preview of where we're going.

```rust
pub fn main() {
    let dsp = noise() >> gain(lfo(10.0) >> gain(-20.0));
    let output: f32 = dsp.generate();
}
```

There's two main elements to this method of graph initialization. First there's the audio node initialization, which is done with `const` functions, and second there's how we're composing these elements into the graph with operators. We'll start by considering the node composition.

### Node Composition

We'll start by implementing chain composition, which will be accomplished by the `>>` right shift operator. To support this we'll need to implement the `Shr` trait. Two elements composed with this operator will feed into each other from left to right.

```rust
pub trait Shr<Rhs = Self> {
    type Output;

    fn shr(self, rhs: Rhs) -> Self::Output;
}
```

We're going to implement the `Shr` trait on a single type, but we want to be able to compose a variety of different processors and generators. So for this to work we'll need to create a `Node` wrapper type that can be composd with our operators, and these node types should be able to play back any processor or generator that it contains.

Here's a simple declaration of our node type.

```rust
#[derive(Copy, Clone)]
pub struct Node<P>(pub P);
```

To drive the conceptual point home, you can see the following diagram, which shows how each generator and processor is wrapped in a `Node` type, and how each of these nodes is being composed with operators.

```
    Node( Generator1 ) >> Node( Processor2 ) >> Node( Processor3 )
```

Now consider each of the operators. How can we use the `Shr` trait to get the desired chaining functionality? We'll use the trait to generate a `struct Chain`, which wraps the elements on either side of the operator and calls them in the appropriate order. A `struct Chain` will contain two elements that are fed into one other, as shown in the diagram below. These chains may be composed recursively.

```
    Chain( Node( Generator1 ), Chain( Node( Processor1 ), Node( Processor2 ) ) )
```

This diagram represents the data structure that we want our operators to generate. Since we already implemented the `Node` struct, we'll need to implement the `Chain` struct next.

```rust
#[derive(Copy, Clone)]
pub struct Chain<P1, P2>(pub P1, pub P2);

impl<In, Between, Out, P1, P2> Processor for Chain<P1, P2> 
    where
        P1: Processor<Input = In, Output = Between>,
        P2: Processor<Input = Between, Output = Out> {

    type Input = In;
    type Output = Out;

    fn prepare(&mut self, sample_rate: f64, block_size: usize) {
        self.0.prepare(sample_rate, block_size);
        self.1.prepare(sample_rate, block_size);
    }

    fn process(&mut self, input: Self::Input) -> Self::Output {
        self.1.process(self.0.process(input))
    }
}

impl<Between, Out, G, P> Generator for Chain<G, P> 
    where
        G: Generator<Output = Between>,
        P: Processor<Input = Between, Output = Out> {

    type Output = Out;

    fn reset(&mut self) {}
    fn prepare(&mut self, _sample_rate: f64, _block_size: usize) {}

    fn generate(&mut self) -> Self::Output {
        self.1.process(self.0.generate())
    }
}
```

The heavy use of generics in this example may look intimidating, but it's actually fairly simple. Take a look at our implementation of `Generator` for `Chain`. you can see that there are generic parameters `G` and `P` for the generator at the beginning of the chain and the processor next in the chain. then their are two generic parameters `Between` and `Out`, which represent the data types between the two elements and the output of the whole chain. These type constraints you see here are meant to ensure that the output of the first element is the same as the input to the second element. Then you can see the process method will first call member `0`, then feed the output into the call to member `1`, exactly what we'd expect from a chain.

Now that we have the `Node` and `Chain` types, we can implement the code that will generate these types from our operators. We'll start with the `Shr` operator for creating chains.

```rust
impl<A, B> std::ops::Shr<Node<B>> for Node<A> {
    type Output = Node<Chain<A, B>>;

    fn shr(self, rhs: Node<B>) -> Self::Output {
        Node(Chain(self.0, rhs.0))
    }
}
```

When the `>>` is applied, it will tranform the nodes on either side into a single node, containing a chain of the previous nodes. Now we've implemented the code we need to compose audio nodes. Our next step is to create the functions that will produce these nodes from our processors and generators.

### Node Initialization

We're going to start with a implementation of a `struct Gain`, which implements `Processor` generically over all samples.

```rust
pub struct Gain<S: Sample>(S::Float);

impl<S: Sample> Gain<S> {
    pub fn from(db: S::Float) -> Gain<S> {
        Gain(db)
    }
}

impl<S: Sample> Processor for Gain<S> {
    type Input = S;
    type Output = S;

    fn prepare(&mut self, _sample_rate: f64, _block_size: usize) {}

    fn process(&mut self, input: Self::Input) -> Self::Output {
        input.gain(self.0)
    }
}
```

We can create a const function that creates an audio node from this struct as shown below.

```rust
pub const fn gain<S: Sample>(db: S::Float) -> Node<Gain<S>> {
    Node(Gain(db))
}
```

### Testing

Now we should have everything we need to start composing nodes. You can assume I repeated this process for some other processors and generators.

```rust
pub fn main() {
    let noise = noise();        // Creates Node(Noise)
    let gain = gain(10.0);      // Creates Node(Gain(10.0))

    let dsp = noise >> gain_1;  // Creates Node(Chain(Node(Noise), Node(Gain(10.0))))
}
```

The last thing we want is to be able to call the generator and processor methods on our nodes. To do this we need to implement these traits on `Node`.


```rust
impl<Out, G> Generator for Node<G>
    where
        G: Generator<Output = Out> {

    type Output = Out;

    fn prepare(&mut self, sample_rate: f64, block_size: usize) {
        self.0.prepare(sample_rate, block_size);
    }

    fn generate(&mut self) -> Self::Output {
        self.0.generate()
    }
}

impl<In, Out, P> Processor for Node<P>
    where
        P: Processor<Input = In, Output = Out> {

    type Input = In;
    type Output = Out;

    fn prepare(&mut self, sample_rate: f64, block_size: usize) {
        self.0.prepare(sample_rate, block_size);
    }

    fn process(&mut self, input: Self::Input) -> Self::Output {
        self.0.process(input)
    }
}
```

As you can see, each trait is only implemented if the element contained by the node also implements the trait. So, for example, the processor trait will only be implemented for the node if the containing member is also a processors.


Now we can freely compose our nodes as is shown below.

```rust
pub fn main() {
    let block = Block::init(0.0, 512);
    let graph = noise() >> gain(10.0) >> gain(-10.0);

    graph.generate_block(&mut block);
}
```

Since this graph starts with a generator, and ends with processors, the generator methods are avaliable on the whole chain. If I switch the first element to a processor, however, then you'll see the generator methods are no longer available and I can intead call the processor methods. This is the magic of type inteference.

```rust
pub fn main() {
    let input = Block::init(0.0, 512);
    let output = Block::init(0.0, 512);
    let graph = distotion(10.0) >> gain(10.0) >> gain(-10.0);

    graph.process_block(&input, &mut output);
}
```

This general strategy can be expaneded to include splitting and combining streams of data, using the `|` and the `&` operatos, for example. Remember, these nodes we've created are generic over all the float, sample, and buffer types we created previously.

## Summary
