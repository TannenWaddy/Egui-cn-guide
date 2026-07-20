# 第七章 容器：面板、窗口与滚动区

egui 的“容器”在 `crates/egui/src/containers/` 下，是用来组织更大块界面的：面板（Panel）、窗口（Window）、滚动区（ScrollArea）、折叠头（CollapsingHeader）、框架（Frame）、菜单（Menu）、弹出层（Popup/Area）等。

## 7.1 CentralPanel：中央面板

最常用的顶层容器，占满剩余空间。

```rust
egui::CentralPanel::default().show(ui, |ui| {
    ui.label("主内容");
});
```

> ⚠ 顶层面板中，`CentralPanel` **必须最后加**。它吃掉所有剩余空间。

## 7.2 SidePanel：侧边面板

```rust
egui::SidePanel::left("left_panel").show(ui, |ui| {
    ui.heading("左侧");
    ui.button("点我");
});

egui::SidePanel::right("right_panel")
    .resizable(true)
    .default_width(200.0)
    .width_range(150.0..=400.0)
    .show(ui, |ui| {
        ui.label("右侧");
    });

egui::CentralPanel::default().show(ui, |ui| {
    ui.label("中间");
});
```

要点：

- `SidePanel::left("id")` / `right("id")` / `top("id")` / `bottom("id")`。
- 第一个参数是 id（字符串），用来记住用户拖动后的宽度。
- `resizable(true)` 让用户能拖动边缘改宽度。
- **添加顺序很重要**：先加的面板在外层，最后加的在最里。`CentralPanel` 永远最后。

## 7.3 TopBottomPanel：上下面板

```rust
egui::TopBottomPanel::top("menu_bar").show(ui, |ui| {
    egui::menu::bar(ui, |ui| {
        ui.menu_button("文件", |ui| {
            if ui.button("新建").clicked() { /* ... */ }
            if ui.button("退出").clicked() { /* ... */ }
        });
        ui.menu_button("帮助", |ui| {
            if ui.button("关于").clicked() { /* ... */ }
        });
    });
});

egui::CentralPanel::default().show(ui, |ui| {
    ui.label("主体");
});
```

典型用法：顶上放菜单栏或工具栏，底下放状态栏。

## 7.4 Window：浮动窗口

```rust
egui::Window::new("设置")
    .open(&mut show_window)   // 是否显示，关闭按钮会把这里设成 false
    .resizable(true)
    .collapsible(true)
    .default_pos([100.0, 100.0])
    .default_size([300.0, 200.0])
    .min_width(200.0)
    .vscroll(true)             // 内置垂直滚动
    .hscroll(false)
    .show(ctx, |ui| {
        ui.label("窗口内容");
    });
```

> 注意 `Window::show` 第一个参数是 `ctx`（Context），不是 `ui`——因为窗口是顶层的，直接挂在 ctx 上。

`Window` builder 还有：

- `.title_bar(true/false)`：要不要标题栏。
- `.drag_area(WindowDrag::TitleBar)`：从哪里拖动窗口。`OnTouch`（默认）是触摸屏任意位置拖、桌面只从标题栏拖。
- `.frame(Frame::none())`：自定义边框。
- `.current_pos(pos)` / `.current_size(size)`：程序控制位置。
- `.order(Order::Foreground)`：层级。
- `.id(Id::new(...))`：手动指定 ID（同名多窗口必须）。

注意：**egui 的 Window 不是操作系统窗口**，是 egui 自己画的浮动层。要开原生 OS 窗口用 viewport（见后）。

## 7.5 ScrollArea：滚动区

```rust
egui::ScrollArea::vertical().show(ui, |ui| {
    for i in 0..1000 {
        ui.label(format!("行 {i}"));
    }
});
```

要点：

- `ScrollArea::vertical()` / `horizontal()` / `both()`。
- **只排可见的部分**，所以 1000 行也不会卡。
- 闭包里的 `ui` 是滚动区内部的 UI。

常用 builder：

```rust
egui::ScrollArea::vertical()
    .auto_shrink([false; 2])      // 不自动收缩
    .stick_to_right(true)         // 内容贴右（聊天窗口）
    .scroll_bar_visibility(egui::ScrollBarVisibility::AlwaysVisible)
    .max_height(400.0)
    .show(ui, |ui| { /* ... */ });
```

`ScrollArea::show` 返回 `ScrollAreaOutput`，里面有 `inner`（闭包返回值）、`id`、`content_size`、`state` 等。

程序化滚动：

```rust
let scroll_id = ui.make_persistent_id("my_scroll");
let mut scroll_state = egui::ScrollArea::State::load(ctx, scroll_id);
scroll_state.offset.y = 0.0; // 滚到顶
scroll_state.store(ctx, scroll_id);
```

或者用 `Response::scroll_to_me`：

```rust
let r = ui.label("看我");
r.scroll_to_me(Some(egui::Align::Center));
```

## 7.6 CollapsingHeader：折叠头

```rust
ui.collapsing("高级选项", |ui| {
    ui.label("这里是被折叠的内容");
    ui.checkbox(&mut opt1, "选项 1");
});

// 或带状态控制
let mut open = true;
ui.collapsing_with_state("高级", &mut open, |ui| {
    ui.label("...");
});
```

builder 版本：

```rust
egui::CollapsingHeader::new("标题")
    .default_open(true)
    .show(ui, |ui| { /* ... */ });
```

## 7.7 Frame：装饰框

`Frame` 给内容加背景、边框、圆角、内边距。

```rust
egui::Frame::group(ui.style())
    .stroke(egui::Stroke::new(1.0, egui::Color32::GRAY))
    .fill(egui::Color32::from_gray(30))
    .inner_margin(8.0)
    .show(ui, |ui| {
        ui.label("被框起来的内容");
    });
```

预设的 Frame：

- `Frame::group(style)`：组框。
- `Frame::side_top_panel(style)` / `side_bottom_panel` 等：各位置面板的默认框。
- `Frame::central_panel(style)`。
- `Frame::popup(style)`：弹出层。
- `Frame::none()`：无装饰。

`CentralPanel` 等也接受自定义 frame：

```rust
egui::CentralPanel::default().frame(egui::Frame::none()).show(ui, |ui| {
    // 无背景无边框
});
```

## 7.8 Area：浮动层（底层）

`Area` 是更底层的“任意位置浮层”，`Window` 就是基于它做的。

```rust
egui::Area::new(egui::Id::new("my_area"))
    .fixed_pos(egui::pos2(100.0, 100.0))
    .order(egui::Order::Foreground)
    .show(ctx, |ui| {
        ui.label("飘在任意位置");
    });
```

写自定义弹出菜单、提示框时常用。

## 7.9 Popup：弹出层

`Popup` 是 `Area` 之上的封装，配合 `ComboBox` 等用。

```rust
let button = ui.button("下拉");
egui::popup::popup_above_or_below_widget(&button, ui, egui::AboveOrBelow::Below, |ui| {
    ui.set_min_width(120.0);
    ui.button("选项 A");
    ui.button("选项 B");
});
```

更常用的右键菜单：

```rust
let r = ui.label("右键我");
r.context_menu(|ui| {
    if ui.button("复制").clicked() { /* ... */ }
    if ui.button("粘贴").clicked() { /* ... */ }
});
```

## 7.10 ComboBox：下拉框

```rust
let mut choice = 0;
egui::ComboBox::from_label("选择")
    .selected_text(format!("选项 {}", choice))
    .show_ui(ui, |ui| {
        ui.selectable_value(&mut choice, 0, "选项 0");
        ui.selectable_value(&mut choice, 1, "选项 1");
        ui.selectable_value(&mut choice, 2, "选项 2");
    });
```

`selectable_value` 也支持任何 `PartialEq` 类型（枚举、字符串等）。

## 7.11 Menu：菜单栏

```rust
egui::menu::bar(ui, |ui| {
    ui.menu_button("文件", |ui| {
        if ui.button("新建").clicked() {}
        if ui.button("打开").clicked() {}
        ui.separator();
        if ui.button("退出").clicked() {}
    });
    ui.menu_button("编辑", |ui| {
        if ui.button("撤销").clicked() {}
    });
});
```

## 7.12 Modal：模态框

`Modal`（`containers/modal.rs`）会盖住整个界面，阻止点击穿透：

```rust
let mut show_modal = true;
if show_modal {
    let modal = egui::Modal::new(egui::Id::new("my_modal"));
    modal.show(ctx, |ui| {
        ui.heading("确认删除？");
        ui.horizontal(|ui| {
            if ui.button("确定").clicked() { show_modal = false; }
            if ui.button("取消").clicked() { show_modal = false; }
        });
    });
}
```

## 7.13 容器小结

| 容器 | 用途 |
|---|---|
| `CentralPanel` | 主区域，必须最后加 |
| `SidePanel` | 左右上下侧栏 |
| `TopBottomPanel` | 顶栏/底栏 |
| `Window` | 浮动窗口（egui 内部，非 OS） |
| `ScrollArea` | 滚动区 |
| `CollapsingHeader` | 折叠 |
| `Frame` | 装饰框 |
| `Area` | 任意位置浮层 |
| `Popup` | 弹出层 |
| `ComboBox` | 下拉框 |
| `menu::bar` | 菜单栏 |
| `Modal` | 模态框 |
| `Sides` | 两端对齐 |

写 egui 界面基本就是“把这些容器一层层嵌套，最里面放控件”。
