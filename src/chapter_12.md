# 第十二章 eframe 与后端集成

egui 本身不知道怎么读输入、怎么画到屏幕。这件事由“集成（integration）”来做。本章讲官方的 `eframe`，以及怎么把 egui 嵌入到现有引擎里。

## 12.1 eframe 是什么

`eframe` 是官方框架，把 `egui` + `egui-winit`（输入）+ `egui_glow` 或 `egui-wgpu`（渲染）组装好。你写一份代码：

- **桌面**：编译成原生可执行文件（`cargo run`）。
- **网页**：编译成 WebAssembly，在浏览器里跑。

`eframe` 名字里的 “frame” 有两个意思：你的应用所在的“框” + framework（框架）。`eframe` 是框架，`egui` 是库。

## 12.2 eframe::App trait

实现这个 trait 就是一个 egui 应用：

```rust
pub trait App {
    fn logic(&mut self, ctx: &egui::Context, frame: &mut Frame) { /* 默认空 */ }

    fn ui(&mut self, ui: &mut egui::Ui, frame: &mut Frame);

    fn save(&mut self, _storage: &mut dyn Storage) {}
    fn on_exit(&mut self) {}
    fn auto_save_interval(&self) -> std::time::Duration { Duration::from_secs(30) }
    fn clear_color(&self, _visuals: &egui::Visuals) -> [f32; 4] { /* ... */ }
    fn persist_egui_memory(&self) -> bool { true }
    fn raw_input_hook(&mut self, _ctx: &egui::Context, _raw_input: &mut egui::RawInput) {}
}
```

关键函数：

- `logic`：每帧调用，在 `ui` 之前。**不能**在里面画 UI。用来更新状态、读输入、调 `ctx.request_repaint`。
- `ui`：每帧调用，画 UI。`ui` 参数是根 `Ui`，没有 margin 和背景，建议先套 `CentralPanel`。
- `save`：被定时调用（`auto_save_interval`），程序退出时也会调。需要开 `persistence` feature。
- `clear_color`：清屏色。
- `raw_input_hook`：在 egui 处理输入前修改原始输入，可以做按键拦截或虚拟键盘。

> 注意：新版 eframe 把传统 `update(ctx, frame)` 改成了 `logic` + `ui` 两步。这是为了分离“状态更新”和“UI 绘制”。

## 12.3 启动

### 桌面

```rust
fn main() -> eframe::Result {
    let options = eframe::NativeOptions {
        viewport: egui::ViewportBuilder::default()
            .with_inner_size([800.0, 600.0])
            .with_title("My App"),
        ..Default::default()
    };
    eframe::run_native(
        "My App",
        options,
        Box::new(|cc| {
            // cc.egui_ctx 可以在这里设置字体、样式
            Ok(Box::new(MyApp::new(cc)))
        }),
    )
}
```

`run_native` 返回 `eframe::Result`，是个别名。返回它给 `main` 即可。

### 网页

```rust
use eframe::WebRunner;

#[wasm_bindgen]
pub async fn start(canvas_id: &str) -> Result<(), eframe::wasm_bindgen::JsValue> {
    let web_options = eframe::WebOptions::default();
    WebRunner::new()
        .start(canvas_id, web_options, Box::new(|cc| Ok(Box::new(MyApp::new(cc)))))
        .await
}
```

HTML 里有一个 `<canvas id="the_canvas_id">`，JS 调 `wasm_bindgen` 暴露的 `start("the_canvas_id")`。完整模板看 <https://github.com/emilk/eframe_template>。

## 12.4 NativeOptions 重要字段

```rust
pub struct NativeOptions {
    pub viewport: egui::ViewportBuilder,  // 窗口设置
    pub multisampling: u16,               // MSAA，默认 0
    pub depth_buffer: u8,                 // 深度缓冲，默认 0
    pub stencil_buffer: u8,               // 模板缓冲，默认 0
    pub renderer: Renderer,               // Glow 或 Wgpu
    pub run_and_return: bool,             // 关窗后是否继续运行
    pub persistence_path: Option<PathBuf>, // 持久化路径
    // ...
}
```

`ViewportBuilder` 上的常用方法：

```rust
egui::ViewportBuilder::default()
    .with_inner_size([800.0, 600.0])
    .with_min_inner_size([400.0, 300.0])
    .with_title("My App")
    .with_resizable(true)
    .with_maximized(false)
    .with_fullscreen(false)
    .with_decorations(true)       // 是否有 OS 标题栏
    .with_transparent(false)
    .with_app_id("com.example.myapp")  // 持久化路径用
    .with_icon(egui::IconData { /* ... */ })
```

## 12.5 渲染器：glow vs wgpu

`eframe` 支持两种渲染后端：

- **glow**（OpenGL）：默认。兼容性最好，所有平台都能跑。
- **wgpu**（WebGPU/Vulkan/Metal/DX12）：性能更好，支持更现代的 GPU 特性。

在 `Cargo.toml` 选：

```toml
[dependencies]
eframe = { version = "0.32", default-features = false, features = ["glow"] }   # 用 OpenGL
# 或
eframe = { version = "0.32", default-features = false, features = ["wgpu"] }   # 用 wgpu
```

`NativeOptions::renderer` 显式指定（如果两个都开了）。

## 12.6 Frame

`eframe::Frame` 是当前帧的“环境”，可以拿到：

- `frame.ctx()`：`&egui::Context`。
- `frame.info()`：窗口信息、显示器信息等。
- `frame.output_mut()`：改输出（光标、剪贴板等）。
- `frame.winit_window()`（桌面）：拿到 `winit::window::Window`。

## 12.7 多视口（多个原生窗口）

egui 2024 起支持“视口（viewport）”，也就是真正的操作系统多窗口。

```rust
ctx.show_viewport_deferred(
    egui::ViewportId::from_hash_of("second"),
    egui::ViewportBuilder::default().with_title("Second"),
    |ctx, class| {
        // 这是个独立的 egui Context
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.label("我在另一个 OS 窗口里");
        });
    },
);
```

需要后端支持（`eframe` 桌面版默认支持，web 不支持）。`Area`、`Window` 还是 egui 内部的浮动层，**不**是新 OS 窗口。

## 12.8 持久化（存档）

开 `persistence` feature 后，egui 会自动存：

- 窗口位置/大小。
- `egui::Memory`（滚动距离、折叠状态等）。
- 你的 `App`（如果你实现了 `serde::Serialize + Deserialize`）。

通过 `App::save` 自定义存档内容：

```rust
fn save(&mut self, storage: &mut dyn Storage) {
    eframe::set_value(storage, eframe::APP_KEY, self);
}
```

桌面存到 `persistence_path` 下的文件，网页存到 Local Storage。

## 12.9 自己写集成

如果你想把 egui 嵌进游戏引擎（bevy、miniquad、自定义引擎），不用 `eframe`。基本循环：

```rust
let mut ctx = egui::Context::default();

loop {
    let raw_input: egui::RawInput = gather_input();  // 1. 收集输入

    let full_output = ctx.run_ui(raw_input, |ui| {   // 2. 跑你的 UI 代码
        egui::CentralPanel::default().show(ui, |ui| {
            ui.label("Hello");
            if ui.button("Click").clicked() {}
        });
    });

    handle_platform_output(full_output.platform_output);  // 3. 处理输出（光标等）
    let clipped_primitives = ctx.tessellate(               // 4. tessellate 成三角形
        full_output.shapes,
        full_output.pixels_per_point,
    );
    paint(full_output.textures_delta, clipped_primitives); // 5. 画
}
```

四件事：

1. **输入**：把鼠标、键盘、屏幕大小塞进 `RawInput`。
2. **UI**：调用你的 UI 代码。
3. **输出**：处理 `PlatformOutput`（光标变化、剪贴板、打开 URL 等）。
4. **绘制**：把 `shapes` 转成三角形，自己用 OpenGL/wgpu 画。

参考实现：

- `crates/egui_glow/src/painter.rs`：OpenGL 渲染参考。
- `crates/egui-wgpu`：wgpu 渲染。
- `crates/egui-winit`：winit 事件转换。

## 12.10 渲染器排错

### 文字模糊

- 确认 `pixels_per_point` 设置正确（高 DPI 屏要 > 1）。
- 检查纹理采样是不是差了半个像素。换成最近邻采样试试。
- egui 默认会处理 `pixels_per_point`，桌面端 `egui-winit` 会自动读系统 DPI。

### 窗口太透明/太黑

egui 用**预乘 alpha**。混合函数必须是 `(ONE, ONE_MINUS_SRC_ALPHA)`。纹理采样器要 clamp（`GL_CLAMP_TO_EDGE`）。

egui 偏好 gamma 颜色空间混合：

- **不要**用 sRGB 感知的纹理（不要 `GL_SRGB8_ALPHA8`）。
- **不要**开 `GL_FRAMEBUFFER_SRGB`。
- 在 gamma 空间做纹理和顶点颜色乘法。

### 边缘锯齿

- 关闭背面剔除。
- 检查 `TessellationOptions` 里的 `feathering`（egui 的抗锯齿方式）。

## 12.11 第三方集成

社区已有的引擎集成（看 <https://github.com/emilk/egui/wiki/3rd-party-integrations>）：

- **bevy_egui**：bevy 引擎。
- **egui-miniquad**：miniquad。
- **egui_sdl2_gl**、**egui_glium** 等。

如果你用的引擎还没有集成，自己写一个也不难——上面四步就够。

## 12.12 小结

| 你要做什么 | 用什么 |
|---|---|
| 写桌面/网页 app | `eframe` |
| 嵌入 bevy/miniquad | 第三方集成（`bevy_egui` 等） |
| 嵌入自己的引擎 | 直接用 `egui` + 自己写 4 步循环 |
| 多个 OS 窗口 | `ctx.show_viewport_deferred` |
| 持久化 | 开 `persistence` feature + 实现 `App::save` |
| 选渲染后端 | `glow`（兼容好）或 `wgpu`（性能好） |
