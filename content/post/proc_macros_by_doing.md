---
title: "Macros Rule! A Simple Guide to Rust Procedural Macros"
date: 2021-06-17T17:55:12+01:00
Description: ""
Tags: []
Categories: []
DisableComments: false
---

A macro is a function that takes some Rust code as input and returns some Rust code (usually a modified version of the input) as the output. There are two ways of generating your own: declarative macros and procedural macros.

* Declarative macros use a simple pattern matching syntax and are generated using another macro called macro_rules! For example, if you wanted a simple macro for printing a single item without having to pass a template string to println! as the first argument, you could use:
    ```rust
    macro_rules! print_it {
        ($x:expr) => { println!("{}", $x) }
    }
    ```
    If I write:
    ```rust
    print_it!(3)
    ```
    Then print_it should transform that to:
    ```rust
    println!("{}", 3);
    ```
    You don't need to worry about the declarative macro syntax for now. It's not why we're here. If you're curious about it though, [Rust by Example's macro_rules! chapter](https://doc.rust-lang.org/beta/rust-by-example/macros.html) is a great starting point.

* Procedural macros are functions which take in one or more streams of rust tokens as arguments. They can be broken down into three types.
    * Function-like macros are called in the same way as you would call a declarative macro (so they look like a function call, hence the name). Their signature looks like this:
        ```rust
        #[proc_macro]
        pub fn my_shiny_new_macro(input: TokenStream) -> TokenStream
        ```
        And you call them like this:
        ```rust
        my_shiny_new_macro!(some_input)
        ```
    * Derive macros are used with the derive annotation, typically to derive an implementation of some trait for the tagged struct. They are defined like:
        ```rust
        #[proc_macro_derive(SomeTraitName)]
        pub fn derive_some_trait_name(input: TokenStream) -> TokenStream
        ```
        And called with:
        ```rust
        #[derive(SomeTraitName)]
        struct BoringStruct {}
        ```
    * Attribute macros are also used with annotation, but unlike derive macros, they can take arguments other than the token stream they annotate. You can define them with:
        ```rust
        #[proc_macro_attribute]
        pub fn transmogrify(args: TokenStream, input: TokenStream) -> TokenStream
        ```
        And invocation looks like:
        ```rust
        #[transmogrify(args, to, transmogrify)]
        struct Transmogrifee {}
        ```

In the rest of the article, I'll be walking you through the implementation for a simple attribute macro (if you can write an attribute macro, you can definitely handle the other two). I'll motivate the example with a use case, then we'll go through how we'd write that use case, line by line. Hopefully, along the way, you'll learn enough to be able to write your own procedural macros in the future.

# The Use Case

Story time. Lets imagine you run an online game and are writing a service to provide real-time analytics on users for the front-end. Whenever a player is picking a costume, you want to give them prompts saying things like "Nice choice! That was the second most worn jacket today. If you're looking for something that goes with it, the Winged Helmet is the most popular hat to wear with it." (Don't worry. There are no microtransactions involved. You just want to help players look great).

To keep everything fast, you have an in-memory representation of the costumes each player picked today and do all the calculations on this data on the fly. So the representation of a costume looks like:

```rust

struct Costume {
    costume_id: u32,
    player_id: u32,
    time_selected: u32,
    hat_id: u32,
    hat_color_id: u32,
    hat_accessory_1_id: u32,
    hat_accessory_2_id: u32,
    footwear_id: u32,
    footwear_color_id: u32,
    footwear_augmentation_id: u32,
    jacket_id: u32,
    // And so on... (I think you get the idea)
}
```

Whenever a costume change event comes in the sevice generates a new Costume and adds it to the vector of costumes. Whenever a query comes in, the service runs iterates over all this data to answer it.

You deployed this service and performance has been okay, but still slower than you'd like. Your ops team are talking about things like 99th Percentile latency and throughput. They don't sound impressed. You need a way to speed it up.

Lately you've been looking into data-oriented design and you think it might be a great fit for this use case. Most queries don't use most fields in a costume, so if instead of your data structure looking like this:

```rust
struct CostumesVectorized {
    costumes: Vec<Costume>
}
```

it looked more like this:

```rust
struct CostumeVectorized {
    costume_id: Vec<u32>,
    player_id: Vec<u32>,
    time_selected: Vec<u32>,
    // Once again, I'll assume the pattern is clear at this point...
}
```

you think that would make better use of the cache, reduce query latency and keep your ops team happy.

Never upset your ops team. They know where the bodies are buried.

You're already considering code re-use. If this change works, you might roll it out to a few more of your services. So now you want a way to generate the vecorized struct that will be easy to reuse across different data models and services. And that's where proc macros come in.

# What do you want the macro's output to look like?

Basically the macro will take some struct BaseType as input.

```rust
struct BaseType {
    field1: Field1Type,
    field2: Field2Type,
    // ...
}
```

Basetype will be mapped to some vectorized type.

```rust
struct BaseTypeVectorized {
    field1: Vec<Field1Type>,
    field2: Vec<Field2Type>,
    // ...
}
```

We want this vectorized struct to have one and only one value for each field of each instance of BaseType. This means that all of the vectors should always have the same length. To maintain this invariant, you want to keep the vector fields private.

So we want a method that will add a new BaseType to the vectorized collection. We also want a method that will give us an immutable reference to a vector to run all our query logic. And a constructor would be nice. So we'll end up with an interface that looks something like:

```rust
impl BaseTypeVectorized {
    fn new() -> Self {
        // ...
    }

    fn push(&mut self, instance: BaseType) {
        // ...
    }

    fn get_field1(&self) -> &Vec<Field1Type> {
        // ...
    }

    fn get_field2(&self) -> &Vec<Field2Type> {
        // ...
    }

    // ...
}
```

And this is what we want our macro to derive. We want to be able to drop an attribute on our struct and have all of the above code be generated for us automagically.

Lets get to work.

# Crate Setup and Boilerplate

Proc macros in rust have to live in a separate proc macro crate. The reason for this is pretty straightforward. Most of the code in your crate is designed to be compiled and emitted in the binary to be executed at run time. However, proc macros generate code that will be executed at _compile_ time. They are plugins that the compiler uses to take some chunk of your code, run the macro on it to generate a different chunk of code and then compile that transformed piece of code into your binary.

So the compiler has to compile the macro before compiling any of your code. Rust handles this by breaking proc macros out into separate crates from your actual code and compiling the macro crates first.

For this project, all we really want to do is:

1. Make our proc macro.
2. Make sure that it works

So let's make one main crate that contains our test case and one subcrate that contains our macro..
```bash
> cargo new proc_macros_learn_by_doing --lib
> cd proc_macros_learn_by_doing
> cargo new vectorize --lib
```

Then we'll let the compiler know our main crate depends on the sub-crate by adding the following to the dependencies section of the main Cargo.toml.

```
[dependencies]
vectorize = { path = "vectorize" }
```

Finally, we need to set up our subcrate for proc macro development. Add the following lines to the Cargo.toml in the vectorize subcrate:

```
[lib]
proc-macro = true # Lets Rust know this is a compiler plugin

[dependencies]
proc-macro2 = "1.0.27" # Gives us versions of macro types that can be used in non-macro contexts
quote = "1.0.9" # Translates Rust syntax trees back into source code.
syn = "1.0.72" # Used for translating token streams to syntax trees.
```

That should give us everything we need to get our macro working.

# Designing Our Macro Interface and Building a Test Case

Above, we designed the interface we want for the code coming out of the macro, but what do we want the macro's own interface to look like.

Well, it takes in a struct and should emit the unchanged code for that struct along with the code for the vectorized struct.

It would be nice to give the programmer a way to specify what the vectorized version of the struct is actually called. Maybe we can pass that in as an argument to the attribute macro.

So that gives us something like:

```rust
#[vectorize(PluralName)]
struct SingularName {}
```

That looks fine. It's straightforward and gives us everything we'd want from a simple vectorize macro.

Now lets build a test case for that interface. Add the following to the lib.rs in your main crate.

```rust
use vectorize::vectorize;

#[vectorize(Potatoes)]
struct Potato {
    starch_content: f32,
}

#[cfg(test)]
mod tests {
    use super::{Potato, Potatoes};

    #[test]
    fn works_on_potatoes() {
        let mut potatoes = Potatoes::new();

        for starch_content in &[0.5, 0.1, 0.8] {
            potatoes.push(Potato {starch_content: *starch_content});
        }

        let total_starch: f32 = potatoes.get_starch_content().iter().sum();
        assert!((total_starch - 1.4).abs() < 0.001);
    }
}
```

Once the above passes, we will know our generated new, push and get methods all are working as intended.

# Writing The Actual Macro

Okay, boilerplate done. Lets write some macros!

Well, one. One macro.

Add the following code to the lib.rs of your vectorize crate:

```rust
mod generate;

use proc_macro::TokenStream;
use syn::{DeriveInput, parse_macro_input};

use generate::VectorizedStructName;

#[proc_macro_attribute]
pub fn vectorize(args: TokenStream, input: TokenStream) -> TokenStream {
    let parsed_input = parse_macro_input!(input as DeriveInput);
    let vec_struct_name = parse_macro_input!(args as VectorizedStructName);
    generate::expand_vectorize(vec_struct_name, parsed_input)
}
```

Whew, there's a lot going on in there for 13 lines of code. We might need to break that into chunks.

```rust
mod generate;

use proc_macro::TokenStream;
use syn::{DeriveInput, parse_macro_input};

use generate::VectorizedStructName;

```

Generate is the name of the module where we'll write most of the actual macro function logic. Rust macro functions have to be at the project root, but having hundreds of lines of macro code in your lib.rs can ruin your readability so we'll do the bare minimum in lib.rs.

proc_macro::TokenStream represents a stream of Rust tokens (probably not shocking). Syn::DeriveInput is a class that represents the parsed input of an attribute macro. parse_macro_input is a macro that tells the compiler to try and parse a tokenstream into that class. It can be used to parse the Rust code representation of any class that implements syn's Parse trait (we'll see an example of this trait in a little bit). And generated::VectorizedStructName is the class we will be using to represent the class name of the output vectorized struct, which the user can provide as an argument.

```rust
#[proc_macro_attribute]
pub fn vectorize(args: TokenStream, input: TokenStream) -> TokenStream {
    let parsed_input = parse_macro_input!(input as DeriveInput);
    let vec_struct_name = parse_macro_input!(args as VectorizedStructName);
    generate::expand_vectorize(vec_struct_name, parsed_input)
}
```

proc_macro_attribute is a builtin Rust attribute to tag proc macro attribute functions. The function must have the interface given above: it accepts two TokenStreams. It will use the first one to read its args if needed and it will interpret the second as the raw Rust tokens of the object which it is being applied to. The output will be another Rust Tokenstream.

Next we get an example of parse_macro_input in action. The macro takes the tokenstream before the as and interprets it as the type given after the as.

And in the last line, we take these two parsed structs and pass them to the internal expand_vectorize function to generate the actual output tokens.

That covers the interface of a macro and parsing its inputs. Everything else happens in generate.rs:

```rust
use proc_macro::TokenStream;                                                    
use proc_macro2::Span;                                                          
use quote::quote;                                                               
use syn::parse::{Parse, ParseStream, Result};                                   
use syn::{Data, DataStruct, DeriveInput, Fields, Ident, Type};                  
                                                                                
pub(crate) struct VectorizedStructName {                                        
    name: Ident,                                                                
}                                                                               
                                                                                
impl Parse for VectorizedStructName {                                           
    fn parse(input: ParseStream) -> Result<Self> {                              
        let name: Ident = input.parse()?;                                       
                                                                                
        Ok(VectorizedStructName { name: name })                                 
    }                                                                           
} 
```

First we need to actually implement the VectorizedStructName type we passed to parse_macro_input. In general, any struct you pass to parse_macro_input is going to represent some piece of information you want to intercept from the input below the macro or the arguments you pass to the macro within the annotation. In this case, we want to intercept a single identifier, passed in though the arguments on the same line, represented by the name field.

The parse trait lets the compiler know how to translate a stream of tokens into this struct. Here it's really simple. We only want one token so we intercept the next ident with a call to parse on the input stream and then return a VectorizedStructName containing that ident (wrapped in Ok so it conforms to the parse trait API).

```rust
pub(crate) fn expand_vectorize(vec_name: VectorizedStructName, input: DeriveInput) -> TokenStream {
    let type_name = &input.ident;                                               
    let vec_name = vec_name.name;                                               
                                                                                
    let fields = match &input.data {                                            
        Data::Struct(DataStruct {                                               
            fields: Fields::Named(fields),                                      
            ..                                                                  
        }) => &fields.named,                                                    
        _ => panic!("expected a struct with named fields"),                     
    };                                                                          
                                                                                
    let field_name: Vec<&Option<Ident>> = fields.iter().map(|field| &field.ident).collect();
    let field_type: Vec<&Type> = fields.iter().map(|field| &field.ty).collect();
                                                                                
    let getter_name: Vec<Ident> = fields                                        
        .iter()                                                                 
        .map(|field| {                                                          
            Ident::new(                                                         
                format!(                                                        
                    "get_{}",                                                   
                    field                                                       
                        .ident                                                  
                        .clone()                                                
                        .expect("Struct contains a field without an ident")        
                        .to_string()                                            
                )                                                               
                .as_str(),                                                      
                Span::call_site(),                                              
            )                                                                   
        })                                                                      
        .collect();                                                             
                                                                                
    TokenStream::from(quote!(
        // We'll tackle all the stuff in here in a minute...

    ));
```

Again, there's a lot going on here. First off we need to unpack the parsed inputs and use them to build the variables for our macro output. We'll need the type name obviously, so we'll grab that and the same for the name of the vecorized type.

Next we need to get the fields from the struct. To get these, we make sure the input we parsed was actually parsed to the Data::Struct variant. If it is, we'll take a reference to them and if not we'll panic (panicking is a little more acceptable in a proc macro than it is in Rust in general).

In the next two lines we unpack the names and types for these fields.

The final thing we need to generate is the names of the getters for our vectorized api. This looks like a bit of a mess but basically we iterate over the fields and for each one create a new ident for the getter function. This ident is built from a string (the function name) and a span (a way for Rust to track which part of the code corresponds to the ident for compiler diagnostics).

Okay, that's all the gritty details of generating our output variables. That leaves us free to move on to even more gritty details.

This is the code we pass to the quote function in vectorize:

```
#input                                                                  
									
#[derive(Clone, Debug)]                                                 
struct #vec_name {                                                      
    #(                                                                  
	#field_name: Vec<#field_type>                                   
    )*                                                                  
}                                                                       
									
impl #vec_name {                                                        
    fn new() -> Self {                                                  
	Self {                                                          
	    #(                                                          
		#field_name: Vec::new()                                 
	    )*                                                          
	}                                                               
    }                                                                   
									
    fn push(&mut self, instance: #type_name) {                          
	#(                                                              
	    self.#field_name.push(instance.#field_name)                 
	)*                                                              
    }                                                                   
									
    #(                                                                  
      fn #getter_name(&self) -> &Vec<#field_type> {                     
	  &self.#field_name                                             
      }                                                                 
    )*                                                                  
}
```

Here, there's a golden rule you want to remember: within the quote macro, everywhere you see a hash sign that isn't an an annotation, that's a placeholder for a variable.

So #input will be replaced with the original input of the macro , #vec_name will be replaced with vec_name and so on.

There is one other pattern you need to know about: you probably noticed the parts of the code that are wrapped in #( )*. For example:


```
#(                                                          
    #field_name: Vec::new()                                 
)* 
```

This is the macro system's way of expressing iteration. Basically, we'll iterate over any variable that shows up inside that wrapper and for each iteration, generate one copy of that section of the output, with the value returned from that iteration. So if the variable field_name was a vector containing the values "foo" and "bar", the above iteration would generate:

```rust
    foo: Vec::new(),
    bar: Vec::new()
```

And that's the final piece of code we needed for our macro to work. If you switch back to the main crate and run cargo test, your test case should pass now. If not, there's a copy of the working code [here](https://github.com/fergaljoconnor/blog_code_examples/tree/main/proc_macros_learn_by_doing) that should help you track down the problem. The macro works, it generalises and our imaginary ops team should be happy again.

# Some Extra Resources
If you want to learn more about precedural macros, the following links should help:
* [The Little Book of Rust Macros](https://danielkeep.github.io/tlborm/book/index.html): A very good reference, though sadly a bit out of date.
* [The Rust Refernce's Section on Procedural Macros](https://doc.rust-lang.org/reference/procedural-macros.html): Concise and a good starting point.
* [David Tolnay's Workshop on Procedural Macros](https://github.com/dtolnay/proc-macro-workshop). If you're going to learn, learn from the best. Tolnay is one of them.
