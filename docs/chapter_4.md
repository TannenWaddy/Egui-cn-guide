# 第四章 核心概念

egui 有几个核心类型，理解了它们，看 egui 文档就轻松了。

## 4.1 `Context`：全局上下文

`Context` 是 egui 的“大脑”，存着所有跨帧的状态：字体、样式、输入、内存、动画等。按惯例变量名写成 `ctx`。

特点：

- 可以被克隆（内部是 `Arc`），克隆是廉价的。
- 用一个 `RwLock` 保护。读数据时用闭包形式，避免忘记解锁导致死锁：

```rust
ctx.input(|i| i.key_down(egui::Key::A));
```

不要这样写：

```rust
// 错误示范：手动拿锁容易死锁
let input = ctx.input_mut();
// ... 这里如果再调 egui API 可能死锁
```

`Context` 的常用方法：

- `ctx.input(|i| ...)`：读输入（键盘、鼠标）。
- `ctx.input_mut(|i| ...)`：改输入（少用）。
- `ctx.style()` / `ctx.set_style(...)`：读/设置全局样式。
- `ctx.fonts(...)` / `ctx.set_fonts(...)` / `ctx.add_font(...)`：字体。
- `ctx.request_repaint()`：从其它线程请求重绘。
- `ctx.request_discard()`：请求多跑一遍 pass。
- `ctx.read_response(id)`：在本帧添加控件之前，先读这个 ID 上帧的交互结果（高级用法）。

## 4.2 `Ui`：当前正在排的区域

`Ui` 表示“屏幕上一块正在被填充的区域”。你拿到一个 `&mut Ui`，就能往里加控件。

`Ui` 内部维护：

- `min_rect`：当前已经占用的最小矩形（会被控件撑大）。
- `max_rect`：这块区域的软上限（文字会在这里换行）。
- `cursor`：下一个控件放在哪。
- `id`：当前 UI 的 ID 前缀。

每个 `Ui` 有一个 `Layout`（布局方向），决定控件是横着排、竖着排、还是别的。

常用方法非常多，列举一些：

| 方法 | 作用 |
|---|---|
| `ui.label("...")` | 放一段文字 |
| `ui.button("...")` | 按钮，返回 `Response` |
| `ui.add(widget)` | 加任意实现 `Widget` 的控件 |
| `ui.horizontal(\|ui\| ...)` | 横着排 |
| `ui.vertical(\|ui\| ...)` | 竖着排 |
| `ui.horizontal_wrapped(\|ui\| ...)` | 横着排并自动换行 |
| `ui.with_layout(layout, \|ui\| ...)` | 指定布局 |
| `ui.allocate_space(size)` | 留白 |
| `ui.allocate_exact_size(size, sense)` | 申请固定大小，返回 `(Rect, Response)` |
| `ui.separator()` | 分隔线 |
| `ui.spacing()` / `ui.spacing_mut()` | 间距等微调 |
| `ui.style()` / `ui.style_mut()` | 样式 |
| `ui.painter()` | 拿到画笔，直接画 |
| `ui.ctx()` | 拿到 `Context` |
| `ui.available_size()` | 还剩多少空间 |
| `ui.make_persistent_id(...)` | 造一个持久化 ID |

## 4.3 `Id`：唯一标识

`Id` 用来给“egui 需要自己存的状态”打标签。比如：

- 窗口位置（按窗口 ID 存）。
- 折叠头是展开还是收起。
- 滚动距离。
- 哪个滑块正在被拖。

大部分时候 `Id` 自动生成（根据调用位置 + 父 UI 的 ID）。少数情况要手动给：

```rust
ui.push_id("my_unique_id", |ui| {
    // 这里面的控件 ID 都会带上 "my_unique_id" 前缀
});

// 或者直接给 Window 指定 ID
egui::Window::new("Settings").id(egui::Id::new("settings_window")).show(ctx, |ui| {
    // ...
});
```

`Id` 内部就是个 `u64`，可以从字符串、整数、指针等造出来。注意它**只在同一个父 UI 内需要唯一**，不同父 UI 下可以重名。

## 4.4 `Response`：交互结果

每个控件返回一个 `Response`，告诉你“这个控件刚才被怎么操作了”。这是 egui 最核心的返回值。

`Response` 上常用方法（按 `response.rs` 源码）：

| 方法 | 含义 |
|---|---|
| `clicked()` | 被点击（按下并松开） |
| `clicked_by(button)` | 被指定鼠标键点击 |
| `secondary_clicked()` | 右键点击 |
| `hovered()` | 鼠标悬停在上面 |
| `contains_pointer()` | 鼠标指针在它的矩形内 |
| `dragged()` | 正在被拖动 |
| `dragged_by(button)` | 被指定键拖动 |
| `drag_delta()` | 这一帧拖动了多少 |
| `interact_pointer_pos()` | 指针位置（如果在该控件上交互） |
| `changed()` | 控件标记了“我变了” |
| `mark_changed()` | 控件主动标记“变了”（自定义控件用） |
| `active()` | 当前正在被交互（按下/拖动） |
| `request_focus()` / `surrender_focus()` | 申请/放弃焦点 |
| `has_focus()` | 是否有焦点 |
| `on_hover_text("...")` | 悬停时显示提示文字 |
| `on_hover_ui(\|ui\| ...)` | 悬停时显示自定义 UI |
| `on_hover_cursor(CursorIcon::PointingHand)` | 悬停时光标变形 |
| `show_tooltip_text("...")` | 显示 tooltip |
| `context_menu(\|ui\| ...)` | 右键菜单 |
| `scroll_to_me(...)` | 把我滚到可见 |
| `widget_info(\|\| ...)` | 设置无障碍信息（屏幕阅读器） |
| `labelled_by(id)` | 把这个控件和某个 label 关联（无障碍） |

例子：

```rust
let response = ui.button("Click me");
if response.clicked() {
    println!("点了");
}
if response.hovered() {
    response.on_hover_text("点我试试");
}
```

`Response` 还能链式调用，所有 `on_hover_*` 之类的方法都返回 `Self`：

```rust
ui.button("Save")
    .on_hover_text("保存当前文件")
    .on_hover_cursor(egui::CursorIcon::PointingHand);
```

## 4.5 `Sense`：控件感知什么交互

每个控件有一个 `Sense`，告诉 egui “我关心哪些交互”：

```rust
pub struct Sense {
    pub click: bool,   // 点击
    pub drag: bool,    // 拖动
    pub focusable: bool, // 能不能拿焦点
}
```

预设：

- `Sense::click()`：只感知点击。按钮默认是这个。
- `Sense::drag()`：只感知拖动。
- `Sense::click_and_drag()`：两个都感知。
- `Sense::focusable()`：只感知焦点（不可点击/拖动）。
- `Sense::hover()`：什么都不感知，只是能被悬停。

为什么这重要？因为**只有声明了对应 Sense 的控件才会“消费”对应的事件**。

比如 `Button` 只有 `Sense::click`，所以你拖动一个按钮时它不会响应 `dragged()`，拖动事件会“穿过”按钮，传到它后面的控件——这正是 `ScrollArea` 能在按钮上拖动滚动的原因（触摸屏特别需要）。

你可以在自己的控件里用 `ui.allocate_exact_size(size, sense)` 或 `ui.interact(rect, id, sense)` 来指定 Sense。

## 4.6 `InnerResponse`

有些容器（如 `horizontal`、`Window::show`）返回 `InnerResponse<R>`，它包含两样东西：

```rust
pub struct InnerResponse<R> {
    pub inner: R,           // 闭包里返回的值
    pub response: Response, // 容器本身的 Response
}
```

用法：

```rust
let result = ui.horizontal(|ui| {
    ui.button("A");
    ui.button("B");
    "horizontal finished" // 闭包返回值
});
println!("{}", result.inner);        // "horizontal finished"
println!("{:?}", result.response.rect); // 这一行整体的矩形
```

## 4.7 数据流总结

把上面这些串起来，egui 一帧的流程是：

```text
1. eframe 收集输入 → 喂给 Context
2. eframe 调用你的 App::ui(ui, frame)
3. 你写 UI 代码：
   - ui.button(...) → egui 现场排版 + 交互检测 + 加入绘制列表 → 返回 Response
   - 你根据 Response 改自己的状态
4. egui 把这一帧的绘制列表（shapes）tessellate 成三角形
5. eframe 用 glow/wgpu 把三角形画到屏幕
6. 下一帧回到第 1 步
```

记住这个循环，看任何 egui 代码都不会迷路。
