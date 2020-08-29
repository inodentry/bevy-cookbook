# Bevy Cookbook

List of concise recipes for how to do common game development tasks in the Bevy game engine.

Please help improve it and keep it up to date by contributing on [GitHub](https://github.com/jamadazi/bevy-cookbook).

If you like this, you should also have a look at the [Bevy Cheatsheet](https://github.com/jamadazi/bevy-cheatsheet).

Table of Contents
=================

* [Input Handling](#input-handling)
* [Convert screen coordinates to world coordinates](#convert-screen-coordinates-to-world-coordinates)
  * [2D games](#2d-games)
  * [3D games](#3d-games)
* [Grabbing the mouse](#grabbing-the-mouse)


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
    ev_keys: Res<Events<KeyboardInput>,
    ev_cursor: Res<Events<CursorMoved>,
    ev_motion: Res<Events<MouseMotion>,
    ev_mousebtn: Res<Events<MouseButtonInput>,
    ev_scroll: Res<Events<MouseWheel>,
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
    let camera_transform = q_camera.get::<Transform>(state.camera_e);

    for ev in state.cursor.iter(&ev_cursor) {
        // get the size of the window that the event is for
        let wnd = wnds.get(ev.id).unwrap();
        let size = Vec2::new(wnd.width as f32, wnd.height as f32);

        // the default orthographic projection is in pixels from the center;
        // just undo the translation
        let p = ev.position - size / 2.0;

        // apply the camera transform
        let pos_wld = transform.value * p.extend(0.0).extend(1.0);
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

