# 第六章 布局

egui 的布局系统简单但灵活。核心是 `Layout` 和 `Ui` 上的一组 `horizontal`/`vertical` 方法。

## 6.1 默认布局

每个 `Ui` 有一个 `Layout`，决定新加的控件往哪个方向排。新建的 `Ui` 默认是 `top_down`（从上往下），左对齐。

```rust
// 等价于默认行为
ui.label("第一行");
ui.label("第二行");
ui.label("第三行");
```

## 6.2 横向布局

```rust
ui.horizontal(|ui| {
    ui.label("名字:");
    ui.text_edit_singleline(&mut name);
    ui.button("确定");
});
```

`horizontal` 闭包里是一个新的 `Ui`，它的 `Layout` 是 `left_to_right`。闭包结束后回到外层 `Ui`，光标换行到下一行。

注意：**横向布局里高度按最高的那个控件算**。

## 6.3 竖向布局

```rust
ui.vertical(|ui| {
    ui.label("上");
    ui.label("下");
});
```

虽然默认就是竖向，但显式 `vertical` 能确保子 UI 独立，不影响外层。

## 6.4 自动换行

```rust
ui.horizontal_wrapped(|ui| {
    for i in 0..50 {
        ui.button(format!("btn {i}"));
    }
});
```

`horizontal_wrapped` 会像文字一样换行——一行排满后自动到下一行。

可以先把间距设为 0 来去掉标签之间的空格（典型场景：radio 按钮紧贴文字）：

```rust
ui.horizontal_wrapped(|ui| {
    ui.spacing_mut().item_spacing.x = 0.0;
    ui.radio_value(&mut v, false, "Off");
    ui.radio_value(&mut v, true, "On");
});
```

## 6.5 指定布局

`Layout` 有这些构造函数（见 `crates/egui/src/layout.rs`）：

```rust
egui::Layout::top_down(align)              // 从上往下，水平对齐
egui::Layout::bottom_up(align)             // 从下往上
egui::Layout::left_to_right(align)         // 从左往右，垂直对齐
egui::Layout::right_to_left(align)         // 从右往左
egui::Layout::top_down_justified(align)    // 从上往下，宽度撑满
egui::Layout::left_to_right_justified(align)
```

`justified` 版本会让子控件尝试填满可用宽度。

```rust
ui.with_layout(egui::Layout::top_down_justified(egui::Align::Center), |ui| {
    ui.button("我会被撑宽");
    ui.button("我也是");
});
```

## 6.6 对齐

`Align` 控制“交叉轴”上的对齐：

```rust
pub enum Align {
    Min,    // 顶端/左端
    Center, // 居中
    Max,    // 底端/右端
}
```

`Align2` 是二维对齐（`Align::Center, Align::Min` 这种组合）。

## 6.7 列：`columns`

```rust
ui.columns(3, |cols| {
    cols[0].label("第 1 列");
    cols[1].label("第 2 列");
    cols[2].label("第 3 列");
});
```

每列宽度相等。

## 6.8 Grid（网格）

`Grid` 适合做表单、表格。它会自动对齐每列宽度。

```rust
egui::Grid::new("my_grid").show(ui, |ui| {
    ui.label("姓名:");
    ui.text_edit_singleline(&mut name);
    ui.end_row();

    ui.label("年龄:");
    ui.add(egui::Slider::new(&mut age, 0..=120));
    ui.end_row();

    ui.label("邮箱:");
    ui.text_edit_singleline(&mut email);
    ui.end_row();
});
```

要点：

- 每行末尾调 `ui.end_row()`。
- `Grid::new("id")` 给个 id。同 id 的 Grid 跨帧记住列宽。
- `Grid` 第一次出现会猜列宽，可能错位一帧——它会自己请求多 pass 纠正，用户看不到。

`Grid` 的常用 builder：

```rust
egui::Grid::new("settings")
    .num_columns(2)
    .spacing([20.0, 8.0])
    .striped(true)            // 隔行换色
    .min_col_width(80.0)
    .show(ui, |ui| { /* ... */ });
```

## 6.9 间距与留白

```rust
// 调整后续控件的间距
ui.spacing_mut().item_spacing = egui::vec2(8.0, 4.0);
ui.spacing_mut().indent = 16.0;

// 手动留白
ui.add_space(10.0);          // 竖向留白 10
ui.allocate_space(egui::vec2(20.0, 0.0)); // 横向留白 20

// 撑满剩余空间（用于把后面的控件推到底/右）
ui.allocate_space(ui.available_size());
```

## 6.10 方向感知

`Layout` 有 `main_dir()`（主方向）、`cross_dir()`（交叉方向）、`horizontal()`（是否横向）、`vertical()`（是否竖向）等方法。写自适应控件时偶尔会用到。

## 6.11 居中一段内容

egui 没有直接的 `center` 方法，常用套路：

```rust
// 横向居中
ui.horizontal(|ui| {
    let space = (ui.available_width() - button_width) * 0.5;
    ui.allocate_space(egui::vec2(space, 0.0));
    ui.button("居中");
});

// 或用 justified + Align::Center
ui.with_layout(egui::Layout::top_down(egui::Align::Center), |ui| {
    ui.button("水平居中");
});
```

## 6.12 两端对齐（sides）

```rust
ui.horizontal(|ui| {
    ui.label("左");
    ui.with_layout(egui::Layout::right_to_left(egui::Align::Center), |ui| {
        ui.button("右");
    });
});
```

egui 也提供了 `egui::Sides` 容器（见 `containers/sides.rs`）来简化这个场景：

```rust
egui::Sides::new().show(ui,
    |ui| { ui.label("左"); },
    |ui| { ui.button("右"); },
);
```

## 6.13 布局原则总结

- 默认竖向，需要横向就 `ui.horizontal`。
- 子 UI 闭包结束后，光标移到下一行/列。
- 表格用 `Grid`。
- 想撑满用 `justified` 或 `allocate_space(ui.available_size())`。
- 居中靠“先留白一半”。

布局在 egui 里是“即时”的——你看到的光标位置就是它现在画到哪了，没有 DOM 那种“等回流”。
