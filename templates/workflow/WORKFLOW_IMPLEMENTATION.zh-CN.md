# 工作流模版实现指南

本文按构建顺序详细解析 **Workflow** 模版的全部实现细节：先挂载编辑器，再加入工作流特有的图形、UI、交互逻辑，最后接入执行模拟器，帮助你在自己的 tldraw 应用中完整复刻同样的体验。

## 1. 挂载具备工作流能力的编辑器

1. **注册自定义图形与绑定。** 在挂载 `<Tldraw />` 时传入工作流节点与连线的 `ShapeUtil`，并同时注册连线绑定工具，这样编辑器才能识别并持久化这些自定义图形。【F:templates/workflow/src/App.tsx†L22-L25】【F:templates/workflow/src/App.tsx†L74-L99】
2. **覆盖核心 UI。** 用 `WorkflowToolbar` 替换默认工具栏，隐藏菜单面板，并在未选中非工作流图形时隐藏样式面板；同时通过 `InFrontOfTheCanvas` 插槽挂载画布上的拾取器与工作流覆盖层。【F:templates/workflow/src/App.tsx†L27-L62】
3. **初始化编辑器行为。** 确保至少存在一个节点、启用吸附，将自定义 `PointingPort` 状态节点挂到选择工具上，保持连线始终位于节点下方，并阻止对工作流图形修改透明度。【F:templates/workflow/src/App.tsx†L80-L99】

> **提示：** 像模版一样把 `editor` 暴露到 `window`，可以在开发阶段调试自定义交互。【F:templates/workflow/src/App.tsx†L80-L83】

## 2. 工作流工具栏与菜单覆盖

1. **填充工作流工具。** 在 `tools` override 中为每种节点定义注册一个专用工具。选中后会在视口中心创建节点，或通过 tldraw 的 helper 支持拖拽创建，然后自动关闭数学菜单。【F:templates/workflow/src/components/WorkflowToolbar.tsx†L65-L98】
2. **垂直工具栏编排。** 渲染出的工具栏将选择工具、工作流节点（含数学菜单与特化的滑杆/条件/地震工具）以及标准绘图图形分组展示，突出节点创建入口。【F:templates/workflow/src/components/WorkflowToolbar.tsx†L101-L140】
3. **节点创建助手。** `createNodeShape` 在历史记录中做标记，按所选节点配置实例化图形，基于投放位置重新居中，并选中图形以便用户立即操作。【F:templates/workflow/src/components/WorkflowToolbar.tsx†L37-L63】

## 3. 节点图形系统

1. **图形元数据。** `NodeShapeUtil` 声明 `'node'` 类型，属性为 `{ node, isOutOfDate }`，关闭原生缩放/旋转控件，并返回由主体矩形与各个端口圆形组成的几何数据，以便手柄避开边界。【F:templates/workflow/src/nodes/NodeShapeUtil.tsx†L24-L108】
2. **渲染逻辑。** 组件读取实时输出值与执行状态，在执行中时切换 CSS class，绘制节点标题、可选输出读数，并通过下文提到的 `NodeBody` 渲染不同类型的主体内容。【F:templates/workflow/src/nodes/NodeShapeUtil.tsx†L151-L189】
3. **选中指示。** 选中指示器会在高亮时遮罩端口区域，同时绘制可见的端口标记，提升节点被选中时的反馈效果。【F:templates/workflow/src/nodes/NodeShapeUtil.tsx†L111-L149】
4. **节点定义。** 每种节点类型都继承抽象的 `NodeDefinition`，需要提供标题、图标、端口布局、输出计算、异步执行以及 React 主体组件。定义会按编辑器实例缓存，并通过 `getNodeDefinition`、`getNodeTypePorts`、`executeNode` 等辅助函数获取。【F:templates/workflow/src/nodes/types/shared.tsx†L23-L120】【F:templates/workflow/src/nodes/nodeTypes.tsx†L1-L91】【F:templates/workflow/src/nodes/nodeTypes.tsx†L92-L140】
5. **主体复用组件。** `NodeRow`、`NodeInputRow` 等组件负责渲染可编辑输入，并在被上游连线时自动只读；`NodeValue` 用于格式化实时输出或在分支被阻塞时显示占位符。【F:templates/workflow/src/nodes/types/shared.tsx†L73-L197】

## 4. 端口与连线状态

1. **端口描述与缓存。** 通过 `createComputedCache` 为每个节点缓存端口几何与连线信息，方便绘制、命中检测和数据传递。输入端口会从已连接的输出端读取数值，输出端则依据节点定义生成结果（包括 `STOP_EXECUTION` 标记）。【F:templates/workflow/src/nodes/nodePorts.tsx†L1-L126】
2. **图遍历。** `getAllConnectedNodes` 会遍历工作流图，用于检测环路与聚合同一区域的节点，依赖前述缓存的连线数据。【F:templates/workflow/src/nodes/nodePorts.tsx†L128-L156】
3. **端口组件。** 渲染交互端口的圆点，在拖拽期间高亮符合条件的端口，并在指针按下时将事件交由自定义的 `PointingPort` 状态节点处理，开启连线流程。【F:templates/workflow/src/ports/Port.tsx†L1-L189】
4. **端口 UI 状态。** `portState` 存在于每个编辑器实例的 `EditorAtom` 中，`updatePortState` 统一更新逻辑，确保拖拽时能在画布上统一高亮提示。【F:templates/workflow/src/ports/portState.ts†L1-L28】【F:templates/workflow/src/utils.ts†L12-L41】
5. **命中检测。** `getPortAtPoint` 会在指针位置查找最近端口（遵循端口方向与可选的边距），并返回几何信息与现有连线，确保拖拽逻辑能维护单输入限制。【F:templates/workflow/src/ports/getPortAtPoint.tsx†L1-L36】【F:templates/workflow/src/ports/getPortAtPoint.tsx†L37-L49】
6. **指向工具。** `PointingPort` 扩展了 tldraw 的状态机：从输出端拖拽会创建新连线（或在输入仅能单连时复用已有连线），点击则会在右侧生成新节点并通过画布拾取器自动连线；`onPointerMove` 与 `onClick` 负责在拖拽过程中管理连线创建、弹出拾取器及取消后的清理。【F:templates/workflow/src/ports/PointingPort.tsx†L1-L205】

## 5. 连线图形行为

1. **图形工具。** `ConnectionShapeUtil` 定义贝塞尔曲线几何，提供两端手柄，关闭吸附以免影响布局，并实时读取绑定获取端点位置。【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L39-L130】【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L397-L421】
2. **手柄交互。** 拖动手柄会查询附近端口，在进入或离开端口时高亮目标并更新绑定，同时通过查看已连接节点防止形成环路；如果松开时没有命中端口，则在端点处弹出拾取器，或在连线失去绑定时删除之。【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L132-L205】【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L205-L275】
3. **中心手柄。** 当连线双端均已绑定时，会在中点显示缩放相关的操作点；点击后以“middle”模式打开拾取器，利用 `insertNodeWithinConnection` 把新节点插入连线中。【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L282-L377】
4. **视觉反馈。** 当上游节点输出 `STOP_EXECUTION` 时，连线会渲染成失活样式，清楚区分被阻断的条件分支。【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L297-L329】
5. **绑定辅助。** `ConnectionBindingUtil` 负责保持连线端点与端口同步，触发节点级的 `onPortConnect/onPortDisconnect` 回调，并提供查询或修改绑定的工具函数，是拖拽交互与编程式图编辑的核心。【F:templates/workflow/src/connection/ConnectionBindingUtil.tsx†L20-L193】
6. **层级维护。** `keepConnectionsAtBottom` 监听图形变更，持续重新排序连线，确保它们位于节点下层，避免线条遮挡节点 UI。【F:templates/workflow/src/connection/keepConnectionsAtBottom.tsx†L1-L99】
7. **连线插入流程。** `insertNodeWithinConnection` 驱动“在连线上插入节点”的体验：展示拾取器、创建并摆放新节点、重连绑定、为下游节点腾出空间，并通过动画更新布局。【F:templates/workflow/src/connection/insertNodeWithinConnection.tsx†L1-L168】

## 6. 画布拾取器

1. **状态存储。** 拾取器的可见性与回调保存在每个编辑器实例的 `EditorAtom` 中。【F:templates/workflow/src/components/OnCanvasComponentPicker.tsx†L23-L33】
2. **对话框定位。** 借助 `useQuickReactor` hook 在连接端点（起点、终点或中点）附近定位 Radix 的无样式对话框，通过“连线空间 → 页面空间 → 视口空间”的转换实时跟随相机移动。【F:templates/workflow/src/components/OnCanvasComponentPicker.tsx†L61-L135】
3. **菜单内容。** 拾取器列出与工具栏同源的数学与逻辑节点定义，保证所有创建入口一致。【F:templates/workflow/src/components/OnCanvasComponentPicker.tsx†L35-L58】
4. **选择处理。** 选择节点时会调用保存的 `onPick`，在端点位置创建节点并重新绑定原有连线（或双连线），然后关闭对话框。【F:templates/workflow/src/components/OnCanvasComponentPicker.tsx†L138-L185】

## 7. 工作流执行模拟器

1. **执行图。** 当用户点击播放时，`ExecutionGraph` 会快照从所选起始节点可达的子图，跟踪每个节点的状态（`waiting/executing/executed`），并在依赖节点产生数值后递归执行下游节点；遇到 `STOP_EXECUTION` 会阻断后续路径。【F:templates/workflow/src/execution/ExecutionGraph.tsx†L1-L167】
2. **状态管理。** `executionState` 通过 `EditorAtom` 存储当前正在运行的执行图，在启动新运行前终止已有执行，并在结束后清空状态。【F:templates/workflow/src/execution/executionState.ts†L1-L49】
3. **画布覆盖层。** `WorkflowRegions` 会把相连节点聚合为区域，绘制带播放/停止按钮的覆盖框，在缩放过远时隐藏，并在相机移动时实时更新位置。【F:templates/workflow/src/components/WorkflowRegions.tsx†L1-L174】
4. **节点反馈。** 执行过程中 `NodeShape` 会把 `isOutOfDate` 设为 `true`，使 UI 暂时降低输出亮度，执行完成后再恢复；连线也会读取该标记以控制失活样式。【F:templates/workflow/src/nodes/NodeShapeUtil.tsx†L151-L189】【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L297-L329】

## 8. 支撑工具与样式约定

1. **几何常量。** 节点宽度、标题高度、端口半径、间距等共享尺寸集中在 `constants.tsx`，确保图形、覆盖层与插入逻辑保持一致。【F:templates/workflow/src/constants.tsx†L1-L12】
2. **透明度守卫。** `disableTransparency` 注册副作用，把工作流图形的 `opacity` 固定为 1，避免样式面板意外降低透明度导致可读性下降。【F:templates/workflow/src/disableTransparency.tsx†L1-L17】
3. **EditorAtom 帮助函数。** 基于 tldraw `atom` API 的薄封装，为拾取器、端口状态、执行状态提供按编辑器实例隔离的响应式存储，互不干扰。【F:templates/workflow/src/utils.ts†L1-L41】
4. **索引工具。** `utils.ts` 中的其他 helper 用于管理分数索引列表，沿用了 tldraw 排序图形的方式；若你需要围绕编辑器状态构建有序集合，这些工具非常实用。【F:templates/workflow/src/utils.ts†L44-L85】

---

按照以上步骤依次实现：挂载带工作流覆盖的编辑器、定义节点与连线行为、通过自定义状态管理端口交互、提供画布拾取器，再叠加执行模拟器，就能复刻模版完整的工作流体验，或在此基础上拓展自己的节点式应用。
