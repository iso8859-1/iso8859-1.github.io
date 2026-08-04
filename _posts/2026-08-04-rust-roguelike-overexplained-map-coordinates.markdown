---
layout: post
title:  "Rust Roguelike Overexplained: Map Coordinates"
date:   2026-08-04 21:30:00 +0200
categories: [roguelike, rust, tutorial, bevy]
---
# Introduction

Today's post will introduce the concept of a map coordinates vs. rendering coordinates.

## AI for learning

If you talk to 10 software developers about AI and its code quality, you'll probably get at least 11 opinions. Everything in this field depends on the environment where it's used. In the context of this project, I would advise using it for learning. That does **not** mean that you ask AI to do things and then read what it did. Instead, I would recommend that you try to write the code yourself. You can discuss whether your solution approach is viable and if you're stuck because your code does not compile, you can ask it to fix your approach. Or let it explain code. But, most important, always try to write the code yourself. This usage of AI reduces the frustration level about missing documentation or examples and reduced frustration means you're going to stay on the project longer. Even if you're successful in writing your code, you can ask AI to review your code and give hints how it can be improved. Especially, if you're coming from a different language like Java or C++, it is helpful to ask the AI how to make your code more idiomatic. Rust offers many possibilities that other languages don't have (e.g. pattern matching, if-expressions, ...) that are easy to miss.

# Step 1: Window size

To simplify our implementation, we will start with a fixed resolution / window size. This avoids that we need to adapt the camera and/or text sizes. It can always be added later.

```rust
fn main() {
    let mut default_plugins = DefaultPlugins.build();
    default_plugins = default_plugins.set(LogPlugin {
        level: bevy::log::Level::DEBUG,
        ..default()
    });
    default_plugins = default_plugins.set(WindowPlugin {
        primary_window: Some(Window {
            title: "Roguelike Overexplained".to_string(),
            resolution: (1920, 1080).into(),
            resizable: false,
            ..default()
        }),
        ..default()
    });
	App::new()
	.add_plugins(default_plugins)
	.add_systems(Startup, setup)
	.add_systems(Update, keyboard_input)
	.add_systems(Update, move_player)
	.run();
}
```

In our updated main function, the `DefaultPlugins` are not built inside the `add_plugins` call. This is easier to read due to the increased customization required.

1. `let mut default_plugins = DefaultPlugins.build();`: creates a [builder](https://en.wikipedia.org/wiki/Builder_pattern). The builder is then used to customize the `DefaultPlugins`.
2. the `LogPlugin` is configured using the `set()` method. The builder returns itself to allow chaining. Since we don't want to chain everything for readability, we re-assign the return value. Feel free to experiment with a chained version of the code and see which one you prefer to read.
3. The [`WindowPlugin`](https://docs.rs/bevy/latest/bevy/window/struct.WindowPlugin.html) configures Bevy's windowing support. It has options that control how the application can be exited, a primary window and how the cursor behaves. We set everything to default using `..default()` except the primary window. `primary_window` is an [Option<Window>](https://doc.rust-lang.org/core/option/enum.Option.html). `Option` is a generic type that is either its type inside the `<>` (here `Window`) or empty, represented by [`None`](https://doc.rust-lang.org/std/option/enum.Option.html#variant.None). Setting `None` here means that the application has no window. Since we want one, we set a window using [`Some`](https://doc.rust-lang.org/std/option/enum.Option.html#variant.Some). Inside `Some`, we construct [`Window`](https://docs.rs/bevy/latest/bevy/prelude/struct.Window.html) using the default settings except the `title`, the `resolution` and we set it to be not `resizable`.
4. `(1920, 1080)` is a tuple. `.into()` converts it into the required [`WindowResolution`](https://docs.rs/bevy/latest/bevy/window/struct.WindowResolution.html). This is part of how Rust supports type conversions. Bevy implemented [`From<(u32, u32)>`](https://docs.rs/bevy/latest/bevy/window/struct.WindowResolution.html#impl-From%3C(u32,+u32)%3E-for-WindowResolution) and the compiler generates `into()` on a `(u32, u32)` tuple for free.

# Step 2: Map-Math

Turn-based movement goes well with tiled maps. Up until now, we dealt with pixel coordinates with 0/0 being the center of the screen. Now we introduce map coordinates, each increment representing a new tile. We also move the origin (0/0) to the top left corner of the screen.

## Constants

```rust
const WINDOW_WIDTH: u32 = 1920;
const WINDOW_HEIGHT: u32 = 1080;
const MAP_WIDTH: u32 = 80;
const MAP_HEIGHT: u32 = 40;
const FIELD_SIZE: f32 = 24.0;
```

`const` introduces a constant. It can't be changed. This is similar to `#define` in C or `constexpr` values in C++. A constant is defined by using `const`, then the name followed by a colon and then its type. Afterwards you have to assign the value with `=`. It is a Rust convention to use capital letters for constants. This makes reading the code easier since you know by looking at a name that it's a constant. Improving readability is a big factor in writing maintainable code because code is read more often than written. The Rust compiler will also issue a warning: "warning: constant `test` should have an upper case name" for the line `const test: f32 = 0.0;`

Having these constants defined increases the readability since it gives numerical values a name. When reading the code, you don't need to remember whether `24.0` is the base value for hit points or the pixel size of the tile - you read the name `FIELD_SIZE` and know that it's the field size.

The next step is replacing all magic numbers with their constants, e.g. `font_size: FontSize::Px(FIELD_SIZE),`. As an added benefit, this means when you want to change the field size, you have to change only one location and not multiple.

## Calculations

```rust
fn map_to_screen_coordinates(map_x: u32, map_y: u32) -> Vec3 {
    let screen_x = -(WINDOW_WIDTH as f32 / 2.0) + (map_x as f32 * FIELD_SIZE) + (FIELD_SIZE / 2.0);
    let screen_y = WINDOW_HEIGHT as f32 / 2.0 - (map_y as f32 * FIELD_SIZE) - (FIELD_SIZE / 2.0);
    Vec3::new(screen_x, screen_y, 0.0)
}
```

Now we need a function that translates map coordinates into pixel coordinates. Instead of writing the calculations everywhere we need them, we put them into a function to use them everywhere. This principle has a name - it's called [**Don't Repeat Yourself**](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) or **DRY**. It's one of the most underestimated principles when starting to program. Again, if we need to change the calculation, we just need to change the function - and we don't need to find all places where we copied the code.

The math is straight forward: pixel 0/0 is in the middle, so we have to correct by half height or half width. Then we calculate the map coordinate times field size to get the pixel offset from the new origin. At the end, we need to correct for half a field since the characters are rendered with their middle at the render position. The different + and - in the x and y coordinations come from a change in direction of the y axis. We return a `Vec3` to use it in the translation. Note that `Vec3` does not only have x and y coordinates but also a z coordinate. In 2D rendering, this is used to indicate that objects are in front of others. We're currently not using it hence we set it to 0. This is likely to change later on.

> Note: `MAP_WIDTH * FIELD_SIZE` (80 * 24 = 1920) matches `WINDOW_WIDTH` exactly, but `MAP_HEIGHT * FIELD_SIZE` (40 * 24 = 960) is smaller than `WINDOW_HEIGHT` (1080). This means the border you'll draw below will touch the left and right edges of the window but leave a gap at the top and bottom. Keep this in mind - it's not a mistake in the math, just a reminder to pick `MAP_HEIGHT` deliberately once you care about the window being fully covered.

We can now immediately use this in our player creation:

```rust
	commands.spawn((
        Text2d::new("@"),
        TextFont {
            font_size: FontSize::Px(FIELD_SIZE),	
            font: default(),
            ..default()
        },
        TextColor(Color::WHITE),
        Transform::from_translation(map_to_screen_coordinates(2, 3)),
        Player,
    ));
```

## Map Border

To ensure that we calculated everything correctly, we'll draw a border of walls (`#`) around our map. This will immediately show whether our map grid is located at the right location. The following code goes into our setup function:

```rust
    for y in 0..MAP_HEIGHT {
        commands.spawn((
            Text2d::new("#"), 
            TextFont { 
                font_size: FontSize::Px(FIELD_SIZE), 
                font: default(),
                ..default()
                },
                TextColor(Color::WHITE), 
                Transform::from_translation(map_to_screen_coordinates(0, y)),
        ));
        commands.spawn((
            Text2d::new("#"), 
            TextFont { 
                font_size: FontSize::Px(FIELD_SIZE), 
                font: default(),
                ..default()
                },
                TextColor(Color::WHITE), 
                Transform::from_translation(map_to_screen_coordinates(MAP_WIDTH - 1, y)),
        ));
    }
    for x in 1..MAP_WIDTH - 1 {
        commands.spawn((
            Text2d::new("#"), 
            TextFont { 
                font_size: FontSize::Px(FIELD_SIZE), 
                font: default(),
                ..default()
                },
                TextColor(Color::WHITE), 
                Transform::from_translation(map_to_screen_coordinates(x, 0)),
        ));
        commands.spawn((
            Text2d::new("#"), 
            TextFont { 
                font_size: FontSize::Px(FIELD_SIZE), 
                font: default(),
                ..default()
                },
                TextColor(Color::WHITE), 
                Transform::from_translation(map_to_screen_coordinates(x, MAP_HEIGHT - 1)),
        ));
    }
```

These are two for loops. One iterates through all possible y coordinates, the other through all possible x coordinates. On each iteration, we place two walls (left/right or up/down respectively). Again, we make use of the function that translates between both coordinate spaces. The second loop omits the leftmost and rightmost characters since they've already been placed by the first loop.

> Note: since character glyphs aren't square, the walls along the x-axis look a bit spaced out. This can be corrected by either using square tile graphics or by adapting the code for non-square tiles. This will be addressed at a later point in time.

# Summary

Today, we drew a frame around our map. We introduced map coordinates based on tiles and simplified our work using a translation function between map and pixel coordinates as well as Rust constants to improve readability and maintainability.