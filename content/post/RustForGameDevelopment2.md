---
title: "Rust For Game Development #2 -- Simple cross-platform graphics"
date: 2022-08-15T10:54:31-07:00
publishdate: 2022-08-15
lastmod: 2022-08-15
draft: false
tags: ["Rust", "Graphics"]
categories: []
---

## Prelude

Follow along this tutorial using the cooresponding repository on [Github](https://github.com/theseatoad/simple_crossplatform_2d_graphics). Snippets here will highlight important structs and methods, but for compiling code, refer to the link above.

This post will be all about graphics. Writing a game will not look like this in practice, unless you write your own engine. If this is enjoyable to you, maybe you can go about writing your own graphics engine. If not, there is plently of higher-level crates to leverage. See [Rust for Game Development #3](https://theseatoad.com/rustforgamedevelopment3) for how to interface with a higher-level crate.

## Introduction

Game development is intrinsically related to computer graphics. Rendering graphics to the display is not an easy feat due to users having both different operating systems and different hardware. It is also not as simple as asking to display a certain RGBA value at a certain XY coordinate. This easy solution quickly falls apart even in 2D games when you think about pixel density on different displays, different windows on the OS, z-depth, etc.

For learning how to simply write code to draw images on the screen, using a crate is a must. Without leveraging pre-existing code written by the community, you would have to be directly interfacing with the your computer's OS and GPU language. 

For this tutorial, we will be using one of rust's most popular cross-platform graphics libraries, [wgpu](https://crates.io/crates/wgpu). One of the main reasons this crate will be useful, is it gives us the ability to write code once, and have it work on a variety of different platforms.

Below is a a diagram on where this crate lays in the scope of getting the pixels to the screen.

![big-picture](/big-picture.png)
*https://github.com/gfx-rs/wgpu*

As you can see, we will be using the wgpu section. This will go on to use wgpu-core to validate shaders using naga. After all of this, it will go on to be translated into the underlying system's graphics API by wgpu-hal. In my case, using Apple's Macbook Pro, my code will be translated to metal-rs to talk with my GPU.

Even with this crate, it is still a journey to render an image to the screen. There is a great tutorial by [sotrh](https://sotrh.github.io/learn-wgpu/#what-is-wgpu) that has inspired the majority of code below. I would highly recommend checking out their tutorial, as I am only expanding on the texture section of that tutorial. It is additionally endorsed by wgpu.

The goal is to draw one sprite to the screen. This would require us to:

1. Create a window
2. Create a drawable surface
3. Draw a texture

Just like any other rust project, run
```
$ cargo new NAME_OF_YOUR_PROJECT
```

With that all set, let's dive into our first computer graphics program.


### Window creation

[Winit](https://crates.io/crates/winit) is a cross-platform window creation library which can be used by wgpu. Let's add this dependency along with all of the other dependencies needed.

```toml
#cargo.toml
[dependencies]
winit = "0.26"
env_logger = "0.9"
log = "0.4"
wgpu = "0.13"
pollster = "0.2.5"
anyhow = "1.0"
bytemuck = { version = "1.4", features = [ "derive" ] }

[dependencies.image]
version = "0.24"
default-features = false
features = ["png", "jpeg"]
```

Making a window is very easy. It requires us to use the winit crate to create both an EventLoop and a WindowBuilder. The EventLoop is used to capture events from the system such as closing the window. This is then passed to the WindowBuilder that will, well, build a window! There is some advanced rust going on here with the move syntax. This simply will extend the lifetime of the event so we can later use it regardless of the scope of returning function.

```rust
use winit::{
    event::*,
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
};

fn main() {
    let event_loop = EventLoop::new();
    let window = WindowBuilder::new().build(&event_loop).unwrap();
    event_loop.run(move | event , event_loop_target_window, control_flow| {});
}
```

Try running this and see what happens. Also, try to use some of those system events like closing the window.

Nothing happens when you click the close button. This is because we have not defined that behavior in the event_loop. It will be nice to implement these behaviors.

To see how this is done, a match statement can be used to match on the events returned from the event_loop.run() method.

```rust
match event {
        Event::WindowEvent {
            ref event,
            window_id,
        } if window_id == window.id() => {
                match event {
                    WindowEvent::CloseRequested | WindowEvent::KeyboardInput =>
                    {
                        //Close window
                    },
                    WindowEvent::Resized(physical_size) =>
                    {
                        //resize(physical_size);
                    },
                    WindowEvent::ScaleFactorChanged { new_inner_size, .. } => {                        
                        //resize(new_inner_size);
                    },
                    _ => {}
                }
        }
        Event::RedrawRequested(window_id) if window_id == window.id() => {
            //Redraw
        }
        Event::MainEventsCleared => {
            //Redraw
        }
        _ => {}
}
```

Window events are mostly used for handling when the window resizes and when the system window close button is hit. The interesting behaviors lie in the when a redraw is requested and MainEventsCleared.

It is still not possible to draw the screen. To do that, we need to use something called a surface.
### Drawable surface

Wgpu defines a [surface](https://docs.rs/wgpu/latest/wgpu/struct.Surface.html) as the area onto which rendered images may be presented. To instantiate one, we need to do multiple things. We need to create a surface on the window. Secondly, we need to add an adapter and configuration for the surface. This allows us to to "link" the hardware configurations and options to our surface.  

```rust
let surface = unsafe { instance.create_surface(window) };
let adapter = instance
    .request_adapter(&wgpu::RequestAdapterOptions {
        //Handle to a physical graphics and/or compute device
    })
let config = wgpu::SurfaceConfiguration {
    //Configures a Surface for presentation.
};
surface.configure(&device, &config);
```

With the surface configured, it is time to introduce the render pipeline. "A pipeline describes all the actions the gpu will perform when acting on a set of data" - [sotrh](https://sotrh.github.io/learn-wgpu/beginner/tutorial3-pipeline/#what-s-a-pipeline).

```rust
let shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
    // Shader file
    source: wgpu::ShaderSource::Wgsl(include_str!("shader.wgsl").into()),
});

let render_pipeline_layout =
    device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
        //Used to describe how the pipeline should be layed out.
    });

let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    layout: Some(&render_pipeline_layout),
    vertex: wgpu::VertexState {
        // Vertex shader in shader module
    },
    fragment: Some(wgpu::FragmentState {
        // Fragment shader in shader module
    }),
});

let vertex_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
    
});
let index_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {

});
```
The addition of the render pipeline will allow us to tell the gpu what vertex shaders and fragment shaders to use.

A vertex is simply a coordinate in 2D or 3D space. Multiple vertices can be used to define a shape, most commonly a triangle. The vertex shader will be code telling the gpu what mathmatical operators to preform on that vertex. If you wanted a completely red triangle, you could define three vertices with a red color.

A fragment shader (also known as a pixel shader) takes input from the vertex shader and allows for shading on the surfaces they create. For example, if you want a triangle with a gradiant from white to black, you can define this here.

### Draw Texture

The last thing we need to do is draw a texture. First, we need to grab the information of the texture from our memory. Then, we need to define where we want to texture to be displayed, and then pass that information to the GPU.

Let's define a struct containing our texture information, and some useful functions for it.

```rust
//texture.rs

pub struct Texture {
    pub texture: wgpu::Texture,
    pub view: wgpu::TextureView,
    pub sampler: wgpu::Sampler,
}

impl Texture {
    pub fn from_bytes(Texture) -> Result<Self> {
        //Load from memory the bytes of the texture
    }

    pub fn from_image(Texture) -> Result<Self> {
        let texture = device.create_texture(&wgpu::TextureDescriptor {
            //Format and sample count
        });

        queue.write_texture(texture);
        let view = texture.create_view(&wgpu::TextureViewDescriptor::default());
        let sampler = device.create_sampler(&wgpu::SamplerDescriptor {
            //Filters and sampling
        });
    }
}
```

Back in main.rs, we will need to add a new layout group to pass to our render pipeline stating that we are giving texture information to it, and how to handle it.
```rust
let texture_bind_group_layout =
    device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
        entries: &[
            wgpu::BindGroupLayoutEntry {
                binding: 0,
                visibility: wgpu::ShaderStages::FRAGMENT,
            }...
        ]
    });

let diffuse_bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
    layout: &texture_bind_group_layout,
    entries: &[
        wgpu::BindGroupEntry {
            // Texture view
            },
        wgpu::BindGroupEntry {
            // Texture sample
        },
    ],
});
```

For our texture, we will want a nice rectangle to contain it. We define both the position in which the vertex lives in world coordinates. This ranges from -1 to 1. Secondly, we define what the texture looks like on top of this vertex. Texture coordinates are layed out as 0,0 for the top left and 1,1 for the bottom right.

```rust
const VERTICES: &[Vertex] = &[
    Vertex {
        position: [-0.5, 0.5, 0.0],
        tex_coords: [0., 0.],
    }, // Top Left
    Vertex {
        position: [-0.5, -0.5, 0.0],
        tex_coords: [0., 1.],
    }, // Bottom Left
    Vertex {
        position: [0.5, -0.5, 0.0],
        tex_coords: [1.,1.],
    }, // Bottom Right
    Vertex {
        position: [0.5, 0.5, 0.0],
        tex_coords: [1., 0.],
    }, // Top Right
];

const INDICES: &[u16] = &[0, 1, 2,
 0, 2, 3];
```

Lastly, some boilerplate of passing our vertices and texture coordinates to the gpu through wgsl.

```wgsl
struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) tex_coords: vec2<f32>,
}

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) tex_coords: vec2<f32>,
}

@vertex
fn vs_main(
    model: VertexInput,
) -> VertexOutput {
    var out: VertexOutput;
    out.tex_coords = model.tex_coords;
    out.clip_position = vec4<f32>(model.position, 1.0);
    return out;
}

@group(0) @binding(0)
var t_diffuse: texture_2d<f32>;
@group(0)@binding(1)
var s_diffuse: sampler;

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    return textureSample(t_diffuse, s_diffuse, in.tex_coords);
}
```

Now, with ALL of this code, we have a simple image appearing in our window.

![big-picture](/simple_texture.png)


### Final words

With about 500 lines to just display one very simple texture to the window, where did we go all wrong? It would be near impossible to write a game like this. To make games of any size, a library on top of wgpu is neccessary. Wgpu gives us the ability to basically directly talk to the gpu. Games need to do that, but it also need to be easier to write. Luckily for us, there are libraries to help with this process. In the next blog post, we will dive into this.