# Newsletter Builder 使用说明

> 本文介绍新增三项功能：**暗色邮件客户端预览**、**WCAG-AA 无障碍审计**、**版本历史时间线**。这三项位于编辑器左侧"检查"页面，全部在浏览器本地运行，无需联网、无需账号。

---

## 一、快速上手

### 打开"检查"页

1. 在 [index.html](file:///e:/MyProject/Newsletter-HTML/index.html)（部署在 GitHub Pages 或本地静态服务器）中进入设计模式
2. 左侧栏点击 **检查** 按钮（`setAssistantTab('inspect')`）

你会在"检查"页顶部看到三个新元素：

- 邮件暗色客户端预览（复选框）
- WCAG-AA 审计（按钮）
- 底部新增一个 **版本历史** section

---

## 二、功能 #2：邮件暗色客户端预览

### 用途

很多收件人（尤其学术圈用户）的邮件客户端默认开暗色：Gmail Dark、iOS Mail Dark、Outlook Dark。如果导出的 HTML 邮件没有 `prefers-color-scheme` 适配，CTA 按钮可能变黑、边框消失、正文背景变深但文字仍是深色——视觉瞬间崩坏。

这个功能做两件事：

1. **预览**：在编辑器里一键切换，看见当前 newsletter 在暗色客户端下的真实样子
2. **导出保护**：导出 HTML 时自动注入 `@media (prefers-color-scheme: dark){...}`，让收件客户端自动套用

### 怎么用

1. 在"检查"页，勾选 **邮件暗色客户端预览** 复选框
2. 中央画布立即套用一组固定暗色变量：`--bg-primary: #0f1218`、`--text-primary: #e8e9ec` 等
3. 浏览 newsletter 每个组件，确认 CTA、链接、边框、按钮不会失色
4. 取消勾选，回到白底预览
5. 确认无误后，点工具栏的 **导出 HTML** —— 导出的 `newsletter.html` 已包含 `prefers-color-scheme: dark` 媒体查询，用户客户端暗色时自动切换

### 注意

- 此 toggle 不影响编辑器本身的暗/亮主题（那由顶部 `data-theme` 控制，是给编辑者的视觉偏好）
- 这里的 toggle 是预览导出端的暗色，独立于编辑器主题
- 暗色配色是固定的学术编辑调（取自 MDPI 暗色风格），目前不支持自定义

### 调色板（供参考）

导出时注入的变量：

| 变量 | 暗色值 |
|---|---|
| `--bg-primary` | `#0f1218` |
| `--bg-secondary` | `#161922` |
| `--bg-tertiary` | `#1c1f29` |
| `--bg-quaternary` | `#252a36` |
| `--text-primary` | `#e8e9ec` |
| `--text-secondary` | `#a0aec0` |
| `--text-muted` | `#6b7280` |
| `--border` | `#2a2e3a` |
| `--border-subtle` | `#22262f` |

---

## 三、功能 #3：WCAG-AA 无障碍审计

### 用途

学术出版对 a11y 要求逐年收紧。旧版"检查"页只算 `theme.bg` vs `theme.text` 一对对比度，组件嵌套（卡片背景 vs 卡片内正文、CTA 按钮 vs 按钮文字）从不被校验。这个功能：

- 遍历画布上每个组件
- 用 `TreeWalker` 找出所有文字叶子节点
- 算每对（背景色，文字色）的对比度
- 按 WCAG 2.1 AA / AAA 评级
- 给出可操作的修复建议（如"把文字色调至 #6c7989 可达 4.5:1"）

### 怎么用

1. 加载或编辑 newsletter 完成后，切到"检查"页
2. 点击 **WCAG-AA 审计** 按钮
3. 下方 `#a11yReport` 区域出现报告：
   - **顶部汇总**："检查 170 项，通过 5 项，失败 165 项"
   - **AA 不达标（必须修复）**：红色标题块，列出所有 < 4.5:1（正文）/ < 3:1（大字）的违规
   - **AAA 不达标（建议修复）**：橙色标题块，列出 < 7:1（正文）/ < 4.5:1（大字）的违规
4. 每条违规包含：组件类型 + 文字预览 + 实测对比度 + 需要值 + 是否大字 + 修复建议
   - 例：`image-text · div "Interaction Regularity o…" 对比度 1.89:1 / 需要 3:1（大字） 建议手动调整颜色满足 3:1`
   - 若算法找到可修复色值：`把文字色调至 #6C7989 可达 4.5:1`
   - 若算法找不到（极端情况）：给出 `建议手动调整颜色满足 4.5:1`

### 评级标准

| 类型 | AA 要求 | AAA 要求 |
|---|---|---|
| 正文文字（< 18px 且非粗体） | ≥ 4.5:1 | ≥ 7:1 |
| 大字（≥ 18px 或 ≥ 14px 加粗） | ≥ 3:1 | ≥ 4.5:1 |

### 怎么修

1. 在报告里看到违规的**组件类型**（如 `image-text`、`heading`、`cta`）
2. 回到画布选中对应组件
3. 在右侧属性面板修改文案颜色 / 卡片背景，目标对比度值已在建议里给出
4. 修改后再点一次 **WCAG-AA 审计** 复检，直到全部通过

### 注意

- 审计只校验文字对比度，不校验 alt 文本/表单 label（那已在其它 preflight 项里覆盖）
- 审计读的是 `getComputedStyle`，所以编辑器的暗/亮模式会影响结果。建议在明色模式（编辑器默认）下做审计，以确保导出明色邮件合规
- 审计不修改任何代码，只读，安全

---

## 四、功能 #4：版本历史时间线

### 用途

旧版 `saveHistory` 只维护一个内存栈，支持 `undo`/`redo`，但刷新页面就清空，也看不到"刚才那个版本长什么样"。这个功能：

- 每次编辑 (`saveHistory`) 都自动保存一帧到浏览器 IndexedDB
- 每帧含完整快照（components + theme + font + spacing）+ SVG 抽象缩略图 + 时间戳
- 可浏览时间线、点击任意帧恢复、删除单帧、清空全部
- 刷新页面、关闭浏览器再回来，历史仍在

### 怎么用

#### 查看历史

1. 编辑 newsletter（每次 `saveHistory` 触发都会记录一帧）
2. 切到"检查"页，滚动到底部 **版本历史** section
3. 你会看到一串垂直列表，每行包含：
   - 左侧：56×40 SVG 缩略图（抽象示意图：每个组件画成一个矩形条，CTA 蓝、图片金色、标题深蓝、其它米色）
   - 中间：时间戳（月/日 时:分）+ 组件数量
   - 右侧：删除按钮（× 鼠标 hover 时出现）

#### 恢复到某一版本

- 点击任意一行的任意位置（除删除按钮外）→ 当前画布立即被该帧的 `components/theme/font/spacing` 覆盖
- 顶部 toast 提示："已回到 10:30"
- 注意：恢复后仍可 undo 回到更早状态（内存栈仍工作）；但若想保存这一恢复点为新版本，会需要再触发一次 `saveHistory`

#### 删除单帧

- 鼠标 hover 该行，右侧出现 × 按钮
- 点击 × → 该帧从 IndexedDB 删除，列表刷新
- toast 提示："已删除"

#### 清空全部历史

- 在"版本历史" section 标题右侧有一个 🗑 按钮
- 点击 → 弹出 confirm："确定清空全部历史？此操作不可撤销。"
- 确认后清空 IndexedDB 的 `frames` store，列表变空，提示："历史已清空"

### 保留策略

- 最多保留 **50 帧**
- 超出时按时间排序淘汰最旧的，自动保留最近 50 帧
- 超出后不会阻塞编辑，淘汰是异步执行的

### 存储位置

- 浏览器 IndexedDB：database 名 `newsletter-history-db`，object store 名 `frames`
- 每帧主键 `id` 形如 `frame_1688765432000_x7y2k`，含时间戳防碰撞
- 配额耗尽会 console.warn，但**不会阻塞**编辑器；内存 undo/redo 仍正常工作

### 缩略图说明

缩略图是抽象 SVG schematic，**不是真实截图**。每个组件画成一个矩形条：

| 组件类型 | 缩略图中的颜色 |
|---|---|
| `cta`（按钮/CTA） | 主题强调色（accent） |
| `image`、`image-text`、`gallery` | 主题金色透明 |
| `heading`、`title` | 主题 accent 实色 |
| 其它 | 米灰 `#d8d6ce` |

矩形宽度按该组件的 `title` 或 `text` 长度估算。

这样设计的好处：零依赖、零延迟、与真实视觉有结构对应关系。当你"想回去"时，看缩略图就能辨别"哦，那时 CTA 在最下面、那时缺了 1 张图"。

---

## 五、与现有功能的关系

| 现有功能 | 三项新功能的影响 |
|---|---|
| **导出 HTML**（顶部"导出"按钮） | 自动注入 `prefers-color-scheme: dark` 到导出的 HTML 头部，但只在导出端；不影响预览端、不影响设计视图。导出文件大小约 +400 字节 |
| **undo/redo**（工具栏箭头） | 完全保留，新版用 IndexedDB 做镜像，不替换原内存栈。`saveHistory` 仍 push 进内存 `history` 数组 |
| **代码模式** | 不受影响。`exportHTML` 在 code 模式下从 `#codeEditor.value` 提取，注入路径一致 |
| **7 个内置模板** | 模板加载不会创建历史帧（只有用户编辑触发 `saveHistory` 才会） |
| **左侧 4 个 tab** | 不受影响。三项新功能全部位于 inspect tab 内 |
| **状态栏** | 不受影响 |

---

## 六、隐藏数据看一眼

如果想确认 IndexedDB 里存了什么，打开浏览器 DevTools → Application → IndexedDB → `newsletter-history-db` → `frames`。每帧字段：

```javascript
{
  id: 'frame_1688765432000_x7y2k',
  ts: 1688765432000,           // Unix ms
  thumbnail: 'data:image/svg+xml;base64,...',
  snapshot: {
    components: [...],          // 完整组件数组深拷贝
    theme: {...},               // theme 对象浅拷贝
    font: {...},
    spacing: {...}
  }
}
```

删除所有数据：DevTools → Application → IndexedDB → 右键 `newsletter-history-db` → Delete database。或在编辑器"检查"页点 🗑 清空。

---

## 七、常见问题

### Q：审计报告里出现大量 `unknown · div "⋮"` 失败项？
A：那是某些组件内部的装饰性元素（如分隔符 `⋮` ）。修复指引：

1. 看违规组件类型是否真需要可见文字（如 `⋮` 是编辑模式的拖拽 handle，导出时不会出现）
2. 真正关心的是带内容的组件（`text`、`image-text`、`heading`），忽略 `div "⋮"` 这类
3. 或者点"删除这些 handle 组件"——但当前版本不提供这个开关，属于后续优化范围

### Q：暗色预览切回去后，编辑器本身变暗了？
A：不应该。`darkPreviewToggle` 只给 `.canvas-frame` 加 `.dark-preview-active` class，编辑器外壳 chrome 由 `document.documentElement.data-theme` 控制，两者完全独立。如出现这个现象请关闭页面后重新打开。

### Q：历史里找不到刚才那个版本？
A：检查一下：
1. 是否实时编辑过——只有调用 `saveHistory` 才会入帧。点工具栏 undo/redo、或工具栏 save 才触发；纯拖拽可能不触发
2. 是否刷新过页面——刷新不会丢历史，但等到编辑器 init 完成自动调一次 `renderHistoryTimeline`；如果 IndexedDB 配额错误写入失败会 console.warn，打开 DevTools 看
3. 是否超过 50 帧后老帧被自动淘汰

### Q：能跨设备同步历史吗？
A：**不能**。IndexedDB 是浏览器本地存储，不同设备/不同浏览器各自独立。这是纯静态站点的设计取舍。如需同步需引入后端，那属于另一个项目（第 6 项"协作评论锚点"在考虑范围）。

### Q：审计通过了 AAA，还需要做什么？
A：很好，但要注意 WCAG 还有图像对比度、表单 label 等其他维度，本工具只校验文字对比度。出版前请配合浏览器 Lighthouse 审计做最终检查。

---

## 八、参考链接

- [WCAG 2.1 对比度指南](https://www.w3.org/WAI/WCAG21/Understanding/contrast-minimum.html)
- [prefers-color-scheme MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme)
- [IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
