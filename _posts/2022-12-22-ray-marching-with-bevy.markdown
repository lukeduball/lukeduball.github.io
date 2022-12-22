---
layout: post
title:  "Ray Marching with Bevy"
date:   2022-12-22 13:53:00 -0600
categories: Bevy Ray Marching
---
# Ray Marching

Ray Marching is an interesting rendering technique that can be used to create fully procedural environments entirely in the fragment shader. In this post I will describe how I set up a very simple scene that is created by Ray Marching in the Bevy game engine. The source code for this project can be found in the [GitHub project](https://github.com/lukeduball/bevy_ray_marching/tree/bp-RayMarchingWithBevy-2022-11-12). I referred to this [post by Michael Walczyk](https://michaelwalczyk.com/blog-ray-marching.html) for a lot of the ray marching algorithm explanation and implementation. Please refer to his post if there are things I did not cover well. Additionally, I learned a lot about Bevy specific programming from both the official [Bevy examples](https://github.com/bevyengine/bevy/tree/main/examples) and from [Logic Project's YouTube channel](https://www.youtube.com/channel/UC7v3YEDa603x_84PgCPytzA). Check out his YouTube channel for some great videos. His [Bevy 0.8 material system video](https://www.youtube.com/watch?v=1zzxqwm5kps&t=3s) has a great explanation of the material system and was very helpful getting the custom shader used for this project working correctly.

## Bevy Screen Space Quad Setup
Before I dig into the ray marching explanation, I am going to explain how I setup Bevy to create the ray marcher. The ray marcher is going to exist entirely in the fragment shader. I setup a screen space quad so that every pixel on the screen will be processed by the fragment shader.

The first thing I did is setup a 2D scene in Bevy.

      {% highlight rust %}
      const WIDTH: f32 = 720.0;
      const HEIGHT: f32 = 720.0;

      fn main() {
            let mut app = App::new();

            app.insert_resource(ClearColor(Color::rgb(0.3, 0.3, 0.3)))
            .add_plugins(DefaultPlugins.set(WindowPlugin {
                  window: WindowDescriptor {
                        width: WIDTH,
                        height: HEIGHT,
                        title: "Ray Marching Scene".to_string(),
                        resizable: true,
                        ..default()
                  },
                  ..default()
            }))
            .add_plugin(Material2dPlugin::<RayMarchingMaterial>::default())

            .add_startup_system(setup);

            app.run();
      }
      {% endhighlight %}

I created the Bevy app and added the DefaultPlugins with some window values for the width, height, title, and set it as resizable. I also created the startup system "setup" below:

      {% highlight rust %}
      fn setup(
            mut commands: Commands,
            mut meshes: ResMut<Assets<Mesh>>,
            mut materials: ResMut<Assets<ColorMaterial>>,
      ) {
            commands.spawn_bundle(Camera2dBundle::default());
            commands.spawn_bundle(MaterialMesh2dBundle) {
                  mesh: meshes.add(Mesh::from(Quad::default())).into(),
                  transform: Transform::default().with_scale(Vec3::splat(128.)),
                  material: materials.add(ColorMaterial.from(Color.PURPLE)),
                  ..default(),
            };
      }
      {% endhighlight %}

If I were to run the application now you would see a purple quad in the middle of the screen.

Currently, this setup uses a default shader and default 2D quad mesh with the 2D camera. To modify this setup to create the ray marching project I created a custom screen space quad mesh.

      {% highlight rust %}
      #[derive(Debug, Copy, Clone)]
      pub struct ScreenSpaceQuad {
            //Scale of the quad in screen space with 1,1 taking the whole screen
            pub scale: Vec2,
      }

      impl Default for ScreenSpaceQuad {
            fn default() -> Self {
                  Self::new(Vec2::ONE)
            }
      }

      impl ScreenSpaceQuad {
            fn new(scale: Vec2) -> Self {
                  Self { scale }
            }
      }

      //Creating the custom vertex data for our ScreenSpaceQuad
      //Note: the normal attribute had to be included for this to work correctly
      impl From<ScreenSpaceQuad> for Mesh {
            fn from(screen_space_quad: ScreenSpaceQuad) -> Self {
                  let vertices = vec![[-1.0*screen_space_quad.scale.x, -1.0*screen_space_quad.scale.y, 0.0],
                                    [-1.0*screen_space_quad.scale.x,  1.0*screen_space_quad.scale.y, 0.0],
                                    [ 1.0*screen_space_quad.scale.x, -1.0*screen_space_quad.scale.y, 0.0],
                                    [ 1.0*screen_space_quad.scale.x,  1.0*screen_space_quad.scale.y, 0.0]];

                  let indices = Indices::U32(vec![0, 2, 1, 2, 3, 1]);

                  let normals = vec![[0.0, 0.0, 1.0],
                                    [0.0, 0.0, 1.0],
                                    [0.0, 0.0, 1.0],
                                    [0.0, 0.0, 1.0]];

                  let uvs = vec![[0.0, 0.0],
                                    [0.0, 1.0],
                                    [1.0, 0.0],
                                    [1.0,1.0]];

                  let mut mesh = Mesh::new(PrimitiveTopology::TriangleList);
                  mesh.set_indices(Some(indices));
                  mesh.insert_attribute(Mesh::ATTRIBUTE_POSITION, vertices);
                  mesh.insert_attribute(Mesh::ATTRIBUTE_NORMAL, normals);
                  mesh.insert_attribute(Mesh::ATTRIBUTE_UV_0, uvs);
                  mesh
            }
      }
      {% endhighlight %}

I created the custom vertex data so that the screen space quad is always in the screen space position of -1.0 to 1.0 in the x and y axes when it has a default scale of 1. The "from" function implemented on the Mesh type allows these attributes to be inserted and passed to our shader. **Note: to get the mesh to pass to the shader correctly, I did have to include the normals as well.**

I also created a custom ray marching material so that I could create custom shaders. 
      
      {% highlight rust %}
      //New material created to setup a custom shader
      #[derive(AsBindGroup, Debug, Clone, TypeUuid)]
      #[uuid = "084f230a-b958-4fc4-8aaf-ca4d4eb16412"]
      pub struct RayMarchingMaterial {
            #[uniform(0)]
            position: Vec4,
      }

      //Setup the RayMarchingMaterial to use the custom shader file for the vertex and fragment shader
      impl Material2d for RayMarchingMaterial {
            fn vertex_shader() -> ShaderRef {
                  "shaders/ray_marching_material.wgsl".into()    
            }

            fn fragment_shader() -> ShaderRef {
                  "shaders/ray_marching_material.wgsl".into()
            }
      }
      {% endhighlight %}

The position uniform specifies the position of our Camera which will be used later in the ray marching logic. Using the "impl Material2d for RayMarchingMaterial" I am able to specify to path to a custom shader file to use. If you were to remove the vertex_shader or fragment_shader function Bevy would continue to use the default shaders.

I also needed to register the 2D material in my main function and updated the setup function to use the custom mesh and custom material.

      {% highlight rust %}
      fn main() {
            ...
            //Custom material registry line
            .add_plugin(Material2dPlugin::<RayMarchingMaterial>::default())
            .add_startup_system(setup)
            ...
      }

      fn setup(
            mut commands: Commands,
            mut meshes: ResMut<Assets<Mesh>>,
            mut materials: ResMut<Assets<RayMarchingMaterial>>,
            ) {
                  commands.spawn(Camera2dBundle::default());
                  commands.spawn(MaterialMesh2dBundle {
                        mesh: meshes.add(Mesh::from(ScreenSpaceQuad::default())).into(),
                        material: materials.add(RayMarchingMaterial { position: Vec4::new(0.0, 1.0, 0.0, 1.0)),
                        ..default()
                  });
            }
      {% endhighlight %}

Now that this is all setup, I created the ray_marching_material.wgsl shader file.

      {% highlight wgsl %}
      //struct to hold the uniform buffer information
      struct Camera {
            position: vec4<f32>,
      }

      @group(1) @binding(0)
      var<uniform> camera: Camera;

      //Define the vertex information passed to the shader
      struct Vertex {
            @location(0) position: vec3<f32>,
            @location(1) normal: vec3<f32>,
            @location(2) uv_coords: vec3<f32>,
      }

      //Output by the vertex shader. @location parameters are passed into the fragment shader
      struct VertexOutput {
            @builtin(position) clip_position: vec4<f32>,
            @location(0) uv_coords: vec2<f32>,
      }

      //Entry point for the vertex shader
      @vertex
      fn vertex(vertex: Vertex) -> VertexOutput {
            var out: VertexOutput;
            //vertex position in screen space which is used by the renderer
            out.clip_position = vec4(vertex.position, 1.0);
            out.uv_coords = vertex.uv_coords;
            return out;
      }

      //Passed from the vertex shader to the fragment shader
      struct FragmentIn {
            @location(0) uv_coords: vec2<f32>,
      }

      //Entry point into the fragment shader
      @fragment
      fn fragment(in: FragmentIn) -> @location(0) vec4<f32> {
            //Returns the uv coords as the colors to show the screen space quad with a custom shader is working.
            return vec4(in.uv_coords.x, in.uv_coords.y, 0.0, 1.0);
      }
      {% endhighlight %}

This completed my initial setup of the screen space quad mesh with a custom material so that a custom shader could be used. The result of this is shown in the image below.
![Initial Setup](/assets/initial_setup_screen_capture.jpg)

## No More Polygons
Now that I described the general setup I will describe ray marching and how I implemented it. Ray marching is a different rendering method than traditional rendering methods using points, lines, triangles, and complex meshes. Instead of using these, ray marching uses signed distance fields (SDFs) to render a scene. SDFs are mathematical functions that tell how far a point in 3D space is to a given surface. The example below shows how we calculate a SDF using a point and a sphere.

First, lets define a point in 3D space in WGSL with coordinates x, y, and z:

      {% highlight wgsl %}
      var point = vec3<f32>(x, y, z)
      {% endhighlight %}

Next, we will have a sphere defined by a point at the sphere's center and a radius. To find the SDF, we need to calculate the shortest distance from the point to the sphere. We can find this distance by finding the distance from the point to the center of the sphere and subtract the radius.

      {% highlight wgsl %}
      fn get_distance_from_sphere(current_position: vec3<f32>, sphere_center: vec3<f32>, radius: f32) -> f32 {
        return length(current_position - sphere_center) - radius;
      }
      {% endhighlight %}

![Sphere SDF Image](/assets/Sphere-SDF-Figure.jpg)

This is just one example of an SDF function for a sphere. There are many more functions for different shapes. The reason this is a "signed" distance field is because the result of the function can result in a negative number, positive number, or zero. These correspond to three different cases.

**Case 1: Negative Number**

The point is inside of the shape.

**Case 2: Zero**

The point lies on the surface of the shape.

**Case 3: Positive Number**
The point is somewhere outside of the shape.

## Rendering Code for Ray Marching
Now that I have discussed what an SDF is, I will talk about how we will use one to render the scene via ray marching. When we are ray marching we are shooting a bunch of rays out of our virtual camera. We will step along each ray at an specific interval and evaluate our scene with the SDF functions each time. This will give us the distance to the closest shape. We will continue to take these steps until we get so close to an object we can consider that we have intersected with it. 

We could take regularly small steps along our ray in order to march but this might not be very effective. If our steps are too small, it will not be very efficient. If our steps are too large, we could move past an object and essentially move through it when the ray would have really intersected it. We will instead take advantage of **distance-aided ray marching**. This works by moving along the ray the distance of the closest object to the point in the scene. Therefore, when we evaluate the SDFs at each interval we will move along the ray the distance of the smallest positive SDF result. The image below shows an example of how this works in 2D space.

![Distance Aided Ray Marching](/assets/distance_aided_ray_marching.jpg)
Our ray begins at the start point and marches to the right. At each point along the ray it calculates the distance to the closest intersection in the scene. It then moves that distance along the ray. This is shown as a circle to show the closest intersection distance and how that translates to the ray. This is continued until the distance becomes so small that it is considered that an intersection has occurred.

### Generating Rays from the Camera
With the information above, the first thing I did was write the code in the fragment shader to generate the rays we are casting into the scene from our camera. To create these rays I shot rays from the origin of the camera into a plane that is one unit in front of the camera. This plane represents the pixels on the screen. I updated the ray_marching_material.wgsl shader file with the updates below.
      
      {% highlight wgsl %}
      @vertex
      fn vertex(vertex: Vertex) -> VertexOutput {
            var out: VertexOutput;
            out.clip_position = vec4(vertex.position, 1.0);
            //Update the vertex coordinates to be in the range of -1 to 1
            out.uv_coords = vertex.uv_coords * 2.0 - 1.0;
            return out;
      }

      @fragment
      fn fragment(in: FragmentIn) -> @location(0) vec4<f32> {
            var camera_origin = camera.position.xyz;
            var ray_origin = camera_origin + vec3<f32>(in.uv_coords, 1.0);
            var ray_direction = normalize(ray_origin - camera_origin);
            ...
      }
      {% endhighlight %}

I used the UV coordinates of the screen space quad to determine the point on the plane to cast the ray through. Since I am using the UV coordinates, I had to remap those coordinates from the range of 0 to 1 to the range of -1 to 1. This is because we want the origin of our camera to be in the middle of the screen with rays being cast to the left and right of the origin as well as up and down from the origin. To generate the ray I calculated the ray origin by taking the camera position and adding the uv coordinates. I also added 1 to the z position. The 1 in the z position acts like the field of view. I then calculated the ray direction by finding the difference between the ray origin and the camera origin.

### Ray Marching Code
Now that I have generated the rays, I set up the code to perform the distance-aided ray marching. First, I updated the fragment function with the following at the end.

      {% highlight wgsl %}
      var color = ray_march(ray_origin, ray_direction);

      return vec4(color.x, color.y, color.z, 1.0);
      {% endhighlight %}

Then I defined the "ray_march" function to perform the distance-aided ray march.

      {% highlight wgsl %}
      fn ray_march(ray_origin: vec3<f32>, ray_direction: vec3<f32>) -> vec3<f32> {
            var total_distance_traveled = 0.0;
            var NUMBER_OF_STEPS = 128;
            var MINIMUM_HIT_DISTANCE = 0.001;
            var MAXIMUM_TRAVEL_DISTANCE = 1000.0;

            //Loop for a finite number of steps for performance reasons
            for(var i = 0; i < NUMBER_OF_STEPS; i++) {
                  var current_position = ray_origin + total_distance_traveled * ray_direction;

                  var distance_to_closest = get_distance_from_world(current_position)

                  if(distance_to_closest < MINIMUM_HIT_DISTANCE) {
                        //Return the color red if a shape is hit
                        return vec3<f32>(1.0, 0.0, 0.0);
                  }

                  if total_distance_traveled > MAXIMUM_TRAVEL_DISTANCE) {
                        //If no hit has occurred after traveling far, break out of the loop
                        break;
                  }

                  total_distance_traveled += distance_to_closest;
            }

            //A miss has occurred so return a background color of black
            return vec3<f32>(0.0, 0.0, 0.0);
      }
      {% endhighlight %}

The function works by looping over a finite number of steps. During these steps it will calculate the distance to the closest object in the scene. If the closest distance to an object in the scene is extremely small, it will be considered a hit. For the time being I returned the color red but will add normals and lighting later. If the distance to the closest object in the scene is not extremely small that means we will continue ray marching by moving the current position along the ray by the distance to the closest object.

I then created the get_distance_from_world function and placed a single sphere within my scene.

      {% highlight wgsl %}
      //SDF function of a sphere from above
      fn get_distance_from_sphere(current_position: vec3<f32>, sphere_center: vec3<f32>, radius: f32) -> f32 {
            return length(current_position - sphere_center) - radius;
      }

      //This function can be modified to have multiple shapes and find the closest distance from all shapes.
      fn get_distance_from_world(current_position: vec3<f32>) -> f32 {
            var sphere_distance = get_distance_from_sphere(current_position, vec3<f32>(0.0, 0.0, 0.0), 1.0);

            return sphere_distance;
      }
      {% endhighlight %}

I am now able to run my project and I get this red circle as a result.

![Intermediate Red Sphere](/assets/red_sphere.jpg)

### Add Normals and Simple Lighting
I am going to now add some normals and a simple point light to the scene to make it more interesting. It would be possible to figure out the shape that is being hit by the ray and then calculate its normals. But instead, I am going to be using the gradient method for normals described in the [post by Michael Walczyk](https://michaelwalczyk.com/blog-ray-marching.html). It works by calculating the gradient vector along the shape by taking small steps in each coordinate axis and finding the closest distance to the world. This method is really nice because it gives us a general way to find the normal for any shape. This is especially nice when distorting the shapes to create cool effects.

      {% highlight wgsl %}
      fn calculate_normal(current_position: vec3<f32>) -> vec3<f32> {
            //Define the small step to take
            var SMALL_STEP = vec2<f32>(0.001, 0.0);

            //Calculate the gradients by calling the distance to world function 6 more times while taking small steps in those axes.
            var gradient_x = get_distance_from_world(current_position + SMALL_STEP.xyy) - get_distance_from_world(current_position - SMALL_STEP.xyy);
            var gradient_y = get_distance_from_world(current_position + SMALL_STEP.yxy) - get_distance_from_world(current_position - SMALL_STEP.yxy);
            var gradient_z = get_distance_from_world(current_position + SMALL_STEP.yyx) - get_distance_from_world(current_position - SMALL_STEP.yyx);

            //Calculate the final normal by normalizing the gradient values
            return normalize(vec3<f32>(gradient_x, gradient_y, gradient_z));
      }
      {% endhighlight %}

I then updated the ray_march function to include the normals and a simple hard-coded point light.

      {% highlight wgsl %}
      ...
      if(distance_to_closest < MINIMUM_HIT_DISTANCE) {
            var normal = calculate_normal(current_position);

            var light_position = vec3<f32>(2.0, -5.0, 3.0);

            var direction_to_light = normalize(current_position - light_position);

            var diffuse_intensity = max(0.0, dot(normal, direction_to_light));

            return vec3<f32>(1.0, 0.0, 0.0) * diffuse_intensity;
      }
      ...
      {% endhighlight %}

The result now looks like this:

![Red Sphere with Lighting](/assets/red_sphere_lighting.jpg)

### Aspect-Ratio Issue
Now that I have a working ray marching scene there is an issue when the screen size is not a square. Here is what it looks like if I stretch the window.

![Aspect Ratio Issue](/assets/aspect_ratio_issue.jpg)

I solved this issue by passing the aspect ratio into the shader and multiplying the UV x coordinate by the aspect ratio. I updated the following in ray_marching_material.wgsl.

      {% highlight wgsl %}
      //Updated the struct to include an entry for the aspect ratio
      struct Camera {
            position: vec4<f32>,
            aspect_ratio: f32,
      };

      //Updated the vertex function to add the aspect ratio line
      @vertex
      fn vertex(vertex: Vertex) -> VertexOutput {
            var out: VertexOutput;
            out.clip_position = vec4(vertex.position, 1.0);
            out.uv_coords = vertex.uv_coords * 2.0 - 1.0;
            //ADDED THIS LINE
            out.uv_coords.x *= camera.aspect_ratio;
            return out;
      }
      {% endhighlight %}

After updating the shader file I had to add some more Bevy code to pass the aspect ratio into the shader. First, I updated the RayMarchingMaterial to include an entry for the aspect ratio.

      {% highlight rust %}
      pub struct RayMarchingMaterial {
            #[uniform(0)]
            position: Vec4,
            #[uniform(0)]
            apsect_ratio: f32,
      }
      {% endhighlight %}

Now that I did that I needed to actually set the aspect ratio. I will set the aspect ratio when I initially create the RayMarchingMaterial but that will not cover the case of changing the aspect ratio when the screen is resized. To do that I will create a resize_event system that will set the aspect ratio when the screen is resized. The aspect ratio is a global resource so I will create a Bevy resource to hold that value. Therefore, I added this struct, registered the resource in my "main" function, and registered the resize_event system in my "main" function. I also added the initial value of aspect_ratio in the RayMarchingMaterial (note: this value is not really important as it will eventually get overwritten by the Global aspect ratio anyway).

      {% highlight rust %}
      //Struct which becomes the Global Resource for the aspect ratio
      #[derive(Default, Resource)]
      pub struct AspectRatio {
            aspect_ratio: f32,
      }

      fn main() {
            ...
            .add_plugin(Material2dPlugin::<RayMarchingMaterial>::default())
            //Add this line to register apsect ratio resource
            .init_resource::<AspectRatio>()
            .add_startup_system(setup)
            //Add this line to register the resize_event funciton
            .add_system(resize_event);
            ...
      }

      fn setup(
            mut commands: Commands,
            mut meshes: ResMut<Assets<Mesh>>,
            mut materials: ResMut<Assets<RayMarchingMaterial>>,
      ) {
            commands.spawn(Camera2dBundle::default());
            commands.spawn(MaterialMesh2dBundle {
                  mesh: meshes.add(Mesh::from(ScreenSpaceQuad::default())).into(),
                  //Added initial value for the aspect ratio
                  material: materials.add(RayMarchingMaterial { position: Vec4::new(0.0, 0.0, -5.0, 1.0), aspect_ratio: WIDTH / HEIGHT}),
                  ..default()
            });   
      )
      {% endhighlight %}

I then created the "resize_event" function by passing the new aspect ratio resource and window resized event to the function.

      {% highlight rust %}
      fn resize_event( 
            mut resize_reader: EventReader<WindowResized>,
            mut aspect_ratio_resource: ResMut<AspectRatio>,
      ) {
            for event in resize_reader.iter() {
                  aspect_ratio_resource.aspect_ratio = event.width / event.height;
            }
      }
      {% endhighlight %}

Now that I did that, I still need to add a bit more code in order to pass the aspect ratio to the shader. Right now, the aspect ratio resource lives in the "Game World". I need to move that data into the "Render World" so that it can be used by the shader. To do this I need to add the extract function and the prepare function. The extract function will take the aspect ratio resource and copy it from the "Game World" to the "Render World". The prepare function can use the aspect ratio resource I just copied and update the uniforms for our ray marching material. Therefore, I updated the "main" function and added the following functions.

      {% highlight rust %}
      fn main() {
            ...
            .add_system(resize_event);

            app.sub_app_mut(RenderApp)
                  //Register the function for the extract stage
                  .add_system_to_stage(RenderStage::Extract, extract_raymarching_material)
                  //Register the function for the prepare stage
                  .add_system_to_stage(RenderStage::Prepare, prepare_raymarching_material);

            app.run();
      }

      //Need a struct of type ShaderType to copy data into our shader's uniform buffer
      #[derive(ShaderType, Clone)]
      struct RayMarchingMaterialUniformData {
            aspect_ratio: f32,
      }

      //Move information from the "Game World" to the "Render World"
      fn extract_raymarching_material(
      mut commands: Commands,
      ray_marching_query: Extract<Query<(Entity, &Handle<RayMarchingMaterial>)>>,
      aspect_ratio_resource: Extract<Res<AspectRatio>>
      ) {
            for (entity, material_handle) in ray_marching_query.iter() {
                  commands.get_or_spawn(entity)
                        .insert(material_handle.clone());
            }

            commands.insert_resource(AspectRatio {
                  aspect_ratio: aspect_ratio_resource.aspect_ratio,
            });
      }

      //Update the buffers with the data taken from the "Game World" and sent to the "Render World" so they can be used by the GPU
      fn prepare_raymarching_material(
      materials: Res<RenderMaterials2d<RayMarchingMaterial>>,
      material_query: Query<&Handle<RayMarchingMaterial>>,
      render_queue: Res<RenderQueue>,
      aspect_ratio_resource: Res<AspectRatio>
      ) {
            for material_handle in &material_query {
                  if let Some(material) = materials.get(material_handle) {
                        for binding in material.bindings.iter() {
                        if let OwnedBindingResource::Buffer(current_buffer) = binding {
                              let mut buffer = encase::UniformBuffer::new(Vec::new());
                              buffer.write(&RayMarchingMaterialUniformData {
                                    apsect_ratio: aspect_ratio_resource.aspect_ratio,
                              }).unwrap();
                              //Write to an offset of 16 bytes to put the aspect ratio data in the right place and not overwrite the 16 bytes of position data (Vec4)
                              render_queue.write_buffer(current_buffer, 16, buffer.as_ref());
                        }
                  }
            }
      }
      {% endhighlight %}

Now if I run my project and resize the screen I will see that the stretching issue no longer occurs.

![Aspect Ratio Fixed](/assets/aspect_ratio_fixed.jpg)

### Displacement Effects
There are some cool effects I created by only displacing the distances in the "get_distance_from_world" function in the ray_marching_material.wgsl shader. As of Bevy 0.9 the time is automatically passed to the shaders from the Globals struct. I use this time along with sine functions to create the effect below.

![Displacement Gif](/assets/displacement_sphere.gif)

Here is the code that was updated to create this effect.

      {% highlight wgsl %}
      //Copied from mesh2d_view_types.wgsl in the bevy_sprite crate (default shaders for 2d meshes in bevy)
      struct Globals {
            // The time since startup in seconds
            // Wraps to 0 after 1 hour.
            time: f32,
            // The delta time since the previous frame in seconds
            delta_time: f32,
            // Frame count since the start of the app.
            // It wraps to zero when it reaches the maximum value of a u32.
            frame_count: u32,
      #ifdef SIXTEEN_BYTE_ALIGNMENT
            // WebGL2 structs must be 16 byte aligned.
            _wasm_padding: f32
      #endif
      }

      //Copied from mesh2d_view_bindings in bevy_sprite crate
      @group(0) @binding(1)
      var<uniform> globals: Globals;

      fn get_distance_from_world(current_position: vec3<f32>) -> f32 {
            var displacement = sin(5.0 * current_position.x + globals.time) * sin(5.0 * current_position.y + globals.time) * sin(5.0 * current_position.z + globals.time) * 0.25;
            var sphere_distance = get_distance_from_sphere(current_position, vec3<f32>(0.0, 0.0, 0.0), 1.0);

            //Add other shapes in this area

            return sphere_distance + displacement;
      }

      {% endhighlight %}

## What's Next?
There are plenty of things I plan to add to this project in the future which includes:
- Moveable camera to navigate the scene
- Multiple different shapes
- CSG operations (unions, intersections, subtractions, etc.)
- Shadows and ambient occlusion
- Movable point light source via a uniform buffer

Thank you for reading. I hope you enjoyed it!

{% include utterances.html %}