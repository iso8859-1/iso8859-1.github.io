---
layout: roguelike-post
title:  "Rust Roguelike Overexplained: Preparing For Combat"
date:   2026-08-16 11:30:00 +0200
categories: [roguelike, rust, tutorial, bevy]
---
# Introduction

This chapter will again be code clean-up to prepare for npcs and combat. It will introduce a Rust [module](https://doc.rust-lang.org/rust-by-example/mod.html) to structure the code into more files and enforce some [encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)). In addition to that, we will introduce a [state machine](https://en.wikipedia.org/wiki/Finite-state_machine) to enable/disable systems. This will allow us to separate a turn into different phases and enable/disable systems based on the phase.

# Modules

Rust modules allow you to divide the source-code of a [crate](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html) into different files. Modules can also be created within a single file. We have seen this already when we introduced tests. This means, so far our project consists of a single crate with two modules. We will create a third module called `map`. This is where the code for the `struct Map` will be placed. Modules allow you to structure code and restrict access. Within a module, you have access to everything. Take for example the map width and height. The design of the struct assumes that it never changes after creation. But right now, we can't ensure that. Similarly, we provide functions to update the `HashMap` within the `Map`. But nothing prevents us from doing it without functions. This makes it hard to reason about problems e.g. when `Map` and `ECS` get out of sync. By moving the `Map` into its own module, we can restrict access from outside the module. If we want to change the internal implementation or if we need to find a bug, the changes are localized to the module.

## Creating a New Module

First, we need to add `mod map;` to main.rs. Then we create a new file called `map.rs` parallel to `main.rs`. You need both. If you forget `mod map;`, `cargo` will not pick-up the file and you'll get a warning. If you don't create `map.rs` (or any other [compatible source-code structure](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html#modules-cheat-sheet)), your module will be empty. Unlike `C` or `C++`, `Rust` enforces certain naming conventions where to find things. This reduces the effort for finding things (once you are used to the convention). People coming from `Java` know this already.

## Access Restrictions & Imports

After that, we move the code related to `Map` over to the new `map.rs` file. Once you have done that, nothing will compile. This has two reasons. 

1. The moved code isn't accessible anymore due to access restrictions
2. The moved code isn't visible anymore due to missing imports

Let's deal with visibility first. This is solved with `use` - exactly what we did before with `HashMap`.

```rust
use map::{Map, MapPosition, ACTORS_Z};
```

This imports from our `map`-module the `Map` resource, `MapPosition` component and `ACTORS_Z` constant. Depending on how much of the code you moved, you might need different imports. You can have a look at the way how I moved the code at [github](https://github.com/iso8859-1/roguelike-overexplained/tree/main/day_7). I didn't find a perfect split to move the code (yet). This is an indicator that the code still has deficits in the structure. Good indicators for a well designed module split are that the module has only one responsibility and that there are no circular `use` statements (aka it is self-contained). I haven't solved the second point due to a problem with the first point. The tile to coordinate calculation will probably be required outside the map as well - hence I left it in `main.rs`. This produces circular dependencies that I hope to resolve later when I tackle things like inventory.

And now accessibility. To make something accessible from other modules so that they can `use` it, it needs to be marked `pub`.

```rust
pub const ACTORS_Z: u32 = 1;
```

makes `ACTORS_Z` accessible so that it can be imported with `use` as shown above. Similarly

```rust
#[derive(Component, Debug, Clone, PartialEq, Eq, Default, Hash)]
pub struct MapPosition {
    pub x: u32,
    pub y: u32,
}

#[derive(Resource)]
pub struct Map {
    width: u32,
    height: u32,
    entities: HashMap<MapPosition, Entity>,
}
```

makes `MapPosition` and `Map` accessible. Please note the difference between the members in `MapPosition` and in `Map`. With `MapPosition`, the struct itself is accessible **and its members x, y as well**. This is due to them being marked `pub` as well. `Map` is different. Neither `width`, `height` nor `entities` are accessible directly. You need to use associated functions that are marked `pub` to access / modify them.

```rust
impl Map {
    pub fn new(width: u32, height: u32) -> Self {
        Self {
            width,
            height,
            entities: HashMap::new(),
        }
    }

    pub fn width(&self) -> u32 {
        self.width
    }

    pub fn height(&self) -> u32 {
        self.height
    }
...
```

The construction function `new()` is marked `pub`. This allows the construction of `Map` objects. The functions `width()` and `height()` give read-only access to the members `width` and `height` (remember the difference between `&self` and `&mut self`). This means, once a `Map` is created, no one can modify `width` and `height` from the outside - but they can read it.

## Further Clean-Up

Since we already have `spawn_wall()`, I created a `spawn_player` function.

```rust
pub fn spawn_player(&mut self, commands: &mut Commands, x: u32, y: u32) {
    let e = commands.spawn((
        Text2d::new("@"),
        TextFont {
            font_size: FontSize::Px(FIELD_SIZE_Y),	
            font: default(),
            ..default()
        },
        TextColor(Color::linear_rgb(1.0,0.0, 0.0)),
        Transform::from_translation(map_to_screen_coordinates(x, y, ACTORS_Z)),
        MapPosition { x, y },
        super::Player,
    ));
    self.add_entity(MapPosition { x, y }, e.id());
}
```

This reduced the coupling between `main.rs` and `map.rs`. Please note, this uses `map_to_screen_coordinates`, which is still in `main.rs`. Submodules can use everything from their parent directly. This is the reason why our tests as submodules were able to test everything without access-restrictions. Here we use it until we can clarify what to make of these functions. This results in `use super::{map_to_screen_coordinates, FIELD_SIZE_Y};` in my `map.rs`

# State-Machines & Bevy

State machines are a useful tool to describe program mechanics. They consist of `states` and `transitions` between states. Bevy supports state-machines nativly with the [state](https://docs.rs/bevy/latest/bevy/state/index.html) module. You can easily define your states like this:

```rust
#[derive(States, Debug, Clone, PartialEq, Eq, Hash, Default)]
enum TurnPhases {
    #[default]
    PlayerInput,
    PlayerMovement,
}
```

as an enum that `derive`s from `States`. Additionally you need to `derive` from `Clone`, `PartialEq`, `Eq`, `Hash`, `Debug` and `Default`. The inital state is marked with `#[default]`. Here we define a small state machine (that we plan on expanding later on) that distinguishes between `PlayerInput` - the phase where we wait for keyboard input - and `PlayerMovement` - the phase where the player moves and keyboard input is ignored.

Once that is done, we register the state and tie the systems to the state they run in.

```rust
	App::new()
	.add_plugins(default_plugins)
	.insert_state(TurnPhases::PlayerInput)
	.add_systems(Startup, setup)
    .add_systems(Update, keyboard_input.run_if(in_state(TurnPhases::PlayerInput)))
    .add_systems(Update, move_entity.run_if(in_state(TurnPhases::PlayerMovement)))
    .insert_resource(Map::default())
	.run();
```

`insert_state` adds the state to Bevy. `run_if` combined with `in_state` configures the scheduling of the systems according to the state. This is in essence the same thing we achieved with the `Movement` component.

The signature of our systems need to change as well, since they modify the state:

```rust
fn move_entity(
    time: Res<Time>,
    mut query: Query<(&mut Transform, &MapPosition), With<Player>>,
    mut next_turn_phase: ResMut<NextState<TurnPhases>>, //new parameter
) {
    for (mut transform, map_position) in query.iter_mut() {
        let speed = 100.0;
        let delta = time.delta_secs();
        let target_position = map_to_screen_coordinates(map_position.x, map_position.y, ACTORS_Z);
        let direction = (target_position - transform.translation).normalize_or_zero();
        let distance = (target_position - transform.translation).length();
        let movement_distance = speed * delta;
        if distance <= movement_distance {
            transform.translation = target_position;
            next_turn_phase.set(TurnPhases::PlayerInput); //new state transition
        } else {
            transform.translation += direction * movement_distance;
        }
    }
}
```

The parameter `next_turn_phase` gives us now the option to switch to a new state. In `move_entity` we do this when the movement is done and we transition back to the `PlayerInput` phase.

## Things we didn't use about states

States in Bevy are very powerful. In addition to modify when systems are run, they also offer additional `Schedule`s to run systems. Bevy creates a schedule before entering and after exiting a state as well as for specific state transitions (from x to y). To do this, you wrap the registration of a system in `add_systems` into an `OnEnter`, `OnExit` or `OnTransition` (e.g. `.add_systems(OnEnter(TurnPhases::PlayerInput), my_system)`. Check this [chapter](https://bevy-cheatbook.github.io/programming/states.html) on states for more information. 

# Summary

Today was a clean-up day again. 

First, we used Rust modules to make our code more readable & findable. The code for `Map` moved into a new `rs`-file. Additionally, we used Rusts access restriction system to encapsule the functionality of the Map there. The goal is that internals of the `Map` stay internal details and can't be messed with from the outside. Read-only access to width and height via access functions reduces the amount of code we need to search through if we have a bug there. Encapsulating `HashMap` as internal detail gives us the option to change it later to a flat array without too many changes in the rest of the program.

As a second step, we used Bevy's state machine mechanism to start splitting up our Turn into different phases. We use only a phase for player input and a phase for movement, but later we can add phases for npc ai and npc movement as well as phases for different ui states like inventory shown or targeting mode for spells.
