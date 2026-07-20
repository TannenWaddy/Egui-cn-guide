# 第十章 字体与样式

egui 的外观完全可以自定义。本章讲怎么改字体、改颜色、改间距。

## 10.1 默认字体

egui 自带四个字体（在 `epaint_default_fonts` crate 里）：

- `Ubuntu-Light.ttf`：默认 proportional 字体。
- `Hack-Regular.ttf`：默认 monospace 字体。
- `NotoEmoji-Regular.ttf`：emoji。
- `emoji-icon-font.ttf`：另一套 emoji 图标。

默认字体**不支持中文**。要显示中文必须自己加字体。

## 10.2 加中文字体

最简单的办法：用 `ctx.add_font` 在已有字体上追加一个。

```rust
use eframe::epaint::text::{FontInsert, InsertFontFamily, FontPriority};

fn add_chinese_font(ctx: &egui::Context) {
    ctx.add_font(FontInsert::new(
        "my_font",
        egui::FontData::from_static(include_bytes!("../assets/SourceHanSansCN-Regular.otf")),
        vec![
            InsertFontFamily {
                family: egui::FontFamily::Proportional,
                priority: egui::epaint::text::FontPriority::Highest, // 优先用它
            },
            InsertFontFamily {
                family: egui::FontFamily::Monospace,
                priority: egui::epaint::text::FontPriority::Lowest, // 等宽兜底
            },
        ],
    ));
}
```

在 `App` 创建时调一次：

```rust
impl MyApp {
    fn new(cc: &eframe::CreationContext<'_>) -> Self {
        add_chinese_font(&cc.egui_ctx);
        Self::default()
    }
}
```

## 10.3 完全替换字体表

如果你想把整个字体表换成自己的：

```rust
fn replace_fonts(ctx: &egui::Context) {
    let mut fonts = egui::FontDefinitions::default();

    fonts.font_data.insert(
        "my_font".to_owned(),
        std::sync::Arc::new(egui::FontData::from_static(include_bytes!(
            "../assets/MyFont-Regular.ttf"
        ))),
    );

    // proportional 优先用 my_font
    fonts
        .families
        .entry(egui::FontFamily::Proportional)
        .or_default()
        .insert(0, "my_font".to_owned());

    // monospace 也加 my_font 作为兜底
    fonts
        .families
        .entry(egui::FontFamily::Monospace)
        .or_default()
        .push("my_font".to_owned());

    ctx.set_fonts(fonts);
}
```

`FontDefinitions` 里：

- `font_data`：所有字体数据，按名字索引。
- `families`：每个 `FontFamily` 是一组字体名（按优先级排序）。egui 找不到字符时按顺序往下找。

把中文字体放在 `Proportional` 列表前几位，中文就能显示了。**保留默认字体**很重要——不然英文字符可能没字体可用，或者 emoji 显示不出来。

## 10.4 字体大小

全局字号：

```rust
let mut style = (*ctx.style()).clone();
style.text_styles.insert(egui::TextStyle::Body, egui::FontId::new(18.0, egui::FontFamily::Proportional));
ctx.set_style(style);
```

`TextStyle` 有几个预设：`Small`、`Body`、`Button`、`Heading`、`Monospace`、`Name(...)`。

局部改字号用 `RichText`：

```rust
ui.label(egui::RichText::new("大字").size(28.0));
ui.label(egui::RichText::new("小字").size(10.0));
```

## 10.5 Style（全局样式）

`ctx.style()` 返回 `Arc<Style>`。要改得 clone 一份再 `set_style`：

```rust
use egui::Style;

let mut style: Style = (*ctx.style()).clone();
style.spacing.item_spacing = egui::vec2(10.0, 6.0);
style.spacing.button_padding = egui::vec2(12.0, 4.0);
style.spacing.indent = 24.0;
style.spacing.scroll_bar_width = 12.0;
ctx.set_style(style);
```

`Style` 包含：

- `spacing: Spacing`：间距、控件高度、缩进等。
- `visuals: Visuals`：颜色（暗色/亮色主题、控件颜色）。
- `text_styles: HashMap<TextStyle, FontId>`：每种文本样式的字号字体。
- `animation_time: f32`：动画时长。
- `interaction: Interaction`：可选中、双击间隔等。

只改一个 `Ui` 内的样式：

```rust
ui.style_mut().spacing.item_spacing = egui::vec2(4.0, 2.0);
```

## 10.6 Visuals（视觉/主题）

切换暗色/亮色：

```rust
ctx.set_visuals(egui::Visuals::dark());
ctx.set_visuals(egui::Visuals::light());
```

`Visuals` 里有：

- `dark_mode: bool`
- `widgets: Widgets`：各种状态（普通/悬停/激活/禁用）下控件的颜色和描边。
- `selection: Selection`：选中时的颜色。
- `window_fill` / `panel_fill` / `extreme_bg_color`：背景色。
- `faint_bg_color`：表格隔行色。

改某个颜色：

```rust
let mut v = ctx.style().visuals.clone();
v.widgets.inactive.bg_fill = egui::Color32::from_rgb(40, 40, 40);
v.selection.bg_fill = egui::Color32::from_rgb(0, 120, 215);
ctx.set_visuals(v);
```

## 10.7 控件视觉的 interact

`Style::interact(&response)` 返回 `&WidgetVisuals`，告诉你这种状态下控件该用什么颜色。写自定义控件时常用：

```rust
let visuals = ui.style().interact(&response);
ui.painter().rect(rect, 4.0, visuals.bg_fill, visuals.bg_stroke, egui::StrokeKind::Inside);
```

`interact_selectable(response, selected)` 给出选中态的视觉。

## 10.8 圆角（Rounding）

```rust
ui.style_mut().visuals.window_shadow = epaint::Shadow::NONE; // 去掉窗口阴影
ui.style_mut().visuals.widgets.noninteractive.rounding = egui::Rounding::same(0.0); // 直角
```

`Rounding::same(8.0)` 是四个角都 8；也可以分别指定 nw/ne/sw/se。

## 10.9 自定义颜色

`Color32` 是 sRGB 8 位颜色（`[r, g, b, a]`）。常用构造：

```rust
egui::Color32::from_rgb(r, g, b)                    // a=255
egui::Color32::from_rgba_unmultiplied(r, g, b, a)
egui::Color32::from_rgba_premultiplied(r, g, b, a)
egui::Color32::from_gray(v)                         // r=g=b=v, a=255
egui::Color32::from_white_alpha(a)                  // 白色透明
egui::Color32::from_black_alpha(a)
```

egui **内部用预乘 alpha**。`from_rgba_unmultiplied` 会自动转换。除非写底层渲染代码，一般不用关心这个细节。

## 10.10 文本样式（RichText）

```rust
ui.label(
    egui::RichText::new("Important")
        .color(egui::Color32::RED)
        .size(20.0)
        .strong()       // 加粗
        .italics()
        .underline()
        .background_color(egui::Color32::from_rgb(60, 0, 0))
);
```

## 10.11 主题切换示例

```rust
impl eframe::App for MyApp {
    fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ui, |ui| {
            ui.horizontal(|ui| {
                if ui.button("Dark").clicked() {
                    ui.ctx().set_visuals(egui::Visuals::dark());
                }
                if ui.button("Light").clicked() {
                    ui.ctx().set_visuals(egui::Visuals::light());
                }
            });
        });
    }
}
```

`ctx.set_visuals` 立刻生效，下一帧就用新主题。

## 10.12 持久化样式

如果你想让用户自定义的样式下次打开还在，把 `Style` 存到 `App` 里，并通过 `eframe::App::save` 持久化（需要开 `persistence` feature）。`egui` 内部也会自动把样式写入 `Memory`，但仅同一进程内。

## 10.13 小结

| 想改什么 | 怎么改 |
|---|---|
| 中文显示 | `ctx.add_font(FontInsert::new(...))` |
| 完全换字体 | `ctx.set_fonts(FontDefinitions { ... })` |
| 全局字号 | `style.text_styles.insert(TextStyle::Body, FontId::new(18.0, ...))` |
| 局部字号 | `RichText::new(s).size(20.0)` |
| 间距 | `style.spacing.item_spacing = ...` |
| 暗/亮主题 | `ctx.set_visuals(Visuals::dark() / light())` |
| 选中色 | `style.visuals.selection.bg_fill = ...` |
| 自定义控件颜色 | `style.interact(&response)` 拿 `WidgetVisuals` |
