# 第十三章 实战技巧与常见问题

本章是一些从源码和 FAQ 里整理出来的实战经验。

## 13.1 输入读取

```rust
// 键盘
ctx.input(|i| i.key_down(egui::Key::A));
ctx.input(|i| i.key_pressed(egui::Key::Enter));

// 鼠标
ctx.input(|i| i.pointer.primary_clicked());   // 这一帧点了左键
ctx.input(|i| i.pointer.is_decidedly_dragging()); // 正在拖
let pos: Option<egui::Pos2> = ctx.input(|i| i.pointer.hover_pos());

// 滚轮
let scroll = ctx.input(|i| i.smooth_scroll_delta);

// 修饰键
let ctrl  = ctx.input(|i| i.modifiers.ctrl);
let shift = ctx.input(|i| i.modifiers.shift);
let alt   = ctx.input(|i| i.modifiers.alt);

// 剪贴板
let s: Option<String> = ctx.input(|i| i.raw.events.iter().find_map(|e| {
    if let egui::Event::Paste(s) = e { Some(s.clone()) } else { None }
}));
```

> 都用 `ctx.input(|i| ...)` 闭包形式，别手动拿锁。

## 13.2 自定义快捷键

在 `App::raw_input_hook` 里拦截/注入事件：

```rust
fn raw_input_hook(&mut self, _ctx: &egui::Context, raw: &mut egui::RawInput) {
    // 例如：吞掉某些按键
    raw.events.retain(|e| !matches!(e, egui::Event::Key { key: egui::Key::F1, .. }));
}
```

或在 `logic` 里检测：

```rust
fn logic(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
    if ctx.input(|i| i.key_pressed(egui::Key::S) && i.modifiers.ctrl) {
        self.save();
    }
}
```

## 13.3 异步任务

GUI 线程**绝不能阻塞**——一 `.await` 整个 UI 就卡住。常见做法：

- **通道**（`std::sync::mpsc`）：后台线程发，GUI 线程 `try_recv`。
- **`Arc<Mutex<T>>`**：后台线程写，GUI 线程读。
- **`poll_promise::Promise`** / **`eventuals::Eventual`** / **`tokio::sync::watch`**。

例子：用 `mpsc` 在后台跑任务：

```rust
use std::sync::mpsc;

struct MyApp {
    rx: mpsc::Receiver<String>,
    logs: Vec<String>,
}

impl eframe::App for MyApp {
    fn logic(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        // 非阻塞地取消息
        while let Ok(msg) = self.rx.try_recv() {
            self.logs.push(msg);
        }
        // 想持续刷新就请求重绘
        ctx.request_repaint();
    }

    fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ui, |ui| {
            for log in &self.logs {
                ui.label(log);
            }
        });
    }
}
```

## 13.4 文件对话框

用 [rfd](https://docs.rs/rfd)（同时支持桌面和网页）：

```rust
let file = rfd::AsyncFileDialog::new()
    .add_filter("图片", &["png", "jpg"])
    .pick_file()
    .await;
```

记得用 async 版本，并在后台线程跑，别阻塞 GUI。

## 13.5 持久化窗口大小/位置

开 `persistence` feature，egui 会自动存。`ViewportBuilder::with_app_id(...)` 决定存到哪个目录。

## 13.6 让控件可拖出文件

egui 本身没有“拖文件进窗口”的内置支持，靠集成：

- 桌面：`egui-winit` 把 winit 的 `DroppedFile` 事件转成 `egui::RawInput`。
- 读取：

```rust
ctx.input(|i| {
    for file in &i.raw.dropped_files {
        println!("{:?}", file.path);
    }
});
```

## 13.7 自动重绘与节能

egui 默认只在有交互或动画时重绘。空闲时不占 CPU。

如果你想定时刷新（比如显示时间），每帧请求重绘：

```rust
ctx.request_repaint_after(Duration::from_secs(1));
```

或从其它线程触发：

```rust
let ctx2 = ctx.clone();
std::thread::spawn(move || {
    std::thread::sleep(Duration::from_millis(500));
    ctx2.request_repaint();
});
```

`Context` 可以被 clone 然后传到任何线程。

## 13.8 长列表虚拟化

`ScrollArea` 已经只画可见部分，但你**还是要避免**在闭包里把 10000 个控件都加进去——光调用 `ui.add` 也要时间。

策略：

- 只加可见范围内的：根据 `ScrollArea` 的 offset 算出可见区间。
- 用 `ui.allocate_ui_with_layout` 跳过不可见的部分。

简单情况（每行高度固定）：

```rust
egui::ScrollArea::vertical().show(ui, |ui| {
    let row_height = 20.0;
    let visible_start = (ui.min_rect().top() / row_height).floor() as usize;
    let visible_end = ((ui.max_rect().bottom() / row_height).ceil() as usize) + 1;

    // 跳过不可见的顶部
    ui.allocate_space(egui::vec2(0.0, visible_start as f32 * row_height));

    for i in visible_start..visible_end.min(data.len()) {
        ui.label(format!("item {}", data[i]));
    }

    // 跳过不可见的底部
    let remaining = data.len().saturating_sub(visible_end);
    ui.allocate_space(egui::vec2(0.0, remaining as f32 * row_height));
});
```

## 13.9 调试

打开 egui 的内置检查器：

```rust
ctx.set_debug_on_hover(true);  // 悬停时显示控件边界、ID 等
```

或者：

```rust
ui.ctx().debug_painting();  // 高亮重绘区域
```

按 `Ctrl+Shift+D`（eframe 默认）可以打开 egui 自带的“inspection”窗口，看内存、ID、布局。

输出当前帧的信息：

```rust
ctx.output_mut(|o| {
    o.status_messages.push("Hello".into());  // 状态栏
});
```

## 13.10 在 egui 里画 3D

两种方式：

### Shape::Callback

egui 在绘制阶段调你的代码，用当时的渲染上下文（eframe+glow 给你 `&glow::Context`）画任意 3D：

```rust
ui.painter().add(egui::Shape::callback(egui::Rect::from_min_size(pos, size), move |rect, painter| {
    // 在这里用 painter.ctx() 拿 glow::Context，画 3D
}));
```

参考示例：`examples/custom_3d_glow/`。

### 渲染到纹理

把 3D 场景渲到纹理，转成 `egui::TextureId`，用 `ui.image(...)` 显示。具体怎么转看后端（`egui-wgpu` 提供 `CallbackResources`，`egui_glow` 提供 `register_native_texture`）。

## 13.11 中文字体推荐

免费可商用中文字体：

- **思源黑体**（Source Han Sans）：`SourceHanSansCN-Regular.otf`，最常用。
- **思源宋体**（Source Han Serif）。
- **更纱黑体**（Sarasa Gothic）：等宽中文，适合代码编辑器。
- **Noto Sans CJK**：Google 版的思源。

加载方法见 [第十章](./chapter_10.md#102-加中文字体)。

## 13.12 常见错误

### “`egui::Context` was locked” / 死锁

你手动拿了 `Context` 的锁后又在闭包里调 egui API。改成 `ctx.input(|i| ...)` 这种闭包形式。

### 控件 ID 冲突

debug 模式 egui 会 panic 报告冲突。用 `ui.push_id("...", |ui| ...)` 给重复结构加唯一前缀。

### 窗口一直闪烁

可能是 `request_repaint` 被每帧调用，导致永远在重绘。检查 `logic` 里有没有无条件 `request_repaint`。

### 滑块/数值显示一闪一闪

`Slider` 等控件的 `text` 内容每帧都重新构造字符串，如果你在闭包里改了状态可能抖动。把状态改到 `App` 字段里，闭包只读不改。

### 重新设置字体不生效

`ctx.set_fonts` 替换整个字体表，会**重置所有 `FontImage` 缓存**。如果你只是想加字体，用 `ctx.add_font(FontInsert::new(...))`。

### `Button` 没反应

可能你套了 `ui.set_enabled(false)` 或 `ui.add_enabled(false, ...)`。检查调用栈上有没有禁用。

## 13.13 性能清单

- 大列表用 `ScrollArea` + 虚拟化（只画可见的）。
- 别在 `ui` 闭包里做重计算（解析、IO），提前算好放 `App` 字段。
- 复杂图片用 `TextureHandle` 缓存，别每帧重新上传。
- 用 `ui.is_rect_visible(rect)` 跳过不可见绘制。
- `cargo run --release` 测真实性能，debug 构建慢得多。

## 13.14 哪里继续学

- **官方文档**：<https://docs.rs/egui>
- **在线 demo**：<https://www.egui.rs/#demo>
- **源码示例**：`examples/` 目录每个都值得跑一遍。
- **demo lib 源码**：`crates/egui_demo_lib/src/demo/` 是个“怎么用 egui 写各种控件”的活教材。
- **GitHub Discussions**：<https://github.com/emilk/egui/discussions>，问问题去这。
- **Discord**：egui 官方 Discord 服务器。

## 13.15 写在最后

egui 的设计哲学是“简单直接”。如果你写代码时觉得“这也太绕了”，大概率是你想多了——回到“每帧画一遍、状态自己存”这两个原则，问题就清楚了。

祝玩得开心。
