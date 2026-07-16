# 期刊网页构建工具 V0.2.0

> **单文件 · 零依赖 · 开箱即用**  
> 面向学术期刊的网页可视化构建工具

[![Version](https://img.shields.io/badge/version-0.2.0-blue.svg?style=flat-square)]()
[![License](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)]()

---

## 项目简介

期刊网页构建工具是一个单文件 HTML 应用，用于可视化构建学术期刊风格的网页展示页面。无需后端、无需构建、无需安装，浏览器打开即用。

**典型产出**：期刊 Highlights 页面、最新录用列表、虚拟专刊展示、会议征稿页、年度报告页面等静态 HTML 内容。

**服务对象**：
- **期刊编辑/内容运营** - 拖拽组件、选择模板、调整配色，零代码产出专业网页
- **前端开发者** — 代码模式手写 HTML，实时预览，精细控制布局

---

## 快速开始

```bash
# 方式一：直接下载
# 下载 index.html，双击打开即可

# 方式二：克隆仓库
git clone https://github.com/CarlosZhou6767/Newsletter-Builder.git
cd Newsletter-Builder
# 双击 index.html 打开
```

**无需 `npm install`、无需构建、无需启动服务器。**

---

## 技术架构

### 单文件架构

所有代码封装在单个 `index.html` 中：

```
index.html
├── HTML 结构（~310 行）
│   ├── 工具栏（品牌标识、模式切换、操作按钮）
│   ├── 代码编辑器/预览面板
│   ├── 助手面板（组件/模板/样式/检查）
│   ├── 画布/属性面板
│   ├── 弹窗（欢迎/确认/全屏/Toast）
│   └── 状态栏
├── CSS 样式（~1000 行）
│   ├── CSS 变量（主题/间距/字体/阴影/圆角）
│   ├── 布局系统（Flex/Grid）
│   ├── 组件样式（工具栏/面板/画布/弹窗）
│   ├── 响应式适配
│   └── 暗色主题
└── JavaScript 逻辑（~3600 行）
    ├── 工具函数（XSS 转义、debounce、颜色转换、DOM 缓存）
    ├── 核心系统（模式切换、主题管理、组件管理）
    ├── 渲染引擎（全量渲染 + 组件级 Diff 增量渲染）
    ├── 拖放系统（排序/插入）
    ├── 模板系统（内置/自定义）
    ├── 样式系统（配色/字体/间距）
    ├── 检查系统（WCAG 审计/发布前检查/暗色模式预览）
    ├── 历史系统（内存 undo/redo + IndexedDB 持久化）
    ├── 导出系统（HTML/JSON 生成与下载）
    ├── 快捷键系统（7 个全局快捷键）
    └── 初始化（数据加载/事件绑定）
```

### 技术栈

| 层级 | 技术 |
|------|------|
| 语言 | HTML5 + CSS3 + ES6+ |
| 存储 | localStorage（设计数据/模板/组件）+ IndexedDB（版本历史） |
| 渲染 | 全量渲染 + 组件级 Diff 引擎（基于 JSON 签名比对） |
| 安全 | XSS 转义 `S()`、URL 白名单 `safeUrl()`、iframe sandbox |
| 性能 | DOM 缓存 `el()`、事件委托、防抖节流、`structuredClone` 深拷贝（含 polyfill）、`requestAnimationFrame` 批量更新、IndexedDB 游标批量删除 |

### 版本历史

| 版本 | 亮点 |
|------|------|
| **v0.2.0** | 项目定位转型为"期刊网页构建工具"、移除邮件相关功能（垃圾邮件评分、Outlook兼容检查）、导出HTML从table布局改为现代CSS div布局、暗色模式预览重命名、发布前检查重命名、品牌全面更新 |
| **v0.1.3** | JSON 导入/导出、剪贴板复制粘贴、搜索过滤、7 键快捷键系统、DOM 缓存、IndexedDB 游标优化、深拷贝去重及 polyfill、CSS 性能优化（通配符/transition/will-change）、无障碍增强（aria-label/role/对比度修复）、Bug 修复（搜索激活/模板加载/未定义变量） |
| **v0.1.2** | 组件级 Diff 渲染引擎、事件委托、防抖节流、XSS 修复、iframe 沙箱加固、JSDoc 注释规范 |
| **v0.1.1** | 安全修复、逻辑 Bug 修复、左侧面板布局优化、欢迎弹窗、版本号统一 |
| **v0.1.0** | 初始版本：三种模式、组件库、模板系统、样式系统、导出、历史撤销 |

---

## 核心功能

### 三种工作模式

| 模式 | 适用人群 | 说明 |
|------|----------|------|
| **代码模式** | 开发者 | 左右分栏：代码编辑器 + 实时预览，支持 Tab 缩进、语法高亮、格式化 |
| **设计模式** | 内容运营 | 三栏布局：助手面板 + 画布 + 属性编辑，拖拽式操作 |
| **预览模式** | 所有人 | 全屏预览最终效果，桌面/手机设备切换，暗色模式预览 |

### 组件系统

19 种内置组件，覆盖学术期刊常见内容块：

`heading` · `text` · `image` · `image-text` · `cta` · `divider` · `spacer` · `list` · `quote` · `callout` · `gallery` · `author` · `journal-cover` · `toc` · `paper-list` · `paper-card` · `event-card` · `banner` · `footer`

每个组件支持：拖拽排序、复制、删除、保存为自定义区块、属性编辑。**搜索过滤**：输入关键词实时过滤组件库和模板列表。

### 渲染引擎

- **全量渲染** `renderCanvasLegacy()`：组件数量变化超 50% 或首次渲染时触发，重建全部 DOM
- **Diff 增量渲染** `patchCanvas()`：常规编辑时触发，比对组件 JSON 签名，只更新变化的节点
- 渲染调度 `renderCanvas()`：`requestAnimationFrame` 合并批量更新，避免布局抖动

### 模板系统

- **内置模板**：7 个预设模板（期刊精选、封面+图文、封面+列表、会议横幅等）
- **自定义模板**：保存当前设计为模板，存储在 localStorage，支持命名/加载/删除

### 样式系统

- **主题色**：主色/背景色/文字色拾取器 + 12 套预设配色 + 3 种智能配色（类比/互补/三角）
- **字体**：5 种字体栈、H1-H3 字号滑块、字重/字号/行高调节
- **间距**：容器宽度(480-680px)、区块间距(8-64px)、内边距(8-48px)、网格基准(8/12/16px)

### 检查系统

| 功能 | 说明 |
|------|------|
| **暗色模式预览** | 一键切换画布暗色模式，预览读者端暗色效果 |
| **WCAG-AA 审计** | 遍历所有组件，计算文字/背景对比度，按 WCAG 2.1 评级并给出修复建议 |
| **发布前检查** | 5 项检查：图片完整性、链接空值、Alt 属性、移动端适配、对比度合规 |
| **版本历史** | 每次编辑自动保存快照到 IndexedDB（最多 50 帧），支持浏览、恢复、删除 |

### 数据持久化

| 数据 | 存储 | 说明 |
|------|------|------|
| 设计内容/模板/主题 | localStorage | 自动保存（防抖 2s），刷新不丢失 |
| 版本历史 | IndexedDB | 最多 50 帧，自动淘汰最旧 |
| 编辑器主题 | localStorage | 亮/暗主题偏好持久化 |

### 导出

- 生成内联 CSS + 语义化 HTML
- 自动注入暗色模式适配（`@media (prefers-color-scheme: dark)`）
- 一键下载 `journal-page.html`
- **JSON 项目导出**：导出完整项目数据（组件、主题、字体、间距），支持跨设备迁移
- **JSON 项目导入**：从 JSON 文件恢复完整项目，工具栏"导入"按钮
- 支持嵌入 CMS、GitHub Pages、iframe 三种发布方式

### 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+Z` / `Ctrl+Shift+Z` | 撤销 / 重做 |
| `Ctrl+S` | 保存到本地存储 |
| `Ctrl+D` | 复制选中组件 |
| `Ctrl+C` / `Ctrl+V` | 剪贴板复制 / 粘贴组件 |
| `Delete` | 删除选中组件 |
| `/` | 聚焦搜索框（设计模式） |
| `Ctrl+Enter` | 刷新预览（代码模式） |

---

## 开发指南

### 环境

- 任意现代浏览器（Chrome / Firefox / Safari / Edge）
- 无需构建工具、无需包管理器

### 项目结构

```
Newsletter-Builder/
├── index.html              # 主入口（单文件应用）
├── README.md               # 开发者文档
├── USER_MANUAL.md          # 用户使用手册
├── .gitignore
└── LICENSE
```

### 代码规范

- JavaScript 使用 ES6+ 语法
- CSS 变量优先，避免硬编码色值
- 组件渲染函数返回 HTML 字符串，不直接操作 DOM
- 用户输入必须通过 `escapeHtml()` / `S()` 转义
- 公共函数使用 JSDoc 注释（`@param` + `@returns`）

### 构建与部署

本项目为零构建项目，直接修改 `index.html` 即可。提交后通过 GitHub Pages 部署：

```bash
git add index.html
git commit -m "feat: add some feature"
git push
```

启用 GitHub Pages 后，访问 `https://<username>.github.io/Newsletter-Builder/` 即可使用最新版本。

### 贡献

欢迎提交 Issue 和 Pull Request：

1. Fork 本仓库
2. 创建特性分支：`git checkout -b feature/your-feature`
3. 提交变更：`git commit -m 'feat: add some feature'`
4. 推送分支：`git push origin feature/your-feature`
5. 发起 Pull Request

提报 Issue 时请提供：浏览器版本、复现步骤、预期行为与实际表现、控制台错误信息。

---

## 许可证

[MIT](LICENSE)

```
MIT License

Copyright (c) 2026 Newsletter-Builder

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 项目链接

- **GitHub**：[https://github.com/CarlosZhou6767/Newsletter-Builder](https://github.com/CarlosZhou6767/Newsletter-Builder)
- **用户手册**：[USER_MANUAL.md](USER_MANUAL.md)
- **作者**：Carlos Zhou