# Equipment Layout Manager 技术文档

一个基于 Vue 3 与 Konva.js 的设备布局可视化工具，用于在大尺寸工程坐标系中加载、查看、搜索、新增和编辑设备矩形，并可叠加由 DXF 等工程图转换得到的 SVG 背景。

项目采用纯前端架构：设备数据和背景文件从 `public/` 目录加载，所有新增、编辑和删除操作只保存在当前页面内存中，刷新页面后会恢复为静态文件中的初始数据。

## 1. 功能概览

- 从 `output_eq.json` 加载设备位置、尺寸和扩展信息
- 使用左下角为原点、Y 轴向上的工程坐标系展示设备
- 支持负坐标，并根据设备和背景图范围动态计算画布边界
- 使用 Konva 分层绘制网格、SVG 背景、设备、提示框和原点标记
- 支持滚轮缩放、拖动画布、滑块缩放、重置比例和自适应视图
- 按设备 ID 搜索、聚焦并高亮设备
- 点击设备打开编辑弹窗，支持修改、删除及只读查看扩展信息
- 通过表单新增设备，并进行 ID、坐标和尺寸校验
- 支持全屏浏览和背景图显隐切换
- 鼠标悬停显示设备信息，移动鼠标时显示当前 DXF 坐标

## 2. 技术栈

| 技术/组件 | 版本 | 用途 |
| --- | --- | --- |
| Vue | `^3.5.32` | 页面组件、响应式状态、组件通信和生命周期管理 |
| Konva | `^10.3.0` | Canvas 场景图、图层管理、图形绘制、命中检测、拖拽和缩放 |
| Vite | `^8.0.10` | 本地开发服务器、Vue 单文件组件编译和生产构建 |
| `@vitejs/plugin-vue` | `^6.0.6` | 为 Vite 提供 Vue SFC 编译支持 |
| CSS | 原生 | 响应式布局、面板、表单、弹窗和全屏模式样式 |
| Fetch API / Blob URL | 浏览器原生 | 加载 JSON、SVG，并将 SVG 文本转换为可供图片节点使用的 URL |
| ResizeObserver | 浏览器原生 | 容器尺寸变化时同步调整 Konva Stage |

项目没有引入路由、状态管理库或后端接口，应用状态集中维护在根组件中。

## 3. 快速开始

### 环境要求

- Node.js：建议使用满足 Vite 8 要求的 Node.js 版本
- npm：随 Node.js 安装

### 安装与运行

```bash
npm install
npm run dev
```

终端会输出本地访问地址，通常为 `http://localhost:5173`。

### 构建与预览

```bash
npm run build
npm run preview
```

生产文件输出到 `dist/`。可用脚本如下：

| 命令 | 说明 |
| --- | --- |
| `npm run dev` | 启动 Vite 开发服务器 |
| `npm run build` | 生成生产构建 |
| `npm run preview` | 在本地预览生产构建 |

## 4. 目录结构

```text
.
├── public/
│   ├── output_eq.json          # 初始设备数据
│   ├── raw_layer.svg           # 工程图背景
│   ├── raw_layer_meta.json     # 背景图坐标范围和元数据
│   ├── icons.svg
│   └── favicon.svg
├── src/
│   ├── components/
│   │   ├── DeviceCanvas.vue    # Konva 画布及全部交互逻辑
│   │   ├── DeviceForm.vue      # 新增设备表单
│   │   └── DeviceEditModal.vue # 编辑、删除设备的弹窗
│   ├── App.vue                 # 数据加载、全局状态及业务编排
│   ├── main.js                 # Vue 应用入口
│   └── style.css               # 全局与组件样式
├── index.html
├── vite.config.js
└── package.json
```

`HelloWorld.vue` 及 `src/assets/` 中的模板资源当前未被业务代码引用，可视为 Vite 模板遗留文件。

## 5. 整体架构

应用遵循“根组件持有状态、子组件负责交互”的单向数据流：

```text
public 静态资源
  ├─ output_eq.json ──┐
  ├─ raw_layer.svg ───┼─ fetch ─> App.vue（状态与业务编排）
  └─ meta.json ───────┘              │
                                     ├─ props ─> DeviceForm
                                     ├─ props ─> DeviceCanvas ─> Konva Stage/Layers
                                     └─ props ─> DeviceEditModal
                                           ↑
                               add/select/zoom/save/delete 事件
```

`App.vue` 是唯一业务状态源，主要维护：

- `devices`：当前设备列表
- `canvasBounds`：世界坐标边界
- `zoom`：当前缩放比例
- `selectedId`：当前选中设备 ID
- `isEditOpen`、`isFullscreen`：界面状态
- `bgSvgUrl`、`bgSvgMeta`、`showBg`：背景图状态

子组件不直接修改设备列表，而是通过 `emit` 将操作交给 `App.vue`：

- `DeviceForm` 发出 `add`
- `DeviceCanvas` 发出 `select`、`zoom`
- `DeviceEditModal` 发出 `save`、`delete`、`close`

## 6. 核心组件实现

### 6.1 `App.vue`：数据加载与业务编排

组件挂载后会并行完成两组资源加载：

1. 请求 `/output_eq.json`，标准化数字字段并生成设备 ID。
2. 请求 `/raw_layer.svg` 和 `/raw_layer_meta.json`，将 SVG 文本包装为 `Blob`，再通过 `URL.createObjectURL` 生成图片 URL。

设备 ID 的生成规则为：

1. 优先使用数据中的非空 `id`。
2. 如果没有 `id`，拼接 `info.direction_flag`、`info.文字` 和 `info.bay_occupation`。
3. 如果以上字段均为空，生成 `M-{序号}`。

`_isDetail` 是前端增加的内部标识：原始数据存在显式 `id` 时为 `true`，用于区分 Detail 和 Macro 两种标签及提示信息展示方式。

边界由 `computeBounds()` 计算。它遍历所有设备矩形，并可将背景元数据中的 `bbox` 合并到范围内；最终在四周增加 `2000` 个世界坐标单位的留白。没有数据时使用 `800000 × 200000` 的默认范围。

搜索只匹配 `_isDetail === true` 的设备，ID 比较不区分大小写。匹配成功后调用画布暴露的 `focusDevice(id)` 完成居中和缩放。

### 6.2 `DeviceCanvas.vue`：Konva 渲染引擎

该组件直接管理 Konva 实例，不通过 Vue 模板逐个渲染图形。创建的场景层级如下：

| 顺序 | Konva 节点 | 职责 |
| --- | --- | --- |
| 1 | `FastLayer` | 绘制无交互的棋盘格背景，降低基础背景开销 |
| 2 | `Layer` | 加载和显示 SVG 工程图背景 |
| 3 | `Layer` | 绘制设备矩形及文字标签，并接收点击和悬停事件 |
| 4 | `Layer` | 绘制不受设备层遮挡的 Tooltip |
| 5 | `Layer` | 标记 DXF 坐标原点 `(0, 0)` |

Stage 本身开启 `draggable`，因此用户可以拖拽浏览。组件同时注册：

- `wheel`：以鼠标位置为缩放中心，缩放倍率为每次 `1.05`
- `dragmove` / `dragend`：约束 Stage 位置，避免画布完全拖出视口
- `click`：点击空白区域取消选择
- `mousemove`：将指针屏幕坐标反算为 DXF 坐标
- `ResizeObserver` 和 `window.resize`：更新 Stage 宽高

设备节点使用 `Map<id, { group, rect, label }>` 缓存。数据变化时复用已有节点、更新属性，并销毁已删除设备的节点，避免每次响应式更新都重建整个设备图层。

选中设备使用黄色填充、橙色描边和阴影；普通设备使用蓝色填充。线宽依据所有设备矩形的平均对角线动态计算，使不同量级的工程图上保持相对可见性。

标签内容分两种模式：

- Detail：显示设备 `id`
- Macro：分行显示 `direction_flag`、`文字`、`bay_occupation`

组件通过 `defineExpose()` 暴露两个方法：

- `autoFit()`：计算所有设备的包围盒，以最大 `1` 倍的比例缩放并居中
- `focusDevice(id)`：根据目标设备尺寸计算比例，将指定设备放大并居中

### 6.3 `DeviceForm.vue`：新增设备

表单使用 `reactive` 保存草稿，提交前检查：

- ID 必填且不能与现有 ID 重复
- 坐标和尺寸必须是有效数字
- X、Y 不能为负数
- Width、Height 必须大于 0
- 设备不能超出默认的 `800000 × 200000` 范围

校验通过后发出 `add` 事件，并恢复默认值。新增设备只包含 `id`、`x`、`y`、`width` 和 `height`。

### 6.4 `DeviceEditModal.vue`：编辑和删除

弹窗监听 `device` 属性，将设备字段复制到本地 `draft`，避免输入过程中直接修改父组件数据。保存时执行与新增表单相同的数值和范围检查，并允许设备保留自己的原 ID。

设备的 `info` 在弹窗中只读展示。对象值会被序列化；包含 `x`、`y` 的对象会显示为 `x=..., y=...`。删除动作由根组件进行二次确认后执行。

## 7. 坐标系统与转换

业务数据使用工程坐标系：原点在左下角，X 向右增长，Y 向上增长。Canvas/Konva 默认使用屏幕坐标系：原点在左上角，Y 向下增长，因此绘制时必须翻转 Y 轴。

设：

- 设备业务坐标为 `(x, y)`
- 设备尺寸为 `(w, h)`
- 世界范围左下角为 `(minX, minY)`
- 世界范围高度为 `canvasHeight`

则设备矩形左上角的 Konva 坐标为：

```text
konvaX = x - minX
konvaY = canvasHeight + minY - y - h
```

减去 `h` 是因为业务坐标描述矩形左下角，而 Konva 的矩形坐标描述左上角。

鼠标位置反算业务坐标时，还需要消除 Stage 的平移与缩放：

```text
worldX = (pointerX - stageX) / scale + minX
worldY = canvasHeight + minY - (pointerY - stageY) / scale
```

SVG 背景根据 `bbox.minX` 和 `bbox.maxY` 定位，同样使用上述坐标翻转规则，从而与设备图层对齐。

## 8. 数据格式

### 8.1 设备数据 `output_eq.json`

基础格式：

```json
[
  {
    "id": "EQ-1001",
    "x": 80,
    "y": 60,
    "width": 180,
    "height": 120,
    "info": {
      "area": "A1",
      "direction_flag": "N",
      "文字": "ETCH",
      "bay_occupation": "B01"
    }
  }
]
```

字段说明：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | string | 否 | 设备唯一标识；为空时由 `info` 字段组合生成 |
| `x` | number/string | 是 | 设备左下角 X 坐标，加载时使用 `Number()` 转换 |
| `y` | number/string | 是 | 设备左下角 Y 坐标 |
| `width` | number/string | 是 | 设备宽度 |
| `height` | number/string | 是 | 设备高度 |
| `info` | object | 否 | Tooltip 和编辑弹窗展示的扩展信息 |

### 8.2 背景元数据 `raw_layer_meta.json`

```json
{
  "layer": "LayerB",
  "source": "/tmp/test_layer.dxf",
  "bbox": {
    "minX": 0,
    "minY": 0,
    "maxX": 200,
    "maxY": 100
  },
  "svgWidth": 200,
  "svgHeight": 100,
  "entityCount": 2
}
```

`bbox` 是背景定位和扩展全局画布边界的必要字段。`svgWidth`、`svgHeight` 可选；缺失时使用 bbox 的宽高差。SVG 本身应与元数据使用同一工程坐标范围。

## 9. 主要交互流程

### 初始加载

1. Vue 挂载页面并请求静态数据。
2. 设备字段被标准化，随后计算设备边界。
3. SVG 与元数据加载完成后，背景 bbox 合并进画布边界。
4. `DeviceCanvas` 监听 props，更新网格、设备和背景节点。
5. 父组件调用 `autoFit()`，使设备区域适应当前视口。

### 缩放与拖拽

工具栏和滚轮最终都更新根组件中的 `zoom`。画布监听该属性并同步设置 Stage scale；滚轮操作会先计算鼠标对应的世界坐标，再调整 Stage position，保证缩放中心停留在鼠标位置。

### 编辑设备

点击矩形后，画布发出 `select(id)`；根组件更新选中 ID 并打开弹窗。弹窗保存后发出完整的新设备对象，根组件使用 `map` 替换对应项，随后画布 watcher 更新缓存节点。

## 10. 样式与全屏实现

全局样式位于 `src/style.css`，使用 CSS Grid 构建左侧表单与右侧画布的双栏布局。面板采用半透明背景、`backdrop-filter` 和阴影。

全屏是应用内部的沉浸式布局，而不是浏览器 Fullscreen API：

- 根节点增加 `page--fullscreen`
- `body` 增加 `is-fullscreen`
- 隐藏页面头部和新增表单
- 让画布区域占满 `100vh`
- 等待布局完成后触发 resize，并重新执行 `autoFit()`
- 按 `Escape` 可退出

## 11. 扩展开发建议

### 接入后端持久化

目前增删改仅修改 `devices` 内存状态。接入 API 时，可将 `addDevice`、`saveEdit`、`deleteDevice` 改为异步请求，在请求成功后再更新列表，并增加加载、错误和冲突处理。

### 支持设备拖拽

可为设备 `Konva.Group` 开启 `draggable`，在 `dragend` 中将 Konva 坐标转换回业务坐标，再向根组件发出更新事件。实现时需保留左下角坐标语义并处理边界约束。

### 大数据量优化

当前设备节点已经缓存，但每次数据变化仍会遍历整个数组。设备达到数万级时可考虑：

- 只更新发生变化的节点
- 按视口范围进行空间裁剪
- 使用空间索引加速搜索和碰撞判断
- 减少阴影、Tooltip 等高成本绘制效果
- 将静态设备与交互设备拆分到不同 Layer

## 12. 已知限制

- 页面刷新后，新增、编辑和删除结果不会保留。
- 新增和编辑表单固定限制在 `800000 × 200000` 范围，而数据加载与画布渲染本身支持动态边界和负坐标；如果业务需要负坐标编辑或完全动态范围，应统一校验规则。
- 搜索仅匹配原始数据中带显式 ID 的 Detail 设备，不搜索自动生成 ID 的 Macro 设备。
- `autoFit()` 根据设备范围计算，不直接按 SVG 背景范围适配；只有背景而没有设备时不会执行适配。
- 当前没有自动化测试、Lint、TypeScript 类型约束和错误上报。
- SVG Blob URL 在组件卸载时尚未调用 `URL.revokeObjectURL()` 回收。
- 字体通过 Google Fonts 在线加载；离线环境会回退到系统字体。

## 13. 部署说明

项目构建后是纯静态站点，可部署到 Nginx、对象存储或任意静态托管平台。部署时必须保证以下资源与站点根路径对应：

```text
/output_eq.json
/raw_layer.svg
/raw_layer_meta.json
```

代码当前使用以 `/` 开头的绝对路径请求资源。如果应用部署到域名子路径，需要同步配置 Vite 的 `base`，并将资源请求改为基于该 base 的路径。
