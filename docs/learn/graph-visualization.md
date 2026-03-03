# LightRAG WebUI 图可视化功能详解

## 目录

1. [概述](#概述)
2. [技术栈](#技术栈)
3. [架构设计](#架构设计)
4. [前端实现](#前端实现)
5. [后端API](#后端api)
6. [数据流程](#数据流程)
7. [核心功能](#核心功能)
8. [文件结构](#文件结构)

---

## 概述

LightRAG WebUI 提供了一个功能强大的交互式知识图谱可视化界面，允许用户：

- 查看和浏览知识图谱中的实体（节点）和关系（边）
- 搜索和定位特定节点
- 编辑节点和边的属性
- 动态扩展/修剪节点
- 多种布局算法切换
- 缩放、平移、全屏等交互操作

---

## 技术栈

### 核心渲染引擎

| 库名称 | 版本 | 用途 |
|--------|------|------|
| **Sigma.js** | ^3.0.2 | WebGL 图渲染引擎，支持大规模图的高性能渲染 |
| **@react-sigma/core** | ^5.0.6 | Sigma.js 的 React 封装，提供 React 集成 |
| **Graphology** | ^0.26.0 | 图数据结构和操作库 |

### 布局算法

| 库名称 | 用途 |
|--------|------|
| **@react-sigma/layout-circular** | 圆形布局 |
| **@react-sigma/layout-circlepack** | 圆形打包布局 |
| **@react-sigma/layout-force** | 力导向布局 |
| **@react-sigma/layout-forceatlas2** | Force Atlas 2 布局（推荐用于大型图） |
| **@react-sigma/layout-noverlap** | 防重叠布局 |
| **@react-sigma/layout-random** | 随机布局 |

### 渲染程序

| 库名称 | 用途 |
|--------|------|
| **@sigma/node-border** | 节点边框渲染 |
| **@sigma/edge-curve** | 曲边渲染 |

### 其他工具

| 库名称 | 用途 |
|--------|------|
| **MiniSearch** | 客户端模糊搜索 |
| **Zustand** | 图状态管理 |
| **seedrandom** | 确定性随机数生成 |

---

## 架构设计

### 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          LightRAG WebUI                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────┐      ┌──────────────────────────────────┐   │
│  │   GraphViewer     │      │       GraphControl               │   │
│  │   (主容器)         │◄─────┤       (事件处理/布局)            │   │
│  │                   │      │                                  │   │
│  │  ┌─────────────┐  │      │  - 点击/悬停事件                 │   │
│  │  │ SigmaContainer│ │      │  - 布局算法应用                 │   │
│  │  └─────────────┘  │      │  - 节点/边高亮                   │   │
│  │                   │      │  - 主题适配                      │   │
│  │  ┌─────────────┐  │      └──────────────────────────────────┘   │
│  │  │ 子组件:      │  │                                            │
│  │  │ - GraphSearch│ │      ┌──────────────────────────────────┐   │
│  │  │ - LayoutCtrl │ │─────┤       useLightragGraph            │   │
│  │  │ - ZoomCtrl   │ │      │       (数据获取/处理)             │   │
│  │  │ - Properties │ │      │                                  │   │
│  │  │ - Legend     │ │      │  - fetchGraph()                 │   │
│  │  └─────────────┘  │      │  - createSigmaGraph()           │   │
│  └───────────────────┘      │  - 节点扩展/修剪                 │   │
│                            └──────────────────────────────────┘   │
│                                     │                               │
│                                     ▼                               │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    useGraphStore (Zustand)                    │  │
│  │                    ────────────────────────                   │  │
│  │  - rawGraph: RawGraph      // 原始图数据                      │  │
│  │  - sigmaGraph: DirectedGraph // Sigma图实例                   │  │
│  │  - selectedNode/focusedNode // 选择/聚焦状态                   │  │
│  │  - searchEngine: MiniSearch // 搜索引擎                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                     │                               │
│                                     ▼                               │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                      API Layer                                │  │
│  │                      lightrag.ts                              │  │
│  │                                                              │  │
│  │  queryGraphs(label, maxDepth, maxNodes)                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Backend API (FastAPI)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  GET  /graphs?label={label}&max_depth={d}&max_nodes={n}             │
│  → 返回指定标签的子图数据                                            │
│                                                                     │
│  GET  /graph/label/list                                             │
│  → 返回所有图标签                                                   │
│                                                                     │
│  GET  /graph/label/popular?limit={n}                                │
│  → 返回按节点度排序的热门标签                                        │
│                                                                     │
│  GET  /graph/label/search?q={query}&limit={n}                      │
│  → 模糊搜索标签                                                     │
│                                                                     │
│  POST /graph/entity/edit                                            │
│  → 更新实体属性（支持重命名和合并）                                  │
│                                                                     │
│  POST /graph/relation/edit                                          │
│  → 更新关系属性                                                     │
│                                                                     │
│  POST /graph/entity/create                                          │
│  → 创建新实体                                                       │
│                                                                     │
│  POST /graph/relation/create                                        │
│  → 创建新关系                                                       │
│                                                                     │
│  POST /graph/entities/merge                                         │
│  → 合并多个实体                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      LightRAG Core                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  rag.chunk_entity_relation_graph.get_knowledge_graph()              │
│  └─→ 图存储层 (NetworkX / Neo4j / PostgreSQL等)                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 前端实现

### 1. 数据结构

#### RawNodeType（原始节点类型）

```typescript
type RawNodeType = {
  id: string                      // 节点唯一标识
  labels: string[]                // 节点标签列表
  properties: Record<string, any> // 节点属性
  size: number                    // 节点大小（根据度计算）
  x: number, y: number           // 节点坐标
  color: string                   // 节点颜色（按实体类型）
  degree: number                  // 节点度（连接数）
}
```

#### RawEdgeType（原始边类型）

```typescript
type RawEdgeType = {
  id: string                      // 边唯一标识
  source: string                  // 源节点ID
  target: string                  // 目标节点ID
  type?: string                   // 边类型
  properties: Record<string, any> // 边属性（包含 weight, keywords 等）
  dynamicId: string               // Sigma图中的动态ID
}
```

### 2. 核心组件

#### GraphViewer.tsx - 主图查看器

**位置**: `lightrag_webui/src/features/GraphViewer.tsx`

主要功能：
- 创建 `SigmaContainer` 作为图渲染容器
- 配置 Sigma 设置（渲染程序、标签颜色、边样式等）
- 管理子组件布局
- 处理主题切换时的图重渲染

关键代码：
```typescript
const createSigmaSettings = (isDarkTheme: boolean): Partial<SigmaSettings> => ({
  defaultNodeType: 'default',
  defaultEdgeType: 'curvedNoArrow',
  renderEdgeLabels: false,
  edgeProgramClasses: {
    arrow: EdgeArrowProgram,
    curvedArrow: EdgeCurvedArrowProgram,
    curvedNoArrow: createEdgeCurveProgram()
  },
  nodeProgramClasses: {
    default: NodeBorderProgram,
    circel: NodeCircleProgram,
    point: NodePointProgram
  },
  labelColor: {
    color: isDarkTheme ? labelColorDarkTheme : labelColorLightTheme,
    attribute: 'labelColor'
  },
  // ...
})
```

#### GraphControl.tsx - 交互控制

**位置**: `lightrag_webui/src/components/graph/GraphControl.tsx`

主要功能：
- 注册图交互事件（点击、悬停、拖拽）
- 实现 nodeReducer 和 edgeReducer（节点/边高亮效果）
- 应用布局算法
- 根据权重动态调整边的大小

事件处理：
```typescript
const events = {
  enterNode: (event) => { setFocusedNode(event.node) },
  leaveNode: (event) => { setFocusedNode(null) },
  clickNode: (event) => { setSelectedNode(event.node) },
  clickStage: () => { clearSelection() }
}
```

#### useLightragGraph.tsx - 数据获取钩子

**位置**: `lightrag_webui/src/hooks/useLightragGraph.tsx`

主要功能：
1. **数据获取**: `fetchGraph(label, maxDepth, maxNodes)`
   - 调用后端 `/graphs` API
   - 计算节点度（degree）
   - 根据度计算节点大小
   - 根据 `entity_type` 分配颜色

2. **图创建**: `createSigmaGraph(rawGraph)`
   - 创建 Graphology UndirectedGraph 实例
   - 添加节点和边
   - 根据权重计算边大小

3. **节点扩展**: `handleNodeExpand(nodeId)`
   - 获取扩展节点的子图（深度2，最大1000节点）
   - 将新节点添加到现有图中
   - 更新节点和边的大小

4. **节点修剪**: `handleNodePrune(nodeId)`
   - 删除节点及其连接边
   - 自动删除会变得孤立的节点

### 3. 状态管理

#### useGraphStore (Zustand)

**位置**: `lightrag_webui/src/stores/graph.ts`

```typescript
interface GraphState {
  // 选择状态
  selectedNode: string | null
  focusedNode: string | null
  selectedEdge: string | null
  focusedEdge: string | null

  // 图数据
  rawGraph: RawGraph | null           // 原始图数据
  sigmaGraph: DirectedGraph | null     // Sigma图实例
  sigmaInstance: any | null            // Sigma实例引用

  // 搜索
  searchEngine: MiniSearch | null

  // UI状态
  moveToSelectedNode: boolean
  isFetching: boolean
  graphIsEmpty: boolean

  // 类型颜色映射
  typeColorMap: Map<string, string>

  // 方法
  setSelectedNode, setFocusedNode, clearSelection
  updateNodeAndSelect, updateEdgeAndSelect
  triggerNodeExpand, triggerNodePrune
  // ...
}
```

### 4. 节点颜色映射

**位置**: `lightrag_webui/src/utils/graphColor.ts`

节点根据 `entity_type` 属性自动分配颜色：

```typescript
const NODE_TYPE_COLORS: Record<string, string> = {
  person:      '#4169E1',  // 蓝色
  creature:    '#bd7ebe',  // 紫色
  organization: '#00cc00', // 绿色
  location:    '#cf6d17',  // 橙色
  event:       '#00bfa0',  // 青色
  concept:     '#e3493b',  // 红色
  method:      '#b71c1c',  // 深红
  content:     '#0f558a',  // 深蓝
  data:        '#0000ff',  // 纯蓝
  artifact:    '#4421af',  // 深紫
  naturalobject: '#b2e061', // 浅绿
  other:       '#f4d371',  // 黄色
  unknown:     '#b0b0b0'   // 灰色
}
```

---

## 后端API

### 图查询 API

**位置**: `lightrag/api/routers/graph_routes.py`

#### 获取子图

```http
GET /graphs?label={label}&max_depth={depth}&max_nodes={nodes}
```

**参数**:
- `label`: 起始节点标签
- `max_depth`: 最大深度（默认3）
- `max_nodes`: 最大节点数（默认1000）

**返回**:
```json
{
  "nodes": [
    {
      "id": "entity_id",
      "labels": ["Entity Name"],
      "properties": {
        "entity_id": "Entity Name",
        "entity_type": "ORGANIZATION",
        "description": "...",
        "source_id": "chunk1<SEP>chunk2"
      }
    }
  ],
  "edges": [
    {
      "id": "source-target",
      "source": "source_id",
      "target": "target_id",
      "properties": {
        "keywords": "relationship_type",
        "weight": 1.0,
        "description": "..."
      }
    }
  ],
  "is_truncated": false
}
```

#### 标签相关 API

| 端点 | 方法 | 描述 |
|------|------|------|
| `/graph/label/list` | GET | 获取所有图标签 |
| `/graph/label/popular` | GET | 获取热门标签（按度排序） |
| `/graph/label/search` | GET | 模糊搜索标签 |

#### 实体编辑 API

| 端点 | 方法 | 描述 |
|------|------|------|
| `/graph/entity/edit` | POST | 更新实体属性（支持重命名和合并） |
| `/graph/relation/edit` | POST | 更新关系属性 |
| `/graph/entity/create` | POST | 创建新实体 |
| `/graph/relation/create` | POST | 创建新关系 |
| `/graph/entities/merge` | POST | 合并多个实体 |

---

## 数据流程

### 1. 初始加载流程

```
用户输入查询标签
       │
       ▼
GraphViewer 挂载
       │
       ▼
useLightragGraph.fetchGraph()
       │
       ├─→ 调用 API: GET /graphs?label=...&max_depth=3&max_nodes=1000
       │
       ▼
后端处理
       │
       ├─→ rag.get_knowledge_graph()
       │   └─→ 从图存储获取子图数据
       │
       ▼
返回数据 { nodes, edges, is_truncated }
       │
       ▼
前端处理
       │
       ├─→ validateGraph() - 验证数据完整性
       ├─→ 计算节点度和大小
       ├─→ 分配节点颜色
       │
       ▼
createSigmaGraph()
       │
       ├─→ 创建 UndirectedGraph 实例
       ├─→ 添加节点（带位置、大小、颜色）
       ├─→ 添加边（带权重）
       │
       ▼
更新 useGraphStore
       │
       ├─→ setRawGraph(rawGraph)
       └─→ setSigmaGraph(sigmaGraph)
       │
       ▼
Sigma 渲染图
```

### 2. 节点编辑流程

```
用户点击节点 → PropertiesView 显示
       │
       ▼
用户修改属性（如 description）
       │
       ▼
EditablePropertyRow 调用 updateEntity()
       │
       ├─→ API: POST /graph/entity/edit
       │   {
       │     "entity_name": "...",
       │     "updated_data": { "description": "new value" },
       │     "allow_rename": false
       │   }
       │
       ▼
后端处理
       │
       ├─→ rag.aedit_entity()
       │   ├─→ 更新图存储
       │   ├─→ 更新向量存储
       │   └─→ 返回更新后的实体数据
       │
       ▼
前端更新
       │
       ├─→ useGraphStore.updateNodeAndSelect()
       │   ├─→ 更新 rawGraph
       │   ├─→ 更新 sigmaGraph
       │   └─→ 触发重新渲染
```

### 3. 节点扩展流程

```
用户点击属性面板中的扩展按钮
       │
       ▼
triggerNodeExpand(nodeId)
       │
       ▼
useLightragGraph.handleNodeExpand()
       │
       ├─→ 获取要扩展的节点信息
       ├─→ 调用 API: GET /graphs?label={nodeLabel}&max_depth=2&max_nodes=1000
       │
       ▼
获取扩展子图数据
       │
       ├─→ 识别新节点（未在当前图中的）
       ├─→ 识别新边
       │
       ▼
添加到当前图
       │
       ├─→ 计算新节点位置（极坐标分布在扩展节点周围）
       ├─→ 更新节点大小（基于新的度）
       ├─→ 更新边大小（基于新的权重范围）
       │
       ▼
重新构建搜索索引
       │
       └─→ resetSearchEngine()
```

---

## 核心功能

### 1. 节点搜索

**组件**: `GraphSearch.tsx`

**实现方式**:
- 使用 **MiniSearch** 进行客户端模糊搜索
- 支持前缀匹配和模糊匹配（fuzzy: 0.2）
- 中间内容匹配补充（当结果少于5个时）

```typescript
const newSearchEngine = new MiniSearch({
  idField: 'id',
  fields: ['label'],
  searchOptions: {
    prefix: true,
    fuzzy: 0.2,
    boost: { label: 2 }
  }
})
```

### 2. 布局算法

**组件**: `LayoutsControl.tsx`

| 算法 | 特点 | 适用场景 |
|------|------|----------|
| **Circular** | 节点排列成圆环 | 少量节点，清晰展示 |
| **Circlepack** | 层级圆环打包 | 有明显层级结构的图 |
| **Random** | 随机分布 | 初始状态 |
| **Noverlaps** | 防止节点重叠 | 后处理优化 |
| **Force Directed** | 力导向布局 | 通用，中等规模图 |
| **Force Atlas** | Force Atlas 2 | 大规模图，效果最佳 |

力导向参数配置：
```typescript
settings: {
  attraction: 0.0003,   // 吸引力
  repulsion: 0.02,      // 排斥力
  gravity: 0.02,        // 重力
  inertia: 0.4,         // 惯性
  maxMove: 100          // 最大移动距离
}
```

### 3. 属性编辑

**组件**: `PropertiesView.tsx`, `EditablePropertyRow.tsx`

**可编辑属性**:

| 元素 | 可编辑属性 |
|------|-----------|
| 节点 | `entity_id`, `entity_type`, `description` |
| 边 | `keywords`, `description` |

**实体重命名/合并**:
```typescript
updateEntity(
  "Old Name",
  { "entity_id": "New Name" },
  allow_rename: true,    // 允许重命名
  allow_merge: true      // 允许合并到已存在的实体
)
```

### 4. 节点扩展和修剪

**节点扩展**:
- 从当前节点获取深度为2的子图
- 新节点按极坐标分布在扩展节点周围
- 自动更新节点/边大小

**节点修剪**:
- 删除选中节点
- 自动删除会变得孤立的关联节点
- 不允许删除所有节点

---

## 文件结构

```
lightrag_webui/src/
├── features/
│   └── GraphViewer.tsx              # 主图查看器组件
│
├── components/graph/
│   ├── GraphControl.tsx             # 图事件控制
│   ├── GraphSearch.tsx              # 节点搜索
│   ├── GraphLabels.tsx              # 标签选择器
│   ├── LayoutsControl.tsx           # 布局控制
│   ├── ZoomControl.tsx              # 缩放控制
│   ├── FullScreenControl.tsx        # 全屏控制
│   ├── PropertiesView.tsx           # 属性查看面板
│   ├── EditablePropertyRow.tsx      # 可编辑属性行
│   ├── PropertyEditDialog.tsx       # 属性编辑对话框
│   ├── MergeDialog.tsx              # 合并对话框
│   ├── FocusOnNode.tsx              # 节点聚焦
│   ├── Legend.tsx                   # 图例
│   ├── LegendButton.tsx             # 图例按钮
│   ├── Settings.tsx                 # 设置按钮
│   └── SettingsDisplay.tsx          # 设置显示
│
├── hooks/
│   └── useLightragGraph.tsx         # 图数据获取和处理钩子
│
├── stores/
│   └── graph.ts                     # 图状态管理 (Zustand)
│
├── utils/
│   └── graphColor.ts                # 节点颜色映射
│
└── api/
    └── lightrag.ts                  # API 调用封装

lightrag/api/routers/
└── graph_routes.py                  # 图相关 API 路由
```

---

## 总结

LightRAG WebUI 的图可视化系统采用了以下设计原则：

1. **性能优先**: 使用 WebGL (Sigma.js) 渲染，支持大规模图
2. **模块化**: 组件职责清晰，易于维护和扩展
3. **状态管理**: Zustand 集中管理图状态
4. **交互丰富**: 支持搜索、编辑、扩展、修剪等多种操作
5. **主题适配**: 支持明暗主题切换
6. **布局灵活**: 提供多种布局算法适应不同场景

该系统实现了从后端图存储到前端可视化的完整数据流，提供了流畅的用户体验。