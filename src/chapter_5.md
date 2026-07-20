# 第五章 内置控件

egui 在 `crates/egui/src/widgets/` 下提供了一组内置控件。本章逐个过一遍。源码里的 `mod.rs` 是入口。

## 5.1 控件的两种写法

每个控件都有两种用法：

**简写**（直接调 `ui` 上的方法）：

```rust
if ui.button("Click me").clicked() { ... }
ui.label("Hello");
```

**完整写法**（构造 builder，再用 `ui.add`）：

```rust
if ui.add(egui::Button::new("Click me")).clicked() { ... }
ui.add(egui::Label::new("Hello").text_color(egui::Color32::RED));
```

简写等价于 `ui.add(Button::new(...))`。需要设置额外参数时用 builder 写法。

> 任何实现 `trait Widget` 的东西都能 `ui.add(...)`。`Widget` trait 长这样：
> ```rust
> pub trait Widget {
>     fn ui(self, ui: &mut Ui) -> Response;
> }
> ```

## 5.2 Label（标签）

显示一段文字。

```rust
ui.label("Hello world");
ui.label(egui::RichText::new("Red and big").color(egui::Color32::RED).size(20.0));
```

`RichText` 是带样式的文字。常用方法：

- `.text_color(c)` / `.color(c)`
- `.size(pt)`
- `.strong()` 加粗
- `.italics()` 斜体
- `.monospace()`
- `.code()` 等宽 + 背景
- `.underline()` / `.strikethrough()`
- `.heading()` / `.name("...")` 用样式表里预定义的字号

Label 的高级 builder：

```rust
ui.add(
    egui::Label::new("Some long text that might wrap")
        .wrap()
        .truncate()            // 截断而不是换行
        .show_tooltip_when_elided(true)  // 截断时悬浮显示完整文字
);
```

## 5.3 Button（按钮）

```rust
if ui.button("Save").clicked() {
    save();
}
```

`Button` builder：

```rust
ui.add(
    egui::Button::new("Save")
        .min_size(egui::vec2(80.0, 24.0))
        .fill(egui::Color32::from_rgb(0, 120, 200))
        .stroke(egui::Stroke::NONE)
        .rounding(egui::Rounding::same(4.0))
        .text_color(egui::Color32::WHITE)
);
```

带图标的按钮：

```rust
ui.add(
    egui::Button::image_and_text(
        egui::include_image!("assets/icon.png"),
        "Save")
);
```

按钮内含富文本：

```rust
ui.add(egui::Button::new(egui::RichText::new("Save").strong().color(egui::Color32::WHITE)));
```

## 5.4 Checkbox / RadioButton

```rust
let mut checked = true;
ui.checkbox(&mut checked, "Enable feature");

#[derive(PartialEq)]
enum Mode { A, B, C }
let mut mode = Mode::A;
ui.horizontal(|ui| {
    ui.radio_value(&mut mode, Mode::A, "A");
    ui.radio_value(&mut mode, Mode::B, "B");
    ui.radio_value(&mut mode, Mode::C, "C");
});
```

`radio_value` 也适用于整数、字符串等任何 `PartialEq` 类型。

## 5.5 Slider（滑块）

```rust
let mut v: f32 = 0.0;
ui.add(egui::Slider::new(&mut v, 0.0..=100.0).text("My value"));
```

builder 常用方法：

```rust
ui.add(
    egui::Slider::new(&mut v, 0.0..=100.0)
        .text("volume")
        .clamping(egui::SliderClamping::Always)  // 拖动时也限制在范围内
        .step_by(1.0)         // 步长
        .fixed_decimals(2)    // 显示几位小数
        .prefix(">")          // 数字前缀
        .suffix("%")          // 数字后缀
        .orientation(egui::SliderOrientation::Vertical) // 垂直滑块
);
```

整数也行：

```rust
ui.add(egui::Slider::new(&mut age, 0..=120).text("age"));
```

## 5.6 DragValue（可拖动的数值）

显示一个数字，鼠标按住可以拖动改值。比 Slider 省地方。

```rust
ui.add(egui::DragValue::new(&mut v).speed(0.1).range(0.0..=100.0).prefix("v="));
```

按住 `Ctrl` 拖动可以更慢（更精细）。双击可以输入数字。

## 5.7 TextEdit（文本框）

详见 [第八章](./chapter_8.md)。这里只放最常用形式：

```rust
let mut s = String::new();
ui.text_edit_singleline(&mut s);   // 单行
ui.text_edit_multiline(&mut s);    // 多行

// 带提示文字 / 密码框
ui.add(
    egui::TextEdit::singleline(&mut password)
        .password(true)
        .hint_text("输入密码")
        .desired_width(200.0)
);
```

## 5.8 Hyperlink（超链接）

```rust
ui.hyperlink("https://github.com/emilk/egui");
ui.hyperlink_to("egui 主页", "https://github.com/emilk/egui");
ui.add(egui::Hyperlink::from_label_and_url("点击这里", "https://example.com").open_in_new_tab(true));
```

点击会在浏览器打开。

## 5.9 Image（图片）

详见 [第九章](./chapter_9.md)。最短用法：

```rust
ui.image(egui::include_image!("assets/ferris.png"));
ui.image("https://example.com/foo.png"); // 需要先装 egui_extras 图片加载器
```

## 5.10 Separator（分隔线）

```rust
ui.label("上");
ui.separator();
ui.label("下");
```

## 5.11 ProgressBar（进度条）

```rust
let progress: f32 = 0.42;
ui.add(egui::ProgressBar::new(progress).text("42%"));
ui.add(egui::ProgressBar::new(progress).animate(true)); // 动画
```

## 5.12 Spinner（加载动画）

```rust
if loading {
    ui.spinner();
}
```

## 5.13 ColorPicker（颜色选择器）

```rust
let mut color: egui::Color32 = egui::Color32::RED;
ui.color_edit_button_srgba(&mut color);
// 或者
ui.add(egui::widgets::color_picker::color_picker_button_srgba(ui, &mut color, egui::Alpha::Blend));
```

## 5.14 控件大小

用 `ui.add_sized` 指定大小：

```rust
ui.add_sized([80.0, 24.0], egui::Button::new("Fixed"));
```

也可以用 `ui.spacing_mut()` 调整后续所有控件的默认间距/高度：

```rust
ui.spacing_mut().button_padding = egui::vec2(16.0, 4.0);
ui.spacing_mut().interact_size = egui::vec2(80.0, 24.0);
```

## 5.15 提示文字（tooltip）

任何 `Response` 都能加 tooltip：

```rust
ui.button("Save").on_hover_text("保存到磁盘");
ui.button("X").on_hover_ui(|ui| {
    ui.label("详细说明");
    ui.label("可以放任意 UI");
});
```

## 5.16 控件去哪查

- 文档：<https://docs.rs/egui>
- 源码：`crates/egui/src/widgets/mod.rs` 是入口，每个控件一个文件（`button.rs`、`slider.rs` 等）。
- 演示：跑 `cargo run -p egui_demo_app`，里面“Widget Gallery”一节基本所有控件都有。
