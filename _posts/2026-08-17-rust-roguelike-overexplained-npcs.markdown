---
layout: roguelike-post
title:  "Rust Roguelike Overexplained: NPCs"
date:   2026-08-17 11:30:00 +0200
categories: [roguelike, rust, tutorial, bevy]
---
# Introduction

In this post, we will include our `Map` in a plugin and add some debugging-help. Then we extend our turn phases to NPCs, spawn an NPC and make it move towards the player.

# Bevy Plugins

On top of Rust `modules`, Bevy supports [`Plugins`](https://bevy.org/learn/quick-start/getting-started/plugins/). In essence, anything can be a `Plugin`, if it implements the interface. What you gain is encapsulation of initialization code for the ECS into a plugin.

```rust
pub struct MapPlugin;

impl Plugin for MapPlugin {
    fn build(&self, app: &mut App) {
        app.insert_resource(Map::default())
    }
}
```

This creates the `MapPlugin`. The struct is empty for now, but we could use it to pass configuration-data for our `Map` into the build function.

1. `impl Plugin for MapPlugin`: this section implements the [`Plugin` interface](https://docs.rs/bevy_app/latest/bevy_app/trait.Plugin.html) for our `MapPlugin` struct. The compiler will complain if we do not create everything required. Once we have done that, `MapPlugin` can be used as Bevy `Plugin`
2. `fn build(&self, app: &mut App)`: the function we are required to implement. It is a member function - that means we need to pass an instance of our plugin. It takes `&self` - that means that our plugin is used read-only. The second parameter is the `App`. That allows us to do our initialization.
3. `insert_resource`: is used to register our `Map`. Note: with parameters for `height` and `width` in our plugin, we could call `new` instead of `default()`. But this is an extension we can do later if required.

With the `MapPlugin` available, we can now `.add_plugins(MapPlugin{})` in our `main()` function.

> This doesn't look like a great benefit right now, but it will help us with the code structure in the long run.

# Spawning Monsters!

To spawn a monster, we do the same thing as for spawning a player. To distinguish monsters from players, we add a `Npc` component and not a `Player` component.

```rust
pub fn spawn_npc(&mut self, commands: &mut Commands, x: i32, y: i32, symbol: &str) {
    let e = commands.spawn((
        Text2d::new(symbol),
        TextFont {
            font_size: FontSize::Px(FIELD_SIZE_Y),
            font: default(),
            ..default()
        },
        TextColor(Color::WHITE),
        Transform::from_translation(map_to_screen_coordinates(x, y, ACTORS_Z)),
        MapPosition { x, y },
        Npc,
    ));
    self.add_entity(MapPosition { x, y }, e.id());
}
```

> Note: this code duplication is not good and needs to be refactored later. The common code between the spawn methods should be pulled out into a separate function. When a bug occurs in the common code, you only need to fix it once and can't forget another location.

# Moving Monsters!

The monster should move directly in our direction after we made our move. To do this, we add two new phases: `NpcAi` to select where to go and `NpcMovement` to animate the movement.

To select where to move, we check all 8 possible movement squares around the monster, discard those that are invalid and pick the one that is "closest" according to some definition of closest. How we determine "closest" influences the path the monster will take.

This method is not very intelligent behavior for our monsters but a sufficient start. It can easily be modified later on.

## Preparation - Math with `MapPosition`

To find the 8 surrounding `MapPosition`s, we could calculate them with loops and conditions based on our current position. Instead we will take the math approach.

```rust
const NEIGHBORS: [MapPosition; 8] = [
    MapPosition { x: 1, y: 1 },
    MapPosition { x: 1, y: 0 },
    MapPosition { x: 1, y: -1 },
    MapPosition { x: 0, y: -1 },
    MapPosition { x: -1, y: -1 },
    MapPosition { x: -1, y: 0 },
    MapPosition { x: -1, y: 1 },
    MapPosition { x: 0, y: 1 },
];
```

`NEIGHBORS` creates a static array (with so called `static` lifetime, i.e. it lives for the whole runtime of your program) with the deviations from the center position. This requires that we refactor our `MapPosition` from `u32` for x and y to `i32`.

Now that we have the neighbors, we can simply add our position to each of them and get our neighbors. To be able to add two types in Rust, you need to implement `Add` (and `Sub` for symmetry).

```rust
impl Add for MapPosition {
    type Output = MapPosition;

    fn add(self, other: MapPosition) -> MapPosition {
        MapPosition {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

impl Sub for MapPosition {
    type Output = MapPosition;

    fn sub(self, other: MapPosition) -> MapPosition {
        MapPosition {
            x: self.x - other.x,
            y: self.y - other.y,
        }
    }
}
```

Since addition could change the output type, you have to declare it with `type Output = MapPosition;`. Then you have to implement `add` and `sub`. The code should be easy enough to understand by now.

At last, we add a function that returns the neighbors of a given `MapPosition`:

```rust
pub fn neighbors_of(center: MapPosition) -> [MapPosition; 8] {
    NEIGHBORS.map(|offset| center + offset)
}
```

This needs explanation. 
1. `[MapPosition;8]`: this is an array with 8 elements of type `MapPosition`. You can't enlarge it or shrink it - it has exactly those 8 elements.
2. `NEIGHBORS.map()`: this is a call to the [`map` function of array](https://doc.rust-lang.org/std/primitive.array.html#method.map). The map function applies a function to each element of the array and returns the result as a new array. In our case, we want to add our position to each of the elements of the NEIGHBORS array.
3. `|offset| center + offset`: a rust [closure](https://doc.rust-lang.org/rust-by-example/fn/closures.html). These are unnamed functions that capture parts of their environment. `map()` requires a function that takes a `MapPosition` and returns a `MapPosition` (e.g. `fn foo(p: MapPosition) -> MapPositions`), but we want our `center` to be part of that function. We can't add a second parameter, otherwise `map()` would not know how to call it. Here is where the closure comes into play. It captures `center` and makes it part of the function. `|offset|` declares the function parameter to be called offset and the function body is `center + offset`. It could also have multiple lines, but in our case, it's not required. Since this is the last statement and it has no `;` at the end, the return value of the function is the value of the statement. That means the closure calculates what we want - it adds an `offset` to our `center` (using the `Add` trait implementation). `map()` calls this for every element in `NEIGHBORS` and the resulting array now contains all neighbors of `center`.  

The next thing we need, is our distance function. We implement it as associated function for `MapPosition`:

```rust
impl MapPosition {
    /// Calculates the Manhattan distance between two MapPositions.
    pub fn distance_to(self, other: MapPosition) -> i32 {
        (self.x - other.x).abs() + (self.y - other.y).abs()
    }
}
```

`abs()` calculates the absolute value of a number. The absolute value of a number is the number itself if it is positive or `-1` times the number if negative (i.e. the `abs()` of `-2` is `-1 * -2 = 2`).

The [Manhattan distance](https://en.wikipedia.org/wiki/Taxicab_geometry) describes the distance if you can only move single units up/down or left/right. This isn't actually what we're doing (the true distance would be the [Chebyshev distance](https://en.wikipedia.org/wiki/Chebyshev_distance)) - but it produces results that favor the diagonals and that's more what the user would expect.

> Once you have everything working, play around by changing the distance function to see the different behaviors caused by it.

We could have used the "normal" distance ([Euclidean](https://en.wikipedia.org/wiki/Euclidean_distance)), but that would have been more expensive to calculate for the same result.

All of this needs unit-tests as well. You can find them in the [repository](https://github.com/iso8859-1/roguelike-overexplained/blob/b856444b5eb623bf23df1b848251543ef1e37932/day_8/src/map.rs#L217)

## Finding the Player

To be able to move towards the player, the monster must be able to query the position of the player. To be able to do this, we need to extend `Map`.

1. Track the player Entity as field of the map: `player: Option<Entity>,`. Since we can't be sure that the player is always there in all states, it's an `Option`. When `spawn_player` is called, the `Option` is set.
2. Allow a reverse-lookup from `Entity` to `MapPosition` via a second `HashMap`: `    positions: HashMap<Entity, MapPosition>,`. All functions need to be updated to track both `HashMap`s.
3. Add functions to help with getting the player position.

```rust
pub fn player(&self) -> Option<Entity> {
    self.player
}

pub fn position(&self, e: Entity) -> Option<MapPosition> {
    self.positions.get(&e).cloned()
}

pub fn player_position(&self) -> Option<MapPosition> {
    match self.player() {
        Some(player) => self.position(player),
        None => None,
    }
}
```

Nothing fancy here - but you can see how small functions play together to compose to something bigger like getting the player position. Although by itself `player_position` doesn't do much, it's a function worth having - if the implementation changes in the future, you would need to search for all occurrences of that `match` block.

The `match` block itself shows how `Options` can be dealt with elegantly in Rust. `Some(player)` checks as condition whether the `Option` contains a value and makes that value available as `player` in the section after the `=>`. This allows calling the next helper function although it does not take an `Option` by itself.

Interesting is also the code for `update_entity_position`:

```rust
pub fn update_entity_position(
    &mut self,
    old_position: &MapPosition,
    new_position: MapPosition,
) {
    if let Some(entity) = self.entities.remove(old_position) {
        self.entities.insert(new_position, entity);
        self.positions.remove(&entity);
        self.positions.insert(entity, new_position);
    }
}
```

`if let Some(entity) = self.entities.remove(old_position)` means: if the `Option` returned by `self.entities.remove(old_position)` contains a value, then execute the block and assign the value to the name `entity`.

## Checking Collisions

```rust
pub fn check_collision(&self, pos: MapPosition) -> bool {
    self.entities.contains_key(&pos)
}
```

We also need to give our map a function to ask for collisions at a `MapPosition`. Luckily that's simply asking our `HashMap` whether the key is contained or not.

## Pulling Everything Together - Moving the Monster

We need a system to execute during the `NpcAi` phase:

```rust
fn npc_ai(
    mut map: ResMut<Map>,
    mut query: Query<&mut MapPosition, With<Npc>>,
    mut next_turn_phase: ResMut<NextState<TurnPhases>>,
) {
    for mut map_position in query.iter_mut() {
        //get player position
        if let Some(player_position) = map.player_position() {
            //calculate all possible next positions
            let mut neighbors: Vec<(MapPosition, i32)> = neighbors_of(*map_position)
                .into_iter()
                .filter(|pos| !map.check_collision(*pos))
                .map(|pos| (pos, pos.distance_to(player_position)))
                .collect();
            neighbors.sort_by_key(|(_, distance)| *distance);
            debug!("neighbors: {:?}", neighbors);
            let current_distance = map_position.distance_to(player_position);
            if let Some((next_position, distance)) = neighbors.first()
                && *distance <= current_distance
            {
                let original_position = *map_position;
                *map_position = *next_position;
                map.update_entity_position(&original_position, *next_position);
            }
        }
    }
    next_turn_phase.set(TurnPhases::NpcMovement);
}
```

1. We ensure that our query runs with every `Npc`.
2. `if let Some(player_position) = map.player_position()` we determine the player position. It should always be available but our function doesn't guarantee that - hence the `if`.
3. `let mut neighbors: Vec<(MapPosition, i32)> = ...`: a mutable `Vec` of tuples of MapPositions and an integer that stores the distance.
4. `neighbors_of(*map_position)`: that `Vec` is calculated from the array of neighbors from our position.
5. `.into_iter()`: then we turn our array into an iterator that can iterate over the array. The iterator provides an abstraction for different containers to walk through all items.
6. `.filter(|pos| !map.check_collision(*pos))`: filter out everything that does not match the condition. Our condition is a closure that returns `true` if the field is clear. `!` is the operator for `NOT`, negating a boolean. This is done lazily while iterating.
7. `.map(|pos| (pos, pos.distance_to(player_position)))`: This turns each `MapPosition` into a tuple with the `MapPosition` and the distance to the player.
8. `.collect();`: turns the iterator into a `Vec` containing the data. Up until that point, we simply chained lazy operations and nothing happened. `collect()` now actually iterates over the iterator and collects the data into a `Vec`.
9. `neighbors.sort_by_key(|(_, distance)| *distance);`: this sorts the `Vec` by distance - shortest distance first.
10. Next, we calculate the current distance and then try to find a point (`neighbors.first()` might be empty) with a distance smaller than the current distance. If so, we update the position of our monster. The `if let Some(...) = ... && *distance <= current_distance` is a so called [let-chain](https://doc.rust-lang.org/edition-guide/rust-2024/let-chains.html): it lets you combine a pattern match with a plain boolean condition in a single `if`, so the block only runs when both the `Option` contains a value *and* the extra condition holds - without it we would need a nested `if` inside the `if let`.
11. `next_turn_phase.set(TurnPhases::NpcMovement);`: always move to the next phase, otherwise the player will never be able to move again.

```rust
fn move_npc(
    time: Res<Time>,
    mut query: Query<(&mut Transform, &MapPosition), With<Npc>>,
    mut next_turn_phase: ResMut<NextState<TurnPhases>>,
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
            next_turn_phase.set(TurnPhases::PlayerInput);
        } else {
            transform.translation += direction * movement_distance;
        }
    }
}
```

`move_npc` is more or less a copy of `move_player`, except that it works with `Npc` and not `Player` and that the next phase is `TurnPhases::PlayerInput`.

And now the monster moves!

# Summary

At first, we made a `MapPlugin` to better organize our code. Then we added functionality to our `MapPosition` to be able to add and subtract them from each other. We learned about `map()` and `filter()` to calculate the neighbors of our current position. Using the Manhattan distance, we selected from our possible positions the "best". This was all done during the new `NpcAi`-phase.