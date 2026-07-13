# 性能优化：组件级 Diff 渲染引擎 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将设计模式画布从全量 innerHTML 重建改为组件级 diff + 局部更新，并对所有热路径加防抖/节流，解决 10-20 组件规模下的编辑卡顿问题。

**Architecture:** 手写组件级 diff 算法替代 innerHTML 全量重建；属性编辑改用单组件局部更新（patchComponent）；滑块操作加 100ms 防抖；历史快照加 500ms 节流合并连续编辑；画布事件改为容器级事件委托。所有改动在单文件 index.html 内完成，不引入外部依赖。

**Tech Stack:** 原生 JavaScript（ES6+）、HTML5 Drag and Drop API、localStorage、IndexedDB、requestAnimationFrame、requestIdleCallback

**Spec:** [docs/superpowers/specs/2026-07-10-performance-optimization-design.md](file:///e:/MyProject/Newsletter-Builder/docs/superpowers/specs/2026-07-10-performance-optimization-design.md)

**重要约定：**
- 本项目无测试框架，所有"验证"步骤为浏览器手动验证
- 现有 CSS 类名为 `canvas-component`（非 spec 中的 `canvas-component-wrapper`），计划沿用现有类名
- 现有 data 属性为 `data-id`（L4847 已存在），事件委托可直接使用，无需新增 `data-component-id`
- 所有行号基于当前 index.html（5555 行），修改后行号会偏移

---

## File Structure

| 文件 | 职责 |
|------|------|
| `index.html` | 唯一修改文件，所有改动在此 |

**改动区域分布：**
- L3916-3935: 状态变量区（新增 diff 相关变量）
- L4139-4272: 10 个滑块函数（renderCanvas -> _debouncedRender）
- L4657-4694: renderCanvas（重构为 diff 引擎）
- L4843-4848: renderComponent（确认 data-id 已存在）
- L4850-4864: selectComponent（改用 updateSelectionState）
- L4892-4902: updateComponentProp（改用 patchComponent + 防抖）
- L5770-5794: saveHistory（增加节流逻辑）
- L5879-5904: undoDesign/redoDesign（saveHistory 调用改为 force）
- L6014-6052: init（调用 initCanvasEvents）

---

## Task 1: 事件委托基础设施

**目标：** 将画布事件从逐组件绑定改为容器级事件委托，为后续 diff 渲染铺路（diff 后无需重绑事件）。

**Files:**
- Modify: `index.html` L4657-4694（renderCanvas 中移除逐组件事件绑定）
- Modify: `index.html` L6014-6052（init 中调用 initCanvasEvents）
- Modify: `index.html` L4850-4864（selectComponent 改用 updateSelectionState）

- [ ] **Step 1: 新增 initCanvasEvents 和 updateSelectionState 函数**

在 `renderCanvas` 函数之前（L4657 之前，即 `let _renderScheduled = false;` 之前）插入两个新函数：

```javascript
    // ===== CANVAS EVENT DELEGATION =====
    function initCanvasEvents() {
      const canvas = document.getElementById('canvasContent');
      if (!canvas) return;
      canvas.addEventListener('click', (e) => {
        const wrapper = e.target.closest('.canvas-component');
        if (!wrapper) {
          selectedComponentId = null;
          updateSelectionState();
          const empty = document.getElementById('propertiesEmpty');
          const body = document.getElementById('propertiesBody');
          if (empty) empty.style.display = 'flex';
          if (body) body.style.display = 'none';
          return;
        }
        e.stopPropagation();
        selectComponent(wrapper.dataset.id);
      });
      canvas.addEventListener('dragstart', (e) => {
        const wrapper = e.target.closest('.canvas-component');
        if (wrapper) onDragStart(e, wrapper.dataset.id);
      });
      canvas.addEventListener('dragend', onDragEnd);
      canvas.addEventListener('dragover', (e) => {
        const wrapper = e.target.closest('.canvas-component');
        if (wrapper) {
          onDragOver(e, wrapper.dataset.id);
        } else {
          e.preventDefault();
        }
      });
      canvas.addEventListener('drop', (e) => {
        const wrapper = e.target.closest('.canvas-component');
        if (wrapper) {
          onDrop(e, wrapper.dataset.id);
        } else {
          onDropToEnd(e);
        }
      });
      canvas.addEventListener('dragenter', (e) => e.preventDefault());
    }

    function updateSelectionState() {
      document.querySelectorAll('.canvas-component.selected')
        .forEach(el => el.classList.remove('selected'));
      if (selectedComponentId) {
        const el = document.getElementById('wrap-' + selectedComponentId);
        if (el) el.classList.add('selected');
      }
    }
```

- [ ] **Step 2: 移除 renderCanvas 中的逐组件事件绑定**

将 renderCanvas 的 requestAnimationFrame 回调中，从 `components.forEach(c => {` 到 `canvas.onclick = () => {` 结束的整段事件绑定代码（L4673-4692）替换为空操作。

找到这段代码（L4673-4692）：
```javascript
        components.forEach(c => {
          const el = document.getElementById('wrap-' + c.id);
          if (!el) return;
          el.onclick = (e) => { e.stopPropagation(); selectComponent(c.id); };
          const handle = el.querySelector('.canvas-component-draghandle');
          if (handle) {
            handle.addEventListener('dragstart', (e) => onDragStart(e, c.id));
            handle.addEventListener('dragend', onDragEnd);
          }
          el.addEventListener('dragover', (e) => onDragOver(e, c.id));
          el.addEventListener('drop', (e) => onDrop(e, c.id));
          el.addEventListener('dragenter', (e) => e.preventDefault());
        });
        canvas.ondragover = (e) => { e.preventDefault(); };
        canvas.ondrop = (e) => { e.preventDefault(); onDropToEnd(e); };
        canvas.onclick = () => {
          selectedComponentId = null;
          renderCanvas();
          selectComponent(null);
        };
```

替换为：
```javascript
        // 事件委托已在 initCanvasEvents 中一次性绑定，此处无需重绑
```

- [ ] **Step 3: 改造 selectComponent 使用 updateSelectionState**

将 selectComponent（L4850-4864）中的 `renderCanvas()` 调用替换为 `updateSelectionState()`。

找到：
```javascript
    function selectComponent(id) {
      selectedComponentId = id;
      renderCanvas();
      const comp = components.find(c => c.id === id);
```

替换为：
```javascript
    function selectComponent(id) {
      selectedComponentId = id;
      updateSelectionState();
      const comp = components.find(c => c.id === id);
```

- [ ] **Step 4: 在 init 中调用 initCanvasEvents**

在 init 区域的 `loadFromStorage();`（L6014）之前插入 `initCanvasEvents();`。

找到：
```javascript
    loadFromStorage();
    showWelcome();
```

替换为：
```javascript
    initCanvasEvents();
    loadFromStorage();
    showWelcome();
```

- [ ] **Step 5: 浏览器验证**

打开 index.html，进入设计模式：
1. 点击组件 -> 应正确选中（高亮 + 属性面板显示）
2. 点击画布空白区域 -> 应取消选中
3. 拖拽组件排序 -> 应正常工作
4. 从左侧组件库拖入新组件 -> 应正常插入

如果以上任一不正常，检查 `data-id` 属性是否正确输出（在 DevTools Elements 面板查看 `wrap-*` 节点）。

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "perf: 画布事件委托优化，移除逐组件事件重绑"
```

---

## Task 2: 组件级 Diff 渲染引擎

**目标：** 将 renderCanvas 从全量 innerHTML 重建改为组件级 diff，只更新变化的组件节点。

**Files:**
- Modify: `index.html` L3916-3935（新增 diff 状态变量）
- Modify: `index.html` L4657-4694（renderCanvas 重构为 diff + legacy 回退）

- [ ] **Step 1: 新增 diff 相关状态变量**

在 `let storageTimer = null;`（L3935）之后插入：

```javascript
    // ===== DIFF RENDER STATE =====
    let _lastSignatures = {};
    let _lastComponentIds = [];
    let _canvasInitialized = false;
```

- [ ] **Step 2: 新增 patchCanvas 和 renderCanvasLegacy 函数**

在 `let _renderScheduled = false;`（L4657）之后，`function renderCanvas()` 之前插入：

```javascript
    function renderCanvasLegacy() {
      const canvas = document.getElementById('canvasContent');
      canvas.style.maxWidth = spacing.containerWidth + 'px';
      const lbl = document.getElementById('canvasSizeLabel');
      if (lbl) lbl.textContent = spacing.containerWidth + 'px';
      if (components.length === 0) {
        canvas.innerHTML = '<div class="canvas-empty-guide"><svg viewBox="0 0 24 24" width="48" height="48" fill="none" stroke="var(--text-muted)" stroke-width="1.5"><rect x="3" y="3" width="18" height="18" rx="2"/><line x1="3" y1="9" x2="21" y2="9"/><line x1="9" y1="21" x2="9" y2="9"/></svg><div class="canvas-empty-title">画布空空如也</div><div class="canvas-empty-desc">切换到左侧「模板」Tab 可快速加载预设模板<br>或<code>/</code>键呼出组件面板，充电宝一样快</div></div>';
      } else {
        canvas.innerHTML = components.map(c => renderComponent(c)).join('');
      }
      // 重建签名缓存
      _lastSignatures = {};
      _lastComponentIds = [];
      components.forEach(c => {
        _lastSignatures[c.id] = JSON.stringify(c.props) + c.type;
        _lastComponentIds.push(c.id);
      });
      _canvasInitialized = true;
    }

    function patchCanvas() {
      const canvas = document.getElementById('canvasContent');
      canvas.style.maxWidth = spacing.containerWidth + 'px';
      const lbl = document.getElementById('canvasSizeLabel');
      if (lbl) lbl.textContent = spacing.containerWidth + 'px';

      // 空画布
      if (components.length === 0) {
        if (!_lastComponentIds.length) return; // 已经是空的
        canvas.innerHTML = '<div class="canvas-empty-guide"><svg viewBox="0 0 24 24" width="48" height="48" fill="none" stroke="var(--text-muted)" stroke-width="1.5"><rect x="3" y="3" width="18" height="18" rx="2"/><line x1="3" y1="9" x2="21" y2="9"/><line x1="9" y1="21" x2="9" y2="9"/></svg><div class="canvas-empty-title">画布空空如也</div><div class="canvas-empty-desc">切换到左侧「模板」Tab 可快速加载预设模板<br>或<code>/</code>键呼出组件面板，充电宝一样快</div></div>';
        _lastSignatures = {};
        _lastComponentIds = [];
        return;
      }

      // 全量回退条件：组件数量变化超过 50% 或首次渲染
      const oldLen = _lastComponentIds.length;
      const newLen = components.length;
      if (!_canvasInitialized || (oldLen > 0 && Math.abs(newLen - oldLen) / Math.max(newLen, oldLen) > 0.5)) {
        renderCanvasLegacy();
        return;
      }

      // 构建旧节点映射
      const oldIdSet = new Set(_lastComponentIds);
      const newIds = components.map(c => c.id);

      // 删除不再存在的节点
      _lastComponentIds.forEach(id => {
        if (!newIds.includes(id)) {
          const el = document.getElementById('wrap-' + id);
          if (el) el.remove();
          delete _lastSignatures[id];
        }
      });

      // 遍历新组件：更新或插入
      let prevNode = null;
      components.forEach(c => {
        const sig = JSON.stringify(c.props) + c.type;
        const existing = document.getElementById('wrap-' + c.id);
        if (existing && oldIdSet.has(c.id) && _lastSignatures[c.id] === sig) {
          // 未变化，跳过
          prevNode = existing;
        } else if (existing) {
          // 已存在但 props 变化 -> 替换
          const isSelected = existing.classList.contains('selected');
          existing.outerHTML = renderComponent(c);
          const newNode = document.getElementById('wrap-' + c.id);
          if (isSelected && newNode) newNode.classList.add('selected');
          _lastSignatures[c.id] = sig;
          prevNode = newNode;
        } else {
          // 新组件 -> 创建并插入
          const tmp = document.createElement('div');
          tmp.innerHTML = renderComponent(c);
          const newNode = tmp.firstElementChild;
          if (prevNode) {
            prevNode.after(newNode);
          } else {
            canvas.prepend(newNode);
          }
          _lastSignatures[c.id] = sig;
          prevNode = newNode;
        }
      });

      _lastComponentIds = newIds;
      _canvasInitialized = true;
    }
```

- [ ] **Step 3: 重构 renderCanvas 为 diff 调度器**

将原 renderCanvas 函数（L4659-4694）替换为：

```javascript
    function renderCanvas() {
      if (_renderScheduled) return;
      _renderScheduled = true;
      requestAnimationFrame(() => {
        _renderScheduled = false;
        patchCanvas();
      });
    }
```

- [ ] **Step 4: 浏览器验证**

打开 index.html，进入设计模式：
1. 加载模板（如"期刊 Highlights"）-> 应触发全量回退路径，组件完整渲染
2. 选中组件 -> 高亮正确
3. 修改属性面板中的标题文字 -> 画布应只更新对应组件
4. 添加新组件 -> diff 插入新节点
5. 删除组件 -> diff 移除节点
6. 清空画布 -> 空状态提示显示
7. 再次加载模板 -> 全量回退后完整渲染

在 DevTools Console 中检查无报错。

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "perf: 组件级diff渲染引擎，替代全量innerHTML重建"
```

---

## Task 3: 属性编辑局部更新

**目标：** 属性编辑时不再触发全量 renderCanvas，改为只更新当前组件 DOM。

**Files:**
- Modify: `index.html` L3935（新增 patchComponent 函数）
- Modify: `index.html` L4892-4902（updateComponentProp 重构）

- [ ] **Step 1: 新增 patchComponent 和 _debouncedSaveHistory**

在 Task 1 新增的 `updateSelectionState` 函数之后插入：

```javascript
    function patchComponent(id) {
      const comp = components.find(c => c.id === id);
      const node = document.getElementById('wrap-' + id);
      if (!comp || !node) return;
      const isSelected = node.classList.contains('selected');
      node.outerHTML = renderComponent(comp);
      const newNode = document.getElementById('wrap-' + id);
      if (isSelected && newNode) newNode.classList.add('selected');
      _lastSignatures[id] = JSON.stringify(comp.props) + comp.type;
    }
```

在状态变量区（`let _canvasInitialized = false;` 之后）新增：

```javascript
    const _debouncedSaveHistory = debounce(() => saveHistory(), 500);
```

- [ ] **Step 2: 重构 updateComponentProp**

找到原函数（L4892-4902）：
```javascript
    function updateComponentProp(id, key, val, e) {
      const comp = components.find(c => c.id === id);
      if (!comp) return;
      comp.props[key] = val;
      const input = e && e.target || event && event.target;
      if (input) {
        validateInput(input, true, '');
      }
      saveHistory();
      renderCanvas();
    }
```

替换为：
```javascript
    function updateComponentProp(id, key, val, e) {
      const comp = components.find(c => c.id === id);
      if (!comp) return;
      comp.props[key] = val;
      const input = e && e.target || event && event.target;
      if (input) {
        validateInput(input, true, '');
      }
      patchComponent(id);
      _debouncedSaveHistory();
    }
```

- [ ] **Step 3: 浏览器验证**

打开 index.html，进入设计模式，加载模板（选 20+ 组件的模板如"季刊完整版"）：
1. 选中一个组件，在属性面板连续快速输入文字 -> 画布应只更新该组件，无卡顿
2. 输入过程中焦点不应丢失
3. 选中态（高亮边框）应保持
4. 停止输入 500ms 后，历史时间线应新增一帧（而非每键一帧）
5. 撤销操作 -> 应回到编辑前的状态（注意：由于节流，可能跳到 500ms 前的状态）

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "perf: 属性编辑局部更新，避免全量画布重渲染"
```

---

## Task 4: 滑块操作防抖

**目标：** 10 个样式控制滑块的 oninput 改用防抖渲染，避免拖动时连续全量渲染。

**Files:**
- Modify: `index.html` L3935 附近（新增 _debouncedRender）
- Modify: `index.html` L4139-4272（10 个滑块函数）

- [ ] **Step 1: 新增 _debouncedRender**

在 `_debouncedSaveHistory` 定义之后新增：

```javascript
    const _debouncedRender = debounce(() => renderCanvas(), 100);
```

- [ ] **Step 2: 将 10 个滑块函数中的 renderCanvas() 替换为 _debouncedRender()**

需要修改的 10 个函数，逐个将 `renderCanvas()` 替换为 `_debouncedRender()`：

1. `updateThemeColor` (L4148): `renderCanvas();` -> `_debouncedRender();`
2. `updateBgColor` (L4159): `renderCanvas();` -> `_debouncedRender();`
3. `updateTextColor` (L4171): `renderCanvas();` -> `_debouncedRender();`
4. `updateHeadingSize` (L4228 附近): `renderCanvas();` -> `_debouncedRender();`
5. `updateFontWeight` (L4234 附近): `renderCanvas();` -> `_debouncedRender();`
6. `updateBodySize` (L4240 附近): `renderCanvas();` -> `_debouncedRender();`
7. `updateLineHeight` (L4246 附近): `renderCanvas();` -> `_debouncedRender();`
8. `updateContainerWidth` (L4253 附近): `renderCanvas();` -> `_debouncedRender();`
9. `updateSectionGap` (L4261 附近): `renderCanvas();` -> `_debouncedRender();`
10. `updatePadding` (L4267 附近): `renderCanvas();` -> `_debouncedRender();`

**注意：** 每个函数中 UI 控件值更新（如 `document.getElementById('xxx').value = v`）保持不变，只有 `renderCanvas()` 调用替换为 `_debouncedRender()`。`checkContrast()` 调用也保持不变（轻量操作）。

使用搜索替换时需注意：每个函数的 `renderCanvas()` 调用需逐个替换，不能全局替换（因为其他函数如 moveUp/moveDown 的 renderCanvas 不应防抖）。

- [ ] **Step 3: 浏览器验证**

打开 index.html，进入设计模式，加载模板：
1. 拖动主色调颜色选择器 -> 画布应在 100ms 内更新，拖动过程流畅
2. 拖动 H1 字号滑块 -> 数值标签即时更新，画布防抖更新
3. 拖动容器宽度滑块 -> 画布尺寸标签即时更新，画布防抖更新
4. 拖动区块间距滑块 -> 画布防抖更新
5. 在颜色 hex 输入框中输入颜色 -> 画布更新（注意：输入框的 oninput 也会触发 updateThemeColor，防抖应正常工作）

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "perf: 滑块操作防抖100ms，避免拖动时连续全量渲染"
```

---

## Task 5: 历史快照节流

**目标：** saveHistory 引入 500ms 节流，合并连续编辑；离散操作使用 force 选项强制保存。

**Files:**
- Modify: `index.html` L5770-5794（saveHistory 重构）
- Modify: 多处 saveHistory() 调用改为 saveHistory({ force: true })

- [ ] **Step 1: 重构 saveHistory 为节流版本**

找到原 saveHistory（L5770-5794）：
```javascript
    function saveHistory() {
      const state = structuredClone({ components, theme, font, spacing });
      if (historyIndex < history.length - 1) history = history.slice(0, historyIndex + 1);
      history.push(state);
      if (history.length > HISTORY_LIMIT) history.shift();
      historyIndex = history.length - 1;
      hasUnsavedChanges = true;
      scheduleSave();
      // 异步镜像到 IndexedDB（不阻塞 undo/redo）
      scheduleIdle(() => {
        saveFrameToDB({
          id: 'frame_' + Date.now() + '_' + Math.random().toString(36).slice(2, 7),
          ts: Date.now(),
          thumbnail: generateSVGThumbnail(),
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

替换为：
```javascript
    const HISTORY_MIN_INTERVAL = 500;
    let _lastHistoryTime = 0;
    let _pendingHistoryTimer = null;

    function saveHistory(opts) {
      if (opts && opts.force) {
        clearTimeout(_pendingHistoryTimer);
        _doSaveHistory();
        return;
      }
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
      const state = structuredClone({ components, theme, font, spacing });
      if (historyIndex < history.length - 1) history = history.slice(0, historyIndex + 1);
      history.push(state);
      if (history.length > HISTORY_LIMIT) history.shift();
      historyIndex = history.length - 1;
      hasUnsavedChanges = true;
      scheduleSave();
      scheduleIdle(() => {
        saveFrameToDB({
          id: 'frame_' + Date.now() + '_' + Math.random().toString(36).slice(2, 7),
          ts: Date.now(),
          thumbnail: generateSVGThumbnail(),
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

- [ ] **Step 2: 将离散操作的 saveHistory() 调用改为 saveHistory({ force: true })**

需要修改的函数（逐个搜索 `saveHistory();` 并替换为 `saveHistory({ force: true });`）：

1. `moveUp` 函数中的 `saveHistory();`
2. `moveDown` 函数中的 `saveHistory();`
3. `duplicateComponent` 函数中的 `saveHistory();`
4. `deleteComponent` 函数中的 `saveHistory();`
5. `clearCanvas` 函数中的 `saveHistory();`
6. `loadTemplate` 函数中的 `saveHistory();`
7. `restoreFrame` 函数中的 `saveHistory();`
8. `onDrop` 函数中的 `saveHistory();`（拖拽排序后）

**注意：** `updateComponentProp` 中的 `saveHistory()` 已在 Task 3 改为 `_debouncedSaveHistory()`，不需要再改。`undoDesign` 和 `redoDesign` 不调用 `saveHistory()`，无需修改。

- [ ] **Step 3: 浏览器验证**

打开 index.html，进入设计模式：
1. 加载模板 -> 历史时间线立即新增一帧（force）
2. 连续快速编辑属性 -> 停止 500ms 后历史时间线新增一帧（合并）
3. 撤销 -> 正确回退到合并前的状态
4. 重做 -> 正确前进
5. 拖拽排序 -> 立即新增一帧（force）
6. 删除组件 -> 立即新增一帧（force）
7. 复制组件 -> 立即新增一帧（force）
8. 清空画布 -> 立即新增一帧（force）
9. 恢复历史帧 -> 立即新增一帧（force）

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "perf: 历史快照节流500ms，合并连续编辑，离散操作强制保存"
```

---

## Task 6: 性能基准测量与回退开关

**目标：** 添加性能测量代码和调试回退开关，对比优化前后数据。

**Files:**
- Modify: `index.html`（新增性能测量工具和回退开关）

- [ ] **Step 1: 添加 FORCE_FULL_RENDER 调试开关**

在 `let _canvasInitialized = false;` 之后新增：

```javascript
    const FORCE_FULL_RENDER = false; // 调试开关：true 时强制全量渲染
```

- [ ] **Step 2: 在 renderCanvas 中添加开关检查**

修改 renderCanvas 函数：

```javascript
    function renderCanvas() {
      if (_renderScheduled) return;
      _renderScheduled = true;
      requestAnimationFrame(() => {
        _renderScheduled = false;
        if (FORCE_FULL_RENDER) {
          renderCanvasLegacy();
        } else {
          patchCanvas();
        }
      });
    }
```

- [ ] **Step 3: 添加性能测量辅助函数**

在 `patchComponent` 函数之后新增：

```javascript
    // 性能测量辅助（开发调试用，可在控制台调用）
    function _benchPatchComponent(id) {
      const t0 = performance.now();
      for (let i = 0; i < 100; i++) patchComponent(id);
      const t1 = performance.now();
      console.log('patchComponent x100: ' + (t1 - t0).toFixed(2) + 'ms, avg: ' + ((t1 - t0) / 100).toFixed(2) + 'ms');
    }
    function _benchRenderCanvas() {
      const t0 = performance.now();
      for (let i = 0; i < 100; i++) renderCanvasLegacy();
      const t1 = performance.now();
      console.log('renderCanvasLegacy x100: ' + (t1 - t0).toFixed(2) + 'ms, avg: ' + ((t1 - t0) / 100).toFixed(2) + 'ms');
    }
```

- [ ] **Step 4: 浏览器性能对比验证**

打开 index.html，进入设计模式，加载"季刊完整版"模板（20+ 组件）：

**基线测量（全量渲染）：**
1. 在 DevTools Console 中执行：
   ```javascript
   _benchRenderCanvas()
   ```
   记录 100 次全量渲染的平均耗时。

**优化后测量（局部更新）：**
2. 选中一个组件，在 Console 中执行：
   ```javascript
   _benchPatchComponent(selectedComponentId)
   ```
   记录 100 次 patchComponent 的平均耗时。

**对比：**
3. patchComponent 耗时应 < renderCanvasLegacy 耗时的 30%
4. 如果不满足，检查 patchComponent 是否正确复用了 DOM 节点

**功能回归验证：**
5. 将 `FORCE_FULL_RENDER` 改为 `true`，刷新页面，验证全量渲染路径仍正常
6. 改回 `false`，刷新页面，验证 diff 路径正常

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "perf: 添加性能测量工具和全量渲染调试开关"
```

---

## Task 7: 最终集成验证

**目标：** 全面验证所有优化后功能正常，无回归。

- [ ] **Step 1: 完整功能验证清单**

打开 index.html，按以下清单逐项验证：

| # | 场景 | 操作 | 预期 |
|---|------|------|------|
| 1 | 加载模板 | 设计模式 -> 模板 Tab -> 点击"期刊 Highlights" | 组件完整渲染，画布正确 |
| 2 | 属性编辑 | 选中组件 -> 属性面板输入文字 | 画布即时更新该组件，无卡顿 |
| 3 | 焦点保持 | 属性面板连续输入 10 个字符 | 焦点不丢失 |
| 4 | 选中态 | 选中组件 -> 编辑属性 | 选中高亮保持 |
| 5 | 颜色滑块 | 拖动主色调选择器 | 100ms 内画布更新，流畅 |
| 6 | 字号滑块 | 拖动 H1 字号滑块 | 数值即时更新，画布防抖 |
| 7 | 拖拽排序 | 拖拽组件 A 到组件 B 前方 | 顺序正确，选中态保持 |
| 8 | 库组件拖入 | 从左侧组件库拖入新组件 | 正确插入到目标位置 |
| 9 | 撤销 | 编辑后点击撤销 | 回到编辑前状态 |
| 10 | 重做 | 撤销后点击重做 | 前进到撤销前状态 |
| 11 | 历史时间线 | 检查页底部历史 | 有记录，不过度（连续编辑合并） |
| 12 | 恢复历史帧 | 点击历史时间线某帧 | 画布恢复到该帧状态 |
| 13 | 删除历史帧 | 点击某帧的 × 按钮 | 该帧删除 |
| 14 | 清空画布 | 点击清空 -> 确认 | 画布清空，空状态显示 |
| 15 | 复制组件 | 选中 -> 点击复制 | 组件复制到下一位置 |
| 16 | 删除组件 | 选中 -> 点击删除 | 组件删除 |
| 17 | 导出 HTML | 点击导出 | 下载 newsletter.html，内容正确 |
| 18 | 预览模式 | 切换到预览模式 | iframe 正确显示 |
| 19 | 代码模式 | 切换到代码模式 | 代码编辑器显示 HTML |
| 20 | 暗色预览 | 检查页 -> 勾选暗色预览 | 画布切换暗色 |
| 21 | WCAG 审计 | 检查页 -> 点击审计 | 报告正常生成 |
| 22 | 保存模板 | 当前设计 -> 保存为模板 | 模板列表新增 |
| 23 | 保存组件 | 选中组件 -> 保存为区块 | 自定义组件列表新增 |
| 24 | 刷新恢复 | 编辑后刷新页面 | 组件恢复，历史时间线保留 |

- [ ] **Step 2: DevTools Console 错误检查**

在完成所有操作后，打开 DevTools Console，确认无任何报错或警告（除了已知的 `history frame save failed` IndexedDB 配额警告）。

- [ ] **Step 3: 最终 Commit**

```bash
git add index.html
git commit -m "perf: 性能优化完成 - diff渲染+局部更新+防抖+节流+事件委托"
```

---

## 自审清单

### Spec 覆盖检查

| Spec 章节 | 对应 Task | 状态 |
|-----------|-----------|------|
| 3. 组件级 Diff 渲染引擎 | Task 2 | ✅ patchCanvas + 签名缓存 + 全量回退 |
| 4. 属性编辑防抖与局部更新 | Task 3 | ✅ patchComponent + _debouncedSaveHistory |
| 5. 滑块操作防抖 | Task 4 | ✅ _debouncedRender 100ms |
| 6. 历史快照节流 | Task 5 | ✅ 500ms 节流 + force 选项 |
| 7. 事件委托优化 | Task 1 | ✅ initCanvasEvents + updateSelectionState |
| 8.3 回退策略 | Task 6 | ✅ FORCE_FULL_RENDER 开关 |
| 8.1 验证矩阵 | Task 7 | ✅ 24 项验证清单 |

### 类型一致性检查

- `patchCanvas()` - 无参数，从全局 components 读取 ✅
- `patchComponent(id)` - 接收 id 字符串 ✅
- `updateSelectionState()` - 无参数 ✅
- `initCanvasEvents()` - 无参数 ✅
- `saveHistory(opts)` - opts 为 `{ force: true }` 或 undefined ✅
- `_doSaveHistory()` - 内部函数，无参数 ✅
- `_debouncedRender` - debounce 包装的 renderCanvas ✅
- `_debouncedSaveHistory` - debounce 包装的 saveHistory()（无参数，触发节流） ✅
- `renderCanvasLegacy()` - 无参数 ✅
- CSS 类名统一使用 `canvas-component`（现有） ✅
- data 属性统一使用 `data-id`（现有 L4847） ✅

### 占位符检查

- 无 TBD/TODO ✅
- 所有代码步骤包含完整代码 ✅
- 所有验证步骤包含具体操作和预期 ✅
