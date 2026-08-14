# Egui 中文指南

这是一个关于 egui 的中文教程项目，使用 [mdBook](https://rust-lang.github.io/mdBook/) 构建静态网站。

## 项目介绍

egui是一个用纯 Rust 编写的即时模式 GUI 库，具有以下特点：

- **简单**：写法直白，没有回调、没有信号槽
- **快速**：debug 构建下也能达到 60 FPS
- **可移植**：同一份代码能编译成桌面程序，也能编译成网页（WebAssembly）

本项目旨在为中文用户提供一个完整的 egui 学习指南，涵盖从基础概念到实战技巧的各个方面。

### 📖 内容来源

本指南的所有内容均由 AI 自动编写。AI 读取了 egui 官方仓库的源码、文档和示例代码，然后生成了这套完整的中文教程。内容力求准确反映 egui 的实际用法和设计理念，但由于是自动生成，可能存在不准确或过时的地方。

## 目录结构

```
Egui中文指南/
├── book.toml          # mdBook 配置文件
├── src/               # Markdown 源文件目录
│   ├── SUMMARY.md     # 目录结构定义
│   ├── chapter_1.md   # 第一章 认识 egui
│   ├── chapter_2.md   # 第二章 快速开始
│   ├── chapter_3.md   # 第三章 即时模式 GUI
│   ├── chapter_4.md   # 第四章 核心概念
│   ├── chapter_5.md   # 第五章 内置控件
│   ├── chapter_6.md   # 第六章 布局
│   ├── chapter_7.md   # 第七章 容器：面板、窗口与滚动区
│   ├── chapter_8.md   # 第八章 文本编辑
│   ├── chapter_9.md   # 第九章 图片
│   ├── chapter_10.md  # 第十章 字体与样式
│   ├── chapter_11.md  # 第十一章 自定义控件
│   ├── chapter_12.md  # 第十二章 eframe 与后端集成
│   └── chapter_13.md  # 第十三章 实战技巧与常见问题
└── book/              # 构建输出目录（已自动生成）
```

## 构建方式

### 前置条件

1. 安装 Rust 和 Cargo
2. 安装 mdBook：

```bash
cargo install mdbook
```

### 构建步骤

1. 克隆仓库：

```bash
git clone https://github.com/TannenWaddy/Egui-cn-guide.git
cd Egui-cn-guide
```

2. 构建静态网站：

```bash
mdbook build
```

构建完成后，静态文件将输出到 `book/` 目录。

3. 本地预览：

```bash
mdbook serve
```

启动本地服务器，默认地址为 `http://localhost:3000`。

### 部署到 GitHub Pages

本项目已配置 GitHub Actions 自动部署。当推送到 `main` 分支时，会自动构建并部署到 GitHub Pages。

在线访问地址：https://tannenwaddy.github.io/Egui-cn-guide/

## 参与贡献

欢迎提交 Issue 和 Pull Request 来改进本项目。

## 相关链接

- [egui 官方仓库](https://github.com/emilk/egui)
- [egui 官方文档](https://docs.rs/egui)
- [mdBook 官方文档](https://rust-lang.github.io/mdBook/)
- [eframe 官方文档](https://docs.rs/eframe)

## 许可证

本项目采用 [MIT 许可证](LICENSE)。