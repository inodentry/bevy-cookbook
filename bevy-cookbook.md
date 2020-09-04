# Bevy Cookbook

List of concise recipes for how to do common game development tasks in the [Bevy game engine](https://github.com/bevyengine/bevy).

Please help improve it and keep it up to date by contributing on [GitHub](https://github.com/jamadazi/bevy-cookbook).

If you like this, you should also have a look at the [Bevy Cheatsheet](https://github.com/jamadazi/bevy-cheatsheet).

Table of Contents
=================

- [Bevy Cookbook](#bevy-cookbook)
- [Table of Contents](#table-of-contents)
- [Recipes](#recipes)
  - [Input Handling](#input-handling)
  - [Convert screen coordinates to world coordinates](#convert-screen-coordinates-to-world-coordinates)
    - [2D games](#2d-games)
    - [3D games](#3d-games)
  - [Grabbing the mouse](#grabbing-the-mouse)
  - [Custom camera projection](#custom-camera-projection)
  - [Pan + Orbit Camera](#pan--orbit-camera)

# Recipes

## Input Handling

To simply check the current state of specific keys or mouse buttons, use the `Input<T>` resource:

```rust
fn my_system(keys: Res<Input<KeyCode>>, btns: Res<Input<MouseButton>>) {
    // Keyboard input
    if keys.pressed(KeyCode::Space) {
        // space is being held down
    }

    // Mouse buttons
    if btns.just_pressed(MouseButton::Left) {
        // a left click just happened
    }
}
```

To detect all activity, use input events. Create a resource to hold the readers and any other state you might need.

```rust
struct MyInputState {
    keys: EventReader<KeyboardInput>,
    cursor: EventReader<CursorMoved>,
    motion: EventReader<MouseMotion>,
    mousebtn: EventReader<MouseButtonInput>,
    scroll: EventReader<MouseWheel>,
}

fn my_input_system(
    mut state: ResMut<MyInputState>,
    ev_keys: Res<Events<KeyboardInput>>,
    ev_cursor: Res<Events<CursorMoved>>,
    ev_motion: Res<Events<MouseMotion>>,
    ev_mousebtn: Res<Events<MouseButtonInput>>,
    ev_scroll: Res<Events<MouseWheel>>,
) {
    // Keyboard input
    for ev in state.keys.iter(&ev_keys) {
        if ev.state.is_pressed() {
            eprintln!("Just pressed key: {:?}", ev.key_code);
        } else {
            eprintln!("Just released key: {:?}", ev.key_code);
        }
    }

    // Absolute cursor position (in window coordinates)
    for ev in state.cursor.iter(&ev_cursor) {
        eprintln!("Cursor at: {}", ev.position);
    }

    // Relative mouse motion
    for ev in state.motion.iter(&ev_motion) {
        eprintln!("Mouse moved {} pixels", ev.delta);
    }

    // Mouse buttons
    for ev in state.mousebtn.iter(&ev_mousebtn) {
        if ev.state.is_pressed() {
            eprintln!("Just pressed mouse button: {:?}", ev.button);
        } else {
            eprintln!("Just released mouse button: {:?}", ev.button);
        }
    }

    // scrolling (mouse wheel, touchpad, etc.)
    for ev in state.scroll.iter(&ev_scroll) {
        eprintln!("Scrolled vertically by {} and horizontally by {}.", ev.y, ev.x);
    }
}
```

## Convert screen coordinates to world coordinates

Bevy does not yet provide functions to help with finding out what the cursor is pointing at. Manual solutions:

### 2D games

Using the built-in 2d orthographic camera:

```rust
struct MyCursorState {
    cursor: EventReader<CursorMoved>,
    // need to identify the main camera
    camera_e: Entity, 
}

fn my_cursor_system(
    mut state: ResMut<MyCursorState>,
    ev_cursor: Res<Events<CursorMoved>>,
    // need to get window dimensions
    wnds: Res<Windows>,
    // query to get camera components
    q_camera: Query<&Transform>
) {
    let camera_transform = q_camera.get::<Transform>(state.camera_e).unwrap();

    for ev in state.cursor.iter(&ev_cursor) {
        // get the size of the window that the event is for
        let wnd = wnds.get(ev.id).unwrap();
        let size = Vec2::new(wnd.width as f32, wnd.height as f32);

        // the default orthographic projection is in pixels from the center;
        // just undo the translation
        let p = ev.position - size / 2.0;

        // apply the camera transform
        let pos_wld = camera_transform.value * p.extend(0.0).extend(1.0);
        eprintln!("World coords: {}/{}", pos_wld.x(), pos_wld.y());
    }
}

fn setup(mut commands: Commands) {
    let camera = Camera2dComponents::default();
    let e = commands.spawn(camera).current_entity().unwrap();
    commands.insert_resource(MyCursorState {
        cursor: Default::default(),
        camera_e: e,
    };
}
```

### 3D games

TODO; raycasting?

## Grabbing the mouse

TODO: show how to grab the mouse for FPS games and such

## Custom camera projection

Bevy currently only offers a standard perspective projection (for 3d) and an orthographic projection based on window pixels (for 2d).

If you need anything else, such as a non-pixel-sized orthographic projection (for 2d or 3d), you need to create your own.

```rust
use bevy::render::camera::{Camera, CameraProjection, DepthCalculation, VisibleEntities};

struct SimpleOrthoProjection {
    aspect: f32,
}

impl CameraProjection for SimpleOrthoProjection {
    fn get_projection_matrix(&self) -> Mat4 {
        Mat4::orthographic_rh(
            -self.aspect, self.aspect, -1.0, 1.0, 0.0, 1000.0
        )
    }

    // what to do on window resize
    fn update(&mut self, width: usize, height: usize) {
        self.aspect = width as f32 / height as f32;
    }

    fn depth_calculation(&self) -> DepthCalculation {
        // for orthographic
        DepthCalculation::ZDifference

        // otherwise
        //DepthCalculation::Distance
    }
}

impl Default for SimpleOrthoProjection {
    fn default() -> Self {
        Self { aspect: 1.0 }
    }
}

fn setup(mut commands: Commands) {
    // same components as the Camera2dComponents bundle,
    // but with our custom projection
    commands.spawn((
        Camera::default(),
        SimpleOrthoProjection::default(),
        VisibleEntities::default(),
        Transform::default(),
        Translation::default(),
        Rotation::default(),
        Scale::default(),
    ));
}

fn main() {
    // need to add a bevy-internal camera system to update
    // the projection on window resizing

    use bevy::render::camera::camera_system;

    App::build().add_default_plugins()
        .add_startup_system(setup.system())
        .add_system_to_stage(
            bevy::app::stage::POST_UPDATE,
            camera_system::<SimpleOrthoProjection>.system(),
        )
        .run();
}
```

## Pan + Orbit Camera

Provide an intuitive camera that pans with left click or scrollwheel, and orbits with right click.

```rust
/// Tags an entity as capable of panning and orbiting.
struct PanOrbitCamera {
    /// The "focus point" to orbit around. It is automatically updated when panning the camera
    pub focus: Vec3
}

impl Default for PanOrbitCamera {
    fn default() -> Self {
        PanOrbitCamera {
            focus: Vec3::zero()
        }
    }
}

/// Hold readers for events
#[derive(Default)]
struct InputState {
    pub reader_motion: EventReader<MouseMotion>,
    pub reader_scroll: EventReader<MouseWheel>,
}

/// Pan the camera with LHold or scrollwheel, orbit with rclick.
fn pan_orbit_camera(
    time: Res<Time>,
    mut windows: Res<Windows>,
    mut state: ResMut<InputState>,
    ev_motion: Res<Events<MouseMotion>>,
    mousebtn: Res<Input<MouseButton>>,
    ev_scroll: Res<Events<MouseWheel>>,
    mut query: Query<(&mut PanOrbitCamera, &mut Translation, &mut Rotation)>
) {
    let mut translation = Vec2::zero();
    let mut rotation_move = Vec2::default();
    let mut scroll = 0.0;
    let dt = time.delta_seconds;

    if mousebtn.pressed(MouseButton::Right) {
        for ev in state.reader_motion.iter(&ev_motion) {
            rotation_move += ev.delta;
        }
    } else if mousebtn.pressed(MouseButton::Left) {
        // Pan only if we're not rotating at the moment
        for ev in state.reader_motion.iter(&ev_motion) {
            translation += ev.delta;
        }
    }

    for ev in state.reader_scroll.iter(&ev_scroll) {
        scroll += ev.y;
    }

    // Either pan+scroll or arcball. We don't do both at once.
    for (mut camera, mut trans, mut rotation) in &mut query.iter() {
        if rotation_move.length_squared() > 0.0 {
            let window = windows.get_primary().unwrap();
            let window_w = window.width as f32;
            let window_h = window.height as f32;

            // Link virtual sphere rotation relative to window to make it feel nicer
            let delta_x = rotation_move.x() / window_w * std::f32::consts::PI * 2.0;
            let delta_y = rotation_move.y() / window_h * std::f32::consts::PI;

            let delta_yaw = Quat::from_rotation_y(delta_x);
            let delta_pitch = Quat::from_rotation_x(delta_y);

            trans.0 = delta_yaw * delta_pitch * (trans.0 - camera.focus) + camera.focus;

            let look = Mat4::face_toward(trans.0, camera.focus, Vec3::new(0.0, 1.0, 0.0));
            rotation.0 = look.to_scale_rotation_translation().1;
        } else {
            // The plane is x/y while z is "up". Multiplying by dt allows for a constant pan rate
            let mut translation = Vec3::new(-translation.x() * dt, translation.y() * dt, 0.0);
            camera.focus += translation;
            *translation.z_mut() = -scroll;
            trans.0 += translation;
        }
    }
}

// Spawn a camera like this:

fn spawn_camera(mut commands: Commands) {
    commands.spawn((PanOrbitCamera::default(),))
        .with_bundle(Camera3dComponents {
            ..Default::default()
        });
}

```
