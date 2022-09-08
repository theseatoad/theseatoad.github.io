---
title: "Rust For Game Development #3 -- Game of Life"
date: 2022-08-30T20:03:00-07:00
publishdate: 2022-08-30
lastmod: 2022-08-30
draft: false
tags: ["Rust", "Game Development", "Bevy"]
categories: []
---

## Prelude

Follow along this tutorial using the cooresponding repository on [Github](https://github.com/theseatoad/conway_game_of_life). Snippets here will highlight important structs and methods, but for compiling code, refer to the link above.

This post is highlighing how the game logic and game state can be abstracted away from the rendering of the game. It is a good habit to decouple the code in this manner as it will be easier to iterate on either systems as well as writing tests. 

There are two flaws in this repository. Firstly, each game tick, the gui redraws the entire board. Secondly, during each game tick, the world iterates over each tile which results in many useless calculations.

## Introduction

One good habit to get into while starting a new game is seperating game logic and game rendering. It is very easy to couple code between these two systems which can result in many bugs down the line. Additionally, bugs will become hard to fix as these sytems will touching many different spots of code. Having two completely seperate systems is one robust way to avoid this class of bug.

For this implementation of [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life), the project will be split into two seperate directories. The first one named cgol will be in charge of the implementation of the rules, tiles, and world. Secondly, the gui directory will be in charge of reading from cgol, and displaying the state of the world.

## Implementing Conway's Game of Life Backend

Let's first initalize our game of life crate.
```
$ cargo new cgol
```

Since this will serve purely as a library for our gui to use, we will need to replace main.rs with lib.rs. This is the convention for writing public code for distribution.

There are two main structures in Conway's Game of Life. There is a world struct containing both the width of the board and the list of tiles, and secondly, a tile struct containing it's state.

```rust
pub struct Tile {
    pub on: bool,
}

pub struct World {
    pub width: i32,
    pub tiles: Vec<Tile>,
}
```

With these two structs created, the rules can now be implemented. The logic of the game is the same every single time; The world iterates over each tile, checking it's state and its neighbour's state, and applying three simple rules. The rules are as follows:

1. Any live cell with two or three live neighbours survives.
2. Any dead cell with three live neighbours becomes a live cell.
3. All other live cells die in the next generation. Similarly, all other dead cells stay dead.

To accomplish this, a function implementation on the tile struct is a good strategy.

```rust
pub fn evolve(
    self,
    left: Option<&Tile>,
    up: Option<&Tile>,
    right: Option<&Tile>,
    down: Option<&Tile>,
    topleft: Option<&Tile>,
    topright: Option<&Tile>,
    bottomleft: Option<&Tile>,
    bottomright: Option<&Tile>,
) -> Tile
```

Optionals is useful here as in this game of life implementation, the world has borders. Because of this, there are many cases where some of the surrounding tiles do not exist. The next step is counting how many alive neighbours there are. Lastly, return a tile with the respective state based on the rules. Below is the code that is responsible for enforcing the rules.

```rust
if self.on == true {
    if alive_neighbours == 2 || alive_neighbours == 3 {
        Tile::new(true)
    } else {
        Tile::new(false)
    }
else {
    if alive_neighbours == 3 {
        Tile::new(true)
    } else {
        Tile::new(false)
    }
}
```

Great. Now, it is useful to create a method on the world struct that iterates over all of the tiles and "evolves" them.

```rust
pub fn tick(&mut self) {
    let mut new_world: Vec<Tile> = Vec::new();
        for y in 0..self.width {
            for x in 0..self.width {
                let new_tile = self.get((x, y)).unwrap().evolve(
                    self.get((x - 1, y)),
                    self.get((x + 1, y)),
                    self.get((x - 1, y-1)),
                    self.get((x + 1, y-1)),
                    self.get((x - 1, y+1)),
                    self.get((x + 1, y+1)),
                    self.get((x, y+1)),
                    self.get((x, y-1)),
                );
                new_world.push(new_tile.clone());
            }
        }
    self.tiles = new_world
}
```

The get method implement for world used above is a quick helper function used to return the optional of the tile at that world location.

It is helpful to write some test cases in this library to ensure the rules are all implemented correctly. Once this is done, it is time to write a gui on top of this.

## GUI

As seen in [tutorial two](https://theseatoad.com/rustforgamedevelopment2/), writing graphics will require a window and a drawable surface. I am not inclined to use wgpu directly this time, so I will use a crate that implements higher-level APIs ontop of wgpu. The crate I will be using is [bevy](https://crates.io/crates/bevy). There is a ton that comes with bevy, but for this tutorial, only the rendering aspects will be highlighted. 

To start, go back to the root level directory for whole repository and
```
$ cargo new gui
```

To use bevy and also be able to reference our game of life library, the cargo.toml needs to be updated with these dependencies.

```toml
[dependencies]
bevy = "0.8"
cgol = {path = "../cgol"}
```

To start out with getting a window using bevy, we need to create a new app, and feed it the plugins we need for our game. As of writing this tutorial, the default plugins are
```rust
/// This plugin group will add all the default plugins:
/// * [`LogPlugin`](bevy_log::LogPlugin)
/// * [`CorePlugin`](bevy_core::CorePlugin)
/// * [`TimePlugin`](bevy_time::TimePlugin)
/// * [`TransformPlugin`](bevy_transform::TransformPlugin)
/// * [`HierarchyPlugin`](bevy_hierarchy::HierarchyPlugin)
/// * [`DiagnosticsPlugin`](bevy_diagnostic::DiagnosticsPlugin)
/// * [`InputPlugin`](bevy_input::InputPlugin)
/// * [`WindowPlugin`](bevy_window::WindowPlugin)
/// * [`AssetPlugin`](bevy_asset::AssetPlugin)
/// * [`ScenePlugin`](bevy_scene::ScenePlugin)
/// * [`RenderPlugin`](bevy_render::RenderPlugin) - with feature `bevy_render`
/// * [`SpritePlugin`](bevy_sprite::SpritePlugin) - with feature `bevy_sprite`
/// * [`PbrPlugin`](bevy_pbr::PbrPlugin) - with feature `bevy_pbr`
/// * [`UiPlugin`](bevy_ui::UiPlugin) - with feature `bevy_ui`
/// * [`TextPlugin`](bevy_text::TextPlugin) - with feature `bevy_text`
/// * [`AudioPlugin`](bevy_audio::AudioPlugin) - with feature `bevy_audio`
/// * [`GilrsPlugin`](bevy_gilrs::GilrsPlugin) - with feature `bevy_gilrs`
/// * [`GltfPlugin`](bevy_gltf::GltfPlugin) - with feature `bevy_gltf`
/// * [`WinitPlugin`](bevy_winit::WinitPlugin) - with feature `bevy_winit`
///
```

Let's create an app and add these.

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .run();
}
```

After this, a window should appear after running the gui crate.

Before writing any more code, we need to think about what steps need to be taken to correctly display the cgol::world.

1. Read inital values for tiles
2. Initalize cgol::world with tiles and width
3. Render tiles
4. Tick world
5. Back to step 3

For the inital values for the tiles, it is easiest to just create a text file with 1's and 0's and read from that to seed the world.

I created a CWorld Struct that is a wrapper ontop of cgol::world. It implements the default trait with the values based on what is in a text file.
```rust
impl Default for CWorld {
    fn default() -> Self {
        let mut tiles: Vec<Tile> = Vec::new();
        let world_string =
            fs::read_to_string("../assets/world.txt").expect("Could not read world file");
        for mut tile in world_string.split(",") {
            tile = tile.trim_matches('\n');
            match tile {
                "0" => {
                    tiles.push(Tile::new(false));
                }
                "1" => {
                    tiles.push(Tile::new(true));
                }
                _ => {
                    println!("Invalid tile");
                }
            }
        }
        CWorld {
            world: World { width: 25, tiles },
        }
    }
}
```

Great, that is steps 1 and 2 complete. Now, we need to loop over the CWorld.world.tiles rendering each to the screen. This is accomplished by grabbing the world from the app, looping over all the tiles, and matching on true or false. In either case, we spawn a new sprite.

```rust
fn draw_world(mut commands: Commands, asset_server: Res<AssetServer>, mut world: ResMut<CWorld>) {
    let mut x = 0;
    let mut y = 0;
    for tile in &world.world.tiles {
        match tile.on {
            true => {
                commands.spawn_bundle(SpriteBundle {
                    sprite: Sprite {
                        color: Color::BLACK.into(),
                        ..default()
                    },
                    texture: asset_server.load("tile.png"),
                    transform: Transform {
                        translation: Vec3 {
                            x: ((x * TILEWIDTH) as f32) - XOFFSET as f32,
                            y: ((y * TILEWIDTH) as f32) - YOFFSET as f32,
                            z: 1.,
                        },
                        ..default()
                    },
                    ..default()
                });
            }
            false => {
                commands.spawn_bundle(SpriteBundle {
                    sprite: Sprite {
                        color: Color::WHITE.into(),
                        ..default()
                    },
                    texture: asset_server.load("tile.png"),
                    transform: Transform {
                        translation: Vec3 {
                            x: ((x * TILEWIDTH) as f32) - XOFFSET as f32,
                            y: ((y * TILEWIDTH) as f32) - YOFFSET as f32,
                            z: 1.,
                        },
                        ..default()
                    },
                    ..default()
                });
            }
        }
        if x == MAPWIDTH - 1 {
            x = 0;
            y += 1;
        } else {
            x += 1
        }
    }
    world.world.tick();
}
```

With this method written, it will need to continually be called at a set interval. Inside of our main method, we will add a new system set. This system set will specify how frequently the draw_world method should be run. We should also add a window descriptor to our app letting winit know how big the window needs to be. This is important as the sprites do not scale to different window sizes. Lastly, we need a startup_system that will spawn a 2D camera so we can see our world!

```rust
fn main() {
    App::new()
        .insert_resource(WindowDescriptor {
            title: "Game of life".to_string(),
            width: 700.,
            height: 800.,
            present_mode: bevy::window::PresentMode::Fifo,
            ..default()
        })
        .insert_resource(ImageSettings::default_nearest())
        .insert_resource(CWorld::default())
        .add_plugins(DefaultPlugins)
        .add_startup_system(startup)
        .add_system_set(
            SystemSet::new()
                .with_run_criteria(FixedTimestep::step(TICKRATE as f64))
                .with_system(draw_world),
        )
        .run();
}
```

With all of this code written, the game should work now. Try adding new values in the world.txt file to see how different seeds progress over time.

## Conclusion

Having seperating of systems made this implementation of game of life easy to read and change. Any change to the renderer would not effect the underlying simulation itself. This is an important concept in game development as artists and game designers do not neccessaily need to know the underlying code of a game. They only need to know how to incorporate their work into the game. Seperation of systems also allows for collaberation between multiple people as library work can happen at the same time level design and art is implemented.

Secondly, bevy made it very easy to create a window, draw sprites, and call functions at set intervals. It comes with many tools that are useful for game development and simulations. In the next tutorials, we will continue to expand on [bevy](https://crates.io/crates/bevy), and how that can help us write games in rust!