---
name: question-tree
description: 检测到用户当前agent为tree时即进入问题树模式。每次用户发言必须先走入口守卫，通过question工具让用户选择覆盖或连接，不得擅自修改树结构。Use ONLY when agent is tree.
---

# 问题树 (Question Tree)

检测到用户当前agent为tree时即进入问题树模式。

## 数据文件

树数据按「项目 → session」两级目录组织，存储在：
`D:\AppData\opencode\TreeData/<项目名>/<session-id>.json`

Session 切换时自动更换树。项目和 session 文件不存在时自动创建。

**`TreeData/` 目录下不得出现裸露的 .json 文件，所有 session 文件必须归于对应的项目文件夹内。**

文件结构：
```json
{
  "topics": {
    "话题名称": {
      "nodes": [
        { "id": 1, "question": "问题内容", "parentId": null },
        { "id": 2, "question": "问题内容", "parentId": 1 }
      ]
    }
  },
  "activeTopic": "当前活跃的话题名称"
}
```

## 项目 (Project)

**项目 = 你在哪个目录下启动了 opencode CLI。** 每个项目目录对应 `TreeData/` 下一个独立文件夹，存放该项目下所有 session 的树文件。切换工作目录即切换项目。

### 首次进入一个项目目录
- 检查 `TreeData/` 下是否存在以当前目录名命名的文件夹
- **不存在** → 该目录首次使用，必须调用 `question` 工具：
  - `question`: "检测到新项目，请确认项目名称："
  - 选项 1：`label`: `"<目录名>（推荐）"`, `description`: `"使用当前目录名作为项目名"`
  - 选项 2：`label`: `"自定义"`, `description`: `"手动输入项目名"`
  - 用户选自定义时继续追问输入名称
  - 确认后在 `TreeData/` 下创建对应文件夹
- **存在** → 直接使用该文件夹

### 切换项目
- 用户说"切换到项目 <名称>"时，后续 session 使用该项目的文件夹

## 话题 (Topic)

话题是 session 内部的逻辑分区，用于在一个 session 下组织不同的根节点树。
不同话题下的节点树相互独立，各自拥有独立的 ID 序列。
activeTopic 标记当前正在操作的话题。

### 首次使用（session 文件为空或 activeTopic 为空）
- 用户第一个问题时自动创建话题（以问题主题命名），设为 activeTopic
- 或用户明确说"新话题 <名称>"

### 创建话题
- 用户说"新话题 <名称>"或"创建话题 <名称>"，在 topics 下创建新条目，设为 activeTopic
- 未指定名称时询问用户

### 切换话题
- 用户说"切换到 <名称>"或"进入话题 <名称>"，将 activeTopic 改为指定话题
- 该话题不存在时询问是否创建

### 列出话题
- 用户说"有哪些话题"或"话题列表"，输出当前 session 下所有话题名

## 核心铁律

**在 tree 模式下，绝对禁止擅自修改树结构。** 无论用户提出任何问题，都必须先通过 `question` 工具让用户做出明确选择，确认后再执行。不允许跳过选择步骤直接操作树。

## 入口守卫（每次用户消息的第一步，不可跳过）

**收到用户的任何消息后，必须立即按以下顺序执行，不得做任何其他事情：**

1. **确定项目名**：获取当前工作目录名，检查 `TreeData/<目录名>/` 是否存在 → 不存在则按「项目」规则调用 question 工具确认项目名，创建文件夹
2. 读取当前 session 数据文件（`TreeData/<项目>/<session-id>.json`），文件不存在则创建
3. 检查当前 activeTopic 是否为空 → 空则先创建话题（以问题主题命名）
4. 检查当前 activeTopic 下的 nodes 是否为空：
   - **nodes 为空** → 立即跳转到「规则 1」
   - **nodes 非空** → 立即跳转到「规则 2」
5. **在完成 question 工具交互并更新树结构之前，禁止回答用户的实质问题，禁止输出任何与树操作无关的内容**

> 触发条件：用户说的任何话都视为"一条新问题"。包括追问、反问、闲聊——全部都需要走入口守卫。
> 仅有的例外：用户明确说"列出话题""有哪些话题""切换到话题""新话题""删除节点 [ID]"时，直接执行话题管理或节点删除操作，不需要走规则 1/2。

## 执行规则

### 规则 1：当前话题树为空 → 确认后创建根节点
- 读取当前 activeTopic 下的 nodes，若为空：
  - **必须调用 `question` 工具**，构建 1 个选项：
    - `label`: `"创建根节点"`
    - `description`: `"将「<用户问题>」作为首个问题添加到话题中"`
  - 参数：`question`: "当前话题树为空，是否将问题添加为根节点？"，`header`: "问题树"，`multiple`: false
  - 等待 question 返回后，创建根节点 `{ id: 1, question: "<用户问题>", parentId: null }`
  - 写入 session 文件
  - 缩进树形输出当前话题的问题树
  - 回答用户的问题
  - 结束

### 规则 2：当前话题树非空 → 两步选择流程（严格按顺序）

#### 第一步：选择操作模式（覆盖 or 连接）

- **必须调用 `question` 工具**，构建 2 个选项：
  - 选项 1：
    - `label`: `"🔁 覆盖"`
    - `description`: `"覆盖已有节点的问题内容（保留其子节点），避免树结构臃肿"`
  - 选项 2：
    - `label`: `"🔗 连接"`
    - `description`: `"将当前问题挂载为已有节点的子问题，或创建新根节点"`
- 参数：`question`: "请选择操作模式："，`header`: "问题树"，`multiple`: false
- 不回答用户问题，等待 question 返回

#### 第二步：根据模式选择目标节点

**如果选了「覆盖」：**
- 读取当前 activeTopic 下的全部节点
- **必须调用 `question` 工具**，每个已有节点作为一个选项：
  - `label`: `"[ID] 问题首行（≤40字）"`
  - `description`: `"覆盖此节点的问题内容"`
- 参数：`question`: "请选择要覆盖的节点："，`header`: "覆盖节点"，`multiple`: false
- 等待 question 返回后：
  - 将选中节点的 question 替换为用户的当前问题
  - 该节点的 id、parentId 和子节点关系保持不变
  - 写入 session 文件
  - 输出更新后的问题树
  - 回答用户的问题
  - 结束

**如果选了「连接」：**
- 读取当前 activeTopic 下的全部节点
- **必须调用 `question` 工具**，构建选项：
  - 每个已有节点作为一个选项：
    - `label`: `"[ID] 问题首行（≤40字）"`
    - `description`: `"挂载为子问题"`
  - 最后一个选项固定：
    - `label`: `"#新根节点"`
    - `description`: `"作为当前话题下独立的根节点"`
- 参数：`question`: "请选择挂载位置："，`header`: "问题树"，`multiple`: false
- 等待 question 返回后：
  - 选 [ID] → 以该节点为 parentId 创建子节点，新 ID = 当前话题最大 ID + 1
  - 选 #新根节点 → parentId 为 null，新 ID = 当前话题最大 ID + 1
  - 写入 session 文件
  - 缩进树形输出当前话题的问题树
  - 回答用户的问题
  - 结束

### 规则 3：删除/修改节点
- 用户要求删除或修改节点时，在当前 activeTopic 内操作，更新 session 文件
- 删除父节点 → 其子节点提升为根节点（parentId: null）
- 操作后自动输出更新后的问题树

### 规则 4：Session 切换
- 每次新 session 开始时自动读取 `D:\AppData\opencode\TreeData/<项目名>/<session-id>.json`
- 文件不存在时创建：`{ "topics": {}, "activeTopic": "" }`
- 如果需要确定或新建项目名，先通过 question 工具让用户选择/输入
- session 之间的树完全独立，互不影响
