# Bevy Cookbook

List of concise recipes for how to do common game development tasks in the [Bevy game engine](https://github.com/bevyengine/bevy).

Please help improve it and keep it up to date by contributing on [GitHub](https://github.com/jamadazi/bevy-cookbook).

The examples in this cookbook assume you are familiar with the basics of programming in Bevy. If you need a refresher, you can refer to the [Bevy Cheatsheet](https://github.com/jamadazi/bevy-cheatsheet).

Table of Contents
=================

- [Bevy Cookbook](#bevy-cookbook)
- [Table of Contents](#table-of-contents)
- [Recipes](#recipes)
  - [Input Handling](#input-handling)
  - [Quitting the App](#quitting-the-app)
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

## Quitting the App

To cleanly shut down bevy, send an `AppExit` event from any system.

To simply exit when the Esc key is pressed, bevy provides a system that you can just add to your App:

```rust
fn main() {
    App::build().add_default_plugins()
        .add_system(bevy::input::system::exit_on_esc_system.system())
        .run();
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
        let size = Vec2::new(wnd.width() as f32, wnd.height() as f32);

        // the default orthographic projection is in pixels from the center;
        // just undo the translation
        let p = ev.position - size / 2.0;

        // apply the camera transform
        let pos_wld = *camera_transform.value() * p.extend(0.0).extend(1.0);
        eprintln!("World coords: {}/{}", pos_wld.x(), pos_wld.y());
    }
}

fn setup(mut commands: Commands) {
    let camera = Camera2dComponents::default();
    let e = commands.spawn(camera).current_entity().unwrap();
    commands.insert_resource(MyCursorState {
        cursor: Default::default(),
        camera_e: e,
    });
}
```

### 3D games

Try the [`bevy_mod_picking` plugin](https://github.com/aevyrie/bevy_mod_picking).

## Grabbing the mouse

**NOTE:** *In the latest git version, Bevy now provides an API for this.
See [the official example](https://github.com/bevyengine/bevy/blob/master/examples/window/window_settings.rs)
for how to use it.*

*The code below is now obsolete, but is kept here, because it is still relevant
to the release version.*

To grab the mouse and make it invisible for FPS style games we can use an event
to hook into the `winit` api as below. Note that this will cause events such as
`CursorMoved` to always trigger with the same position due to the position being
locked. However, `MouseMotion` events will still fire so the delta can be used.

An example system to lock/unlock the mouse, triggered by a custom event:

```rust
use bevy::{window::WindowId, winit::WinitWindows};

// To track if we want the cursor to be locked or unlocked.
enum MouseLockEvent {
    Locked,
    Unlocked,
}

#[derive(Default)]
struct MouseCaptureState {
    mouse_lock_event_reader: EventReader<MouseLockEvent>,
}

fn mouse_capture_system(
    mut state: ResMut<MouseCaptureState>,
    mouse_lock_events: Res<Events<MouseLockEvent>>,
    windows: Res<WinitWindows>,
) {
    // If we receive a MouseLockEvent then enable or disable the cursor.
    if let Some(event) = state.mouse_lock_event_reader.latest(&mouse_lock_events) {
        // We need to get a winit Window to call the appropriate methods on.
        let window = windows.get_window(WindowId::primary()).unwrap();
        match event {
            MouseLockEvent::Locked => {
                // Locks the cursor in its current place.
                window.set_cursor_grab(true).unwrap();
                // Hides the cursor.
                window.set_cursor_visible(false);
            }
            MouseLockEvent::Unlocked => {
                // Unlocks the cursor so the user can move it again.
                window.set_cursor_grab(false).unwrap();
                // Shows the cursor.
                window.set_cursor_visible(true);
            }
        }
    }
}
```

## Custom camera projection

Bevy currently only offers a standard perspective projection (for 3d) and an orthographic projection based on window pixels (for 2d).

If you need anything else, such as a non-pixel-sized orthographic projection (for 2d or 3d), you need to create your own.

```rust
use bevy::render::camera::{Camera, CameraProjection, DepthCalculation, VisibleEntities};

struct SimpleOrthoProjection {
    far: f32,
    aspect: f32,
}

impl CameraProjection for SimpleOrthoProjection {
    fn get_projection_matrix(&self) -> Mat4 {
        Mat4::orthographic_rh(
            -self.aspect, self.aspect, -1.0, 1.0, 0.0, self.far
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
        Self { far: 1000.0, aspect: 1.0 }
    }
}

fn setup(mut commands: Commands) {
    // same components as the Camera2dComponents bundle,
    // but with our custom projection

    let proj = SimpleOrthoProjection::default();

    // Need to set the camera name to one of the bevy-internal magic constants,
    // depending on which camera we are implementing (2D, 3D, or UI).
    // Bevy uses this name to find the camera and configure the rendering.
    // Since this example is a 2d camera:
    let cam_name = bevy::render::render_graph::base::camera::CAMERA2D;

    let mut camera = Camera::default();
    camera.name = Some(cam_name.to_string());

    commands.spawn((
        camera,
        proj,
        VisibleEntities::default(),
        // position the camera like bevy would do by default for 2D:
        Transform::from_translation(Vec3::new(0.0, 0.0, proj.far - 0.1)),
        GlobalTransform::default(),
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

**NOTE:** *this code uses new bevy APIs not in the released version `0.2.1`. Use bevy from git, until a new version is released.

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
    mut query: Query<(&mut PanOrbitCamera, &mut Transform)>
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
    for (mut camera, mut trans) in &mut query.iter() {
        if rotation_move.length_squared() > 0.0 {
            let window = windows.get_primary().unwrap();
            let window_w = window.width() as f32;
            let window_h = window.height() as f32;

            // Link virtual sphere rotation relative to window to make it feel nicer
            let delta_x = rotation_move.x() / window_w * std::f32::consts::PI * 2.0;
            let delta_y = rotation_move.y() / window_h * std::f32::consts::PI;

            let delta_yaw = Quat::from_rotation_y(delta_x);
            let delta_pitch = Quat::from_rotation_x(delta_y);

            trans.translation = delta_yaw * delta_pitch * (trans.translation - camera.focus) + camera.focus;

            let look = Mat4::face_toward(trans.translation, camera.focus, Vec3::new(0.0, 1.0, 0.0));
            trans.rotation = look.to_scale_rotation_translation().1;
        } else {
            // The plane is x/y while z is "up". Multiplying by dt allows for a constant pan rate
            let mut translation = Vec3::new(-translation.x() * dt, translation.y() * dt, 0.0);
            camera.focus += translation;
            *translation.z_mut() = -scroll;
            trans.translation += translation;
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
