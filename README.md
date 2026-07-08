# tree-agent

opencode 问题树 agent - 通过树形结构组织和追踪问题。每次提问需先选择操作模式（覆盖/连接），再选择目标节点，树结构完全由用户控制。

## 安装

### 1. 克隆仓库

```powershell
git clone <repo-url> D:\Projects\INVEN\opencode-tree-agent
```

### 2. 创建数据目录

```powershell
New-Item -ItemType Directory -Force -Path "D:\AppData\opencode\TreeData"
```

### 3. 配置 opencode

在 `~/.config/opencode/opencode.json` 中添加：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["D:/Projects/INVEN/opencode-tree-agent/skills"]
  }
}
```

重启 opencode 使配置生效。

## 使用

在 opencode 中切换到 tree agent 即可使用。

### 操作流程

1. 用户提问 → 入口守卫自动触发
2. 新项目目录首次使用 → 确认项目名
3. 空树 → 确认创建根节点
4. 非空树 → 两步选择：
   - **覆盖**：用当前问题替换已有节点内容（保留子节点），优化臃肿树结构
   - **连接**：挂载为子问题或创建新根节点
5. 树更新完成后才回答用户问题

## 目录结构

```
opencode-tree-agent/
├── .gitignore
├── README.md
└── skills/
    └── question-tree/
        └── SKILL.md
```

## 数据文件

树数据存储在 `D:\AppData\opencode\TreeData/`：

```
TreeData/
├── project-a/
│   ├── session-01.json
│   └── session-02.json
└── project-b/
    └── session-03.json
```

数据文件不入库，每个用户独立生成。
