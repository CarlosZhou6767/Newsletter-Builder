# Newsletter Builder

**单文件 · 零依赖 · 开箱即用的学术期刊网站简报制作工具**

[![GitHub release](https://img.shields.io/badge/version-1.1.0-blue.svg?style=flat-square)]()
[![License](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)]()
[![Features](https://img.shields.io/badge/features-3%20new-orange.svg?style=flat-square)](docs/USER_GUIDE.md)

> 无需后端 · 无需安装 · 无需注册 · 打开即用。

## 项目简介

Newsletter Builder 是一款面向学术期刊编辑和开发者的网站简报制作工具。专为学术期刊官网设计，用于制作期刊 Highlights、最新录用、虚拟专刊等 HTML 内容页面，直接嵌入期刊网站或通过 GitHub Pages 独立发布。项目采用单文件架构，所有代码封装在单个 HTML 文件中，无需后端服务、无需安装依赖、无需注册账号，直接打开即可运行。

项目同时服务两类用户：

- **期刊编辑 / 内容运营** — 通过可视化设计模式，拖拽组件、选择模板、调整配色，零代码产出专业期刊简报页面
- **前端开发者** — 手写 HTML 代码，实时预览效果，精细控制布局与样式

**交付物**：[`index.html`](index.html)（主入口，打开即用）

**典型使用场景**：MDPI 期刊官网的"Highlights"页面、"Recent Articles"展示栏、"Special Issue"征稿页、期刊年度报告、编委会名单等静态 HTML 内容块。

## 功能特性

- **三种工作模式** — 代码模式、设计模式、预览模式一键切换，内容实时同步不丢失
- **丰富组件库** — 学术期刊专用组件：论文卡片、作者列表、期刊封面、目录、CTA 按钮、图表等；也支持通用文本与媒体组件
- **预设模板** — 期刊 Highlights、最新录用、虚拟专刊、征稿启事、年度报告、编委会等，快速起步
- **智能标识助手** — 颜色、字体、间距、组件、模板、检查六大面板，精细调控
- **HTML 导出** — 生成内联 CSS + 语义化 HTML，可直接嵌入期刊网站或部署到 GitHub Pages
- **本地持久化** — 自动保存至 `localStorage`，支持多步历史撤销
- **暗色主题** — 编辑器亮色/暗色主题自由切换，设置自动持久化
- **设备预览** — 预览模式下支持桌面 / 手机 / 平板宽度切换，确保多端阅读体验
- **隐私安全** — 纯本地处理，不上传任何代码到服务器

### 1.1.0 新增三项功能

> 详见 [用户使用指南](docs/USER_GUIDE.md)。三项功能均位于"检查"面板，纯前端实现，不引入外部依赖。

- **🌃 暗色网站模式预览** — 在"检查"页一键预览简报页面在读者浏览器暗色模式下的效果；导出 HTML 时自动注入 `@media (prefers-color-scheme: dark)` 媒体查询，让网站自动适配暗色主题。固定学术编辑暗色调色板（`--bg-primary: #0f1218` 等），零配置开箱即用
- **♿ WCAG-AA 无障碍审计** — 遍历画布每个组件，用 `getComputedStyle` + `TreeWalker` 算每对（背景色, 文字色）的对比度，按 WCAG 2.1 AA/AAA 评级并给出可操作修复建议（如"把文字色调至 #6C7989 可达 4.5:1"）。复制对照评级：正文 < 4.5:1 / 大字 < 3:1 即标记 AA 失败
- **⏱️ 版本历史时间线** — 每次编辑自动通过 IndexedDB 持久化一帧（含 components + theme + font + spacing 完整快照 + SVG 抽象缩略图 + 时间戳），可滚动浏览、点击任意帧一键恢复、删除单帧或清空全部。最多保留 50 帧，自动淘汰最旧的；刷新浏览器、关闭再打开仍在

## 快速开始

### 环境要求

- 任意现代浏览器（Chrome / Firefox / Safari / Edge 最新版本）

### 方式一：直接打开

1. 下载本仓库中的 [`index.html`](index.html) 文件
2. 双击打开即可使用

```bash
# Windows
start index.html

# macOS
open index.html

# Linux
xdg-open index.html
```

### 方式二：克隆仓库

```bash
git clone https://github.com/CarlosZhou6767/Newsletter-HTML.git
cd Newsletter-HTML
# 直接打开 index.html 文件
```

**无需 `npm install`、无需构建、无需启动服务器。下载即用。**

## 使用指南

### 代码模式

适合开发者手写 HTML 进行精细化控制。

- **布局**：左右分栏 — 左侧代码编辑器（带行号与语法高亮）、右侧实时预览
- **功能**：
  - Tab 缩进支持
  - 代码实时预览更新
  - HTML 统计信息（字符数、行数）
  - 底部错误控制台，捕获并显示渲染错误
- **操作**：导出 HTML、复制代码、格式化代码

### 设计模式

适合非技术用户通过可视化界面构建内容。

- **布局**：三栏结构
  - **左侧**：智能标识助手面板（6 个 Tab）
  - **中间**：画布区，组件可按需排列，支持拖拽排序
  - **右侧**：实时预览 + 微编辑面板（选中组件后显示）
- **底部**：状态栏（组件数、主题色、操作提示）

**设计模式六个面板：**

| 面板 | 功能 |
|------|------|
| **颜色** | 主题色/背景色/文字色拾取器；预设配色；智能和谐配色生成；实时对比度检查 |
| **字体** | 字体栈切换；H1/H2/H3 字号滑块；字重 300–700；正文字号 12–20px；行高 1.4–2.0 |
| **间距** | 容器宽度 480–680px；区块间距 8–64px；内边距 8–48px；网格基准 8/12/16px |
| **组件** | 组件卡片网格，点击插入到画布；支持自定义区块保存与复用 |
| **模板** | 预设模板 + 自定义模板保存/加载/删除 |
| **检查** | 运行发送前检查：图片完整性、链接空值、Alt 属性、移动端适配、对比度合规；**1.1.0 新增**：邮件暗色客户端预览 toggle、WCAG-AA 审计按钮、底部新增版本历史时间线 section |

### 预览模式

模拟最终页面效果，全面检查呈现质量。

- 全屏预览最终页面效果
- **设备预览**：支持桌面 / 手机 / 平板宽度切换
- 支持全屏弹窗预览
- **暗色模式预览**（1.1.0）：一键切换暗色模式，提前查看读者端暗色效果

## 组件交互

| 操作 | 说明 |
|------|------|
| **插入** | 点击组件卡片 → 添加到画布末尾 |
| **选中** | 点击画布上组件 → 边框高亮 + 微编辑面板 |
| **排序** | 拖拽左侧拖拽手柄调整组件上下顺序 |
| **操作** | 上移 / 下移 / 复制 / 删除 |
| **编辑** | 微编辑面板修改文字、图片 URL、颜色等属性 |
| **保存为区块** | 单个组件可保存为自定义区块，供后续复用 |

## 模板系统

模板 = 预设组件序列 + 预设样式配置。

| 模板 ID | 说明 |
|---------|------|
| `academic_journal` | 期刊 Highlights：封面 + 目录 + 论文列表 + 征稿 + 亮点 + 横幅 |
| `academic_empty` | 期刊 Highlights（空白版）：同上结构，内容为空 |
| `quarterly_journal` | 季刊完整版：封面 + 寄语 + 论文 + 研究 + 新闻 + 会议 + 编委会 |
| `quarterly_empty` | 季刊空白版：同上结构，内容为空 |
| `general` | 通用简报：标题 + 图文 + CTA + 页脚 |
| `ecommerce` | 商品推广：标题 + CTA + 产品列表 + CTA + 页脚 |
| `education` | 课程介绍：标题 + 双图文卡片 + CTA + 页脚 |
| `saas` | 产品更新：标题 + 双图文 + 分隔线 + CTA + 页脚 |
| `festival` | 节日活动：标题 + CTA + 图文 + 社交 + 页脚 |
| `skeleton` | 纯骨架：空画布 |

### 自定义模板

- 可将当前设计保存为个人模板，存储于 `localStorage`
- 支持命名、加载、删除

## 样式系统

### 主题色

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `primary` | 主色 | `#667eea` |
| `bg` | 背景色 | `#ffffff` |
| `text` | 文字色 | `#1f2937` |
| `accent2` | 副色（用于和谐配色生成） | — |

**预设配色方案**（共 12 套）：科技蓝 · 电商红 · 教育绿 · 节日橙 · SaaS蓝 · 创意粉 · 靛蓝 · 青绿 · 橙色 · 紫罗兰 · 青色 · 草绿

### 字体栈

| 标识 | 字体栈 |
|------|--------|
| `system` | `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif` |
| `pingfang` | 苹方优先 |
| `microsoft` | 微软雅黑优先 |
| `serif` | Georgia 衬线 |
| `mono` | Consolas 等宽 |

### 间距与网格

| 参数 | 默认值 | 范围 |
|------|--------|------|
| 容器宽度 | `600px` | 480–680px |
| 区块间距 | `24px` | 8–64px（8px 倍数） |
| 内边距 | `24px` | 8–48px |
| 网格基准 | `8px` | 8 / 12 / 16px |

### 输出格式

- 所有组件渲染为语义化 HTML（`<section>`、`<article>`、`<h1>`—`<h3>`、`<p>` 等）
- 样式内联，可直接嵌入期刊网站 CMS 系统（如 MDPI 自有平台、WordPress 等）
- 默认 600px 内容宽度，适配期刊官网常见内容栏宽度（480–680px 可调）
- 导出 HTML 可通过 GitHub Pages / 期刊网站后台 / `<iframe>` 嵌入三种方式发布

## 导出与持久化

### 导出操作

| 操作 | 说明 |
|------|------|
| **导出 HTML** | 生成内联 CSS + 语义化 HTML 页面，可直接嵌入期刊网站或部署到 GitHub Pages |
| **复制代码** | 复制完整 HTML 到剪贴板 |
| **格式化代码** | 简易 HTML 格式化（代码模式下） |

### 数据持久化

| 机制 | 说明 |
|------|------|
| **设计内容** | 浏览器 `localStorage`，自动保存间隔 5 秒；历史撤销支持多步回退 |
| **自定义模板** | `localStorage` key `newsletter-custom-templates`，永久保留直到清缓存 |
| **自定义区块** | `localStorage`，与设计内容共享配额（~5MB）|
| **版本历史帧**（1.1.0 新增） | 浏览器 `IndexedDB` database `newsletter-history-db`，独立配额几十至几百 MB；最多 50 帧自动淘汰 |

**数据范围**：组件序列、主题配色、字体配置、间距配置、自定义模板、自定义区块、版本历史帧

**注意**：localStorage / IndexedDB 均为浏览器本地存储，不同设备、不同浏览器各自独立，不跨设备同步。清除浏览器数据会丢失；建议通过"导出 HTML"做异地备份。

## 常见问题

**Q: 数据存储在哪里？**  
A: 所有数据存储在浏览器本地：设计内容、自定义模板、自定义区块在 `localStorage`（~5MB，清缓存会丢失）；版本历史帧在 `IndexedDB`（独立配额几十 MB 级）。建议定期用"导出 HTML"做异地备份。

**Q: 我的自定义模板/历史能跨设备同步吗？**  
A: **不能**。`localStorage` 与 `IndexedDB` 均为浏览器本地，不同设备、不同浏览器各自独立。GitHub Pages 仅托管单文件静态资源。

**Q: 暗色网站模式预览 toggle 切回去后编辑器变暗了？**  
A: 不应该。`darkPreviewToggle` 只对 `.canvas-frame` 加 `.dark-preview-active` class；编辑器外壳由 `document.documentElement.data-theme` 控制，两者完全独立。如出现请刷新页面。

**Q: WCAG-AA 审计里出现大量 `unknown · div "⋮"` 失败项怎么处理？**  
A: 那是组件内部装饰性元素（如拖拽 handle `⋮`），导出 HTML 时不会出现。重点关注带内容的组件（`text`、`image-text`、`heading`），忽略 `div "⋮"` 这类。

**Q: 版本历史里找不到刚才那个版本？**  
A: 1. 只有调用 `saveHistory`（如工具栏 undo/redo、save）才会入帧，纯拖拽不触发；2. 超过 50 帧后最旧帧被自动淘汰；3. 若 IndexedDB 配额错误会 console.warn，但内存 undo/redo 仍工作。

**Q: 支持导入已有的 HTML 吗？**  
A: 代码模式下可直接粘贴 HTML，设计模式暂不支持从 HTML 反向解析。

## 贡献指南

欢迎为 Newsletter Builder 贡献代码、提交 Issue 或提出建议。

### 提报 Issue

- 使用 [GitHub Issues](https://github.com/CarlosZhou6767/Newsletter-HTML/issues) 提交 bug 报告或功能请求
- 描述问题时请提供：
  - 浏览器型号与版本
  - 复现步骤
  - 预期行为与实际表现
  - 控制台错误信息（如有）

### 提交 Pull Request

1. Fork 本仓库
2. 创建特性分支：
   ```bash
   git checkout -b feature/your-feature-name
   ```
3. 提交更改：
   ```bash
   git commit -m 'feat: add some feature'
   ```
4. 推送分支：
   ```bash
   git push origin feature/your-feature-name
   ```
5. 发起 Pull Request 至 `main` 分支

### 开发须知

- 本项目为单文件架构，所有核心代码位于 [`index.html`](index.html) 中
- 修改时请确保不引入外部依赖
- 导出的 HTML 应保持语义化标签，方便嵌入期刊 CMS
- 使用 `escapeHtml()` 处理用户输入文本

### 代码规范

- JavaScript 使用 ES6+ 语法
- CSS 变量优先，避免硬编码色值
- 组件渲染函数返回 HTML 字符串，不直接操作 DOM

## 许可证

本项目基于 MIT 许可证开源。

```
MIT License

Copyright (c) 2026 Newsletter-HTML

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

## 联系方式

- **项目主页**：[GitHub - CarlosZhou6767/Newsletter-HTML](https://github.com/CarlosZhou6767/Newsletter-HTML)
- **提交 Issue**：[GitHub Issues](https://github.com/CarlosZhou6767/Newsletter-HTML/issues)
- **用户指南**：[docs/USER_GUIDE.md](docs/USER_GUIDE.md)（三项新功能详细说明）
- **设计文档**：[docs/superpowers/specs/2026-07-03-builder-features-2-3-4-design.md](docs/superpowers/specs/2026-07-03-builder-features-2-3-4-design.md)
- **实施计划**：[docs/superpowers/plans/2026-07-03-builder-features-2-3-4.md](docs/superpowers/plans/2026-07-03-builder-features-2-3-4.md)
- **作者**：Carlos Zhou

---

<div align="center">

**如果这个项目对你有帮助，欢迎 Star ⭐ 支持！**

</div>
