---
title: "Zero_v - Cutting Out VTable Calls with the Rust Type System"
date: 2021-05-30T21:18:56+01:00
Description: ""
Tags: []
Categories: []
DisableComments: false
---

A couple of months ago, I was writing an LC3 virtual machine/ emulator that allows users to add their own plugins. So a user could write something like:

```Rust
let mut vm = VM::new();
vm.add_plugin(command_logger);
vm.add_plugin(infinite_loop_detector);
vm.load_program(my_program);
vm.run()
```
And the VM would call the logger and loop detector whenever a command executed. The way the vm integrated these plugins was simple. There was a plugin trait that defined the interface for a plugin:

```Rust
pub trait Plugin {
    fn handle_event(&mut self, event: &Event, vm: &mut VM)
}
```

The VM contained a vector of these plugins and whenever there was an event the vm would report it with:

```Rust
for plugin in &mut plugins {
    plugin.handle_event(self, event)?
}
```
There were some complications related to ownership and not modifying the VM while using self.plugins directly (since the plugins are part of the vm, modifying the vm while executing a plugin which is behind a reference to the VM is a bad idea) but we can safely ignore those for now.

This is a natural way to approach the problem and in most cases, the right one. It's straightforward to implement, flexible and makes it easy for the end user to write and add their own plugins.

Unfortunately, it does have a small problem. Inside the VM, the plugins field is defined as:

```Rust
pub struct VM {
    // Some fields
    plugins: Vec<Box<dyn Plugin>>,
    // Some more fields
}
```
We are using trait objects, which means dynamic polymorphism, which means to use a plugin we have to go through a vtable.

And that means vtable overhead.

Most of the time, vtable overhead isn't a big deal. For it to matter you really need to be in a situation where performance is critical, you're counting nanoseconds and you're going to be calling into the vtable a lot. In the case of the VM, performance is important and plugins are called every time the VM sees an event we want to report. Every new command, every memory read or write, every register read or write and every interaction with an IO device will trigger a virtual function call.

So we need to worry about overhead.

This raised the question of whether there was a way for users to inject plugins that:
1. Avoids virtual function calls.
2. Doesn't require the library user to write much extra boilerplate.
3. Provides the compiler with as much information about the plugins at compile time as possible.

For those upsides, I was prepared to make the following concessions:
1. The user has to specify their plugins at compile time rather than at run time.
2. The library itself could contain a significant amount of internal boilerplate.

This article walks through the solution I came up with, the path I took to get there and some of the tools that helped along the way. Hopefully you'll find some of the ideas interesting or learn something new.

# A Simplified Problem Statement

First, the problem has a bunch of extra details from the VM use case that we don't need. Lets make something simpler.

```Rust
pub trait IntOp {
    fn run(&self, arg: usize) -> usize;
}
```
That's a bit more straightforward. No VM cruft, no events. Just one integer as input and one as output.

# Designing a Solution

Instead of treating our collection as a homogenous array of trait objects, we want some way to iterate over function outputs without going through a vtable. One way to do this would be to go through an enum. There's actually a crate that does this, [enum_dispatch.](https://crates.io/crates/enum_dispatch) Enum dispatch is great, but it does violate one of our requirements. Lets say a user wants to make their own plugin. Our crate provides an adder they want to use, but they want to add a custom square op. If we use enum_dispatch, then our user has to do something like:

```Rust
#[enum_dispatch(Op)]
enum OpEnum{
    Adder,
    Square
}

let ops = [
    OpEnum::from(Adder::new()),
    OpEnum::from(Squarer::new())
];
```

This is okay, but it's still a chunk of boilerplate. This is also slightly inefficient in terms of memory usage. Every member of our collection is going to be as large as the largest variant of the enum pus the size of the enum tag.

Now, on the other hand, enum dispatch lets collection composition vary at run time and provides a nice interface when you're iterating over the array which is similar to the way you would iterate over trait objects.

Still, lets see if we can find a solution that deals with memory usage and boiler plate.

So the obvious representation of a memory-efficient, tightly packed collection of heterogenous objects is a struct. But in order to have their trait automatically be implemented for a user-defined struct, the intermediate library author would have to write their own procedural macro, which is a lot of heavy lifting.

Lets try something different. What are our options instead of a simple struct?

Well, we want to define a collection which contains one or more heterogenous objects. And we want to be able to define an iterator over this collection. This sounds like a job for generics.

Except for the fact that we won't know how many objects the library user will want in their collection, so we don't know how many objects to be generic over.

One option for this would be to define one type of struct for every number of elements in the struct. So one struct called OneMemberStruct, one called TwoMemberStruct and so on, with the corresponding number of fields over which the types will be generic:

```rust
struct OneMemberStruct<A> {
    member1: A
}

struct TwoMemberStruct<A, B> {
    member1: A,
    member2: B
}
```
Then we create a trait that defines the behavior of returning an iterator and implement that trait for each struct type. For all collection sizes.

My fingers are starting to hurt just typing that much.

Even with macros, the above solution seems quite ugly. The Rust standard library actually had to do something analogous to this for arrays of different sizes before const generics came along, and while it was workable, it was never ideal. Unfortunately, our collections are heterogenous, so we don't get to use the const generics escape hatch.

So can we create a recursive solution?

```Rust
struct MyStruct<A, B> {
    data: A
    next: B
}
```

If B is another MyStruct, we might have something. It's a little like a linked list. Every node contains some data and the final next field will just be a pointer to the next chunk of data. Except that's not how linked lists work, because the final pointer should be a null. So what does a null look like in Rust? It's usually just an Option. But a None in an option is an object that is null and we want to express a type that is null. The closest thing to this is probably the unit type, ().

So lets take another go at defining our struct. We know we want the next field to either be a MyStruct or the unit type. And we'll change the type name to reflect the linked list idea.

```Rust
struct Node<A, B: NextNode> {
    data: A,
    next: B
}

trait NextNode {}
impl NextNode for ()
impl<A, B:NextNode> NextNode for MyStruct<A, B> {}
```
But what does an empty collection look like? We could use the unit type as the empty collection, but implementing more behavior than we have to for the unit type seems like a bad idea.

Instead, lets have a simple type representing the entry point to the collection for the user.

```Rust
struct Composite<A: NextNode> {
    head: A
}
```

Great, we have a type. Lets imagine we have some constructors defined too (omitted here, the new() functions will most likely look like what you expect) But what does client code look like right now?

```Rust
let collection = Composite::new(
        Node:::new(object1,
        Node::new(object2,
        Node::new(object3, ())
        ))
    );
```

Ugh. Lets not do that.

So, we have a collection where there can be an arbitrary amount of objects of different types fed to the constructor. This looks like a job for macros.

First we want a macro that will look something like:

```Rust
let nodes = compose_nodes!(object1, object2, object3);
```

Which will give us our nodes without the user having to worry about the nested calls to the node constructor.

```Rust
macro_rules! compose_nodes {
    () => {
        ()
    };
    ($val: expr) => {
       Node::base($val)
    };
    ($left: expr, $($right: expr), +) => {
        Node::new($left, compose_nodes!( $($right), +))
    };
}
```

This is simple enough. If it's empty, return the unit type. If it's a single value, return a node with that value as the data field and the unit type as the next field (this is all Node::base does) and finally, if it's two or more values, return a node with the first value in the data field and the macro result of the remaining values in the next field.

Let's add a simpler helper macro to put a Composite::new call on top of that.

```Rust
macro_rules! compose {
    ($($right: expr), *) => {
        Composite::new(compose_nodes!( $($right), *))
    };
}
```

And that gives the end-user a pretty interface when they're defining their collection:

```Rust
let composite = compose!(
    ObjectType1::new(),
    ObjectType2::new(),
    ObjectType3::new()
);
```

# Defining Iteration Over This Collection

Great. That's one problem sorted. But now how does a library writer define the behavior they want over a collection.

This is where the tradeoff with the enum solution comes back to bite us. If the collection was composed of enums, you could iterate over it in a pretty standard way, by returning an iterator over &EnumType. All our objects can be of different types, so that's not an option. We could iterate over &dyn TraitType, but then, even though we keep the compact memory representation of the composite solution, we are back to virtual functions and pointer chasing.

So is there a way we can have our cake and eat it too?

Sort of. Lets look at the definition of our trait again.

```Rust
pub trait IntOp {
    fn run(&self, arg: usize) -> usize;
}
```

We can't iterate over a collection of these, because they might all have different types. But we do have one escape hatch. The return type of the function is always going to be the same, so what if we set that to our iterator return type?

```Rust
struct CompositeIterator<'a, Nodes: NextNode> {
    nodes: &'a Nodes,
    index: usize
}

impl<'a, Nodes: NextNode> Iterator for CompositeIterator<'a, Nodes> {
    type Item = usize;
    fn next(&mut self) -> Option<Self::Item> {
        // We'll get back to this implementation later.
    }
    
}
```

A major problem with this implementation as it stands is that there's no guarantee that members of the collection implement IntOp. So we'll need a marker trait that can prove they do.

```Rust
trait IntOpCollection {}
impl<A: IntOp, B: IntOpCollection> IntOpCollection for Node<A, B> {}
impl IntOpCollection for () {}
```

Now we can change the bounds on our CompositeIterator to:

```Rust
struct CompositeIterator<'a, Nodes: NextNode + IntOpCollection> {
    //...
}
```

That means we know our collection contains ony objects implementing our trait. But how do we iterate over it?

Well, our iterator will keep an index that we'll incremement between iterations. So we just need a way to get the output of the run function for the element at that index.

But the curse of heterogenous collections strikes again. We can't index into the collection, because the return type of the index operation could be any of the types in our collection.

Maybe we can use the recursive nature of IntOpCollection to do something?

```Rust
trait IntOpCollection {
    fn run_at_index(&self, arg: usize, index: usize) -> Option<usize>;
}

impl<A: IntOp, B: IntOpCollection> IntOpCollection for Node<A, B> {
    fn run_at_index(&self, arg: usize, index: usize) -> Option<usize> {
        if index == 0 {
            Some(self.data.run(arg))
        } else {
            self.next.run_at_index(arg, index - 1)
        }
    }
}

impl IntOpCollection for () {
    fn run_at_index(&self, arg: usize, index: usize) -> Option<usize> {
        None
    }
}
```

The real action here happens in the implementation for a Node. If the index has reached zero, we return the result of running the operation at the head of that collection. Otherwise we decrement the index and pass it to the run_at_index method for the subcollection in the next node. Eventually, when the starting index gets big enough, it will reach the unit type at the end of the collection before the index hits zero and return None, which is exactly what we want.

That gives us the behavior we want. Back to our iterator.

The iterator needs to keep track of the index, hold a reference to the collection and feeds the input arguments to the right element of the collection on each iteration.

```Rust
struct CompositeIterator<'a, Nodes: NextNode> {
    nodes: &'a Nodes,
    index: usize,
    arg: usize
}

impl<'a, Nodes: NextNode> Iterator for CompositeIterator<'a, Nodes> {
    type Item = usize;
    fn next(&mut self) -> Option<Self::Item> {
        let output = self.nodes.run_at_index(self.index);
        self.index += 1;
        output
    }
}
```

This gives us something like a reverse of the map function on a collection. Map takes a single function and runs it using each item of the collection as an input. Here we take a single input, and run each function in the collection on it. Well, you could also think of this as using map over a collection of functions where the function passed to map is a closure that calls the function on the single input. Either way, this is only one of the things you might want to do with a collection. For example, what if we wanted to pass the output of one function to the input of the next for an accumulator?

This can be done, as long as we can set the value of the input field, by doing something like:

```Rust
let last_output = input;
while let Some(output) = composite_iterator.next() {
    composite_iterator.arg = output;
    last_output = output;
};
```

Changing the input field mid-iteration should let us get all the behavior we'll need from the iterator.

So that's it for the implementation. It took fifty lines of boilerplate, but we got there. Let's look at all of that together.

```rust
trait IntOp {
    fn execute(&self, input: usize) -> usize;
}

trait IntOpAtLevel {
    fn execute_at_level(&self, input: usize, level: usize) -> Option<usize>;
}

impl<A: IntOp, B: NextNode + IntOpAtLevel> IntOpAtLevel for Node<A, B> {
    fn execute_at_level(&self, input: usize, level: usize) -> Option<usize> {
        if level == 0 {
            Some(self.data.execute(input))
        } else {
            self.next.execute_at_level(input, level - 1)
        }
    }
}

impl IntOpAtLevel for () {
    fn execute_at_level(&self, _input: usize, _level: usize) -> Option<usize> {
        None
    }
}

struct CompositeIterator<'a, Nodes: NextNode + IntOpAtLevel> {
    level: usize,
    input: usize,
    parent: &'a Nodes,
}

impl<'a, Nodes: NextNode + IntOpAtLevel> CompositeIterator<'a, Nodes> {
    fn new(parent: &'a Nodes, input: usize) -> Self {
        Self {
            parent,
            input,
            level: 0,
        }
    }

impl<'a, Nodes: NextNode + IntOpAtLevel> Iterator for CompositeIterator<'a, Nodes> {
    type Item = usize;

    fn next(&mut self) -> Option<Self::Item> {
        let result = self.parent.execute_at_level(self.input, self.level);
        self.level += 1;
        result
    }
}

trait IterExecute<Nodes: NextNode + IntOpAtLevel> {
   fn iter_execute(&self, input: usize) -> CompositeIterator<'_, Nodes>;
}

impl<Nodes: NextNode + IntOpAtLevel> IterExecute<Nodes> for Composite<Nodes> {
   fn iter_execute(&self, input: usize) -> CompositeIterator<'_, Nodes> {
       CompositeIterator::new(&self.head, input)
   }
}
```

# Performance Tuning

The above works, but is it actually giving the expected speedups? After all, if
it isn't then users are jumping through a series of boilerplate hoops for
nothing. We need to justify their effort.

Before I get into the becnhmarks, a few quick caveats.
* Performance varies between different hardware setups. If you care about 
performance enough to try something like this, please test on your own
production setup to make sure these figures reflect your experience.
* I saw some odd behaviour when running these benchmarks. Basically, changing
the way functions were inlined for the static solution seemed to affect times
for the dynamic solution while the two should be independent. At first I
assumed it was due to the laptop throttling the CPU if the static run was fast
enough, and the dynamic benches after it being hit by the throttling as a
result. But swapping the order of the benchmarks made no difference, which
would have been the case if throttling was the issue. For now I'm putting it
down to either some compiler oddity or something in the benchmark framework
itself and given the performance I saw, I do believe this solution
absolutely smokes the dynamic solution in terms of speed.

The benchmarks are divided into four groups, two groups using dynamic calls
and two using static calls. Within each of these sets, one benchmark uses
an object with the parameter embedded in the object and one uses a small
extra tweak where the parameter is baked in at compile time using const
generics.

Basically, you're looking at something like this in the parameter case:

```rust
struct Adder {
    value: usize,
}

impl Adder {
    fn new(value: usize) -> Self {
        Self { value }
    }
}

impl IntOp for Adder {
    fn execute(&self, input: usize) -> usize {
        input + self.value
    }
}
```

And in the const case, an operation will look like this:

```rust
struct ConstAdder<const VALUE: usize> {}

impl<const VALUE: usize> ConstAdder<VALUE> {
    fn new() -> Self {
        Self {}
    }
}

impl<const VALUE: usize> IntOp for ConstAdder<VALUE> {
    fn execute(&self, input: usize) -> usize {
        input + VALUE
    }
}
```

In addition, there was one baseline benchmark that takes the sledgehammer
apparoach, by pushing all the operations into a single function with
no other abstractions. Spoilers, this one crushes both the virtual and static
benchmarks in terms of performance, and it is not close. If you really need
the performance, sometimes it pays to be nice to your compiler.

Oh, and the main code for the benchmarks is included in the footnotes[^1]. 
That's all the legalese taken care of. Onto the results.

![Benchmark Results with No Inlining](/images/violin_no_inline.jpg)

I wasn't joking when I said the baseline crushes the other two methods.
It beats the parameter-based vtable run by a couple of orders of magnitude,
clocking in at just over a nanosecond per iteration.

Looking at the static versus dynamic results though, we've got a pretty
clear victory for the static dispatch benchmarks, clocking in at around 3 times
as fast as dynamic dispatch, which seems like a pretty clear vindication of the
effort involved. Assuming you really need those cycles.

So, can we make it any faster?

Sure. I'll save you the results of the benchmarks for the experiments along the
way (imagine doing the hokey pokey, but with inline function annotations)
and get straight to the optimizations that worked. Basically, the best run
involved adding inline annotations to three methods.

```rust
impl IntOpAtLevel for () {
    #[inline]
    fn execute_at_level(&self, _input: usize, _level: usize) -> Option<usize> {
        None
    }
}

impl<A: IntOp, B: NextNode + IntOpAtLevel> IntOpAtLevel for Node<A, B> {
    #[inline]
    fn execute_at_level(&self, input: usize, level: usize) -> Option<usize> {
        if level != 0 {
            self.next.execute_at_level(input, level - 1)
        } else {
            Some(self.data.execute(input))
        }
    }
}

impl<'a, Nodes: NextNode + IntOpAtLevel> Iterator for CompositeIterator<'a, Nodes> {
    type Item = usize;

    #[inline]
    fn next(&mut self) -> Option<Self::Item> {
        let result = self.parent.execute_at_level(self.input, self.level);
        self.level += 1;
        result
    }
}
```

Which resulted in the following changes to the benchmarks:

```
Integer Ops/Static/Arg  time:   [25.491 ns 25.611 ns 25.761 ns]                                    
                        change: [-19.267% -18.424% -17.520%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 10 outliers among 100 measurements (10.00%)
  3 (3.00%) high mild
  7 (7.00%) high severe
Integer Ops/Vtable/Arg  time:   [109.78 ns 110.13 ns 110.60 ns]                                   
                        change: [-7.5058% -7.2445% -6.9897%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 13 outliers among 100 measurements (13.00%)
  7 (7.00%) high mild
  6 (6.00%) high severe
Integer Ops/Static/Const                                                                             
                        time:   [21.950 ns 21.978 ns 22.011 ns]
                        change: [-37.411% -36.599% -35.701%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 10 outliers among 100 measurements (10.00%)
  3 (3.00%) high mild
  7 (7.00%) high severe
Integer Ops/Vtable/Const                                                                            
                        time:   [102.87 ns 103.12 ns 103.48 ns]
                        change: [+12.436% +12.884% +13.492%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 15 outliers among 100 measurements (15.00%)
  9 (9.00%) high mild
  6 (6.00%) high severe
Integer Ops/Baseline    time:   [1.1747 ns 1.1758 ns 1.1771 ns]                                  
                        change: [-0.3156% -0.0230% +0.3152%] (p = 0.90 > 0.05)
                        No change in performance detected.
Found 7 outliers among 100 measurements (7.00%)
  1 (1.00%) high mild
  6 (6.00%) high severe

```

So, first off, I want to stress again that something weird is going on here.
Those vtable benchmarks shouldn't be affected by the changes I made to the
static benchmarks. However, even with that oddness, it's clear that those
inline annotations have bought us another 20% in performance for very little
effort (the set of functions was small enough that I could skip going through
profiling and head straight to the step where I spend a few minutes dropping
inline annotations on functions and seeing what the numbers tell me).

# Conclusion

So it looks like we've achieved everything we set out to accomplish. The crate
is small and simple. It does involve a decent bit of boilerplate on the part of
the intermediate library author, but once that's in place, their users only 
ever need to create a collection with the compose macro, which is exactly what
we had in mind. And our benchmarks confirm we're getting the expected
performance jump compared to dynamic allocation.

I've [published the code on crates](https://crates.io/crates/zero_v) in case
anyone finds it useful. Thanks for reading, and feel free to contact me with
any questions.

[^1]: Benchmarks Code. [Full Source On Github](https://github.com/fergaljoconnor/zero_v/blob/main/benches/integer_ops.rs)
    ```rust
    fn bench_composed<NodeType, Composed>(input: usize, composed: &Composed) -> usize
    where
        NodeType: IntOpAtLevel + NextNode,
        Composed: IterIntOps<NodeType>,
    {
        composed.iter_execute(input).sum()
    }
    
    fn bench_trait_objects(input: usize, ops: &Vec<Box<dyn IntOp>>) -> usize {
        ops.iter().map(|op| op.execute(input)).sum()
    }
    
    fn bench_baseline(input: usize) -> usize {
        (input + 0)
            + (input << 1)
            + (input + 2)
            + (input * 3)
            + (input + 4)
            + (input * 5)
            + (input + 6)
            + (input * 7)
            + (input + 8)
            + (input * 9)
            + (input + 10)
            + (input >> 11)
            + (input + 12)
            + (input >> 13)
    }
    
    pub fn criterion_benchmark(c: &mut Criterion) {
        let mut group = c.benchmark_group("Integer Ops");
    
        let ops_dyn: Vec<Box<dyn IntOp>> = vec![
            Box::new(Adder::new(0)),
            Box::new(LShifter::new(1)),
            Box::new(Adder::new(2)),
            Box::new(Multiplier::new(3)),
            Box::new(Adder::new(4)),
            Box::new(Multiplier::new(5)),
            Box::new(Adder::new(6)),
            Box::new(Multiplier::new(7)),
            Box::new(Adder::new(8)),
            Box::new(Multiplier::new(9)),
            Box::new(Adder::new(10)),
            Box::new(RShifter::new(11)),
            Box::new(Adder::new(12)),
            Box::new(RShifter::new(13)),
        ];
    
        let ops_dyn_const: Vec<Box<dyn IntOp>> = vec![
            Box::new(ConstAdder::<0>::new()),
            Box::new(ConstLShifter::<1>::new()),
            Box::new(ConstAdder::<2>::new()),
            Box::new(ConstMultiplier::<3>::new()),
            Box::new(ConstAdder::<4>::new()),
            Box::new(ConstMultiplier::<5>::new()),
            Box::new(ConstAdder::<6>::new()),
            Box::new(ConstMultiplier::<7>::new()),
            Box::new(ConstAdder::<8>::new()),
            Box::new(ConstMultiplier::<9>::new()),
            Box::new(ConstAdder::<10>::new()),
            Box::new(ConstRShifter::<11>::new()),
            Box::new(ConstAdder::<12>::new()),
            Box::new(ConstRShifter::<13>::new()),
        ];
    
        let ops = compose!(
            Adder::new(0),
            LShifter::new(1),
            Adder::new(2),
            Multiplier::new(3),
            Adder::new(4),
            Multiplier::new(5),
            Adder::new(6),
            Multiplier::new(7),
            Adder::new(8),
            Multiplier::new(9),
            Adder::new(10),
            RShifter::new(11),
            Adder::new(12),
            RShifter::new(13)
        );
    
        let ops_const = compose!(
            ConstAdder::<0>::new(),
            ConstLShifter::<1>::new(),
            ConstAdder::<2>::new(),
            ConstMultiplier::<3>::new(),
            ConstAdder::<4>::new(),
            ConstMultiplier::<5>::new(),
            ConstAdder::<6>::new(),
            ConstMultiplier::<7>::new(),
            ConstAdder::<8>::new(),
            ConstMultiplier::<9>::new(),
            ConstAdder::<10>::new(),
            ConstRShifter::<11>::new(),
            ConstAdder::<12>::new(),
            ConstRShifter::<13>::new()
        );
    
        group.bench_function("Static/Arg", |b| {
            b.iter(|| bench_composed(black_box(20), black_box(&ops)))
        });
    
        group.bench_function("Vtable/Arg", |b| {
            b.iter(|| bench_trait_objects(black_box(20), black_box(&ops_dyn)))
        });
    
        group.bench_function("Static/Const", |b| {
            b.iter(|| bench_composed(black_box(20), black_box(&ops_const)))
        });
    
        group.bench_function("Vtable/Const", |b| {
            b.iter(|| bench_trait_objects(black_box(20), black_box(&ops_dyn_const)))
        });
    
        group.bench_function("Baseline", |b| {
            b.iter(|| bench_baseline(black_box(20)))
        });
    }
    
    criterion_group!(benches, criterion_benchmark);
    criterion_main!(benches);
    ```
