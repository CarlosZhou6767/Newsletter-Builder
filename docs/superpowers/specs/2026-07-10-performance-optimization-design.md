# 性能优化设计文档：组件级 Diff 渲染引擎

**日期**: 2026-07-10
**项目**: Newsletter Builder
**版本**: V0.1.0 -> V0.2.0
**目标场景**: 10-20 个组件的中等规模简报
**改动范围**: 深度重构（手写轻量 diff，不引入外部依赖）

---

## 1. 问题分析

### 1.1 当前性能瓶颈

| 瓶颈 | 位置 | 触发条件 | 影响 |
|------|------|----------|------|
| 全量 innerHTML 重建 | `renderCanvas()` L4659 | 每次 oninput、滑块拖动 | 20 组件时每次击键重建整个画布 DOM |
| 属性编辑无防抖 | `updateComponentProp()` L4892 | oninput 每次击键 | 每次击键都触发 saveHistory + renderCanvas |
| 滑块操作无防抖 | `updateThemeColor()` 等 10 个函数 L4139-4272 | 拖动滑块 | 连续触发全量 renderCanvas |
| 历史快照过频 | `saveHistory()` L5770 | 每次编辑 | 每次 structuredClone + SVG 缩略图 + IndexedDB 写入 |
| 事件监听器重建 | `renderCanvas()` L4673-4684 | 每次渲染 | 20 组件 = 120 个监听器全部重建 |

### 1.2 热路径分析

用户在设计模式编辑组件属性时，每次击键的调用链：

```
oninput -> updateComponentProp (L4892)
  -> comp.props[key] = val
  -> saveHistory() (L5770)
    -> structuredClone({ components, theme, font, spacing })  // 深拷贝全部状态
    -> scheduleIdle -> saveFrameToDB
      -> generateSVGThumbnail()  // 遍历所有组件生成 SVG
      -> IndexedDB write
      -> renderHistoryTimeline()  // 重建历史列表 DOM
  -> renderCanvas() (L4659)
    -> canvas.innerHTML = components.map(c => renderComponent(c)).join('')  // 全量重建
    -> 120 个事件监听器重建
```

20 个组件场景下，单次击键需要：1 次深拷贝 + 1 次 SVG 生成 + 1 次 IndexedDB 写入 + 1 次历史列表重建 + 1 次画布全量重建 + 120 个事件重绑。

---

## 2. 架构总览

### 2.1 优化子系统

| 子系统 | 当前问题 | 优化方案 |
|--------|----------|----------|
| 画布渲染 `renderCanvas` | 全量 innerHTML + 事件重绑 | 组件级 diff：对比新旧组件列表，只增删改变化的节点 |
| 属性编辑 `updateComponentProp` | oninput -> saveHistory + renderCanvas | 即时局部更新当前组件 DOM（patchComponent）+ 防抖 500ms 历史保存 |
| 滑块操作 `updateThemeColor` 等 | oninput -> renderCanvas 全量 | 防抖 100ms，用 requestAnimationFrame 合并 |
| 历史快照 `saveHistory` | 每次编辑都 structuredClone + SVG + IndexedDB | 节流 500ms 合并连续编辑，SVG 缩略图延迟到 idle |
| 事件绑定 | 每组件 4 个 addEventListener | 事件委托到 canvasContent，一次绑定永久有效 |

### 2.2 核心原则

- 不引入外部依赖，手写 diff 算法
- diff 粒度为组件级（`wrap-{id}` 节点），组件内部仍用 innerHTML 替换
- 导出/预览路径也走统一渲染函数，但不做 diff（直接生成 HTML 字符串）

### 2.3 渲染路径统一

```
renderComponentHTML(c)     ← 组件级 HTML 生成（共用基础，不变）
        ↓
renderComponent(c)         ← 包装层（加选中态、工具栏、data-component-id，仅设计模式用）
        ↓
    ┌─────┴─────┐
    ↓           ↓
patchCanvas()  generateHTML()
(diff渲染)      (完整HTML字符串)
    ↓           ↓
设计模式画布   预览/导出
```

- `renderCanvas()` 内部改用 `patchCanvas()` diff 引擎
- `generateHTML()` 保持不变，仍调用 `renderComponentHTML()`
- `updatePreviewMode()` 和 `exportHTML()` 保持不变
- 新增 `initCanvasEvents()` 在启动时调用一次

---

## 3. 组件级 Diff 渲染引擎

### 3.1 核心算法

`patchCanvas()` 从全局 `components` 数组和当前 DOM 状态对比，生成 DOM 操作补丁。

```
算法流程：
1. 从当前 DOM 构建旧节点映射：oldMap = { id -> DOMNode }（通过 querySelectorAll('[data-component-id]')）
2. 遍历新组件数组（全局 components）：
   a. 若 id 在 oldMap 中存在 -> 检查签名是否变化（对比 _lastSignatures[id]）
      - 变化 -> 替换该 wrap-{id} 节点的 outerHTML（局部更新）
      - 未变 -> 跳过
   b. 若 id 不在 oldMap 中 -> 创建新节点，插入到正确位置
3. 删除 oldMap 中不再存在的节点
4. 同步位置：用 insertBefore 调整 DOM 顺序与 components 数组一致
5. 更新选中态 class（不重建 DOM）
6. 重建 domCache 和 _lastSignatures
```

### 3.2 关键实现细节

**DOM 节点缓存**：`renderCanvas` 首次渲染后维护 `domCache = { id -> HTMLElement }`，后续渲染从缓存读取。

**props 变化检测**：每个组件维护一个轻量签名 `signature = JSON.stringify(c.props) + c.type`，只有签名变化才更新该组件 DOM。维护 `_lastSignatures = { id -> signature }` 映射。

**位置调整**：组件顺序变化时，用 `insertBefore` 移动现有 DOM 节点，不重建。

**组件内部更新**：当某组件 props 变化时，只执行：
```javascript
const node = domCache[id];
node.outerHTML = renderComponent(c); // 替换单个组件节点
// 事件委托后无需重新绑定事件
```

**全量回退**：当组件数量变化超过 50%（如加载模板、清空画布）时，回退到全量 innerHTML 重建，避免 diff 开销大于重建。判定条件：`Math.abs(newLen - oldLen) / Math.max(newLen, oldLen, 1) > 0.5`。

**签名缓存清理**：组件被删除时，同步清理 `_lastSignatures[id]` 和 `domCache[id]`。

**全量回退后缓存失效**：当触发全量回退（`renderCanvasLegacy()`）时，`innerHTML` 替换会销毁所有旧 DOM 节点，因此 `domCache` 和 `_lastSignatures` 必须在回退后重建。`renderCanvasLegacy()` 末尾需重新遍历 DOM 构建 `domCache` 和 `_lastSignatures`。

**两条独立更新路径**：
- `patchComponent(id)` - 属性编辑热路径，只更新单个组件 DOM，不触碰 `domCache`（直接用 `getElementById`）
- `patchCanvas()` - 通用画布渲染，diff 全部组件，维护 `domCache` 和 `_lastSignatures`

两者互不依赖。`patchComponent` 执行后，`_lastSignatures[id]` 需同步更新以避免下次 `patchCanvas` 重复更新该组件。

### 3.3 renderComponent 改造

每个组件包装层添加 `data-component-id` 属性，供事件委托定位：

```javascript
function renderComponent(c) {
  const isSelected = selectedComponentId === c.id;
  return `<div class="canvas-component-wrapper${isSelected ? ' selected' : ''}"
    id="wrap-${c.id}"
    data-component-id="${c.id}">
    ${renderComponentHTML(c)}
    <div class="canvas-component-toolbar">...</div>
  </div>`;
}
```

---

## 4. 属性编辑防抖与局部更新

### 4.1 三阶段更新

```
阶段 1：即时数据更新（0ms）
  - comp.props[key] = val  ← 立即更新内存数据
  - 局部更新当前组件 DOM（只替换 wrap-{id} 的 innerHTML）

阶段 2：防抖历史保存（500ms）
  - debounce(saveHistory, 500ms)
  - 连续编辑合并为一次历史快照

阶段 3：防抖 localStorage 保存（5000ms，已有）
  - scheduleSave() 保持不变
```

### 4.2 具体实现

```javascript
// 新增：局部更新单个组件
function patchComponent(id) {
  const comp = components.find(c => c.id === id);
  const node = document.getElementById('wrap-' + id);
  if (!comp || !node) return;
  // 只替换组件内部 HTML，保留选中态
  const isSelected = node.classList.contains('selected');
  node.outerHTML = renderComponent(comp);
  if (isSelected) {
    document.getElementById('wrap-' + id)?.classList.add('selected');
  }
  // 更新签名缓存
  _lastSignatures[id] = JSON.stringify(comp.props) + comp.type;
}

// 修改：updateComponentProp
const _debouncedSaveHistory = debounce(saveHistory, 500);
function updateComponentProp(id, key, val, e) {
  const comp = components.find(c => c.id === id);
  if (!comp) return;
  comp.props[key] = val;
  const input = e && e.target || event && event.target;
  if (input) {
    validateInput(input, true, '');
  }
  // 局部更新，不触发全量 renderCanvas
  patchComponent(id);
  _debouncedSaveHistory();
}
```

### 4.3 焦点保持

`patchComponent` 只更新画布 DOM，不触碰属性面板 DOM（`buildPropertyEditor` 的结果保持不变），因此输入框焦点不会丢失。

### 4.4 选中态保持

更新前检查 `node.classList.contains('selected')`，更新后恢复。由于事件委托方案下事件绑定在容器上，不需要重新绑定组件事件。

---

## 5. 滑块操作防抖

### 5.1 受影响函数

10 个函数的 `oninput` 直接调用 `renderCanvas()`：
- `updateThemeColor` L4139
- `updateBgColor` L4151
- `updateTextColor` L4163
- `updateHeadingSize` L4227
- `updateFontWeight` L4233
- `updateBodySize` L4239
- `updateLineHeight` L4245
- `updateContainerWidth` L4252
- `updateSectionGap` L4260
- `updatePadding` L4266

### 5.2 优化方案

```javascript
// 新增：防抖画布渲染
const _debouncedRender = debounce(() => renderCanvas(), 100);

// 修改每个滑块函数，将 renderCanvas() 替换为 _debouncedRender()
// 但 UI 控件值（如滑块数值显示）仍即时更新
function updateThemeColor(v) {
  // ...验证逻辑不变...
  theme.primary = v;
  document.getElementById('primaryColorPicker').value = v;  // 即时
  input.value = v;                                            // 即时
  theme.accent2 = adjustColor(v, -20);
  _debouncedRender();  // 防抖渲染（原 renderCanvas）
  checkContrast();      // 即时（轻量）
}
```

10 个函数统一改造：直接调用 `renderCanvas()` 的位置全部替换为 `_debouncedRender()`。UI 控件同步（色块、数值标签）保持即时更新，因为这些操作轻量。

---

## 6. 历史快照节流

### 6.1 当前问题

`saveHistory()`（L5770）每次调用都执行：
- `structuredClone({ components, theme, font, spacing })` - 深拷贝
- `generateSVGThumbnail()` - 遍历所有组件生成 SVG + base64 编码
- `saveFrameToDB()` - IndexedDB 异步写入

在 `oninput` 场景下，连续输入 10 个字符 = 10 次完整快照。

### 6.2 节流方案

引入最小快照间隔，合并连续编辑：

```javascript
const HISTORY_MIN_INTERVAL = 500; // ms
let _lastHistoryTime = 0;
let _pendingHistoryTimer = null;

function saveHistory(opts = {}) {
  // 强制保存（如加载模板、清空画布等非连续操作）
  if (opts.force) {
    clearTimeout(_pendingHistoryTimer);
    _doSaveHistory();
    return;
  }
  // 节流：500ms 内的连续调用合并为一次
  const now = Date.now();
  if (now - _lastHistoryTime < HISTORY_MIN_INTERVAL) {
    clearTimeout(_pendingHistoryTimer);
    _pendingHistoryTimer = setTimeout(_doSaveHistory, HISTORY_MIN_INTERVAL);
    return;
  }
  _doSaveHistory();
}

function _doSaveHistory() {
  _lastHistoryTime = Date.now();
  // 内存快照（structuredClone 保留）
  const state = structuredClone({ components, theme, font, spacing });
  if (historyIndex < history.length - 1) history = history.slice(0, historyIndex + 1);
  history.push(state);
  if (history.length > HISTORY_LIMIT) history.shift();
  historyIndex = history.length - 1;
  hasUnsavedChanges = true;
  scheduleSave();
  // IndexedDB 帧写入：SVG 缩略图延迟到 idle
  scheduleIdle(() => {
    saveFrameToDB({
      id: 'frame_' + Date.now() + '_' + Math.random().toString(36).slice(2, 7),
      ts: Date.now(),
      thumbnail: generateSVGThumbnail(), // idle 时生成
      snapshot: {
        components: JSON.parse(JSON.stringify(components)),
        theme: Object.assign({}, theme),
        font: Object.assign({}, font),
        spacing: Object.assign({}, spacing)
      }
    }).then(() => pruneOldFrames())
      .then(() => renderHistoryTimeline())
      .catch(e => console.warn('history frame save failed:', e));
  });
}
```

### 6.3 强制保存场景

以下离散操作调用 `saveHistory({ force: true })`，立即记录历史：

- `loadTemplate` - 加载模板
- `clearCanvas` - 清空画布
- `deleteComponent` - 删除组件
- `duplicateComponent` - 复制组件
- `moveUp` / `moveDown` - 移动组件
- `restoreFrame` - 恢复历史帧
- `onDrop` - 拖拽排序（画布内排序和库组件插入）
- undo / redo 操作

只有 `updateComponentProp` 的连续输入使用节流合并（通过 `_debouncedSaveHistory`）。

### 6.4 与属性编辑防抖的协作

```
updateComponentProp (oninput)
  -> patchComponent(id)          // 即时局部更新
  -> _debouncedSaveHistory()     // debounce 500ms
       -> saveHistory()           // 500ms 后触发
            -> 节流检查（已在 500ms 外，直接执行）
            -> _doSaveHistory()
```

由于 `_debouncedSaveHistory` 本身已延迟 500ms，`saveHistory` 的节流检查通常直接通过。两者协同确保：连续输入时只产生一次历史快照。

---

## 7. 事件委托优化

### 7.1 当前问题

`renderCanvas()` 每次执行后，对每个组件绑定 4 个事件监听器（L4673-4684），20 个组件 = 120 个事件监听器，每次渲染全部重建。

### 7.2 事件委托方案

事件委托到 `canvasContent` 容器，一次绑定永久有效：

```javascript
// 新增：初始化时绑定一次（在 init 或 DOMContentLoaded 中）
function initCanvasEvents() {
  const canvas = document.getElementById('canvasContent');

  // 点击选中/取消选中
  canvas.addEventListener('click', (e) => {
    const wrapper = e.target.closest('.canvas-component-wrapper');
    if (!wrapper) {
      // 点击空白区域，取消选中
      selectedComponentId = null;
      updateSelectionState();
      selectComponent(null);
      return;
    }
    e.stopPropagation();
    const id = wrapper.dataset.componentId;
    selectComponent(id);
  });

  // 拖拽：事件委托
  canvas.addEventListener('dragstart', (e) => {
    const wrapper = e.target.closest('.canvas-component-wrapper');
    if (wrapper) onDragStart(e, wrapper.dataset.componentId);
  });
  canvas.addEventListener('dragend', onDragEnd);
  canvas.addEventListener('dragover', (e) => {
    const wrapper = e.target.closest('.canvas-component-wrapper');
    if (wrapper) onDragOver(e, wrapper.dataset.componentId);
    else e.preventDefault(); // 允许拖到末尾
  });
  canvas.addEventListener('drop', (e) => {
    const wrapper = e.target.closest('.canvas-component-wrapper');
    if (wrapper) onDrop(e, wrapper.dataset.componentId);
    else onDropToEnd(e);
  });
  canvas.addEventListener('dragenter', (e) => e.preventDefault());
}
```

### 7.3 选中态轻量更新

新增 `updateSelectionState()` 函数，只切换 class，不重建 DOM：

```javascript
function updateSelectionState() {
  document.querySelectorAll('.canvas-component-wrapper.selected')
    .forEach(el => el.classList.remove('selected'));
  if (selectedComponentId) {
    const el = document.getElementById('wrap-' + selectedComponentId);
    if (el) el.classList.add('selected');
  }
}
```

### 7.4 selectComponent 改造

`selectComponent` 调用 `updateSelectionState()` 替代全量 `renderCanvas()`：

```javascript
function selectComponent(id) {
  selectedComponentId = id;
  updateSelectionState();  // 轻量更新选中态 class
  const comp = id ? components.find(c => c.id === id) : null;
  // 显示/隐藏属性面板
  const empty = document.getElementById('propertiesEmpty');
  const body = document.getElementById('propertiesBody');
  if (!comp) {
    if (empty) empty.style.display = '';
    if (body) body.style.display = 'none';
    return;
  }
  if (empty) empty.style.display = 'none';
  if (body) body.style.display = '';
  buildPropertyEditor(comp);
}
```

### 7.5 收益

- 事件监听器从 120 个降至 6 个
- diff 渲染时无需重新绑定事件
- 选中态切换从 `renderCanvas()` 降级为 `updateSelectionState()`（O(1)）

---

## 8. 测试验证方案

### 8.1 手动验证矩阵

无单元测试框架，采用手动验证 + 关键路径检查：

| 验证场景 | 验证点 | 预期 |
|----------|--------|------|
| 属性编辑连续输入 | 画布是否卡顿 | 20 组件下输入流畅 |
| 属性编辑焦点 | 输入框焦点是否丢失 | 焦点保持 |
| 颜色滑块拖动 | 画布更新是否跟手 | 100ms 内更新，无明显延迟 |
| 组件拖拽排序 | 拖放是否正常 | 拖放位置正确，选中态保持 |
| 加载模板 | 组件是否完整渲染 | 全量回退路径正常 |
| 撤销/重做 | 历史是否正确 | 连续编辑合并后撤销跳回正确状态 |
| 历史时间线 | IndexedDB 帧是否记录 | 节流后仍记录，不过度 |
| 选中态切换 | class 是否正确更新 | 选中/取消选中视觉正确 |
| 导出 HTML | 导出内容是否正确 | 与优化前一致 |
| 预览模式 | 预览是否正常 | 与优化前一致 |
| 清空画布 | 全量回退是否正常 | 画布清空，空状态提示显示 |
| 复制/删除组件 | diff 增删是否正常 | 组件正确增删，选中态正确 |
| 移动端上移/下移 | 按钮是否正常工作 | patchComponent 后按钮可用 |

### 8.2 性能基准

**优化前基线**（需在实施前测量）：
- 20 组件下属性编辑单次击键 -> `renderCanvas` 耗时（用 `performance.now()` 测量）
- 颜色滑块拖动 1 秒内的 `renderCanvas` 调用次数
- `saveHistory` 单次执行耗时

**优化后目标**：
- 属性编辑单次击键 -> `patchComponent` 耗时 < 基线的 30%
- 颜色滑块拖动 -> `renderCanvas` 调用次数降至原来的 1/10
- `saveHistory` 连续编辑合并率 > 80%

### 8.3 回退策略

如果 diff 引擎出现兼容性问题，保留全量渲染回退路径：

```javascript
const FORCE_FULL_RENDER = false; // 调试开关
function renderCanvas() {
  if (FORCE_FULL_RENDER) return renderCanvasLegacy();
  return patchCanvas();
}
```

---

## 9. 改动清单

### 9.1 涉及文件

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `index.html` | 修改 | 所有改动均在单文件内 |

### 9.2 涉及函数

| 函数 | 改动类型 | 说明 |
|------|----------|------|
| `renderCanvas()` | 重构 | 改用 diff 引擎，保留 legacy 回退 |
| `renderComponent(c)` | 修改 | 添加 `data-component-id` 属性 |
| `updateComponentProp()` | 重构 | 改用 `patchComponent` + 防抖 `saveHistory` |
| `saveHistory()` | 重构 | 增加节流逻辑，SVG 缩略图延迟 |
| 10 个滑块函数 | 修改 | `renderCanvas()` -> `_debouncedRender()` |
| `selectComponent()` | 重构 | 调用 `updateSelectionState()` 替代全量渲染 |
| `patchCanvas()` | 新增 | diff 渲染核心 |
| `patchComponent(id)` | 新增 | 单组件局部更新 |
| `updateSelectionState()` | 新增 | 选中态轻量更新 |
| `initCanvasEvents()` | 新增 | 事件委托初始化 |
| `_debouncedRender` | 新增 | 防抖渲染（debounce 100ms） |
| `_debouncedSaveHistory` | 新增 | 防抖历史保存（debounce 500ms） |
| `renderCanvasLegacy()` | 新增 | 原 renderCanvas 逻辑保留为回退 |
| init / DOMContentLoaded | 修改 | 调用 `initCanvasEvents()` |

### 9.3 不变函数

以下函数保持不变：
- `renderComponentHTML(c)` - 组件 HTML 生成
- `generateHTML()` - 完整 HTML 文档生成
- `updatePreviewMode()` - 预览模式渲染
- `exportHTML()` - 导出
- `loadTemplate()` - 模板加载
- `addComponent()` / `createComponent()` - 组件创建
- `moveUp()` / `moveDown()` / `duplicateComponent()` / `deleteComponent()` - 仅 `saveHistory` 调用改为 `{force: true}`
- `onDragStart()` / `onDragOver()` / `onDrop()` / `onDropToEnd()` - 拖拽逻辑不变，仅调用方式改为事件委托
- IndexedDB 操作函数 - 不变
- localStorage 操作函数 - 不变

---

## 10. 实施顺序

建议按以下顺序逐步实施，每步完成后验证：

1. **事件委托** - `initCanvasEvents()` + `renderComponent` 添加 `data-component-id` + `selectComponent` 改用 `updateSelectionState()`。验证：选中/拖拽/点击空白正常工作。
2. **Diff 渲染引擎** - `patchCanvas()` + 签名缓存 + 全量回退。验证：组件增删改、加载模板、清空画布正常。
3. **属性编辑局部更新** - `patchComponent()` + `updateComponentProp` 改造。验证：连续输入流畅、焦点保持、选中态正确。
4. **滑块防抖** - 10 个函数 `renderCanvas()` -> `_debouncedRender()`。验证：颜色/字号/间距调整流畅。
5. **历史快照节流** - `saveHistory` 节流 + 强制保存场景。验证：撤销/重做正确、历史时间线正常。
6. **性能基准测量** - 对比优化前后数据。
