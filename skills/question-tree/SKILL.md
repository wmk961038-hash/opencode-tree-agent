# Skill: question-tree

# 问题树 (Question Tree)

检测到用户当前agent为tree时即进入问题树模式。

## 数据文件

每个项目使用独立子目录，每个会话使用独立 JSON 文件。存储于全局数据目录下：
`~/.config/opencode/skills/question-tree/trees/<project-slug>/<session-id>.json`

目录结构：
```
trees/
  <project-slug>/
    <session-id>.json
```

文件结构：
```json
{
  "topic": "会话主题",
  "nodes": [
    { "id": 1, "question": "问题内容", "parentId": null, "timestamp": "..." },
    { "id": 2, "question": "问题内容", "parentId": 1, "timestamp": "..." }
  ]
}
```

## ⚠️ 项目和会话双重隔离

**每个项目有独立子目录，每个会话有独立 .json 文件。** 严禁跨项目、跨会话读取或展示不属于当前会话的节点。

- 进入树模式后，按以下流程确定文件：
  1. 根据用户对话主题确定 project slug（简短英文，如 `"calculus"`）。如果 `trees/<project-slug>` 目录已存在，复用该项目；否则创建该子目录
  2. **每次新对话均为独立 session**——为该会话生成一个新的 session id（如 `session-01`），在项目子目录下创建 `<session-id>.json`，**绝不复用已有 session 文件**
  3. 即使同一项目下有多个 session 文件，当前对话只操作当前 session 的 .json 文件
- 所有读取、写入、展示操作**只操作当前会话的 .json 文件**
- 永远不要读取或引用同一个项目中其他 session 文件或其他项目子目录中的数据
- 不使用 `activeSession` 或 `activeTopic` 概念——每次对话天然是新 session 新文件

## 会话独立原则

同一项目下的多个对话会创建多个 session 文件（如 `session-01.json`、`session-02.json`...），**每个文件完全独立**，互不读取。即使同一用户在不同时间谈论同一项目，不同对话的问题树也**必须严格隔离**，各自从根节点开始。

## 执行规则

### 规则 1：当前 session 文件为空或不存在 → 首个问题自动创建根节点
- 进入树模式后，当前对话是一个新 session，自动创建新文件：
  - 确定 project slug，确保 `trees/<project-slug>/` 子目录存在
  - 生成新 session id，创建 `trees/<project-slug>/<session-id>.json`
- 用户第一个问题自动作为根节点（parentId: null），分配 ID=1
- 写入 session 文件
- Read 确认，缩进树形输出当前会话的问题树
- 回答用户的问题

### 规则 2：当前 session 文件非空 → 强制挂载流程（严格按顺序）

**步骤 ① - question 工具选父节点（必须）**
- 读取当前 session 文件下的全部节点
- **必须调用 `question` 工具**（不是文本输出），构建选项：
  - 每个已有节点作为一个选项：
    - `label`: `"[ID] 问题首行（≤40字）"`
    - `description`: `"挂载为子问题"`
  - 最后一个选项固定：
    - `label`: `"#新根节点"`
    - `description`: `"作为独立的根节点"`
- 参数：`question`: "请选择挂载位置："，`header`: "问题树"，`multiple`: false
- 不回答用户问题，等待 question 返回

**步骤 ② - 创建节点**
- 选 [ID] → 以该节点为 parentId 创建子节点，分配新 ID = 最大 ID + 1
- 选 #新根节点 → parentId 为 null，新 ID = 最大 ID + 1
- 写入当前 session 文件

**步骤 ③ - 输出完整问题树**
- Read 确认写入无误
- 用缩进树形输出：
  ```
  问题树:
  ├── [1] 根问题
  │   ├── [3] 子问题
  │   └── [4] 子问题
  └── [2] 根问题
  ```

**步骤 ④ - 回答用户的问题**

### 规则 3：删除/修改节点
- 用户要求删除或修改节点时，在当前 session 文件内操作
- 删除父节点 → 其子节点提升为根节点（parentId: null）
- 操作后自动输出更新后的问题树

### 规则 4：Session 切换
- 每次新对话自动创建新的 session 文件
- 文件不存在时创建：`{ "topic": "", "nodes": [] }`
- Session 之间的树完全独立，互不影响

## 示例

项目「微积分」—— 会话1（文件 trees/calculus/session-01.json）:
用户: `三重积分是什么？`
→ 新项目 calculus，新目录 trees/calculus/
→ 新会话，创建 session-01.json
→ 创建 [1] 根节点
→ 输出:
问题树:
└── [1] 三重积分是什么？
→ 回答

用户: `那二重积分呢？`
→ 弹出选项: [1] 三重积分是什么？ / #新根节点
→ 用户选 [1]
→ 创建 [2] 子节点，parentId=1
→ 输出:
问题树:
└── [1] 三重积分是什么？
    └── [2] 那二重积分呢？
→ 回答

---

项目「微积分」—— 会话2（第二天，重新对话）:
用户: `什么是导数？`
→ 项目 calculus 已存在，复用目录 trees/calculus/
→ 🔴 新对话 = 新 session，创建 session-02.json（绝不复用 session-01.json）
→ 创建 [1] 根节点
→ 输出:
问题树:
└── [1] 什么是导数？
→ 回答
→ session-02.json 中完全看不到 session-01.json 的三重积分/二重积分节点

---

项目「线性代数」（完全不同的话题，新对话）:
用户: `什么是向量？`
→ 新项目 linear-algebra，创建 trees/linear-algebra/
→ 新会话，创建 trees/linear-algebra/session-01.json
→ 创建 [1] 根节点
→ 输出:
问题树:
└── [1] 什么是向量？
→ 回答
