---
layout: roguelike-post
title:  "Rust Roguelike Overexplained: Code Cleanup"
date:   2026-08-06 17:50:00 +0200
categories: [roguelike, rust, tutorial, bevy]
---
# Introduction

Nobody gets code right on the first try. That's why it's important to step away from implementing features and clean up the code. The goal is to make it easier in the future to implement more features. [Test Driven Development (TDD)](https://en.wikipedia.org/wiki/Test-driven_development) encodes this permanently in its workflow.

# Step 1: Non-Square Fields

To correct the funny look of our wall, we need non-square fields. Luckily that's easy. Short research showed that most terminals use a 2:1 ratio between height and width. 2:1 means 12 pixels for the width. I went with `16.0` instead of the strict `12.0` from the 2:1 ratio. Although not traditional, I prefer the slightly spaced look.

```rust
const FIELD_SIZE_X: f32 = 16.0;
const FIELD_SIZE_Y: f32 = 24.0;
```

Then our lookup function needs to be adapted to use both coordinates:

```rust
fn map_to_screen_coordinates(map_x: u32, map_y: u32) -> Vec3 {
    let screen_x = -(WINDOW_WIDTH as f32 / 2.0) + (map_x as f32 * FIELD_SIZE_X) + (FIELD_SIZE_X / 2.0);
    let screen_y = WINDOW_HEIGHT as f32 / 2.0 - (map_y as f32 * FIELD_SIZE_Y) - (FIELD_SIZE_Y / 2.0);
    Vec3::new(screen_x, screen_y, 0.0)
}
```

The x-coordinate calculation uses `FIELD_SIZE_X`, and y-coordinate uses `FIELD_SIZE_Y`.

The keyboard input system needs adaption too:

```rust
fn keyboard_input(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    query: Query<Entity, (With<Player>, Without<Movement>)>,
    mut commands: Commands
) {
    for player in query.iter() {
        let mut direction = Vec3::ZERO;
        if keyboard_input.pressed(KeyCode::KeyW) {
            direction.y += FIELD_SIZE_Y;
        }
        if keyboard_input.pressed(KeyCode::KeyS) {
            direction.y -= FIELD_SIZE_Y;
        }
        if keyboard_input.pressed(KeyCode::KeyA) {
            direction.x -= FIELD_SIZE_X;
        }
        if keyboard_input.pressed(KeyCode::KeyD) {
            direction.x += FIELD_SIZE_X;
        }
        if direction != Vec3::ZERO {
            debug!("Player {player} moving toward {direction:?}");
            let movement = Movement { target: direction };
            commands.entity(player).insert(movement);
        }
    }
}
```

The direction vector needs to change differently, depending on the axis.

> **Disclaimer**: I re-created the code from the Day 4 code. It might not work as posted here. It will change later during the clean-up.

As a last step, every `font_size: FontSize::Px(FIELD_SIZE),` line needs to change to `font_size: FontSize::Px(FIELD_SIZE_Y),`. Luckily we had constants and by changing the name of the constant, the compiler will tell us which places to change.

> This differs from other languages like JavaScript or Python, where you don't have a compiler. In this case, you have to search for all places that change.

# Step 2: Z-Coordinates

Up until now, we used always the same z-coordinate. But we can use it to our advantage. Higher z-coordinates mean (in Bevy, in 2D) that something is closer to the camera. This means that we can introduce `layers` to put entities of a certain type to ensure consistent rendering.

Right now, we will use 2 layers:

```rust
const TERRAIN_Z: u32 = 0;
const ENTITIES_Z: u32 = 1;
```

Terrain will render below entities like player & monster.

To use it, we update our coordinate translation function:

```rust
fn map_to_screen_coordinates(map_x: u32, map_y: u32, z_level: u32) -> Vec3 {
    let screen_x = -(WINDOW_WIDTH as f32 / 2.0) + (map_x as f32 * FIELD_SIZE_X) + (FIELD_SIZE_X / 2.0);
    let screen_y = WINDOW_HEIGHT as f32 / 2.0 - (map_y as f32 * FIELD_SIZE_Y) - (FIELD_SIZE_Y / 2.0);
    Vec3::new(screen_x, screen_y, z_level as f32)
}
```

our `spawn` commands take the appropriate constant when setting up the translation

```rust
Transform::from_translation(map_to_screen_coordinates(x, y, TERRAIN_Z))
```

# Step 3: Refactoring - Walls

[Refactoring](https://en.wikipedia.org/wiki/Code_refactoring) is a term that describes the process of re-structuring code without changing the behavior. We will do just that and reduce the amount of code duplications by pulling out a function for spawning walls:

```rust
fn spawn_wall(commands: &mut Commands, x: u32, y: u32) {
    commands.spawn((
        Text2d::new("#"), 
        TextFont { 
            font_size: FontSize::Px(FIELD_SIZE_Y), 
            font: default(),
            ..default()
            },
            TextColor(Color::WHITE), 
            Transform::from_translation(map_to_screen_coordinates(x, y, TERRAIN_Z)),
            MapTile,
    ));
}
```

this makes the `for`-loop in our `setup` function so much more readable

```rust
fn setup(mut commands: Commands) {
	commands.spawn(Camera2d);

	commands.spawn((
        Text2d::new("@"),
        TextFont {
            font_size: FontSize::Px(FIELD_SIZE_Y),	
            font: default(),
            ..default()
        },
        TextColor(Color::linear_rgb(1.0,0.0, 0.0)),
        Transform::from_translation(map_to_screen_coordinates(2, 3, ENTITIES_Z)),
        MapPosition { x: 2, y: 3 },
        Player,
    ));

    info!("Player spawned");

    for y in 0..MAP_HEIGHT {
        spawn_wall(&mut commands, 0, y);
        spawn_wall(&mut commands, MAP_WIDTH - 1, y);
    }
    for x in 1..MAP_WIDTH - 1 {
        spawn_wall(&mut commands, x, 0);
        spawn_wall(&mut commands, x, MAP_HEIGHT - 1);
    }
}
```

# Step 4: Refactoring - Movement

Our first movement implementation was a little bit wonky. Now that we have map coordinates to work with, we will change it.

1. We store a `MapPosition` component. This component stores the x and y coordinates in map-space for the entity.
2. `Movement` degrades to a tag-type that just indicates that the movement animation is on.
3. The keyboard handling immediately updates the `MapPosition` of the entity.
4. The `move_entity` (renamed from `move_player`) function calculates the updated translation from the current position to the target position.

```rust
#[derive(Component)]
struct Movement;

#[derive(Component, Debug, Clone, PartialEq, Eq, Default)]
struct MapPosition {
    x: u32,
    y: u32,
}
```

The updated components. `Movement` lost its members. `MapPosition` has integer components in map coordinates for x and y. The `derive` statement adds a few common things to it.

- `Component`: this is the standard marker needed by Bevy for a component.
- [`Debug`](https://doc.rust-lang.org/rust-by-example/hello/print/print_debug.html): this trait allows the `MapPosition` to be printed - with the `?` that we had earlier for `Vec3`.
- [Clone](https://doc.rust-lang.org/std/clone/trait.Clone.html): provide copy functionality.
- [PartialEq](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html): provides comparison via `==`(equality) or `!=` (not equal).
- [Eq](https://doc.rust-lang.org/std/cmp/trait.Eq.html): this is on top of `PartialEq`. It requires the mathematical requirements (symmetric, transitive, consistent) to be fulfilled. This isn't true for floating-point values because of `NaN` (remember `a != a` is true if `a` is `NaN`).
- [Default](https://doc.rust-lang.org/std/default/trait.Default.html): provides a `default()` function to generate an instance. The members are initialized with 0 in this case.

next is the updated keyboard handling

```rust
fn keyboard_input(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut query: Query<(Entity, &mut MapPosition), (With<Player>, Without<Movement>)>,
    mut commands: Commands
) {
    for (entity, mut map_position) in query.iter_mut() {
        let original_position = map_position.clone();
        if keyboard_input.pressed(KeyCode::KeyW) {
            map_position.y = map_position.y-1;
        }
        if keyboard_input.pressed(KeyCode::KeyS) {
            map_position.y = map_position.y+1;
        }
        if keyboard_input.pressed(KeyCode::KeyA) {
            map_position.x = map_position.x-1;
        }
        if keyboard_input.pressed(KeyCode::KeyD) {
            map_position.x = map_position.x+1;
        }
        if original_position != *map_position {
            debug!("Player {entity} moved to ({}, {})", map_position.x, map_position.y);
            commands.entity(entity).insert(Movement{});
        }
    }
}
```

1. The query now needs the `MapPosition` and the `Entity`. Since we potentially modify the `MapPosition`, it needs to be mutable. Both are in a tuple for the first parameter of the query.
2. The filter filters now for `(With<Player>, Without<Movement>)`. Keyboard should only be accepted for the player - other entities are not keyboard controlled - and only if it's currently not moving.
3. `let original_position = map_position.clone();`: we make a copy of the original position. This allows to to check whether the keyboard input resulted in a movement or not.
4. the `if` expressions update the `MapPosition` component of the entity.
5. `if original_position != *map_position {`: here we determine if the `MapPosition` component changed. The `*` is required because `map_position` is a [`pointer type`](https://doc.rust-lang.org/reference/types/pointer.html). If we detect a change, we add the `Movement` tag type. This will trigger the `move_entity` system.

```rust
fn move_entity(time: Res<Time>, mut query: Query<(&mut Transform, &MapPosition, Entity), With<Movement>>, mut commands: Commands) {
    for (mut transform, map_position, entity) in query.iter_mut() {
        let speed = 100.0;
        let delta = time.delta_secs();
        let target_position = map_to_screen_coordinates(map_position.x, map_position.y, ENTITIES_Z);
        let direction = (target_position - transform.translation).normalize_or_zero();
        let distance = (target_position - transform.translation).length();
        let movement_distance = speed * delta;
        if distance <= movement_distance {
            transform.translation = target_position;
            commands.entity(entity).remove::<Movement>();
        } else {    
            transform.translation += direction * movement_distance;
        }
    }
}
```

1. The filter of the query is not just `With<Movement>`. This way, the system can be used to animate **any** entity, as long as it uses the same way to describe a movement (`MapPosition` & `Movement` components).
2. We need `Transform` and `MapPosition` to calculate the new position. `Entity` is required to remove `Movement` once done.
3. `let target_position = map_to_screen_coordinates(map_position.x, map_position.y, ENTITIES_Z);`: calculation of the target position from the screen coordinates of the `MapPosition`.
4. `let direction = (target_position - transform.translation).normalize_or_zero();`: calculate the normalized direction.
5. `let distance = (target_position - transform.translation).length();`: calculation of the remaining distance to move.
6. `let movement_distance = speed * delta;`: calculate the distance that can be moved in that frame.
7. `if distance <= movement_distance`: if there's less distance to move to the target position than we can, we set the translation to the target position and remove `Movement`.
8. `else`: we change the translation by direction multiplied with our calculated distance that we can move in this frame.

# Step 5: Adding Unit-Tests

So far we've tested our code manually. This is simpler in the beginning, but not really sustainable. To ensure that everything **still** works, you would need to test a lot on a larger game. That is where automated tests in form of [unit tests](https://en.wikipedia.org/wiki/Unit_testing) come into play. `Cargo` & `Rust` have integrated support [out-of-the-box](https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html) for unit-testing.

You declare a section with [`#[cfg(test)]`](https://doc.rust-lang.org/book/ch11-03-test-organization.html). This tells rust that this section is only for testing. The code inside will be run when calling `cargo test` and not `cargo run`. Inside the section, tests are declared with `#[test]` before a function. When calling `cargo test`, all tests will be run. If you're using `VS Code`, you can also run single tests via the gui.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_map_to_screen_coordinates() {
        for x in 0..MAP_WIDTH {
            for y in 0..MAP_HEIGHT {
                let screen_coords = map_to_screen_coordinates(x, y, ENTITIES_Z);
                let (map_x, map_y) = screen_to_map_coordinates(screen_coords.x, screen_coords.y);
                assert_eq!((map_x, map_y), (x, y));
            }
        }
    }

    #[test]
    fn test_screen_to_map_coordinates() {
        let (reference_map_x, reference_map_y) = (5, 7);
        let center = map_to_screen_coordinates(reference_map_x, reference_map_y, ENTITIES_Z);
        let left_edge_x = center.x - FIELD_SIZE_X / 2.0;
        let bottom_edge_y = center.y - FIELD_SIZE_Y / 2.0;
        for x in 0..(FIELD_SIZE_X as i32) {
            for y in 0..(FIELD_SIZE_Y as i32) {
                let screen_x = left_edge_x + x as f32;
                let screen_y = bottom_edge_y + y as f32;
                assert_eq!(
                    screen_to_map_coordinates(screen_x, screen_y),
                    (reference_map_x, reference_map_y),
                    "mismatch at x={x}, y={y}"
                );
            }
        }
    }
}
```

I added this test section at the end of `main.rs`. It has two tests. The first test verifies that the round-trip from map to screen to map coordinate space works for **all** map coordinates. The second test verifies that all screen coordinates inside a tile map to the same tile.

For these tests to work, a screen to map coordinates function was needed as well:

```rust
fn screen_to_map_coordinates(screen_x: f32, screen_y: f32) -> (u32, u32) {
    let map_x = ((screen_x + (WINDOW_WIDTH as f32 / 2.0)) / FIELD_SIZE_X).floor() as u32;
    let map_y = (((WINDOW_HEIGHT as f32 / 2.0 - screen_y) / FIELD_SIZE_Y).ceil() - 1.0) as u32;
    (map_x, map_y)
}
```

There is one magic line: `use super::*;`. This is related to the Rust [module system](https://doc.rust-lang.org/reference/items/modules.html). Modules are required to organize the code. This allows for [encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)) and reduces the name clutter. In this case, `mod tests` creates a new sub module. By default, the code above is not visible in the sub-module. The `use super::*` imports everything (`*`) from the parent module (`super`). The tests can then access the functions and use them to test.

## AI

The second test was mainly written with the help of AI. The math is a bit tricky and I didn't feel like sitting down with pen and paper to fully figure it out. I created the tests and then let AI figure out the math. Afterwards, I reviewed the changes.

# Summary

We introduced non-square fields and then cleaned up the code. We reduced code duplication with extra functions, added the use of z-coordinates to ensure rendering order. We switched the movement to an easier system that relies on precise target coordinates and does not require rounded comparison. As a last step, we introduced first unit-tests to avoid regressions.