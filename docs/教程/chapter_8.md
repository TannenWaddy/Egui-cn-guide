# 第八章 文本编辑

`TextEdit` 是 egui 的文本编辑控件，源码在 `crates/egui/src/widgets/text_edit/`。功能比看起来强：单行/多行、复制粘贴、撤销重做、emoji、密码框、可选可编辑文本等。

## 8.1 最常用形式

```rust
// 单行
let mut name = String::new();
ui.text_edit_singleline(&mut name);

// 多行
let mut notes = String::new();
ui.text_edit_multiline(&mut notes);
```

## 8.2 完整 builder

`TextEdit` 是 builder。需要更多控制时用 `ui.add`：

```rust
ui.add(
    egui::TextEdit::singleline(&mut password)
        .password(true)
        .hint_text("输入密码")
        .desired_width(200.0)
        .interactive(true)
        .text_color(egui::Color32::WHITE)
        .clip_text(true)
);
```

多行：

```rust
ui.add(
    egui::TextEdit::multiline(&mut code)
        .code_editor()              // 当代码编辑器（等宽 + 行号风格）
        .desired_width(f32::INFINITY)
        .desired_rows(10)
        .lock_focus(true)
);
```

## 8.3 TextBuffer trait

`TextEdit::new` 接受任何实现 `TextBuffer` 的类型，不只是 `String`。`String` 和 `Rc<String>` 都实现了。你可以给自己的字符串类型实现 `TextBuffer` 来用 egui 编辑。

## 8.4 密码框

```rust
let mut pwd = String::new();
ui.add(egui::TextEdit::singleline(&mut pwd).password(true));
```

显示成圆点，剪贴板操作也会被禁用。

## 8.5 提示文字（hint_text）

输入框为空时显示的灰字：

```rust
ui.add(egui::TextEdit::singleline(&mut search).hint_text("搜索..."));
```

## 8.6 撤销/重做

`TextEdit` 内置撤销/重做，按 `Ctrl+Z` / `Ctrl+Y` 即可。状态按 `Id` 持久化。

## 8.7 选中文本

用户可以鼠标拖选。要程序化获取选区：

```rust
let response = ui.add(egui::TextEdit::singleline(&mut s));
if let Some(state) = egui::TextEdit::State::load(&response.ctx, response.id) {
    let cursor = state.cursor;
    // cursor.char_range 给出选中范围
}
```

也可以让 label 的文字可选中：

```rust
ui.add(egui::Label::new("可以选中复制的文字").selectable(true));
// 或全局开启
ui.style_mut().interaction.selectable_labels = true;
```

## 8.8 限制输入

egui 不直接提供“只能输数字”这类限制，自己在每帧检查并修正即可：

```rust
ui.text_edit_singleline(&mut input);
// 只保留数字
input.retain(|c| c.is_ascii_digit());
```

## 8.9 输入事件回调

`TextEdit` 没有传统的“onChange”回调。即时模式下你只需要在每帧之后看 `&mut s` 的值是否变了。如果要区分“用户按了回车”：

```rust
let r = ui.add(egui::TextEdit::singleline(&mut s).lock_focus(true));
if r.lost_focus() && ui.input(|i| i.key_pressed(egui::Key::Enter)) {
    // 用户按回车了
    submit(&s);
}
```

`Response::lost_focus()` 表示这个控件这一帧失去了焦点，配合按键检测就能模拟“回车提交”。

## 8.10 IME（中文输入法）

`TextEdit` 默认支持 IME。在桌面上由 `egui-winit` 处理 IME 事件。如果你写自定义控件想接 IME，看 `widgets/text_edit/` 里的实现。

## 8.11 代码编辑器

`TextEdit::multiline(...).code_editor()` 会切换成等宽字体 + 行号样式。但 egui 不带语法高亮——要高亮自己用 `LayoutJob` 给文字上色：

```rust
use egui::text::LayoutJob;

let mut job = LayoutJob::default();
// 把字符串切片按 token 加进 job，每段指定颜色
job.append("fn ", 0.0, egui::TextFormat { color: egui::Color32::from_rgb(180, 120, 255), ..Default::default() });
job.append("main", 0.0, egui::TextFormat { color: egui::Color32::from_rgb(255, 200, 100), ..Default::default() });
// ...

ui.label(job);
```

要完整语法高亮，可以用 `syntect` 之类的 crate，配合 `egui_extras::syntect`。

## 8.12 多人协作 / 异步更新

`TextEdit` 直接绑定 `&mut String`。如果数据来自另一个线程，先把数据 clone 到 `App` 字段里再绑给它。改完之后再发回去。

## 8.13 控件大小

```rust
ui.add(
    egui::TextEdit::multiline(&mut s)
        .desired_width(400.0)    // 宽度
        .desired_rows(8)         // 行数
);
```

## 8.14 小结

`TextEdit` 是个功能比较全的编辑控件。日常用法：

- 单行：`ui.text_edit_singleline(&mut s)`
- 多行：`ui.text_edit_multiline(&mut s)`
- 密码：加 `.password(true)`
- 提示：加 `.hint_text(...)`
- 代码风格：加 `.code_editor()`
- 检测回车：`r.lost_focus() && ui.input(|i| i.key_pressed(Key::Enter))`
