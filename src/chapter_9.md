# 第九章 图片

egui 在 `crates/egui/src/widgets/image.rs` 提供了 `Image` 控件。

## 9.1 最常用形式

### 编译时嵌入图片

```rust
ui.image(egui::include_image!("assets/ferris.png"));
```

`include_image!` 在编译时把图片嵌进二进制，类型是 `ImageSource`。这是最省心的方式——不用管加载、不用管路径。

### 运行时加载文件

```rust
ui.image(egui::Image::from_uri("file://path/to/image.png"));
ui.image(egui::Image::from_uri("path/to/image.png")); // 部分后端也支持
```

### 从 URL 加载（网络）

```rust
ui.image("https://example.com/foo.png");
```

**网络/文件加载需要先装图片加载器**：

```rust
eframe::run_native(
    "app",
    options,
    Box::new(|cc| {
        egui_extras::install_image_loaders(&cc.egui_ctx);
        Ok(Box::new(MyApp::default()))
    }),
)
```

`egui_extras::install_image_loaders` 装的是 `file://`、`http://`、`https://` 的加载器。加载是异步的，第一帧显示空，加载完会自动重绘。

## 9.2 已有纹理（高级）

如果你已经有 GPU 纹理（比如自己用 wgpu/glow 上传的），可以转成 `TextureId`：

```rust
let texture_id: egui::TextureId = /* 来自集成 */;
ui.image((texture_id, egui::vec2(640.0, 480.0)));
```

典型场景：把 3D 场景渲染到纹理，再用 `ui.image` 显示在 egui 里。

## 9.3 Image builder

```rust
ui.add(
    egui::Image::new(egui::include_image!("ferris.png"))
        .fit_to_exact_size(egui::vec2(200.0, 200.0))
        .tint(egui::Color32::from_white_alpha(180)) // 整体染色
        .rounding(egui::Rounding::same(8.0))
        .texture_options(egui::TextureOptions::LINEAR)
);
```

`Image` 的常用方法：

| 方法 | 作用 |
|---|---|
| `fit_to_exact_size(size)` | 指定大小 |
| `fit_to_original_size(scale)` | 按图片原始尺寸 × scale |
| `maintain_aspect_ratio()` | 保持长宽比 |
| `tint(color)` | 整体染色 |
| `rounding(r)` | 圆角 |
| `texture_options(opts)` | 采样方式（最近邻/线性等） |
| `sense(Sense::click())` | 让图片可交互 |

## 9.4 图片大小模式（ImageFit）

```rust
ui.add(
    egui::Image::new(src)
        .fit_to_exact_size([200.0, 200.0].into())
        .image_fit(egui::ImageFit::Contain) // 或 Cover / Fill / None
);
```

- `Contain`：完整显示，可能留白。
- `Cover`：填满，可能裁剪。
- `Fill`：拉伸填满，变形。
- `None`：原始尺寸。

## 9.5 加载状态

异步加载时，想知道加载完了没：

```rust
let max_size = egui::vec2(200.0, 200.0);
let image = egui::Image::from_uri(url).fit_to_exact_size(max_size);
if image.load_for_size(ctx, max_size).is_err() {
    ui.spinner();
} else {
    ui.add(image);
}
```

更细致的控制用 `ctx.try_load_texture` 等 API。

## 9.6 GIF / 动图

egui 支持 GIF 和 WebP 动图（带 `has_gif_magic_header`、`decode_animated_image_uri` 等辅助函数）。装好 `egui_extras` 加载器后直接 `ui.image(uri)` 即可，会自动播放。

## 9.7 SVG

需要开启 `egui_extras/svg` feature。然后：

```rust
ui.image(egui::Image::from_uri("file://icon.svg"));
```

## 9.8 资源路径

`include_image!` 用相对当前源文件的路径。`from_uri("file://...")` 是绝对路径或相对工作目录的路径。

跑示例：

```bash
cargo run --example images
```

里面有从 `include_image!`、文件、URL 加载图片的完整对照。

## 9.9 图片按钮

把图片放按钮里：

```rust
ui.add(
    egui::Button::image(egui::include_image!("icon.png"))
        .min_size(egui::vec2(48.0, 48.0))
);
// 或带文字
ui.add(
    egui::Button::image_and_text(egui::include_image!("icon.png"), "保存")
);
```

## 9.10 小结

| 场景 | 写法 |
|---|---|
| 嵌入静态资源 | `ui.image(egui::include_image!("path.png"))` |
| 文件 | `ui.image(egui::Image::from_uri("file://..."))` |
| 网络 | `ui.image("https://...")`，需装 `install_image_loaders` |
| 已有纹理 | `ui.image((texture_id, size))` |
| 指定大小 | `.fit_to_exact_size([w, h])` |
| 保持比例 | `.maintain_aspect_ratio()` + `ImageFit::Contain` |
| 染色 | `.tint(color)` |

加载图片非常简单，95% 的场景 `ui.image(egui::include_image!(...))` 就够了。
