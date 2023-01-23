---
layout: post
title:  "Ray Marching with Bevy 2: Animated Camera and Refactoring"
date:   2023-01-21 19:52:00 -0600
categories: Bevy Ray Marching
---
# Overview
In the last blog post I setup the ray marching project and created a simple scene with a single sphere that was animated. Everything was working but the main.rs file was starting to get very crowded. In this first section I will describe how I refactored the code into separate files and organized the RayMarchingMaterial into its own plugin. Our scene in the last project was also extremely static, so in the second section I will describe how I implemented a very simple fly-camera in Bevy. The github project and branch of the code at the end of this blog post can be found [here](https://github.com/lukeduball/bevy_ray_marching/tree/bp-RayMarchingWithBevy2-2023-01-04).

## Refactoring main.rs
The first thing I did was refactor main.rs. There are a couple of obvious divisions in our current code. First is the RayMarchingMaterial and its associated code. The second is the ScreenSpaceQuad mesh. I will start with the ScreenSpaceQuad mesh.

### Refactoring ScreenSpaceQuad into screen_space_quad.rs
First, I created a new file in the same directory as main.rs and called it screen_space_quad.rs. I then copied the contents of the ScreenSpaceQuad code into the new screen_space_quad.rs file.

    {% highlight rust %}
    use bevy::{prelude::*, render::{mesh::Indices, render_resource::PrimitiveTopology}};

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
    //Note: the normal and uv attribute had to be included for this to work. Seems to be some Bevy limitation
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

I also added the missing imports that were required to screen_space_quad.rs. I then updated main.rs to add the following so it can use the code from screen_space_quad.rs.

    {% highlight rust %}
    //Adds the file into the project
    mod screen_space_quad;
    //Import the ScreenSpaceQuad struct from screen_space_quad.rs
    use crate::screen_space_quad::ScreenSpaceQuad;

    {% endhighlight %}

### Refactoring RayMarchingMaterial into ray_marching_material.rs
Next, I refactored the RayMarchingMaterial code into ray_marching_material.rs. I moved the structs for the RayMarchingMaterial and RayMarchingMaterialUniformData first.

    {% highlight rust %}
    //New material created to setup custom shader
    #[derive(AsBindGroup, Debug, Clone, TypeUuid)]
    #[uuid = "084f230a-b958-4fc4-8aaf-ca4d4eb16412"]
    pub struct RayMarchingMaterial {
        //Set the uniform at binding 0 to have the following information - connects to Camera struct in ray_marching_material.wgsl
        #[uniform(0)]
        pub position: Vec4,
        #[uniform(0)]
        pub aspect_ratio: f32,
    }

    //Setup the RayMarchingMaterial to use the custom shader file for the vertex and fragment shader
    //Note: one of these can be removed to use the default material 2D bevy shaders for the vertex/fragment shader
    impl Material2d for RayMarchingMaterial {
        fn vertex_shader() -> ShaderRef {
            "shaders/ray_marching_material.wgsl".into()    
        }

        fn fragment_shader() -> ShaderRef {
            "shaders/ray_marching_material.wgsl".into()
        }
    }

    //Uniform data struct to move data from the "Game World" to the "Render World" with the ShaderType derived
    #[derive(ShaderType, Clone)]
    struct RayMarchingMaterialUniformData {
        apsect_ratio: f32,
    }
    {% endhighlight %}

Next, I moved the extract_raymarching_material and prepare_raymarching_material functions into ray_marching_material.rs because both of these functions perform tasks specific to the RayMarchingMaterial.

    {% highlight rust %}
    //Added at the top of the file to import the AspectRatio resource
    use crate::AspectRatio;

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
                        //Write to an offset in the buffer so the position data is not over-written
                        render_queue.write_buffer(current_buffer, 16, buffer.as_ref());
                    }
                }
            }
        }
    }
    {% endhighlight %}

Notice, I also added an import for the AspectRatio resource because it is used by the two functions. Currently, there are entries for both the RayMarchingMaterial and the render system stages functions (prepare and extract) in the main function in main.rs. These entries really only apply to the RayMarchingMaterial so it would make more sense to move these entries to ray_marching_material.rs. In Bevy, you can do this by creating a plugin and then registering that with the App in the main function. The plugin will hold an instance of the app so that we can register these entries in ray_marching_material.rs. I added the following lines to the top of ray_marching_material.rs to create the plugin and register the entries.

    {% highlight rust %}
    pub struct RayMarchingMaterialPlugin;

    impl Plugin for RayMarchingMaterialPlugin {
        fn build(&self, app: &mut App) {
            app.add_plugin(Material2dPlugin::<RayMarchingMaterial>::default());

            //Add our custom extract and prepare systems to the app
            app.sub_app_mut(RenderApp)
            .add_system_to_stage(RenderStage::Extract, extract_raymarching_material)
            .add_system_to_stage(RenderStage::Prepare, prepare_raymarching_material);
        }
    }
    {% endhighlight %}

Lastly, in ray_marching_material.rs I added all the required imports at the top of the file.

    {% highlight rust %}
    use bevy::{prelude::*, render::{render_resource::{AsBindGroup, ShaderRef, ShaderType, OwnedBindingResource, encase}, Extract, renderer::RenderQueue, RenderApp, RenderStage}, reflect::TypeUuid, sprite::{Material2d, RenderMaterials2d, Material2dPlugin}};
    {% endhighlight %}

Since I made those updates and created the plugin I had to make a few more updates to main.rs. I removed the entries in the main function that were moved to the plugin and replaced them with the line registering the RayMarchingMaterialPlugin. I also added the import for the RayMarchingMaterial and RayMarchingMaterialPlugin from the new file.

    {% highlight rust %}
    mod ray_marching_material;
    use crate::ray_marching_material::{RayMarchingMaterial, RayMarchingMaterialPlugin};

    fn main() {
        ...
        app.insert_resource(ClearColor(Color::rgb(0.3, 0.3, 0.3)))
        ...
        //Added this line
        .add_plugin(RayMarchingMaterialPlugin)
        ...
    }
    {% endhighlight %}

## Animated Camera
After I refactored the code above I worked on adding in an animated camera that could move in the scene. WASD keys move the camera side-to-side and forward. R and F keys move the camera up and down. Holding down the right mouse button while moving the mouse will rotate the camera. Before I describe the code updates I made, I will describe the implementation at a high level.

### High Level Implementation
The standard Bevy coordinate system is a right-handed coordinate system with y up. That means by default, our camera should be looking into the -z direction with the positive y coordinate up and the positive x coordinate to the right. My current implementation has this a little backwards but I will fix that before implementing any new code changes. I mention that now because it is important to the explanation of the camera implementation. Currently, if the implementation was correct our camera would be looking down the -z axis toward our sphere. The x and y coordinates to cast our ray through are calculated by using the uv coordinates of our screen space quad. 

![2D Framebuffer](/assets/camera-frustum.jpg)

Since the camera is not rotated at all, the u coordinate maps completely to the x axis while the v coordinate maps completely to the y axis.

![3D Representation](/assets/3D-representation-framebuffer.jpg)

The image above shows the 3D representation of this. We have our camera which is aligned to shoot rays onto the frame buffer (rectangle) in the -z axis. The sphere represents the sphere currently in the scene. I want to be able to rotate the camera and that means that the u and v coordinates used to create our ray will no longer map exactly to the x and y axis. Instead, I will find a unit vector in each direction from our camera (forward, right, and up). I am then able to position the frame buffer one unit in front of the camera by adding the forward vector to the camera position. To find the other coordinates of the ray I can now include our u coordinate by multiplying the u coordinate by the right vector and adding it to our new position. I can do the same with the v coordinate by multiplying it by the up vector and adding it to the current position. 

![Front,Up,Right](/assets/right-forward-up-framebuffer.jpg)

This image shows how everything can be remapped to the unit vectors where the up vector is (x=0, y=1, z=0), the forward vector is (x=0, y=0, z=-1), and the right vector is (x=1, y=0, z=0).

Now that I have defined these vectors, I will also use them for movement. For example, by moving the camera position by the forward vector, I can move forward in the scene. If I move the camera position by opposite the forward vector, I can move backward in the scene.

### Fixing the Past Mistake
As I said earlier there is a slight issue in the current implementation. I did not have things aligned with the Bevy coordinate system and actually flipped the z axis. To resolve this I updated the following in main.rs.

    {% highlight rust %}
    fn setup(
        ...
    ) {
        ...
        commands.spawn(MaterialMesh2dBundle {
            ...
            //Changed -5.0 in z coordinate to 5.0
            material: materials.add(RayMarchingMaterial { position: Vec4::new(0.0, 0.0, 5.0, 1.0), aspect_ratio: WIDTH / HEIGHT}),
        });
    }
    {% endhighlight %}

With this update, the lighting will be different because I had this flipped in ray_marching_material.wgsl as well. I updated the following to get the scene looking the same.

    {% highlight wgsl %}
    //Updated 3.0 to -3.0 in the z coordinate
    var light_position = vec3<f32>(2.0, -5.0, -3.0);
    {% endhighlight %}

Now that everything is updated, the scene should look the same once it is run. It is also now oriented in the default Bevy coordinate system so it will be easy to see our camera is working correctly once we implement it.

### Updating Camera Uniforms
As I discussed in the section above I will use the forward, right, and left unit vectors to calculate the rays we are casting into the scene. Fortunately, the Camera2d object already in the scene contains a camera position and functions to calculate these vectors via its transform. Therefore, we will use this Camera2d transform to pass those values to our shader. However, only the aspect ratio and camera position are being passed to the shader right now. So the first thing we are going to do is update our shader uniforms in ray_marching_material.rs.

    {% highlight rust %}
    pub struct RayMarchingMaterial {
        //Set the uniform at binding 0 to have the following information - connects to Camera struct in ray_marching_material.wgsl
        #[uniform(0)]
        pub camera_position: Vec3,
        #[uniform(0)]
        pub camera_forward: Vec3,
        #[uniform(0)]
        pub camera_horizontal: Vec3,
        #[uniform(0)]
        pub camera_vertical: Vec3,
        #[uniform(0)]
        pub aspect_ratio: f32,
    }

    impl RayMarchingMaterial {
        pub fn new() -> RayMarchingMaterial {
            RayMarchingMaterial { 
                camera_position: Vec3::new(0.0, 0.0, 0.0), 
                camera_forward: Vec3::new(0.0, 0.0, -1.0), 
                camera_horizontal: Vec3::new(1.0, 0.0, 0.0), 
                camera_vertical: Vec3::new(0.0, 1.0, 0.0), 
                aspect_ratio: 1.0, 
            }
        }
    }

    //Uniform data struct to move data from the "Game World" to the "Render World" with the ShaderType derived
    #[derive(ShaderType, Clone)]
    struct RayMarchingMaterialUniformData {
        camera_position: Vec3,
        camera_forward: Vec3,
        camera_horizontal: Vec3,
        camera_vertical: Vec3,
        aspect_ratio: f32,
    }
    {% endhighlight %}

I updated the shader uniforms by adding uniforms for the forward vector, right (horizontal), and up (vertical) vector. I also added a function "new" for the ray marching material to create a default RayMarchingMaterial. I did this because after the update, the initial conditions set in the function "setup" will no longer affect the scene since those values will always be overwritten when it is connected to the Camera2d. Lastly, I updated RayMarchingMaterialUniformData because all of the values we get from Camera2d will need to be copied to the shader.

Next, I updated the extract function so that the transform from our Camera2d is extracted to the render world along with the RayMarchingMaterial.

    {% highlight rust %}
    //Move information from the "Game World" to the "Render World"
    fn extract_raymarching_material(
        mut commands: Commands,
        ray_marching_query: Extract<Query<(Entity, &Handle<RayMarchingMaterial>)>>,
        aspect_ratio_resource: Extract<Res<AspectRatio>>,
        //Query added
        camera_query: Extract<Query<&Transform, With<Camera2d>>>
    ) {
        for (entity, material_handle) in ray_marching_query.iter() {
            //Updated to include the camera transform with the material
            let mut entity = commands.get_or_spawn(entity);
            entity.insert(material_handle.clone());
            for transform in camera_query.iter() {
                entity.insert(*transform);
            }
    }
    {% endhighlight %}

I then updated the prepare function so that the correct things were copied to the shader.

    {% highlight rust %}
        fn prepare_raymarching_material(
        materials: Res<RenderMaterials2d<RayMarchingMaterial>>,
        material_query: Query<&Handle<RayMarchingMaterial>>,
        //Added transform to the query
        material_query: Query<(&Transform, &Handle<RayMarchingMaterial>)>,
        render_queue: Res<RenderQueue>,
        aspect_ratio_resource: Res<AspectRatio>,
        ) {
            ...
            //Include transform in the loop
            for (transform, material_handle) in &material_query {
                if let Some(material) = materials.get(material_handle) {
                    for binding in material.bindings.iter() {
                        if let OwnedBindingResource::Buffer(current_buffer) = binding {
                            let mut buffer = encase::UniformBuffer::new(Vec::new());
                            buffer.write(&RayMarchingMaterialUniformData {
                                //Write the position, forward vector, right vector, and up vector.
                                camera_position: transform.translation,
                                camera_forward: transform.forward(),
                                camera_horizontal: transform.right(),
                                camera_vertical: transform.up(),
                                apsect_ratio: aspect_ratio_resource.aspect_ratio,
                            }).unwrap();
                            //Start writing from 0 because we want to write the whole uniform buffer each time
                            render_queue.write_buffer(current_buffer, 0, buffer.as_ref());
                        }
                    }
                }
            }
        }
    {% endhighlight %}

Next, I updated the ray_marching_material.wgsl shader to include these changes to the uniform variables.

    {% highlight wgsl %}
        struct Camera {
        position: vec4<f32>,
        forward: vec3<f32>,
        horizontal: vec3<f32>,
        vertical: vec3<f32>,
        aspect_ratio: f32,
    };
    {% endhighlight %}

I also updated ray_marching_material.wgsl to use these uniform values to calculate our scene rays.

    {% highlight wgsl %}
    fn fragment(in: FragmentIn) -> @location(0) vec4<f32> {
        var camera_origin = camera.position;
        var ray_origin = camera_origin + camera.forward + (in.uv_coords.x * camera.horizontal) + (in.uv_coords.y * camera.vertical);
        ...
    }
    {% endhighlight %}

By adding the camera.forward vector I move the ray origin one unit in front of the camera. I also added the u coordinate multiplied by the horizontal vector. Remember above, the u coordinate corresponds to the local x coordinate of the frame buffer we our casting our ray into. So by multiplying by the horizontal vector, the point is moving to the correct place to the right or left of the camera origin on our frame buffer. The y coordinate multiplied by the vertical vector corresponds to the local y coordinate of the frame buffer. This will move the point to the correct place up and down of the camera origin on the frame buffer. 

The last thing I did was update the "setup" function in main.rs so that the camera is positioned at (0.0, 0.0, 5.0) and the RayMarchingMaterial uses the "new" function defined for it.

    {% highlight rust %}
    fn setup (
        ...
    ) {
        commands.spawn(Camera2dBundle {
            transform: Transform::from_xyz(0.0, 0.0, 5.0),
            ..default()
        });
        commands.spawn(MaterialMesh2dBundle {
            mesh: meshes.add(Mesh::from(ScreenSpaceQuad::default())).into(),
            material: materials.add(RayMarchingMaterial::new()),
            ..default()
        });
    }
    {% endhighlight %}

Now after running the program again, everything still looks the same.

### Moving the Camera
Everything is now setup to move the camera in the scene. The first thing I did was add this system to the "main" function in main.rs to register the camera movement function.

    {% highlight rust %}
    fn main() {
        ...
        .add_plugin(resize_event)
        //Added this line
        .add_plugin(process_camera_translation)
        ...
    }
    {% endhighlight %}

I then created the "process_camera_translation" function below.

    {% highlight rust %}
    fn process_camera_translation(
        keys: Res<Input<KeyCode>>,
        mut camera_query: Query<&mut Transform, With<Camera2d>>,
        time: Res<Time>, 
    ) {
        const SPEED: f32 = 1.0;
        for mut transform in camera_query.iter_mut() {
            let forward_vector = transform.forward();
            let horizontal_vector = transform.right();
            let vertical_vector = transform.up();
            if keys.pressed(KeyCode::W) {
                transform.translation += forward_vector * SPEED * time.delta_seconds();
            }
            if keys.pressed(KeyCode::S) {
                transform.translation -= forward_vector * SPEED * time.delta_seconds();
            }
            if keys.pressed(KeyCode::A) {
                transform.translation -= horizontal_vector * SPEED * time.delta_seconds();
            }
            if keys.pressed(KeyCode::D) {
                transform.translation += horizontal_vector * SPEED * time.delta_seconds();
            }
            if keys.pressed(KeyCode::R) {
                transform.translation += vertical_vector * SPEED * time.delta_seconds();
            }
            if keys.pressed(KeyCode::F) {
                transform.translation -= vertical_vector * SPEED * time.delta_seconds();
            }
        }
    }
    {% endhighlight %}

I queried for the Camera2d along with the Camera2d transform. I also got the resource for the Time and keyboard input. Inside of the function, I updated camera transform by the transforms forward, right, or up vectors based on which key was pressed.

### Rotating the Camera
I also wanted to rotate the camera in the pitch and yaw directions using the mouse while the right mouse button was pressed. So I first updated the "main" function in main.rs to add another system called process_camera_rotation.

    {% highlight rust %}
    fn main() {
        ...
        .add_plugin(resize_event)
        .add_plugin(process_camera_translation)
        //Added this line
        .add_plugin(process_camera_rotation);
        ...
    }
    {% endhighlight %}

I then created the "process_camera_rotation" function below.

    {% highlight rust %}
    fn process_camera_rotation(
        mut motion_event: EventReader<MouseMotion>,
        mouse_buttons: Res<Input<MouseButton>>,
        mut camera_query: Query<&mut Transform, With<Camera2d>>,
        time: Res<Time>
    ) {
        for event in motion_event.iter() {
            const ROTATION_SPEED: f32 = 0.1;
            if mouse_buttons.pressed(MouseButton::Right) {
                for mut transform in camera_query.iter_mut() {
                    transform.rotate_local_x(-event.delta.y * ROTATION_SPEED * time.delta_seconds());
                    transform.rotate_local_y(-event.delta.x * ROTATION_SPEED * time.delta_seconds());
                }
            }
        }
    }
    {% endhighlight %}

Again, I queried for the camera along with its transform. I also got the MouseMotion and Input event for the mouse buttons. When the right mouse button is pressed, I updated the rotation around the local x (local pitch) and local y (local yaw). If you do not rotate around the local x and y of the camera you will see a much different behavior than what you would expect moving the mouse.

## Final Result
After all of this, I have a basic fly-camera that I can use to move around the scene.

![fly-cam-demo](/assets/fly-cam-demo.gif)

## What's Next
There are plenty of things I plan to add to this project in the future which includes:
- Multiple different shapes
- CSG operations (unions, intersections, subtractions, etc.)
- Shadows and ambient occlusion
- Movable point light source via a uniform buffer

Thank you for reading. I hope you enjoyed it!

{% include utterances.html %}