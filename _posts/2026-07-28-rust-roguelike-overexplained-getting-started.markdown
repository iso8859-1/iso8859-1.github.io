---
layout: post
title:  "Rust Roguelike Overexplained: Getting Started"
date:   2026-07-28 08:41:00 +0200
categories: [roguelike, rust, tutorial, bevy]
---
# Introduction

This will be a tutorial series on using [Rust](https://rust-lang.org/) together with [Bevy](https://bevy.org/). The preliminary goal is to create something that resembles a roguelike. I will be following [Roguelike Tutorials](https://rogueliketutorials.com/tutorials/tcod/v2/part-1/) while mixing in parts of [Roguelike Tutorial - in Rust](https://bfnightly.bracketproductions.com/chapter_0.html) and the free parts of [The Impatient Programmer's Guide to Bevy and Rust](https://aibodh.com/books/the-impatient-programmers-guide-to-bevy-and-rust/).

The main point of this is to give ME a better understanding of the programming language Rust and of the [ECS](https://en.wikipedia.org/wiki/Entity_component_system) architectural pattern. By writing it down, I will learn more about the topic than by simply reading about it.

I will be working on Windows for this tutorial series, but everything should also work on Linux.

## Overexplained

I once read "Accelerated C++" by Andrew Koenig and Barbara Moo and was astonished by the amount of explanation they provided for a simple "Hello World" program in C++. I want to try and replicate some of this by trying to understand each and every line of code and each and every action - and explaining them.

# Getting Started

## Setting Up Rust

To get started with programming in Rust, you need a so-called toolchain. It contains the tools you need to translate the texts you write into something the computer can actually understand. A computer program is a series of [bytes](https://en.wikipedia.org/wiki/Byte) (numbers between 0 and 255) that follow a specific convention. On Windows, this convention is called [Portable Executable](https://en.wikipedia.org/wiki/Portable_Executable); on Linux, it's the [Executable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format). Inside those, the instructions from the program are encoded as sequences of [Opcodes](https://en.wikipedia.org/wiki/Opcode). The toolchain is responsible for converting the program text into the [Binary File](https://en.wikipedia.org/wiki/Binary_file) - the sequence of bytes - that your computer understands. To do this, it uses the Rust [compiler](https://en.wikipedia.org/wiki/Compiler) to translate your source code into [object code](https://en.wikipedia.org/wiki/Object_code). Note that Rust does not compile file by file the way C or C++ does: the unit of compilation is the [crate](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html) - one whole program or one whole library, no matter how many files it is spread over. Once every crate is compiled, the [linker](https://en.wikipedia.org/wiki/Linker_(computing)) combines the separate parts of your program and libraries like the [Rust Standard Library](https://doc.rust-lang.org/std/) or Bevy into a single binary file that can be executed by your computer. It will also throw away the parts of the code that aren't actually used to keep the binary small.

Rust has a small tool called [rustup](https://rustup.rs/) that helps you set everything up. After you have followed the instructions, you have all the basic tools to get started. To check whether everything works, go into your development directory and execute the following:

```powershell
cargo init roguelike-overexplained
cd .\roguelike-overexplained\
cargo run
```

`cargo init` should create a new directory named `roguelike-overexplained`, initialize a [git](https://git-scm.com/) repository inside it and place the code for a very basic application there. (If you are already inside a git repository, cargo will skip the git part and leave the existing repository alone.) `cargo run` compiles the application, links the binary and runs it. After a few seconds of compiling, the command line should print `Hello, world!`. ["Hello World"](https://en.wikipedia.org/wiki/Hello,_world) is an established tradition for a first program.

> [!IMPORTANT]
> If this does not work, do not continue but try to fix your Rust installation (e.g. by repeating the steps from rustup)!

### Cargo

Cargo is the Rust package manager. A package manager is a piece of software that is responsible for managing external libraries (external as in "not part of the standard delivery"). Other package managers you might have heard of are `pip` for Python or `nuget` for C#. Cargo in Rust expands on that by also being the de facto standard build system. It is your interface to the complete toolchain. If you want to read more about cargo, feel free to check out [The Cargo Book](https://doc.rust-lang.org/cargo/).

Cargo is controlled by your project settings in the `Cargo.toml` file. It is a [TOML](https://en.wikipedia.org/wiki/TOML) file. The file format is similar in syntax to INI files. It contains named sections and supports comments. If you are not confident that you can get the syntax right, use cargo commands to modify it. The `package` section describes the project. `edition` is the [edition](https://doc.rust-lang.org/edition-guide/editions/) of Rust. Since Rust updates very frequently - every 6 weeks - it needs a mechanism to cope with outdated code. Editions are Rust's solution to that at the source-code level; a new edition appears roughly every 3 years, and your code stays on the edition you pin here until you decide to migrate. For comparison, `C++` gets a new standard every 3 years and avoids breaking changes at the source level, while `C` updates even less frequently - the last few standards came about 6 years apart - and also maintains source-level compatibility.

The `dependencies` section is where your dependencies to other libraries are added (e.g. Bevy). There is a huge ecosystem of open source libraries available at [crates.io](https://crates.io/). Each library entry has the cargo command listed to add the library as a dependency or alternatively the entry that you need to add to your `Cargo.toml`.

### Git

`git` is a [software configuration management](https://en.wikipedia.org/wiki/Software_configuration_management) tool. Its main purpose is to track changes to the source code. Once the change is verified to be good, it is added to the controlled state (or committed). If the change is not successful, it can easily be reverted without the need to remember what was changed. For larger changes, a branch can be opened to track the changes there and merge them back into `main` only once you are satisfied that it is working.

As a software developer, you want to use some form of SCM as it gives you a security blanket for your changes. The default in Rust is git as it is currently the de facto standard SCM tool for software development. Have a look into the [Pro Git](https://git-scm.com/book/en/v2) book. You don't need to read the whole book, but having an understanding of the basics is vital for efficient software development. Additionally, I can recommend getting a graphical git client (e.g. [Sourcetree](https://www.sourcetreeapp.com/)). I personally use Sublime Merge - but it's not free. Other people believe that the command line is the only true access to Git; it is a steeper hill to climb, although it does give you more control.

### Editor / IDE

To write source code, you need a [text editor](https://en.wikipedia.org/wiki/Text_editor) that is capable of editing plain text files. `Microsoft Word` or other [word processing programs](https://en.wikipedia.org/wiki/Word_processor#Word_processing_software) are not suitable for this task. A good text editor for programming provides at least [syntax highlighting](https://en.wikipedia.org/wiki/Syntax_highlighting) for your programming language to help you understand the source code using visual cues.

The next step up is an [Integrated Development Environment](https://en.wikipedia.org/wiki/Integrated_development_environment) for the programming language (e.g. [RustRover](https://www.jetbrains.com/rust/)). But the boundaries between text editor and IDE are blurred due to plugins extending text editors to the point where they provide (nearly) the same functionality.

If you don't know what I'm talking about, I'd recommend using [Visual Studio Code](https://code.visualstudio.com/) together with the [rust-analyzer](https://rust-analyzer.github.io/). You can find instructions on how to install it here: [Rust in Visual Studio Code](https://code.visualstudio.com/docs/languages/rust).


### The Source Code

The source code generated by `Cargo` is located in the `src` subfolder, in `main.rs`. This is not by accident, but part of the [cargo package layout](https://doc.rust-lang.org/cargo/guide/project-layout.html). `main.rs` indicates an executable. A library would live in a file called `lib.rs`.

`main.rs` looks like this:
```rust
fn main() {
    println!("Hello, world!");
}
```

Here we see three lines of code. Let's take them apart piece by piece.

1. `fn` indicates a [function](https://en.wikipedia.org/wiki/Function_(computer_programming)) - a callable sequence of instructions. A function has a name and can - but is not required to - have [parameters](https://en.wikipedia.org/wiki/Parameter_(computer_programming)) that are passed into the function. Additionally, it can have a [return value](https://en.wikipedia.org/wiki/Return_statement), the result of the function. The body of the function is enclosed in curly braces (`{}`). The function here has the name `main`, no parameters (indicated by the empty parentheses `()`) and no return value (indicated by the lack of a `->` after the parameters).
2. `main` is a special function. Although it can be called the same way as a regular function, it also indicates the entry point to the executable. The body of the `main` function is executed after the application has been properly initialized - the runtime hands over the command line arguments, sets up the machinery for panics and so on. Unlike in C++, there is no phase in which [global variables](https://en.wikipedia.org/wiki/Global_variable) are computed and assigned: a Rust `static` is a value the compiler has already baked into the binary. For most of your reasoning, `main` is the first thing that runs.
3. `{`: the beginning of the function body of `main`. Everything after the opening curly brace and before the closing curly brace is your main function.
4. `println!`: a Rust [macro](https://doc.rust-lang.org/book/ch20-05-macros.html) that prints text to the console. You can recognize that `println!` is a macro because of the `!` at the end - but there are other types of macros as well. Macros themselves are not functions. Functions are code whereas macros generate code. In this case, the macro "reacts" to the arguments you wrote down and generates the code that formats and prints them. Note that the macro itself only sees the text of your arguments, not their types - picking the right way to print a value happens in the generated code, through the traits that the value implements. Macros in Rust are fundamentally different from macros in C or C++ (the `#define` directive, which is plain text replacement). They are a form of [metaprogramming](https://en.wikipedia.org/wiki/Metaprogramming). They are simple to use safely and, unless you need to write one yourself, not very complicated.
5. `"Hello, world!"`: The argument for `println!` is a `string literal`. A string in programming is a sequence of characters to be used in the program. Here the intention is to print this character sequence on the command line via `println!`. In Rust, strings are [UTF-8](https://en.wikipedia.org/wiki/UTF-8) encoded. This means, every letter can be represented in Rust strings - even a mixed set of letters (e.g. Korean mixed with Cyrillic) can be represented. If you come from C++ or C, this is different as those languages make no assumption on the character encoding. Another thing to understand here is that UTF-8 means that 1 byte does not equal 1 character. To be able to represent all possible characters, UTF-8 encodes the character in sequences of variable length (1 byte up to 4 bytes). String literals in Rust are of type `&str` with `static` lifetime ([expr.literal.string.type](https://doc.rust-lang.org/reference/expressions/literal-expr.html#r-expr.literal.string.type)). `static` lifetime means that they exist as long as the program runs. `&str` is one of the two main string types (`String` being the other) and represents a slice of a string ([type.str.intro](https://doc.rust-lang.org/reference/types/str.html)). It can't be modified.
6. `;`: the semicolon ends the `println!` statement and is required.
7. `}`: the closing curly brace closes the function body of the `main` function.

When the compiler compiles the program, it generates the code for the main function. The `println!` macro is part of the Rust standard library and the compiler expands it before generating code. When linking everything together, the linker combines the code with the required parts of the standard library and then generates an executable. The "Hello, world!" text is not built at run time - it is stored verbatim in a read-only section of the executable, and the generated code only points at it. Starting the program therefore means: the runtime does its small amount of setup, then calls `main`.

## Setting Up Bevy

Bevy is a "data-driven game engine built in Rust" and it's open source, dual-licensed under [MIT](https://opensource.org/licenses/MIT) and [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0). For our game we want to use Bevy because it brings the functionality our game needs, like displaying graphics, reacting to keyboard events or playing sound.

To add Bevy to our project, we follow the instructions in the [Getting Started Guide](https://bevy.org/learn/quick-start/getting-started/):

```powershell
cargo add bevy
```

This will, at the time of writing, add Bevy 0.19 to your `Cargo.toml`. If you follow this tutorial later, you might get a newer version. The newer version might have an incompatible [API](https://en.wikipedia.org/wiki/API) and the tutorial might not work without adapting it to the new API. To prevent this, you can ask `cargo` to add version 0.19 using `cargo add bevy@0.19`.

Note that the entry `bevy = "0.19"` is not an exact version but a requirement: it means "any 0.19.x". What actually keeps your builds reproducible is the `Cargo.lock` file that cargo writes next to your `Cargo.toml`. It records the exact version of every single dependency that was resolved, and cargo will reuse those versions until you explicitly ask it to update. We will come back to that file when we commit our work to git.

Now call

```powershell
cargo build
```

This will rebuild your project with the new dependencies. As you will notice from the output, Bevy is not the only dependency that will be compiled. This is because Bevy itself relies on many other dependencies and these might again depend on other software. Because this gets complicated very quickly, modern software development relies on package managers to resolve all software dependencies.

A quick look into the `Cargo.toml` shows that the `dependencies` section was updated to contain Bevy (and only Bevy - the rest is managed by cargo).

```toml
[dependencies]
bevy = "0.19"
```

Adding the dependency manually to the `Cargo.toml` file is an alternative way to add a dependency.

### Optimizations

The Bevy startup guide [suggests](https://bevy.org/learn/quick-start/getting-started/setup/) adding the following lines to your `Cargo.toml` file.

```toml
# Enable a small amount of optimization in the dev profile.
[profile.dev]
opt-level = 1

# Enable a large amount of optimization in the dev profile for dependencies.
[profile.dev.package."*"]
opt-level = 3
```

These settings are optimization settings for your compiler. Modern computers are complex machines. To make the most of them without making software development unbearably complex, compilers do not translate the source code 1:1 into machine instructions, but apply optimizations instead. The goal of these optimizations is to transform what was written into something that the computer can execute faster - without changing the observable result. As long as a change is not observable within the rules of the language, the compiler is allowed to make it.

Here's a short program. After you understand the code, I'll show you what a compiler is allowed to do with it.

```rust
fn main() {
    let mut sum = 0;
    for n in 1..4 {
        sum = sum + n;
    }
    println!("{}", sum);
}
```

The [program](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=bfbf59545635419aa03ef38ca9d0f23a) calculates the sum of the numbers 1, 2 and 3 and prints the result. Follow the link to run it in the playground - leave your own `main.rs` untouched, we will need it as it is in a moment.

1. `let mut sum = 0`: this defines a [variable](https://en.wikipedia.org/wiki/Variable_(high-level_programming_language)) within the function. A variable is a space to store a value compatible with its [type](https://en.wikipedia.org/wiki/Type_system).
The [`let`](https://doc.rust-lang.org/std/keyword.let.html) keyword binds a value to a variable (here `0` to `sum`) and makes it available for usage within the [scope](https://doc.rust-lang.org/rust-by-example/variable_bindings/scope.html) (here denoted by the block enclosed by `{}` aka the function body of main).
The [`mut`](https://doc.rust-lang.org/std/keyword.mut.html) keyword (from mutable) indicates that you want to be able to modify that value. In Rust, variables are immutable by default and need to be marked mutable explicitly. (Immutable is not the same as a [`const`](https://doc.rust-lang.org/std/keyword.const.html) - that is a separate concept for values that are fixed at compile time. We'll get to it when we need it.) Classical programming languages have their default the other way round (e.g. in C++ `int a = 0;` is mutable, `const int a = 0;` is not). This is a trade-off in language design where "immutable by default" is geared toward fewer bugs and easier maintenance whereas "mutable by default" is geared toward easier writing.
The variable has a type, but the type is inferred automatically by the compiler. In this case, the compiler deduces the type from the literal `0`. Since the literal is [unconstrained](https://doc.rust-lang.org/rust-by-example/types/literals.html), it further tries to deduce it from its usage. If the usage does not add additional constraints, it is determined to be an `i32` for integer numbers or `f64` for floating-point numbers. This means that `sum` is of type `i32` - a 32 bit [integer](https://en.wikipedia.org/wiki/Integer_(computer_science)). Alternatively the type can be specified with the variable name (e.g. `let mut sum: i32 = 0`).
2. Integer data types are fixed-width data types, i.e. they occupy a fixed number of bytes. They store a whole number and come in the flavors "signed" (`i`) and "unsigned" (`u`). The following types exist in Rust:
    - i8: 8 bits or one byte storing values from -128 to 127
    - u8: 8 bits or one byte storing values from 0 to 255
    - i16: 16 bits or two bytes storing values from -32,768 to 32,767
    - u16: 16 bits or two bytes storing values from 0 to 65,535
    - i32: 32 bits or 4 bytes storing values from -2,147,483,648 to 2,147,483,647
    - u32: 32 bits or 4 bytes storing values from 0 to 4,294,967,295
    - i64: 64 bits or 8 bytes storing values from -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807
    - u64: 64 bits or 8 bytes storing values from 0 to 18,446,744,073,709,551,615
    - i128 and u128: 128 bits or 16 bytes, for the rare cases where 64 bits are not enough

    Additionally there are `isize` and `usize`. They are as wide as a memory address on the machine you compile for - 64 bits on a typical desktop computer today. You will meet `usize` early and often, because it is the type Rust uses for sizes and for indexing into collections.
3. `for n in 1..4`: the start of a [for-loop](https://doc.rust-lang.org/rust-by-example/flow_control/for.html). A for loop can be used to repeat the contents of the following block for a specific number of iterations. In this case `1..4` gives the number of iterations (from 1 up to, but excluding, 4) and is the equivalent to `for (int n=1; n<4; ++n)` in C++. `n` is the counting variable and contains the value of the current iteration.
4. `sum = sum + n;`: This statement adds `n`, the value of the current iteration of the `for`-loop to the current value of `sum` and then *replaces* the current value of `sum` with the new one. So in the first iteration, `sum` is 0 and `n` is 1 and after it has been executed, the new value of `sum` is 1. In the next iteration, `sum` is 1 and `n` is 2. Adding these together is 3 and this will then be the new value for `sum`. In the last iteration, `sum` is 3 and `n` is 3. After this line is executed, `sum` will be 6. So after the `for`-loop, the variable `sum` will contain the value 6.
5. `println!("{}", sum);`: This is another usage of the `println!` macro. The string to be printed is `"{}"` where the curly braces denote a placeholder. The placeholder will be replaced by the value of the variable passed as second argument (here `sum`). In our case, this will simply print `6` because the string contains only a placeholder. Alternatively, we could have written `println!("The sum of the numbers from 1 to 3 is: {}", sum);` and the output would have been: `The sum of the numbers from 1 to 3 is: 6`.

Now here are some optimizations that the compiler is allowed to do:

**loop unrolling**: the optimizer replaces the loop with the code for each iteration:
```rust
fn main() {
    let mut sum = 0;
    sum = sum + 1;
    sum = sum + 2;
    sum = sum + 3;
    println!("{}", sum);
}
```

**pre-calculating the value**: since the loop-execution is not visible from the outside, it can be replaced with the actual result
```rust
fn main() {
    println!("{}", 6);
}
```

There are many more optimizations that the compiler is allowed to do. All this is done to make your code run faster. But it also comes with a drawback. Take the last optimization for example. You would not be able to observe the loop iteration with a [debugger](https://en.wikipedia.org/wiki/Debugger) or the value change in sum - since the variable and the loop do not exist anymore.

This is why there are multiple compile profiles available. The `dev` profile reduces the amount of optimizations at the expense of runtime, to improve the debugging experience. The `release` profile has maximum optimizations to improve the runtime experience at the expense of the debugging experience and increased compile time.

The settings that the "Getting Started" guide proposes increase the optimizations for the Bevy dependencies, since most people do not want to debug/modify them and assume that they are more or less bug-free. Additionally, some optimizations are enabled for your own source code as well to increase the overall performance without diluting the debugging experience too much.

Since our project won't be graphics / performance heavy, both settings are probably unnecessary - but it's good practice to follow the Getting Started recommendations and to understand what they do.

When you change these settings, you need to recompile with `cargo build` for them to take effect.

## Updating the Source Code

To understand what's going on in the updated source code, an introduction to additional Rust concepts is required.

### Structs, Associated Functions & Methods

Rust allows you to create new data types by combining existing ones. The following example is taken from [Rust By Example](https://doc.rust-lang.org/rust-by-example/fn/methods.html):

```rust
struct Point {
    x: f64,
    y: f64,
}
```

The [example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=525bcceb519e5e5ec457a01ee5c6c9bb) defines a new data type named `Point`. The new data type consists of two independent fields named `x` and `y`. If you have a `Point`, the values of `x` or `y` can be accessed with the dot operator:

```rust
fn x(p: Point) -> f64 {
    p.x
}

fn main() {
    let p = Point { x: 1.0, y: 2.0 };
    println!("{}", x(p));
}
```

1. `fn x(p: Point)`: a function that takes a parameter with name `p` and type `Point`. Note that the `Point` is passed *by value*: the function takes ownership of `p`, so the caller cannot use `p` afterwards. We'll come back to ownership in a later part.
2. `-> f64`: the return value of the function is a single `f64`.
3. `p.x`: an expression that accesses the `x` of Point `p`. It uses the [dot operator](https://doc.rust-lang.org/nomicon/dot-operator.html). **Important**: the missing `;` at the end makes it the *tail expression* of the block, and a block evaluates to its tail expression - which is how the value gets returned from the function.
4. `let p = Point { x: 1.0, y: 2.0 };`: an immutable variable named `p` of type `Point` with `x` equal to 1.0 and `y` equal to 2.0.
5. `x(p)` calls the function `x` with argument `p`. It will return 1.0. That value is then used for the `println!` statement.

It is possible to associate functions with certain types. This indicates that the function belongs to that type and that its name is to be seen in that context. This is done using an `impl` block:

```rust
impl Point {
    fn origin() -> Point {
        Point { x: 0.0, y: 0.0 }
    }

    fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }

    fn distance_from_origin(&self) -> f64 {
        f64::sqrt(self.x * self.x + self.y * self.y)
    }
}
```

The [example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=a2b7e896ee1e18b68de926655312ef8c) above shows the two different kinds of function you can put into an `impl` block. `origin` and `new` do not take `self` as their first parameter - these are called *associated functions*. They belong to the type rather than to an instance of it and are called using the `::` syntax (`Point::new` or `Point::origin`). Rust has no constructors as a language feature; `new` returning a `Point` is simply a naming convention that everybody follows. `distance_from_origin` does take a form of `self` as its first parameter (here `&self`, a borrowed reference to the instance) - that makes it a *method*, and it can only be called when you actually have a `Point`, using the `.` operator (`let distance = p.distance_from_origin()`).

In `new`, note the `Point { x, y }`: when the field name and the variable name are identical, you can write the name once instead of `x: x`. This is called field init shorthand. The usage looks like this:

```rust
fn main() {
    let p = Point::new(1.0, 2.0);
    println!("{}", p.distance_from_origin());
}
```

### Bevy App

Since we didn't change the source code, `cargo run` will still display `Hello, world!` on the command line. To actually make use of Bevy, we need to modify our `main.rs`:

```rust
use bevy::prelude::*;

fn main() {
    App::new().run();
}
```

When you now `cargo run` the application, it starts and stops immediately and nothing happens. Let's understand what's happening in the program.

1. `use`: the [`use`](https://doc.rust-lang.org/std/keyword.use.html) keyword imports elements from other crates or modules and declares that you want to use them in this file. It allows the programmer to use a shorter name instead of the full name. In our example, the full name of `App` is actually `bevy::app::App`. By using the `use` declaration, this can be shortened.
2. `bevy::prelude::*`: a prelude for a library is a collection of the types that are most commonly used when using the library. It is a shortcut for the developer. The contents are described in the [prelude documentation](https://docs.rs/bevy/latest/bevy/prelude/index.html) of Bevy. Since we're only using `App`, we could have used `use bevy::app::App;` instead, and the application would still compile.
3. [`App::new()`](https://docs.rs/bevy/latest/bevy/app/struct.App.html): creates a new instance of `App` with sensible default values. It is Bevy's primary API for writing user applications.
4. `App::new().run();` first calls an associated function to create a new instance of `App` and then calls the method `run()` on the newly created instance.

What happens is that `App` gets initialized and then told to run. It recognizes that there is nothing scheduled to do and it exits.

### Open a Window

Change `main.rs` to the following code:

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .run();
}

fn setup(mut commands: Commands) {
    commands.spawn(Camera2d);
}
```

As you can see, `App` allows chaining of method calls. This works because each of these methods hands the `App` back to you (as a mutable reference), so the next `.` has something to attach to. The fact that each `.` operator is on a new line is not relevant - they are all part of the same statement. It ends with the `;`. This type of API is called [fluent interface](https://en.wikipedia.org/wiki/Fluent_interface).

What we can see here compared to the previous example are two new method calls on the `App` and a new function.

1. `add_plugins(DefaultPlugins)`: This installs a collection of plugins. A Bevy [plugin](https://bevy.org/learn/quick-start/getting-started/plugins/) is the way to organize code & functionality in Bevy. The [DefaultPlugins](https://docs.rs/bevy/latest/bevy/struct.DefaultPlugins.html) provide the most commonly used functionality required in games, e.g. for rendering, audio, handling input and more. This is also where our window comes from: opening it and keeping it alive is the job of the window and rendering plugins inside `DefaultPlugins`, not of anything we wrote ourselves.
2. `add_systems(Startup, setup)`: This adds a system to the scheduler. Systems are one part of the Entity Component System architectural pattern. They will be explained in the next post. For now it is sufficient to know that this call schedules the function `setup` to run during startup.
3. `fn setup(mut commands: Commands)`: a function that can be scheduled as a system by Bevy. [`Commands`](https://docs.rs/bevy/latest/bevy/prelude/struct.Commands.html) lets us request changes to what Bevy calls the [World](https://docs.rs/bevy/latest/bevy/prelude/struct.World.html) - the collection of "things" available to Bevy. Important: these requests are *deferred*. They are collected in a queue and only applied later, when Bevy reaches a point where it is safe to modify the world. So nothing has happened yet when the call returns. The parameter is declared `mut` because putting something into that queue modifies it.
4. `commands.spawn(Camera2d);`: Queues the spawning of a [2D camera](https://docs.rs/bevy/latest/bevy/camera/struct.Camera2d.html) entity that will be used by Bevy for rendering.

When we now run this changed source code, a window pops open and the application runs until this window is closed.

### Commit

Now that we are happy with what we achieved so far, we need to commit it to git. The following will use the command line to commit our source code. A GUI git client can achieve the same - look for "staging" changes and then "commit".

```powershell
git status
```

Running the above command gives you an overview of the changes in the repository. For me, it shows multiple untracked files. Untracked files are files that are within the directory managed by git, but that git doesn't know about yet.

**.gitignore**: this is a special file. It contains patterns of filenames that should be ignored by git. This is important if your IDE/editor adds subdirectories or if your build system (cargo) creates directories with build output. Cargo already added the important one for us: the `target` directory, where all build output ends up.

**Cargo.lock**: as mentioned above, this file records the exact version of every dependency. The recommendation by the Rust team is to commit it when developing applications and to leave it out when developing libraries. Since we're working on an application, it should be added to git.

**src/**: the directory containing our source code. The most important part to track.

**Cargo.toml**: the package configuration. The build relies on it and it is thus important as well. Needs to be tracked via git.

```powershell
git add .
```

Running this command will add all files to git for tracking (the `.` is the current directory, and git descends into it recursively, skipping everything that matches `.gitignore`). For files that git already tracks, the same command stages their changes for the next commit.

```powershell
git commit -m "Initial setup"
```

This commits the current state to git and gives it a message ("Initial setup"). The message should be meaningful, so that when you later read through the commits you get a rough idea of what was done.

```powershell
git log
```

This should now display information about the last commit: its hash (a 40-character hexadecimal string), which can be used to identify the change, the author, the date & time as well as the commit message. Once we have more commits, the command will display them as well.

## Conclusion

Today we learned the first basics of programming. How a program is created from source code via compiler & linker. How dependencies are managed using cargo. We had a first look at source code and its structural elements - functions and the special main function, for-loops, the dot operator, structs with their associated functions and methods. We successfully compiled our first program, then expanded it into a Bevy `App` that opens a window, and finally committed the result to git.

Next time, we will add elements to display and handle keyboard input.