---
title: "Zero_v - Cutting Out VTable Calls with the Rust Type System"
date: 2021-05-30T21:18:56+01:00
Description: ""
Tags: []
Categories: []
DisableComments: false
---

# Introduction
A couple of months ago, I was writing an LC3 virtual machine/ emulator that allows users to add their own plugins to the virtual machine. So a user could write something like:

```Rust
let mut vm= VM::new();
vm.add_plugin(command_logger);
vm.add_plugin(infinite_loop_detector);
vm.load_program(my_program);
vm.run()
```
and the vm integrated these plugins in the most obvious way. There was a plugin trait that defined the interface for a plugin:

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

Most of the time, vtable overhead isn't a big deal. For it to matter you really need to be in a situation where performance is critical, you're counting nanoseconds and you're going to be calling into the vtable a lot. In the case of the VM, performance is important and plugins are called every time the VM sees an event we want to report. Every new command, every memory read or write, every register read or write and every interaction with an IO device will trigger a virtual function call. So we probably have to worry about overhead.

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

Now, instead of treating our collection as a homogenous array of trait objects, we want some way to iterate over function outputs without going through any vtables. One way to do this would be to go through an enum. There's actually a crate that does this, [enum_dispatch.](https://crates.io/crates/enum_dispatch) Enum dispatch is great, but it does violate one of our requirements. Lets say a user wants to make their own plugin. Our crate provides an adder they want to use, but they want to add a custom square op. If we use enum_dispatch, then our user has to do something like:

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

Now, on the other hand, enum dispatch lets you vary collection composition at run time and provides a nice interface when you're iterating over the array which is very similar to the way you would iterate over trait objects.

Still, lets see if we can find a solution that deals with memory usage and boiler plate.

So the obvious representation for a memory-efficient, tightly packed collection of heterogenous objects is a struct. But in order to have their trait automatically be implemented for a user-defined struct, the intermediate library author would have to write their own procedural macro, which is a lot of heavy lifting.

Lets try something different. What are our options instead of a simple struct?

Well, we want to define a type which contains one or more instances of heterogenous objects. And we want to be able to define an iterator over this collection of objects.

This sounds like a job for generics, except for the fact that we won't know how many objects the library user is going to want in their collection in their code, so we don't know how many objects we want to be generic over.

One option for this would be to define one type of struct for every number of elements in the struct. So one struct called OneMemberStruct, one called TwoMemberStruct and so on, with the corresponding number of fields over which the types will be generic:

```Rust
struct OneMemberStruct<A> {
    member1: A
}

struct TwoMemberStruct<A, B> {
    member1: A,
    member2: B
}
```
Then we create a trait that defines the behavior of returning an iterator and implement that trait for each struct type. For all likely collection sizes.

My fingers are starting to hurt just typing that.

Even with macros, the above solution seems quite ugly. The Rust standard library actually had to do something analogous to this for arrays of different sizes before const generics came along, and while it was workable, it was never ideal. Unfortunately, our collections are heterogenous, so we don't get to use the const generics escape hatch.

So can we create a recursive solution?

```Rust
struct MyStruct<A, B> {
    data: A
    next: B
}
```
If B is another MyStruct, we might have something.It's a little like a linked list. Every node contains some data and the final next pointer will just be a pointer to the next chunk of data. Except that's not how linked lists work, because the final pointer should be a null. So what does a null look like in Rust? It's usually just an Option. But a None in an option is an object that is null and we want to express a type that is null. The closest thing to this is probably the unit type, ().

So lets take another go at defining our struct. We want the next field to either be a MyStruct or the unit type. And lets change the type name to reflect the linked list idea.

```Rust
struct Node<A, B: NextNode> {
    data: A,
    next: B
}

trait NextNode {}
impl NextNode for ()
impl<A, B:NextNode> NextNode for MyStruct<A, B> {}
```
But what does an empty collection look like? We could just use the unit type as the empty collection, but implementing more behavior than we have to for the unit type seems like a bad idea.

Instead, lets have a simple type representing the entry point to the collection for the user.

```Rust
struct Composite<A:NextNode> {
    head: A
}
```

Great, we have a type. Lets say we have some constructors defined for everything too (omitted here, the new() functions will look like what you expect) But what does client code look like right now?

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

And that gives the end-user a pretty nice interface when they're defining their collection:

```Rust
let composite = compose!(
    ObjectType1::new(),
    ObjectType2::new(),
    ObjectType3::new()
);
```

# Defining Iteration Over This Collection

Great. So we have one problem sorted. But now how does a library writer define the behavior they want over a collection this.

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

A major problem with this implementation as it stands is that don't have a guarantee that members of the collection implement IntOp. So we'll need a marker trait showing that all the members of a collection implement IntOp.

```Rust
trait IntOpCollection {}
impl<A: IntOp, B: IntOpCollection> IntOpCollection for Node<A, B> {}
impl IntOpCollection for () {}
```

So now we can change the bounds on our CompositeIterator to:

```Rust
struct CompositeIterator<'a, Nodes: NextNode + IntOpCollection> {
    //...
}
```

Now we know our collection contains objects with our trait. How do we iterate over it?

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

Okay, that gives us the behavior we want. Back to our iterator.

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

This gives us something like a reverse of the map function on a collection. Map takes a single function and runs it on using each item of the collection as an input. Here we take a single input, and run each function in the collection on it. Well, you could also think of this as using map over a collection of functions where the function passed to map is a closure that calls the function on the single input. Either way, this is only one of the things you might want to do with a collection. What if we wanted to pass the output of one function to the input of the next for an accumulator for example?

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
