---
layout: roguelike-post
title:  "Rust Roguelike Overexplained: Collision Detection"
date:   2026-08-11 21:18:00 +0200
categories: [roguelike, rust, tutorial, bevy]
---
# Introduction

So far, we have a player that we can move and walls that are initialized at startup. But the player can move through walls. As a next step, we prevent the player from moving through walls.

# Step 1: A Map

## General Idea

To find out whether we can enter a field or not, we need to check what's in that field. As of now, this can only be a wall. Since a wall is an entity, we could iterate over all entities and see if each entity's Transform component matches the location we want to go. This is a valid approach for the size of the map that we have and the amount of entities expected. Assuming that we can have terrain, actors and loot/items and every map field has all three of them, the maximum number of entities would be `120*40*3 = 14400` entities - probably much less. This is doable in iterations - but it's inefficient.

To increase the efficiency, we create a Bevy `Resource` called `Map`. The `Map` tracks the position of all entities for fast lookup. If you have the coordinates, you can check whether a field contains an entity or not. There are many viable approaches for this. 

Option 1: Reserving space for each field on the map and storing an `Option<Entity>` in that field. To reserve the space, you can use a `Vec` since the 2 dimensional coordinates can be mapped easily onto the linear array in `Vec` e.g., `index = x + y * map_width`.

Option 2: Use a dictionary datastructure that takes the position as key and returns the contents if found. Rust's `HashMap` is well suited for this task. It uses a mathematical function to map the key to an index and uses the index to look up the content. This is very fast, usually no search is required. In computer science this is called [`O(1)`](https://en.wikipedia.org/wiki/Big_O_notation), meaning the lookup takes constant time. This is the approach we will be using for now. We might switch in the future though - depending on future requirements. The main benefit of this approach is that it uses less memory for large maps.

> **Note**: The `HashMap` does not support multiple entries per key. To achieve that, you do not store the key itself but a vector of keys.

Option 1 and 2 both have the drawback that we need to keep them in sync with the Entities managed by Bevy.

## Implementation of the Map

```rust
#[derive(Component, Debug, Clone, PartialEq, Eq, Default, Hash)]
struct MapPosition {
    x: u32,
    y: u32,
}

#[derive(Resource)]
struct Map {
    width: u32,
    height: u32,
    entities: HashMap<MapPosition, Entity>,
}

impl Map {
    fn new(width: u32, height: u32) -> Self {
        Self {
            width,
            height,
            entities: HashMap::new(),
        }
    }

    fn width(&self) -> u32 {
        self.width
    }

    fn height(&self) -> u32 {
        self.height
    }

    fn add_entity(&mut self, position: MapPosition, entity: Entity) {
        self.entities.insert(position, entity);
    }

    fn remove_entity(&mut self, position: &MapPosition) {
        self.entities.remove(position);
    }

    fn get_entity(&self, position: &MapPosition) -> Option<&Entity> {
        self.entities.get(position)
    }

    fn update_entity_position(&mut self, old_position: &MapPosition, new_position: MapPosition) {
        if let Some(entity) = self.entities.remove(old_position) {
            self.entities.insert(new_position, entity);
        }
    }

    fn spawn_wall(&mut self, commands: &mut Commands, x: u32, y: u32) {
        let e = commands.spawn((
            Text2d::new("#"), 
            TextFont { 
                font_size: FontSize::Px(FIELD_SIZE_Y), 
                font: default(),
                ..default()
                },
                TextColor(Color::WHITE), 
                Transform::from_translation(map_to_screen_coordinates(x, y, TERRAIN_Z)),
                Terrain,
        ));
        self.add_entity(MapPosition { x, y }, e.id());
    }
}

impl Default for Map {
    fn default() -> Self {
        Self::new(MAP_WIDTH, MAP_HEIGHT)
    }
}
```

Now that's a lot of code. Let's go through this piece by piece.

```rust
#[derive(Component, Debug, Clone, PartialEq, Eq, Default, Hash)]
struct MapPosition {
    x: u32,
    y: u32,
}
```

This defines a struct with x and y coordinates as unsigned integers. The `derive` ensures
- that it is a Bevy `Component`
- that it can be printed for debugging purposes
- that it can be cloned via the `clone()` method to create a copy
- that it can be compared for equality and inequality
- that it is actually fully ordered
- that it has a `default()` method to construct it (with x and y being equal to 0)
- that it can be hashed, meaning that the special mathematical function for the HashMap is available.

```rust
#[derive(Resource)]
struct Map {
    width: u32,
    height: u32,
    entities: HashMap<MapPosition, Entity>,
}
```

This defines the struct. `#[derive(Resource)]` ensures that everything that's required for Bevy resources is generated by the compiler. The struct itself contains fields for `width` and `height` of the map as well as `HashMap<MapPosition, Entity>`. In the `HashMap` a `MapPosition` is used as key and a Bevy `Entity` is stored as value.

```rust
impl Map {
    //code
}
```

This block defines the methods associated with the `Map` type. We used those methods before when we called `App::new()` or `commands.spawn()`. Blocks like these define them. Depending on their signature, you call them either with `::` or with `.` (technically, any associated method can be called with `::` - but we will see that later).

```rust
fn new(width: u32, height: u32) -> Self {
    Self {
        width,
        height,
        entities: HashMap::new(),
    }
}
```

This is a construction method. Since the parameter-list does not contain `self`, it is called with `::`. Since it returns `Self`, it is a construction method. `Self` in this context means `Map`. The method constructs a `Map` type with a new, empty `HashMap` and the map-size as given in the parameters.

```rust
fn width(&self) -> u32 {
    self.width
}

fn height(&self) -> u32 {
    self.height
}
```

These two methods give access to `width` and `height` of the map. The first parameter is `&self`. That means that the function can be called with `.`. This is syntactic sugar to make the code more readable.

```rust
let map = Map::new(10,10);
let height = map.height();
let height2 = Map::height(map);
```

As you can see in the code snippet above, the second line with the `.` operator is less visual noise and easier to read. But semantically, line two and three mean the same. Whenever a method is callable with `.`, a variant of `self` is the first parameter. Both methods `height()` and `width()` take `self` as reference. This means that these methods do not change the state of the object instance. In C++ terms, these methods are `const`.

```rust
fn add_entity(&mut self, position: MapPosition, entity: Entity) {
    self.entities.insert(position, entity);
}
```

This method takes `&mut self` as first parameter. It changes the state of the map instance. It adds an Entity to the map by storing it in the `HashMap`. `remove_entity` does the inverse.

```rust
fn get_entity(&self, position: &MapPosition) -> Option<&Entity> {
    self.entities.get(position)
}
```

`get_entity` retrieves the `Entity` at a specific position. It returns an `Option` to be able to indicate that no `Entity` was found at that location. 

```rust
fn update_entity_position(&mut self, old_position: &MapPosition, new_position: MapPosition) {
    if let Some(entity) = self.entities.remove(old_position) {
        self.entities.insert(new_position, entity);
    }
}
```

This method "moves" an entity from one position to another. To do that, it looks up the entity from the `old_position` if available and inserts it at `new_positon`. Notable here is the `if let Some(entity) = ...` syntax. It means "if the `Option` returned by `remove()` is `Some` (aka not empty), then assign the value inside the `Option` to entity and execute the block". In languages like C++, this needs multiple lines.

```rust
fn spawn_wall(&mut self, commands: &mut Commands, x: u32, y: u32) {
    let e = commands.spawn((
        Text2d::new("#"), 
        TextFont { 
            font_size: FontSize::Px(FIELD_SIZE_Y), 
            font: default(),
            ..default()
            },
            TextColor(Color::WHITE), 
            Transform::from_translation(map_to_screen_coordinates(x, y, TERRAIN_Z)),
            Terrain,
    ));
    self.add_entity(MapPosition { x, y }, e.id());
}
```

This is our `spawn_wall` function, moved from a free function to a method of `Map`. The most notable change is that it calls `add_entity()` with the entity returned by `commands.spawn()` to keep the state between the ECS of Bevy and the `Map` in-sync.

```rust
impl Default for Map {
    fn default() -> Self {
        Self::new(MAP_WIDTH, MAP_HEIGHT)
    }
}
```

This block is a block to implement the methods for a [`Trait`](https://doc.rust-lang.org/rust-by-example/trait.html). `Traits` in Rust define contracts for common behavior. We have seen `Traits` in disguise before. `#[derive(Clone)]` is a macro to provide a default implementation for the [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html) trait. But you can also implement it yourself. In our case, we want the [`Default`](https://doc.rust-lang.org/std/default/trait.Default.html) trait to be implemented. We could have used `#[derive(Default)]`, but that would have resulted in a default-constructed `Map` having `height = width = 0`. Instead we want them to be our pre-calculated sizes from the constants. To achieve this, we implement `Default` ourselves. `impl Default for Map` means exactly what it reads. This block implements the `Default` trait for `Map`. We do this by providing the required method `default()` that returns a `Map` constructed with the predefined constants.

## Using the Map

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
            resolution: (WINDOW_WIDTH, WINDOW_HEIGHT).into(),
            resizable: false,
            ..default()
        }),
        ..default()
    });
	App::new()
	.add_plugins(default_plugins)
	.add_systems(Startup, setup)
    .add_systems(Update, keyboard_input)
    .add_systems(Update, move_entity)
    .insert_resource(Map::default())
	.run();
}
```

When setting up Bevy, we need to call `insert_resource()` with our `Map` to make it available via `Res<Map>` in our systems.

```rust
fn setup(mut map: ResMut<Map>, mut commands: Commands) {
	commands.spawn(Camera2d);

	let e = commands.spawn((
        Text2d::new("@"),
        TextFont {
            font_size: FontSize::Px(FIELD_SIZE_Y),	
            font: default(),
            ..default()
        },
        TextColor(Color::linear_rgb(1.0,0.0, 0.0)),
        Transform::from_translation(map_to_screen_coordinates(2, 3, ACTORS_Z)),
        MapPosition { x: 2, y: 3 },
        Player,
        Being,
    ));
    map.add_entity(MapPosition { x: 2, y: 3 }, e.id());

    info!("Player spawned");

    for y in 0..map.height() {
        let width = map.width();
        map.spawn_wall(&mut commands, 0, y);
        map.spawn_wall(&mut commands, width - 1, y);
    }
    for x in 1..map.width() - 1 {
        let height = map.height();
        map.spawn_wall(&mut commands, x, 0);
        map.spawn_wall(&mut commands, x, height - 1);
    }
}
```

Our setup function needs to add the player-entity to the map as well using `map.add_entity()`. The `Map` is available since we requested it as a mutable resource via `ResMut<Map>` in the system signature. When spawning our walls, we call `map.spawn_wall()`.

> **Note**: you need to call `map.height()` and `map.width()` outside the `spawn_wall` command due to Rust's [borrow checker](https://rustc-dev-guide.rust-lang.org/borrow-check.html). The borrow checker is part of Rust's magic. It enforces access restrictions to eliminate certain categories of errors completely without restricting the developer too much. One of the rules is that if there is a mutable reference taken, nobody else may access the object - not even for reading. `spawn_wall` is a mutating function so additional calls to members of `map` are disallowed. By moving them outside with `let height = map.height()`, we avoid this problem.
>
> To make it clear - the borrow checker is a great piece of software that prevents many bugs - but it will change how you program aka old programming patterns will not always work.

```rust
fn keyboard_input(
    mut map: ResMut<Map>,
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
            if map.get_entity(&map_position).is_some() {
                debug!("Player {entity} collided with an entity at ({}, {})", map_position.x, map_position.y);
                *map_position = original_position;
            } else {
                debug!("Player {entity} moved to ({}, {})", map_position.x, map_position.y);
                commands.entity(entity).insert(Movement{});
                map.update_entity_position(&original_position, map_position.clone());
            }
        }
    }
}
```

Our keyboard system needs to update the map. The decision here is to update the position on the `Map` before the movement is completed. This makes sense since, once started, the movement will complete. Logically, the entity is already at the new location - its display position just hasn't updated yet. In this method, we also check for the collision with a wall.

1. `if original_position != *map_position {`: we check only if the keyboard input calculated a new position. Otherwise we would detect a collision with ourselves.
2. `if map.get_entity(&map_position).is_some() {`: if there is actually something at the new position then we reset the `*map_position = original_position;` - no movement happened.
3. otherwise we insert `Movement` to start the animation and update the map with `map.update_entity_position(&original_position, map_position.clone());`. Here we need a copy / clone of the `map_position`.

# Summary

We implemented collision detection. To detect the collision, we implemented a Bevy resource to track the position of entities on the map and allow fast lookup via `HashMap`. Then we added the new type to our ECS and added the code for synchronization.

Interesting points that might change in the future:
- HashMap vs. Vec as representation for the `Map`
- Code Cleanup: main.rs is getting bigger. Additionally, `Map` is sufficiently separate to move it to a different file. This also benefits the structure since access restrictions can be added then.
- Unit-Tests: no unit-test were added. Also no additional debugging facilities. This should probably be done. There are examples how to unit-test with an ECS in the bevy examples but I still have to fully grasp their capabilities. Additionally, debug-system with sanity-checks could be added (e.g. verify that each entity is stored only once in the map). Running these regularly combined with error output would be helpful in finding synchronization bugs.

Code is available on [github](https://github.com/iso8859-1/roguelike-overexplained/tree/main/day_6)