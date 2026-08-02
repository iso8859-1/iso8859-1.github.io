---
layout: post
title:  "Rust Roguelike Overexplained: Movement 1"
date:   2026-08-02 12:00:00 +0200
categories: [roguelike, rust, tutorial, bevy]
---
# Introduction

[Last time](https://iso8859-1.github.io/roguelike/rust/tutorial/bevy/2026/07/28/rust-roguelike-overexplained-getting-started.html) we setup our development environment and started a skeleton application in Rust using Bevy. Today, we will start displaying the traditional `@` for the player and use the keyboard to move it around.

## Entity Component System

Before we start programming, we need to understand the architectural pattern called "entity component system" or ECS. It is the backbone of Bevy. [Software architecture](https://en.wikipedia.org/wiki/Software_architecture) is the term we use when we describe how the different parts of the software interact with each other. A clean and good software architecture helps you reason about the software and makes it easy to extend it with desired functionality. Software architecture makes trade-offs - ideally the right ones - so that functionality you need can be added easily whereas other things can't be done easily.

ECS is a software architecture pattern that evolved in the gaming industry and their needs. It deals with Entities or "things" in your program that have properties or Components in ECS terms. They are processed using algorithms or Systems. The ECS programming pattern is great when you can't easily create strong hierarchies (as in this `is-a` ...) like in OOP. It allows free composition of Components in entities compared to OOP, that favors strong types with properties related to those types. Systems process matching entities i.e. entities that have the required components for the algorithm. A scheduler determines when each system is run.

Last time, we already saw an entity with components and a system.

```rust
fn setup(mut commands: Commands) {
    commands.spawn(Camera2d);
}
```

The setup function is a system (with no requirements). `.add_systems(Startup, setup)` scheduled it to run at Startup. `spawn` creates an entity. `Camera2d` is a [bundle](https://bevy-cheatbook.github.io/programming/bundle.html). Bundles are templates for creating entities with specific components.

# Display the Player

## Creating a Player Component

```rust
#[derive(Component)]
struct Player;
```

This declares that the struct Player is a new Bevy component. Last time, we learned that structs create new data types in Rust. This struct is empty and has no members - it only **is**. These empty types are called tag types. The only purpose of those types is to identify things. In terms of ECS, the player tag allows us to identify entities that are players and not - for example - monsters.

`#[derive(Component)]` invokes [macro-magic](https://doc.rust-lang.org/book/ch20-05-macros.html) again. Bevy uses this to generate the code it requires for components. This type of macro invocation is very common in Rust and used in many places. [`derive`](https://doc.rust-lang.org/rust-by-example/trait/derive.html) is used in many places to provide default implementations e.g., `#[derive(PartialEq)]` for an implementation that allows you to compare the type with `==`. Rust [Unit Tests](https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html) use `#[test]` to identify tests.  

## Displaying the @ Sign

Extend the setup function to look like this:

```rust
fn setup(mut commands: Commands) {
	commands.spawn(Camera2d);

	commands.spawn((
        Text2d::new("@"),
        TextFont {
            font_size: FontSize::Px(24.0),	
            font: default(),
            ..default()
        },
        TextColor(Color::WHITE),
        Transform::from_translation(Vec3::ZERO),
        Player,
    ));
}
```

The second `spawn` command uses a Rust [tuple](https://doc.rust-lang.org/std/primitive.tuple.html) as a bundle. Tuples are similar to structs - they combine multiple values, but have no name. If you create the same combination of types as tuple over and over again, consider giving it a name and making it a struct. Since we have only one call here, a tuple is sufficient. You can recognize the tuple by the extra pair of `()` compared to the `spawn(Camera2d)` call above.

> **Important**
> If you are not used to writing programs in Rust or any programming language for that matter, it is highly likely that you don't know what's important and what's not important to look at. I highly recommend you to **NOT** copy the examples from this blog post but type it. You will make mistakes (e.g. forgetting the extra pair of brackets or a semi colon, adding a space where none is allowed, ...) and you'll have to search for the difference between what was typed and what is written in the example. But that will train your brain to recognize what's important and what's not.
> Learning to program is in that sense similar to learning a language. Just because you can *read* and *understand* a language, does not mean you can *write* your own sentences.

The tuple consists of multiple elements:

1. a [`Text2d`](https://docs.rs/bevy/latest/bevy/prelude/struct.Text2d.html), created with the `new` function. It takes as parameter the text to be displayed - in our case the @ symbol.
2. a [`TextFont`](https://docs.rs/bevy/latest/bevy/prelude/struct.TextFont.html). It is constructed using the normal construction syntax with `{}` where the struct members are initialized in the form `member: value`. There is a specialty though - `..default()`. This means that the missing members of the struct are initialized with the values from the `default()` construction method.
3. [`TextColor`](https://docs.rs/bevy/latest/bevy/prelude/struct.TextColor.html) specifies a color for the text. In this case via `Color::WHITE`, a predefined constant. It can also be defined via the [Color](https://docs.rs/bevy/latest/bevy/color/enum.Color.html) class in various different ways.
4. [`Transform`](https://docs.rs/bevy/latest/bevy/prelude/struct.Transform.html) describes the position of an entity via translation, rotation & scale. In this case, it is generated via a construction method that simply takes a translation vector & sets the rotation to 0 and scale to 1.
5. [`Vec3::ZERO`](https://docs.rs/bevy/latest/bevy/prelude/struct.Vec3.html) is a constant of a 3 dimensional vector where all f32 floating point coordinates are 0. f32 is a 4-byte floating-point number. The other common floating-point format is f64. They differ in range and precision of the numbers they can handle. For more details, see [Wikipedia](https://en.wikipedia.org/wiki/IEEE_754).
6. Our `Player` tag-type to identify this entity as the player.

Feel free to play around with the code. Things you can try out:

1. change the value of the text
2. change the font height
3. change the color
4. change the position

revert the changes back once you feel confident about the effect of the different components.

### Floating-Point Knowledge

Floating-point numbers in computers allow storing values of a wide range. The precision that can be expressed depends on the value. The larger the value, the lower the precision. They are similar to the scientific notation with a fixed number of digits. Take this example with 3 digits precision: although 111E-3 is equal to 0.111 and thus can express values in the 1/1000 range, 1112 & 1113 are indistinguishable in this notation - they are both 111E1.

The [IEEE754](https://en.wikipedia.org/wiki/IEEE_754) standard defines many different variants, the most common one being float & double (or f32 and f64 in Rust). Each variant differs in the amount of [bits](https://en.wikipedia.org/wiki/Bit) they assign to the mantissa (the number of digits before the E in the scientific notation) and the exponent (the number after the E). But the encoding and rules for calculation stay the same for each format.

The representation & calculation rules result in some quirks that are worth knowing:

1. the closer to 0 you are, the more precise you can be. Float is approx. 7 digits precise at maximum, Double is approximately 16 digits precise.
2. Special numbers: IEEE754 supports special numbers - plus and minus infinity and "not a number" (NaN). These are "sticky" in a sense that you can't get them out of your calculation once they are in. Due to the special numbers, floating-point numbers are not ordered. Rust takes that seriously and it's by default not possible to sort an array of floating-points.
3. There is a positive and a negative 0
4. NaN is weird. It is not equal to itself.
5. Basic math rules do not necessarily apply due to rounding. a+a+a is not always equal to 3*a. This means you need to be careful when comparing values and usually, you need to apply a margin or error to account for rounding. Doing this right can be tricky.
6. IEEE754 has the same properties as the decimal representation. Decimal has fractions that are easily displayed with a finite number of digits like 1/10 and those that aren't (1/3), and IEEE754 has them as well - they are just not the same. This is due to one being based on a decimal number system, the other on a binary one.

More details in the great [Floating Point Guide](https://floating-point-gui.de/)

### Enums

Color mentioned above is an [enum](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html). Enums are cousins to structs. They are a way to define new data types. Whereas each field in a struct is available all the time, with enums only one field can be active at any given time. To make the difference more clear - envision all possible values that a struct can take and count them. The number of combinations is the **product** of the number of possible values for each field. For an enum, the number of combinations is the **sum** of all possible values for each option. Structs are product types, enums are sum types (thx to Ben Dean for his great talk ["Using Types Effectively"](https://www.youtube.com/watch?v=ojZbFIQSdl8) on the topic).

Bevy uses the Enum to represent the different ways colors can be specified.

# Movement

Now that we can display the player, we need to be able to move it. This requires additional things:

1. code that runs regularly (e.g. every frame) to update the picture.
2. code that reacts on inputs like the keyboard.

## Scheduler

A scheduler is a piece of software that runs code at a given time. In your OS, the scheduler distributes the CPU time to your different programs that are running (e.g. VS Code, your browser, music player, ...). Inside a process (roughly what you see as entries in your task manager), the computing time is distributed between different threads. Distributing work over multiple threads became important in the early 2000s with the advent of multi-core processors. Modern PC hardware always has multiple cores - my 7800X3D for example has 8 different cores. There's also the concept of [logical cores](https://en.wikipedia.org/wiki/Hyper-threading) - but for us right now, that's just a concept that CPU vendors implemented to use their hardware more efficiently. Important for us - Bevy takes care of utilizing multiple threads using its [Schedules](https://docs.rs/bevy/latest/bevy/prelude/struct.Schedule.html). We tell Bevy when certain systems need to be run and Bevy takes care of distributing them across multiple cores where possible. Possible Schedules are documented in the [App](https://docs.rs/bevy/latest/bevy/app/index.html) documentation.

## Moving the Player

According to ECS, we need a new system that is scheduled regularly. Bevy offers a schedule that runs with every frame called [Update](https://docs.rs/bevy/latest/bevy/app/struct.Update.html).

```rust
fn main() {
	App::new()
	.add_plugins(DefaultPlugins)
	.add_systems(Startup, setup)
    .add_systems(Update, move_player) //added code
	.run();
}
```
The call to `.add_systems(Update, move_player)` schedules the system `move_player` to be run with the Update schedule - every frame. Now we need a system called `move_player`. As we know, systems are Rust functions with a specific signature - depending on what you want to do.

```rust
fn move_player(time: Res<Time>, mut query: Query<&mut Transform, With<Player>>) {
    for mut transform in query.iter_mut() {
        let speed = 100.0;
        let delta = time.delta_secs();
        let direction = Vec3::new(1.0, 0.0, 0.0); // Move right
        transform.translation += direction * speed * delta;
    }
}
```

There is a lot to unpack here.

1. a function called `move_player`. It has parameters called `time` and `query`. `time` is constant and can't be modified, `query` is not. This information is important for Bevy because it uses it for parallelization (systems that modify the same component can't run in parallel). The system has no return value because it misses the `->` after the parameters.
2. `time: Res<Time>`: a parameter that references the [Bevy resource](https://bevy.org/learn/quick-start/getting-started/resources/) [`Time`](https://docs.rs/bevy_time/latest/bevy_time/struct.Time.html). Resources represent globally unique data in your application like time or textures.
3. `mut query: Query<&mut Transform, With<Player>>`: this is the matching magic for your system. The signature tells Bevy that the system runs on all entities with a `Transform` component that also have the `Player` component. The first element in angle brackets (`<>`) is the data that is queried. `&mut Transform` indicates that we want to modify the component. If the `mut` was missing, it would be an `immutable` query. [`With<Player>`](https://docs.rs/bevy/latest/bevy/ecs/prelude/struct.With.html) is a filter that ensures that we're only looking at entities that have the `Player` component. There are many other possibilities of filters like [Changed](https://docs.rs/bevy/latest/bevy/ecs/prelude/struct.Changed.html) and filters can be combined with operators like [Or](https://docs.rs/bevy/latest/bevy/ecs/prelude/struct.Or.html).
4. `<>`: The parameters contain many angle brackets. These indicate [generic data types](https://doc.rust-lang.org/book/ch10-01-syntax.html). In our case, `Res` and `Query` are generic data types i.e., they work together with multiple other data types. Let's have a look at [`Vec`](https://doc.rust-lang.org/std/vec/struct.Vec.html). `Vec` represents an array of data i.e. the data is indexable by element number and it is placed in memory one after another without space in between. Think of it like a row of boxes, each containing an item of the same type. How large the box needs to be depends on what's inside. That's where generic programming kicks in. `Vec` defines the behavior independent of the actual data type but the implementation details (like how large the box needs to be aka how much memory is required) are derived from the data type of the content. `Vec<f64>` behaves similarly to a `Vec<i32>`, but is different in the details.
5. `for ... in`: again, a loop that iterates over things one after another. In this case, it's not numbers. `query.iter_mut()` is a function call on the query and it returns the list of components matched by the query. The `_mut` indicates that the items can be modified. `mut transform` gives the current item in the iteration the name `transform`. Again, it is marked as mutable.
6. `let speed = 100.0;`: creates a constant with name `speed` and the value `100.0` and auto-deduces the type to `f32`. If you have a good development environment, it will show you the deduced type. Note that `f64` is actually Rust's default type for floating-point literals - `speed` is inferred as `f32` here only because it's later multiplied with `delta` (an `f32`) and a `Vec3` (which is built from `f32` components). Removing the `.0` will change the deduced type to `i32`.
7. `let delta = time.delta_secs();`: creates a constant with name `delta`. [`delta_secs()`](https://docs.rs/bevy/latest/bevy/time/struct.Time.html#method.delta_secs) returns the number of seconds as `f32` since the last `Update`.
8. `let direction = Vec3::new(1.0, 0.0, 0.0);`: Creates a new [vector](https://en.wikipedia.org/wiki/Vector_(mathematics_and_physics)) that represents the movement direction.
9. `direction * speed * delta`: Mathematically, this is a vector multiplied with two scalar values ([scalar multiplication](https://en.wikipedia.org/wiki/Euclidean_vector#Scalar_multiplication)). `speed` is in pixel / s. Multiplying it with `delta` (unit: seconds) gives the number of pixels the player moves. That number is then used to scale up the movement.
10. `+=`: `x += y` is a shorthand notation for `x = x + y`.
11. `transform.translation +=`: using [vector addition](https://en.wikipedia.org/wiki/Euclidean_vector#Addition_and_subtraction) this modifies the translation of the entity by adding the calculated movement vector.

**Short summary**: the new system works only on entities that have a `Transform` component and a `Player` component. It modifies the `Transform` component by a movement vector. The size of the movement vector is calculated from a speed in pixel/s and the duration since the last `Update`.

### Math Interlude: Movement Vector

In the previous section, we calculated a movement vector that is then added to the current translation to move the entity. The actual speed of the movement is the [length](https://en.wikipedia.org/wiki/Euclidean_vector#Length) of the vector. If you draw the vector onto a paper grid, it is the actual length of the arrow. If you analyze the calculation above closely, you'll notice a small potential bug. It only works because our direction vector has a length of 1. Taking a different vector e.g., `Vec3::new(1.0, 1.0, 0.0)` will result in a faster movement.

Bevy has a function associated to `Vec3` to help with that: [Vec3::normalize()](https://docs.rs/bevy/latest/bevy/prelude/struct.Vec3.html#method.normalize). Using this function, you can easily create the correct movement vector by first creating a vector to the intended target and then normalizing it before using it in the movement calculation. The length can also be calculated using [Vec3::length()](https://docs.rs/bevy/latest/bevy/prelude/struct.Vec3.html#method.length). If the calculation needs to be fast and you can live with the squared result, use [Vec3::length_squared()](https://docs.rs/bevy/latest/bevy/prelude/struct.Vec3.html#method.length_squared) instead. It avoids the costly square root calculation.

# Keyboard

## Controlling the Player

Now change `move_player` to look like this:

```rust
fn move_player(input: Res<ButtonInput<KeyCode>>, time: Res<Time>, mut query: Query<&mut Transform, With<Player>>) {
    for mut transform in query.iter_mut() {
        let speed = 100.0;
        let delta = time.delta_secs();
        //let direction = Vec3::new(1.0, 0.0, 0.0); // Move right
        let mut direction = Vec3::ZERO;
        if input.pressed(KeyCode::KeyW) {
            direction += Vec3::new(0.0, 1.0, 0.0); // Move up
        }
        if input.pressed(KeyCode::KeyS) {
            direction += Vec3::new(0.0, -1.0, 0.0); // Move down
        }
        if input.pressed(KeyCode::KeyA) {
            direction += Vec3::new(-1.0, 0.0, 0.0); // Move left
        }
        if input.pressed(KeyCode::KeyD) {
            direction += Vec3::new(1.0, 0.0, 0.0); // Move right
        }
        transform.translation += direction.normalize_or_zero() * speed * delta;
    }
}
```

The changes are:

1. a new `Resource` called `input`. It is a [`ButtonInput`](https://docs.rs/bevy/latest/bevy/input/prelude/struct.ButtonInput.html) resource. `ButtonInput` is generic - here it's parameterized with `KeyCode` for keyboard buttons, but it can also be used with `GamepadButton` to detect gamepad buttons.
2. `if ... {}`: a [conditional](https://en.wikipedia.org/wiki/Conditional_(computer_programming)). It indicates that the code in `{}` will only be executed if the condition `...` is true. Condition is an [expression](https://doc.rust-lang.org/reference/expressions.html) that returns a [boolean](https://doc.rust-lang.org/reference/types/boolean.html) value (a value that can be `true` or `false`). Please note - in Rust `if` conditionals are also expressions and produce a value. In other languages like C++ or Java, they are statements and do not produce a value. Rust's approach to this can produce very clean, easy to read code once you are used to this. In our case here, we don't use this property of `if`.
3. [`input.pressed(KeyCode::...)`](https://docs.rs/bevy/latest/bevy/input/prelude/struct.ButtonInput.html#method.pressed): a method that checks whether one or multiple buttons are pressed at the point of calling. In our case, we only check for one key on the keyboard. The function returns a `bool` and can thus be directly used as condition in the if-expression.
4. The bodies of the `if-expressions` add a direction vector to the initial vector `Vec3::ZERO`. This follows the normal vector addition mentioned above. Since all vectors that can be added have the same length, opposing vectors cancel out during the addition (e.g. up+down = 0).
5. `.normalize_or_zero()`: This normalizes the direction if possible (i.e. it does not have a length of 0) or returns the 0-vector otherwise. This ensures that diagonal movement is at the same speed as movement along one of the axes (e.g. up).

And with this, we can move our @ sign around.

# Summary

After a first short introduction to the main architectural pattern of Bevy - the Entity Component System - we created our first component. A tag component to designate the player. Then we attached multiple other components to it via a tuple to display an `@` sign. We brushed the topics of floating-point representation of numbers in Rust as well as enums, a sum type compared to struct, a product type.

Then we started moving the player. Using Bevy schedules, we introduced operations that run every frame and updated the `Transform` component of our player to move. This gave us a glance at vectors in math and we used them to describe the movement. We adapted the speed according to the frame-rate using the `Time` resource.

After we knew how to move our player, we hooked it up to the keyboard using another Bevy resource - `ButtonInput`. Using our knowledge of vector math and Rust conditional expressions (`if`), we changed the movement vector according to the keys pressed. Then we normalized it to make up for even speed in all directions.