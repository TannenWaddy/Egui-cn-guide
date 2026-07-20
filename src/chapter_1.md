# 第一章 认识 egui

## 1.1 egui 是什么

egui（读作 “e-gooey”）是一个用纯 Rust 写的 GUI 库。它有三个特点：

1. **简单**：写法直白，没有回调、没有信号槽。
2. **快**：debug 构建下也能跑到 60 FPS。
3. **可移植**：同一份代码能编译成桌面程序，也能编译成网页（WebAssembly）。

它不依赖任何操作系统提供的控件，所有按钮、文字、窗口都是 egui 自己画出来的。所以它长得不像 Windows 或 macOS 的原生界面，但好处是到哪里都长一个样。

egui 自己只是个“画界面”的库，它不知道自己在什么系统上跑、不知道怎么读鼠标键盘、不知道怎么把像素画到屏幕上。这些事交给“集成（integration）”来做。官方提供的集成叫 **eframe**，它把 egui 包装成一个可以直接跑的程序框架。

## 1.2 egui 能做什么

按官方 README 列举的功能：

- **控件**：标签（label）、按钮、超链接、复选框、单选按钮、滑块、可拖动的数值、文本编辑、颜色选择器、加载动画（spinner）。
- **图片**：支持加载并显示。
- **布局**：水平、垂直、分列、自动换行。
- **文本编辑**：多行、复制粘贴、撤销、支持 emoji。
- **窗口**：可拖动、可缩放、可命名、可最小化、可关闭，自动定位与大小。
- **区域**：可缩放、垂直滚动、折叠标题、面板。
- **渲染**：线、圆、文字、凸多边形的抗锯齿渲染。
- **悬浮提示（tooltip）**：鼠标悬停时显示。
- **可访问性**：通过 AccessKit 支持屏幕阅读器。
- **可选文本**：标签里的文字可以被选中。

## 1.3 egui 适合谁用

- 你在用 Rust 写一个交互式程序，想加个简单界面。
- 你在写游戏，想在游戏里加个调试面板或设置界面。
- 你想写一个网页小程序，又不想学前端那一套。

egui **不**适合：

- 不用 Rust 的人。
- 想要原生外观的人。
- 想要一次写好、以后升级不坏的人（egui 还在快速迭代，新版本经常有破坏性改动）。

## 1.4 仓库结构

egui 是一个 monorepo，由多个 crate 组成。理解这些 crate 的分工很重要：

| crate | 作用 |
|---|---|
| `emath` | 最基础的 2D 数学：`Vec2`、`Pos2`、`Rect`、`lerp`、`remap`。 |
| `ecolor` | 颜色类型。 |
| `epaint` | 能变成带纹理三角形的 2D 形状和文字。 |
| `epaint_default_fonts` | 内嵌的默认字体。 |
| `egui` | 主库。所有控件、布局、`Context`、`Ui`、`Response` 都在这里。只依赖 `emath` 和 `epaint`。 |
| `egui_extras` | 在 `egui` 之上的附加功能，比如图片加载器。 |
| `egui-winit` | 把 [winit](https://crates.io/crates/winit) 的事件翻译给 egui。 |
| `egui_glow` | 用 [glow](https://github.com/grovesNL/glow)（OpenGL）把 egui 画出来。 |
| `egui-wgpu` | 用 [wgpu](https://crates.io/crates/wgpu)（WebGPU）画。 |
| `eframe` | 官方框架，把上面这些组装起来，让你写一份代码同时跑在桌面和网页。 |
| `egui_demo_lib` | 演示代码，可以当参考。 |
| `egui_demo_app` | 把 `egui_demo_lib` 打包成可执行程序或网页。 |
| `egui_kittest` | 基于 kittest 和 AccessKit 的测试框架。 |
| `egui_plot` | 折线图等绘图。 |

> 简单说：`emath`/`epaint` 是底层；`egui` 是核心；`eframe` 是让你能跑起来的框架。

## 1.5 一段最小示例

```rust
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
```

这段代码做的事：放一个标题，放一行（标签 + 文本框），放一个滑块，放一个按钮，按钮被点就 `age += 1`，最后显示问候语。

读这段代码你就明白 egui 的风格了：**你写 UI 的过程，就是 UI 真正在显示的过程**。没有“先创建按钮、再注册回调”这一套。下一章我们会从零跑起来。
