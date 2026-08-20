# 第十一章 自定义控件

egui 写自定义控件非常简单——只要实现 `Widget` trait 就行。本章用官方的 toggle switch 示例（`crates/egui_demo_lib/src/demo/toggle_switch.rs`）讲解套路。

## 11.1 Widget trait

```rust
pub trait Widget {
    /// 分配空间、处理交互、绘制、返回 Response
    fn ui(self, ui: &mut Ui) -> Response;
}
```

注意 `self` 是按值消费的——控件是 builder，不是持久的对象。

任何 `FnOnce(&mut Ui) -> Response` 也自动实现 `Widget`，所以你也可以用闭包当控件：

```rust
ui.add(move |ui: &mut egui::Ui| -> egui::Response {
    // ...
});
```

## 11.2 写控件的四步

官方推荐的四步：

1. **决定大小**：算出控件想要多大。
2. **分配空间**：`ui.allocate_exact_size(size, sense)` 拿到 `Rect` 和 `Response`。
3. **处理交互**：根据 `Response` 改状态。
4. **绘制**：用 `ui.painter()` 画。

## 11.3 完整示例：iOS 风格开关

下面是 egui 官方示例的简化版：

```rust
use egui::*;

pub fn toggle_ui(ui: &mut Ui, on: &mut bool) -> Response {
    // 1. 决定大小：标准按钮高 × 2:1
    let desired_size = ui.spacing().interact_size.y * vec2(2.0, 1.0);

    // 2. 分配空间，声明感知点击
    let (rect, mut response) = ui.allocate_exact_size(desired_size, Sense::click());

    // 3. 处理交互
    if response.clicked() {
        *on = !*on;
        response.mark_changed();  // 告诉调用者我变了
    }

    // 无障碍信息
    response.widget_info(|| {
        WidgetInfo::selected(WidgetType::Checkbox, ui.is_enabled(), *on, "")
    });

    // 4. 绘制（只在可见时画）
    if ui.is_rect_visible(rect) {
        // 动画：开关滑动的过渡
        let how_on = ui.ctx().animate_bool_responsive(response.id, *on);
        // 拿当前状态下的颜色
        let visuals = ui.style().interact_selectable(&response, *on);
        let rect = rect.expand(visuals.expansion);
        let radius = 0.5 * rect.height();

        // 画背景胶囊
        ui.painter().rect(
            rect,
            radius,
            visuals.bg_fill,
            visuals.bg_stroke,
            StrokeKind::Inside,
        );

        // 画圆点（带动画）
        let circle_x = lerp((rect.left() + radius)..=(rect.right() - radius), how_on);
        let center = pos2(circle_x, rect.center().y);
        ui.painter()
            .circle(center, 0.75 * radius, visuals.bg_fill, visuals.fg_stroke);
    }

    response
}

/// 包装成 Widget，使用更顺手：`ui.add(toggle(&mut my_bool))`
pub fn toggle(on: &mut bool) -> impl egui::Widget + '_ {
    move |ui: &mut egui::Ui| toggle_ui(ui, on)
}
```

使用：

```rust
let mut enabled = true;
ui.add(toggle(&mut enabled));
```

## 11.4 关键 API 解析

### `ui.allocate_exact_size(size, sense) -> (Rect, Response)`

在当前 `Ui` 里申请一块固定大小的空间，返回这块的矩形（屏幕绝对坐标）和交互响应。

`Sense::click()` 表示这块区域感知点击。其它选项见 [第四章](chapter_4.md#45-sense控件感知什么交互)。

### `ui.allocate_ui(size, |ui| ...) -> InnerResponse<R>`

更高级：分配空间的同时在里面给你一个**子 `Ui`**，可以在里面继续放控件。常用于“复合控件”。

```rust
let response = ui.allocate_ui(egui::vec2(200.0, 40.0), |ui| {
    ui.horizontal(|ui| {
        ui.label("x:");
        ui.add(egui::DragValue::new(&mut x));
    });
}).response;
```

### `ui.painter()`

返回 `&Painter`，可以画各种东西：

```rust
ui.painter().rect(rect, rounding, fill, stroke, StrokeKind::Inside);
ui.painter().circle(center, radius, fill, stroke);
ui.painter().line_segment([a, b], stroke);
ui.painter().text(pos, anchor, "hello", FontId::proportional(14.0), color);
ui.painter().rect_filled(rect, rounding, fill);
ui.painter().rect_stroke(rect, rounding, stroke, StrokeKind::Middle);
```

坐标系是**屏幕绝对坐标**，不是相对当前 `Ui`。`rect` 给的就是绝对坐标，直接用即可。

### `ui.is_rect_visible(rect)`

剪裁优化。如果这块矩形不在可见区（比如被 `ScrollArea` 滚出去了），就不用画了。`ScrollArea` 里这个检查很重要——能省很多绘制。

### `response.mark_changed()`

告诉调用者“这个控件的值变了”。调用者可以用 `response.changed()` 检测。

### `ui.ctx().animate_bool_responsive(id, bool) -> f32`

egui 的动画 API。给定一个 bool 和 id，返回 0~1 之间的平滑过渡值。bool 变化时返回值会插值过去。非常适合做开关、悬停高亮等过渡。

### `response.widget_info(|| WidgetInfo::...)`

给屏幕阅读器提供信息。`WidgetInfo::selected(...)`、`WidgetInfo::button(...)`、`WidgetInfo::label(...)` 等。可访问性是可选的，但加上是好习惯。

## 11.5 复合控件（多个简单控件组合）

不需要画图，把现有控件组合起来就行：

```rust
pub fn slider_vec2(value: &mut egui::Vec2) -> impl egui::Widget + '_ {
    move |ui: &mut egui::Ui| {
        ui.horizontal(|ui| {
            ui.add(egui::Slider::new(&mut value.x, 0.0..=1.0).text("x"));
            ui.add(egui::Slider::new(&mut value.y, 0.0..=1.0).text("y"));
        })
        .response
    }
}

// 用
let mut v = egui::Vec2::new(0.5, 0.5);
ui.add(slider_vec2(&mut v));
```

注意返回的是 `.response`（外层 horizontal 容器的 Response），不是 `inner`。

## 11.6 带状态的控件

如果控件需要跨帧记住状态（比如光标位置、滚动），用 `egui::Memory`：

```rust
#[derive(Clone, Default)]
struct MyState {
    cursor: usize,
}

// 读
let state: Option<MyState> = ui.ctx().memory(|m| m.data.get_temp(response.id));
// 写
ui.ctx().memory_mut(|m| m.data.insert_temp(response.id, state));
```

需要持久化（重启还在）的话用 `data.get_persisted` / `insert_persisted`。

## 11.7 检测重复 ID

如果你写控件时不小心撞了 ID，egui 在 debug 模式会 panic 提示。用 `ui.make_persistent_id(...)` 或 `ui.next_auto_id()` 来手动管理 ID。

## 11.8 发布你的控件

如果你写了一个有用的控件，可以独立成 crate 发布到 crates.io。社区维护的第三方控件清单见 <https://github.com/emilk/egui/wiki/3rd-party-egui-crates>。

## 11.9 小结

写自定义控件的口诀：

> **算大小 → 分配空间 → 处理交互 → 画 → 返回 Response**

记住这五步，看任何 egui 控件源码都能看懂。源码里 `crates/egui/src/widgets/` 下的 `button.rs`、`slider.rs`、`checkbox.rs` 都是这么写的，建议挑一个完整读一遍。
