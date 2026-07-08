# Skill: question-tree

# 问题树 (Question Tree)

检测到用户当前agent为tree时即进入问题树模式。

## 数据文件

每个 session 独立一个 JSON 文件，存储在全局数据目录下：
`~/.config/opencode/skills/question-tree/trees/<session-id>.json`

Session 切换时自动更换树。文件不存在时自动创建空文件。

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

## 执行规则

### 规则 1：当前话题树为空 → 首个问题自动创建根节点
- 读取当前 activeTopic 下的 nodes，若为空：
  - 直接创建根节点 `{ id: 1, question: "<用户问题>", parentId: null }`
  - 写入 session 文件
  - 缩进树形输出当前话题的问题树
  - 回答用户的问题
  - 结束

### 规则 2：当前话题树非空 → 强制挂载流程（严格按顺序）

**步骤 ① - question 工具选父节点（必须）**
- 读取当前 activeTopic 下的全部节点
- **必须调用 `question` 工具**（不是文本输出），构建选项：
  - 每个已有节点作为一个选项：
    - `label`: `"[ID] 问题首行（≤40字）"`
    - `description`: `"挂载为子问题"`
  - 最后一个选项固定：
    - `label`: `"#新根节点"`
    - `description`: `"作为当前话题下独立的根节点"`
- 参数：`question`: "请选择挂载位置："，`header`: "问题树"，`multiple`: false
- 不回答用户问题，等待 question 返回

**步骤 ② - 创建节点**
- 选 [ID] → 以该节点为 parentId 创建子节点，分配新 ID = 当前话题最大 ID + 1
- 选 #新根节点 → parentId 为 null，新 ID = 当前话题最大 ID + 1
- 写入 session 文件

**步骤 ③ - 输出完整问题树**
- Read 确认写入无误
- 用缩进树形输出：
  ```
  【当前话题: 微积分-重积分】
  ├── [1] 根问题
  │   ├── [3] 子问题
  │   └── [4] 子问题
  └── [2] 根问题
  ```

**步骤 ④ - 回答用户的问题**

### 规则 3：删除/修改节点
- 用户要求删除或修改节点时，在当前 activeTopic 内操作，更新 session 文件
- 删除父节点 → 其子节点提升为根节点（parentId: null）
- 操作后自动输出更新后的问题树

### 规则 4：Session 切换
- 每次新 session 开始时自动读取 `~/.config/opencode/skills/question-tree/trees/<session-id>.json`
- 文件不存在时创建：`{ "topics": {}, "activeTopic": "" }`
- session 之间的树完全独立，互不影响
