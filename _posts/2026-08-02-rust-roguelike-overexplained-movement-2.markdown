---
layout: post
title:  "Rust Roguelike Overexplained: Movement 2"
date:   2026-08-02 17:30:00 +0200
categories: [roguelike, rust, tutorial, bevy]
---
# Introduction

Last time, we made the `@` sign move and follow our keyboard commands. But the movement itself wasn't the style of movement you'll expect in a turn-based roguelike. It was continuous as long as you pressed a movement key. In this part of the series, we're going to change to a more turn-based style.

## Disclaimer

So far, I stuck relatively close to other tutorials and thus was very confident that what I explained was correct. Now we're slowly but steadily leaving that realm. We'll make progressive use of ECS and its scheduling capabilities. I don't know whether the structure that I will propose will scale to game level - I just know that it works. I will explain my rationale and where it might give problems - but in the end, only the future will show whether the decisions were good.

# Turn-Based Movement

With turn-based movement, you issue a movement command via a single button click and then the movement continues until your `@` reaches the next field. In the original [**rogue**](https://en.wikipedia.org/wiki/Rogue_(video_game)), the graphics were a text terminal with rows and columns of characters. A character's position was always aligned to a row and column - it did not move smoothly between cells but jumped from one to the next instead. We have the benefit of a graphics engine and can move the `@` sign pixel by pixel smoothly. The following code will make use of this.

## Solution Outline

When a key is pressed, we don't move the character instantly but instead we will store a location where the character will move to. In a second system, we'll evaluate whether there is a target to move to and calculate the translation for this frame and move the character accordingly.

## Code

### Logging

[Logging](https://en.wikipedia.org/wiki/Logging_(computing)) is a technique in computer programming that helps narrowing down programming errors. It is done by inserting statements into the code that print out important information into a file or on console. Log-messages are usually associated with severities and logging can be configured to log only messages above a certain level. The levels in bevy are `trace`, `debug`, `info`, `warn` and `error` (in order of severity from lowest to highest).

```rust
use bevy::log::LogPlugin;

fn main() {
	App::new()
	.add_plugins(DefaultPlugins.set(LogPlugin {
	    level: bevy::log::Level::DEBUG,
	    ..default()
	}))
	.run();
}
```

The code above initializes logging to the console for your Bevy application. The `level` determines which messages are logged. In this case everything except `trace` is logged. 


### A New Component

```rust
#[derive(Component)]
struct Movement
{
    target: Vec3
}
```

As a first step, we need a new component that stores the movement. For now it simply stores the movement vector required to reach the target.

> This is most likely not the final solution. As you can see later, there is a potential rounding issue that can add up. Additionally, you'll most likely want to have the *destination* stored to interact with the map. For now, this is sufficient and once we deal with the map, it might change.

### Keyboard System

This time, we'll have a dedicated system for keyboard input. It is scheduled with every frame (`Update` schedule) for fast reaction time.

```rust
fn keyboard_input(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    query: Query<Entity, (With<Player>, Without<Movement>)>,
    mut commands: Commands
) {
    let field_size = 24.0;
    for player in query.iter() {
        let mut direction = Vec3::ZERO;
        if keyboard_input.pressed(KeyCode::KeyW) {
            direction.y += 1.0;
        }
        if keyboard_input.pressed(KeyCode::KeyS) {
            direction.y -= 1.0;
        }
        if keyboard_input.pressed(KeyCode::KeyA) {
            direction.x -= 1.0;
        }
        if keyboard_input.pressed(KeyCode::KeyD) {
            direction.x += 1.0;
        }
        if direction != Vec3::ZERO {
            direction *= field_size;
            debug!("Player {player} moving toward {direction:?}");
            let movement = Movement { target: direction };
            commands.entity(player).insert(movement);
        }
    }
}
```

1. `Query<Entity, (With<Player>, Without<Movement>)>`: This queries for the Entity and not a component of that entity. It does have an extended filter. You can read that filter as "every entity that has a `Player` component and at the same time does not have a `Movement` component". This means, although the system is scheduled every frame, it will not run the body of the function when there is either no player or the player is moving. We have seen [`With`](https://docs.rs/bevy/latest/bevy/ecs/query/struct.With.html) before. [`Without`](https://docs.rs/bevy/latest/bevy/ecs/query/struct.Without.html) is similar but checks for the absence of a component. Both conditions are combined with the [logical AND](https://en.wikipedia.org/wiki/Boolean_algebra#Basic_operations) by placing them in a tuple (indicated by the extra parantheses). If you haven't heard of boolean logic, now is a good time to read up on it. It will come up often in programming & computer science.
2. `mut commands: Commands`: gives us access to the command queue that allows us to modify things. Bevy requires this for inserting/removing components since it is a structural change.
3. `let field_size = 24.0;`: a local constant to make a field 24*24 pixel. It is always better to use named constants and not [magic numbers](https://en.wikipedia.org/wiki/Magic_number_(programming)). This makes the code more readable and easier to change.
4. `if`: determine the movement vector in "fields".
5. `if direction != Vec3::ZERO {`: run the following code only when there is a movement command issued.
6. `direction *= field_size;`: scaling up the vector according to the size of a field.
7. `let movement = Movement { target: direction };`: create a Movement component instance.
8. `commands.entity(player).insert(movement);`: schedule the insertion of the component into the player. After this line, the player entity will have a movement component. Since we excluded entities with `Movement`, this code will not be called again until the component is removed again.
9. `debug!("Player {player} moving toward {direction:?}");`: a message that helps debugging the code. It will be printed on the console. As an experiment, you should move the `debug!` statement before the `if` and see what happens. `{}` means that the element inside the brackets is not resolved to some readable representation. It does not print the contents literally. `{player}` for example prints the entity number. `direction` is a `Vec3` and `:?` selects the debug representation of `Vec3`. Omitting it also works, since `Vec3` also has a text representation. It will print the coordinates inside `[]` and leave out the type name.

**Summary**: The system reacts only on players without movement. If there is such an entity, it checks whether one or multiple of WASD keys are pressed. If so, it calculates a translation vector to the next field. Please note, this vector is not normalized - it points directly to the next field and *must* be longer with diagonal movement. It inserts this vector as part of a `Movement` component into player. This results in it not being called again until `Movement` is removed from player.

### Player Movement

This will react on Players that have `Movement` and `Transform` components.

```rust
fn move_player(time: Res<Time>, mut query: Query<(&mut Transform, &mut Movement, Entity), With<Player>>, mut commands: Commands) {
    for (mut transform, mut movement, entity) in query.iter_mut() {
        let speed = 100.0;
        let delta = time.delta_secs();
        let direction = movement.target;
        let displacement = if direction.length_squared()>1.0 {
            direction.normalize_or_zero() * speed * delta
        } else {
            direction
        };
        transform.translation += displacement;
        movement.target -= displacement;
        if movement.target.length_squared() < 0.1 {
            debug!("Player {entity} reached target");
            commands.entity(entity).remove::<Movement>();
        }
    }
}
```

1. we don't need the keyboard in this system.
2. `mut query: Query<(&mut Transform, &mut Movement, Entity), With<Player>>`: read this as every entity that has `Transform` and `Movement` filtered by components that have `Player`. `Transform` and `Movement` will be changed and the entity id should also be provided. Here the tuple is with the elements that we want to work with in the system. This is hard to understand. Read it multiple times and try to understand the differences between this query and the one in keyboard.
3. `mut commands: Commands`: access to the command queue as with the `keyboard_input`.
4. `let direction = movement.target;`: we use the direction vector stored in the `Movement` component
5. `let displacement = if direction.length_squared()>1.0 {`: check if the length is > 1.0. This can be done on the squared length to save some CPU cycles. Please note that here `if` is used as an expression that returns a value. For this to work the last lines in each part of the if body denoted by `{}` **must not** have a `;`.
6. `} else {`: run the following block encased in `{}` in case the condition of the if is **not** met.
7. The combined if-else expression returns either the normalized and speed adjusted vector for vectors with a length larger than 1.0 or the vector itself. That means if the distance is smaller than 1 pixel, we simply complete the movement.
8. `transform.translation += displacement;`: do the actual movement
9. `movement.target -= displacement;`: subtract what we moved from the way that we still need to go.
10. `if movement.target.length_squared() < 0.1 {`: if we are close to the target. This is to accommodate potential rounding errors during the floating-point calculations.
11. `debug!("Player {entity} reached target");`: emit a debug message
12. `commands.entity(entity).remove::<Movement>();`: issue a command to remove the `Movement` component. After this step is completed, the `keyboard_input` system will be able to react to keyboard inputs again.

This is still not perfect. It will not work if `delta` is large due to overshooting. But fixing this right now is not worth it due to upcoming changes. Try to imagine what happens if you're close to your target point but suddenly `delta` is very large.

### Schedule Setup

For completeness

```rust
fn main() {
	App::new()
	.add_plugins(DefaultPlugins.set(LogPlugin {
	    level: bevy::log::Level::DEBUG,
	    ..default()
	}))
	.add_systems(Startup, setup)
	.add_systems(Update, keyboard_input) //added code
	.add_systems(Update, move_player) //added code
	.run();
}
```

# Summary

We implemented step-wise movement. We did this by separating the keyboard logic from the rendering logic using an extra component. As a first step, logging was added to be able to extract information from the running program to help in finding bugs. As a next step, we added a new component. Then the separate systems were set up. Both with very different query logic. The systems "communicate" via the presence/absence of the `Movement` component - forming a small [state machine](https://en.wikipedia.org/wiki/Finite-state_machine). Inserting and removing components in Bevy is costly. In our case this does not matter since we do it rarely. In other cases, other Bevy mechanisms like [States](https://docs.rs/bevy/latest/bevy/prelude/trait.States.html) can be a better choice.

## Potentially Wrong Decisions

1. adding/removing components: it is costly. The reason for this approach is that this allows filtering in the query (which is fast). Otherwise the value inside the `Movement` component would have to be checked.
2. not storing the target point but the movement itself: this is most likely wrong - but good enough for now. Once we figured out how maps work, this will be changed.
3. allowing multiple key presses affect the movement: as user experience in turn based movement, you probably want a special key for diagonal movement. Otherwise the success of diagonal movement depends on the keyboards capabilities & user dexterity. This will be re-visited.
4. storing the un-normalized movement vector: since `Movement.target` holds the raw per-axis step, a diagonal target is `√2` times longer than an orthogonal one. Because `move_player` moves at a constant normalized speed, this means diagonal steps currently take noticeably longer (in real time) to complete than orthogonal ones - unlike in the previous part, where the input vector was normalized so that diagonal and orthogonal movement had the same speed. This falls out of point 2 above and should be revisited together with it.