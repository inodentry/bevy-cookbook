# Livro de receitas para Bevy Game Engine

Lista de receitas concisas de como fazer tarefas comuns de desenvolvimento de jogos no [Bevy Game Engine](https://github.com/bevyengine/bevy).

Ajude a melhorá-lo e mantê-lo atualizado, contribuindo em [GitHub](https://github.com/jamadazi/bevy-cookbook).

Os exemplos neste livro de receitas pressupõem que você esteja familiarizado com os fundamentos da programação em Bevy. Se você precisar de uma atualização, pode consultar o [Folha de dicas para Bevy Game Engine](https://github.com/jamadazi/bevy-cheatsheet).

Tabela de Conteúdos
=================

- [Livro de receitas para Bevy Game Engine](#livro-de-receitas-para-bevy-game-engine)
- [Tabela de Conteúdos](#tabela-de-conteúdos)
- [Receitas](#receitas)
  - [Manipulação de Entrada](#manipulação-de-entrada)
  - [Saindo do Aplicativo](#saindo-do-aplicativo)
  - [Converta as Coordenadas da Tela em Coordenadas Mundiais](#converta-as-coordenadas-da-tela-em-coordenadas-mundiais)
    - [Jogos 2D](#jogos-2D)
    - [Jogos 3D](#jogos-3D)
  - [Pegando o Mouse](#pegando-o-mouse)
  - [Projeção de Câmera Personalizada](#projeção-de-câmera-personalizada)
  - [Câmera Panorâmica e Orbital](#câmera-panorâmica-e-orbital)

# Receitas

## Manipulação de Entrada

Para simplesmente verificar o estado atual de teclas específicas ou botões do mouse, use o recurso `Input<T>`:

```rust
fn meu_sistema(teclas: Res<Input<KeyCode>>, botoes: Res<Input<MouseButton>>) {
    // Entrada de teclado
    if teclas.pressed(KeyCode::Space) {
        // o espaço está sendo mantido pressionado
    }

    // Botões do mouse
    if botoes.just_pressed(MouseButton::Left) {
        // um clique esquerdo acabou de acontecer
    }
}
```

Para detectar todas as atividades, use eventos de entrada. Crie um recurso para conter os leitores e qualquer outro estado que você possa precisar.

```rust
struct MeuEstadoDeEntrada {
    teclas: EventReader<KeyboardInput>,
    cursor: EventReader<CursorMoved>,
    movimento: EventReader<MouseMotion>,
    mousebotao: EventReader<MouseButtonInput>,
    scroll: EventReader<MouseWheel>,
}

fn meu_sistema_de_entrada(
    mut estado: ResMut<MeuEstadoDeEntrada>,
    ev_teclas: Res<Events<KeyboardInput>>,
    ev_cursor: Res<Events<CursorMoved>>,
    ev_movimento: Res<Events<MouseMotion>>,
    ev_mousebotao: Res<Events<MouseButtonInput>>,
    ev_scroll: Res<Events<MouseWheel>>,
) {
    // Entrada de teclado
    for ev in estado.keys.iter(&ev_teclas) {
        if ev.estado.is_pressed() {
            eprintln!("Tecla apenas pressionada: {:?}", ev.key_code);
        } else {
            eprintln!("Tecla recém-liberada: {:?}", ev.key_code);
        }
    }

    // Posição absoluta do cursor (em coordenadas da janela)
    for ev in estado.cursor.iter(&ev_cursor) {
        eprintln!("Cursor em: {}", ev.position);
    }

    // Movimento relativo do mouse
    for ev in estado.motion.iter(&ev_movimento) {
        eprintln!("Mouse se moveu {} pixels", ev.delta);
    }

    // Botões do mouse
    for ev in estado.mousebtn.iter(&ev_mousebotao) {
        if ev.estado.is_pressed() {
            eprintln!("Botão do mouse pressionado: {:?}", ev.button);
        } else {
            eprintln!("Botão do mouse recém-liberado: {:?}", ev.button);
        }
    }

    // rolagem (roda do mouse, touchpad, etc.)
    for ev in estado.scroll.iter(&ev_scroll) {
        eprintln!("Rolou verticalmente por {} e horizontalmente por {}.", ev.y, ev.x);
    }
}
```

## Saindo do Aplicativo

Para encerrar o Bevy de forma limpa, envie um evento `AppExit` de qualquer sistema.

Para simplesmente sair quando a tecla Esc for pressionada, o Bevy fornece um sistema que você pode simplesmente adicionar ao seu aplicativo:

```rust
fn main() {
    App::build()
        .add_default_plugins()
        .add_system(bevy::input::system::exit_on_esc_system.system())
        .run();
}
```

## Converta as Coordenadas da Tela em Coordenadas Mundiais

O Bevy ainda não fornece funções para ajudar a descobrir para onde o cursor está apontando. Soluções manuais:

### Jogos 2D

Usando a câmera ortográfica 2D embutida:

```rust
struct EstadoDoMeuCursor {
    cursor: EventReader<CursorMoved>,
    // precisa identificar a câmera principal
    camera_e: Entity,
}

fn sistema_do_meu_cursor(
    mut estado: ResMut<EstadoDoMeuCursor>,
    ev_cursor: Res<Events<CursorMoved>>,
    // precisa obter as dimensões da janela
    wnds: Res<Windows>,
    // consulta para obter componentes da câmera
    consulta_da_camera: Query<&Transform>
) {
    let transformar_camera = consulta_da_camera.get(estado.camera_e).unwrap();

    for ev in estado.cursor.iter(&ev_cursor) {
        // obter o tamanho da janela para a qual o evento se destina
        let wnd = wnds.get(ev.id).unwrap();
        let size = Vec2::new(wnd.width() as f32, wnd.height() as f32);

        // a projeção ortográfica padrão é em pixels a partir do centro;
        // apenas desfaça a tradução
        let p = ev.position - size / 2.0;

        // aplicar a transformação da câmera
        let pos_wld = transformar_camera.compute_matrix() * p.extend(0.0).extend(1.0);
        eprintln!("World coords: {}/{}", pos_wld.x(), pos_wld.y());
    }
}

fn setup(mut comandos: Commands) {
    let camera = Camera2dComponents::default();
    let e = comandos.spawn(camera).current_entity().unwrap();
    comandos.insert_resource(EstadoDoMeuCursor {
        cursor: Default::default(),
        camera_e: e,
    });
}
```

### Jogos 3D

Tente o plugin [`bevy_mod_picking`](https://github.com/aevyrie/bevy_mod_picking).

## Pegando o Mouse

Isso pode ser feito por meio da [API de configurações da janela](https://github.com/bevyengine/bevy/blob/master/examples/window/window_settings.rs).

## Projeção de Câmera Personalizada

Atualmente, o Bevy oferece apenas uma projeção em perspectiva padrão (para 3D) e uma projeção ortográfica baseada em pixels da janela (para 2D).

Se precisar de mais alguma coisa, como uma projeção ortográfica sem tamanho de pixel (para 2D ou 3D), você precisa criar a sua própria.

```rust
use bevy::render::camera::{Camera, CameraProjection, DepthCalculation, VisibleEntities};

struct ProjecaoOrthoSimples {
    distancia: f32,
    aspecto: f32,
}

impl CameraProjection for ProjecaoOrthoSimples {
    fn get_projection_matrix(&self) -> Mat4 {
        Mat4::orthographic_rh(
            -self.aspecto, self.aspecto, -1.0, 1.0, 0.0, self.far
        )
    }

    // o que fazer no redimensionamento da janela
    fn update(&mut self, largura: usize, altura: usize) {
        self.aspecto = largura as f32 / altura as f32;
    }

    fn depth_calculation(&self) -> DepthCalculation {
        // para ortográfico
        DepthCalculation::ZDifference

        // de outra forma
        // DepthCalculation::Distance
    }
}

impl Default for ProjecaoOrthoSimples {
    fn default() -> Self {
        Self { distancia: 1000.0, aspecto: 1.0 }
    }
}

fn setup(mut comandos: Commands) {
    // mesmos componentes do pacote Camera2dComponents,
    // mas com nossa projeção customizada

    let proj = ProjecaoOrthoSimples::default();

    // Precisa definir o nome da câmera para uma das constantes mágicas internas do grupo,
    // dependendo de qual câmera estamos implementando (2D, 3D ou UI).
    // Bevy usa esse nome para encontrar a câmera e configurar a renderização.
    // Como este exemplo é uma câmera 2d:
    let nome_da_camera = bevy::render::render_graph::base::camera::CAMERA2D;

    let mut camera = Camera::default();
    camera.name = Some(nome_da_camera.to_string());

    comandos.spawn((
        camera,
        proj,
        VisibleEntities::default(),
        // posicione a câmera como o Bevy faria por padrão em 2D:
        Transform::from_translation(Vec3::new(0.0, 0.0, proj.far - 0.1)),
        GlobalTransform::default(),
    ));
}

fn main() {
    // precisa adicionar um sistema de câmera interno Bevy para atualizar
    // a projeção no redimensionamento da janela

    use bevy::render::camera::camera_system;

    App::build()
        .add_default_plugins()
        .add_startup_system(setup.system())
        .add_system_to_stage(
            bevy::app::stage::POST_UPDATE,
            camera_system::<ProjecaoOrthoSimples>.system(),
        )
        .run();
}
```

## Câmera Panorâmica e Orbital

Fornece uma câmera intuitiva que gira com o botão esquerdo ou o botão de rolagem e orbita com o clique direito.

```rust
/// Marca uma entidade como capaz de fazer panorâmica e orbitar.
struct CameraPanOrbit {
    /// O "ponto de foco" para orbitar. É atualizado automaticamente ao girar a câmera
    pub foco: Vec3,
}

impl Default for CameraPanOrbit {
    fn default() -> Self {
        CameraPanOrbit {
            foco: Vec3::zero(),
        }
    }
}

/// Retenha leitores para eventos
#[derive(Default)]
struct EstadoDeEntrada {
    pub movimento_do_leitor: EventReader<MouseMotion>,
    pub scroll_do_leitor: EventReader<MouseWheel>,
}

/// Faça panorâmicas da câmera com Hold e roda de rolagem, orbite com clique.
fn camera_pan_orbit(
    tempo: Res<Time>,
    janelas: Res<Windows>,
    mut estado: ResMut<EstadoDeEntrada>,
    ev_movimento: Res<Events<MouseMotion>>,
    mouse_botao: Res<Input<MouseButton>>,
    ev_scroll: Res<Events<MouseWheel>>,
    mut consulta: Query<(&mut CameraPanOrbit, &mut Transform)>,
) {
    let mut translation = Vec2::zero();
    let mut moviment_de_rotacao = Vec2::default();
    let mut scroll = 0.0;
    let tempo_delta = tempo.delta_seconds;

    if mouse_botao.pressed(MouseButton::Right) {
        for ev in estado.movimento_do_leitor.iter(&ev_movimento) {
            moviment_de_rotacao += ev.delta;
        }
    } else if mouse_botao.pressed(MouseButton::Left) {
        // Pan apenas se não estivermos girando no momento
        for ev in estado.movimento_do_leitor.iter(&ev_movimento) {
            translation += ev.delta;
        }
    }

    for ev in estado.scroll_do_leitor.iter(&ev_scroll) {
        scroll += ev.y;
    }

    // Pan + scroll ou arc ball. Não fazemos os dois ao mesmo tempo.
    for (mut camera, mut trans) in consulta.iter_mut() {
        if moviment_de_rotacao.length_squared() > 0.0 {
            let janela = janelas.get_primary().unwrap();
            let janela_w = janela.width() as f32;
            let janela_h = janela.height() as f32;

            // Vincule a rotação da esfera virtual em relação à janela para torná-la mais agradável
            let delta_x = moviment_de_rotacao.x() / janela_w * std::f32::consts::PI * 2.0;
            let delta_y = moviment_de_rotacao.y() / janela_h * std::f32::consts::PI;

            let delta_yaw = Quat::from_rotation_y(delta_x);
            let delta_pitch = Quat::from_rotation_x(delta_y);

            trans.translation =
                delta_yaw * delta_pitch * (trans.translation - camera.foco) + camera.foco;

            let look = Mat4::face_toward(trans.translation, camera.foco, Vec3::new(0.0, 1.0, 0.0));
            trans.rotation = look.to_scale_rotation_translation().1;
        } else {
            // O plano é x/y enquanto z está "para cima". Multiplicar por tempo_delta permite uma taxa de panorâmica constante
            let mut translation = Vec3::new(-translation.x() * tempo_delta, translation.y() * tempo_delta, 0.0);
            camera.foco += translation;
            *translation.z_mut() = -scroll;
            trans.translation += translation;
        }
    }
}

// Crie uma câmera como esta:

fn spawn_camera(mut comandos: Commands) {
    comandos.spawn(CameraPanOrbit::default(),)
        .with_bundle(Camera3dComponents {
            ..Default::default()
        });
}
```
