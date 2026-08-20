# 第二章 快速开始

本章带你从零跑起一个 egui 程序。我们用官方的 `eframe` 框架，因为它最省事。

## 2.1 建工程

```bash
cargo new my_egui_app
cd my_egui_app
```

编辑 `Cargo.toml`：

```toml
[package]
name = "my_egui_app"
version = "0.1.0"
edition = "2021"

[dependencies]
eframe = "0.36"      # 版本号按你本地 egui 源码的版本填
egui = "0.36"
env_logger = "0.11"
```

> egui 还在快速迭代，每个版本都有破坏性改动。如果你的代码在某版跑不动，先看 CHANGELOG。

## 2.2 最短能跑的程序

`src/main.rs`：

```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")] // release 下隐藏控制台

use eframe::egui;

fn main() -> eframe::Result {
    env_logger::init(); // 初始化日志

    let options = eframe::NativeOptions {
        viewport: egui::ViewportBuilder::default().with_inner_size([320.0, 240.0]),
        ..Default::default()
    };

    eframe::run_native(
        "My egui App",
        options,
        Box::new(|cc| {
            // 如果你需要加载图片，装一下官方图片加载器：
            // egui_extras::install_image_loaders(&cc.egui_ctx);
            Ok(Box::<MyApp>::default())
        }),
    )
}

struct MyApp {
    name: String,
    age: u32,
}

impl Default for MyApp {
    fn default() -> Self {
        Self {
            name: "Arthur".to_owned(),
            age: 42,
        }
    }
}

impl eframe::App for MyApp {
    fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ui, |ui| {
            ui.heading("My egui Application");
            ui.horizontal(|ui| {
                let name_label = ui.label("Your name: ");
                ui.text_edit_singleline(&mut self.name)
                    .labelled_by(name_label.id);
            });
            ui.add(egui::Slider::new(&mut self.age, 0..=120).text("age"));
            if ui.button("Increment").clicked() {
                self.age += 1;
            }
            ui.label(format!("Hello '{}', age {}", self.name, self.age));
        });
    }
}
```

运行：

```bash
cargo run
```

你会看到一个 320×240 的窗口，里面有一个名字输入框、一个年龄滑块、一个按钮和一行问候语。

## 2.3 程序结构解读

整个程序只有三块：

### （1）`main`：启动

`eframe::run_native` 接三个参数：

- 窗口标题。
- `NativeOptions`：里面有个 `viewport`，控制窗口大小、图标等。
- 一个闭包，接收 `&CreationContext`（简称 `cc`），返回你的 `App` 实例。`cc.egui_ctx` 可以在程序启动前用来设置字体、样式等。

### （2）`MyApp`：你的状态

egui 是即时模式，**它不替你存状态**。你的所有数据（名字、年龄、滚动位置等）都得自己存。`MyApp` 这个结构体就是你的状态容器。

### （3）`impl eframe::App for MyApp`：每帧调用

`eframe::App` 这个 trait 最关键的函数是 `ui`：

```rust
fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame)
```

它**每帧都会被调用**（60 FPS 或更高）。你在里面写 UI 代码。`ui` 参数是当前帧的根 `Ui`，你已经可以往里加控件了。

> 注意：新版的 `eframe::App::ui` 直接给你一个 `&mut egui::Ui`，并且**没有 margin 和背景色**。所以你一般会先套一个 `CentralPanel::default().show(ui, |ui| { ... })` 来获得标准的内边距和背景。

## 2.4 不写 struct 的更短写法

如果你只是想画两下界面、懒得定义结构体，可以用 `eframe::run_ui_native`：

```rust
use eframe::egui;

fn main() -> eframe::Result {
    let options = eframe::NativeOptions {
        viewport: egui::ViewportBuilder::default().with_inner_size([320.0, 240.0]),
        ..Default::default()
    };

    let mut name = "Arthur".to_owned();
    let mut age = 42u32;

    eframe::run_ui_native("My egui App", options, move |ui, _frame| {
        egui::CentralPanel::default().show(ui, |ui| {
            ui.heading("My egui Application");
            ui.horizontal(|ui| {
                ui.label("Your name: ");
                ui.text_edit_singleline(&mut name);
            });
            ui.add(egui::Slider::new(&mut age, 0..=120).text("age"));
            if ui.button("Increment").clicked() {
                age += 1;
            }
            ui.label(format!("Hello '{name}', age {age}"));
        });
    })
}
```

这种写法 egui 不管理状态，`name` 和 `age` 在闭包外存活。**适合一次性脚本或简单示例**，正式项目还是建议用 `App` trait。

## 2.5 跑网页版

想让同一份代码跑在网页上：

1. 看 <https://github.com/emilk/eframe_template> 模板，里面有完整的 wasm 配置。
2. 大致流程是：加 `wasm-bindgen`、` trunk` 或 `wasm-pack`，把 `run_native` 换成 `WebRunner::new()` + `eframe::WebRunner::start`。

具体配置看模板，不要硬背。

## 2.6 跑示例

egui 源码的 `examples/` 目录里有几十个示例，非常值得一个个看：

```bash
cd egui-main
cargo run --example hello_world
cargo run --example hello_world_simple
cargo run --example custom_font
cargo run --example custom_3d_glow
cargo run --example images
cargo run --example popups
cargo run --example keyboard_events
```

完整列表见 `examples/README.md`。这是学 egui 最快的方式。
