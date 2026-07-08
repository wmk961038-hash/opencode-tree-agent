# tree-agent

opencode 问题树 agent - 通过树形结构组织和追踪问题。

## 安装

```powershell
# 1. 克隆仓库
git clone https://github.com/yourname/opencode-tree-agent.git ~/opencode-tree-agent

# 2. 确保数据目录存在
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\skills\question-tree\trees"
```

### 配置 opencode

在 `~/.config/opencode/opencode.json` 中添加引用：

```json
{
  "skills": {
    "paths": ["~/opencode-tree-agent/skills"]
  },
  "plugin": [
    "~/opencode-tree-agent/tools/interactive.ts"
  ]
}
```

重启 opencode 使配置生效。

## 使用

在 opencode 中切换到 tree agent 即可使用。每次提问会自动构建问题树。

详见 [SKILL.md](./skills/question-tree/SKILL.md)。

## 目录结构

```
opencode-tree-agent/
├── skills/question-tree/SKILL.md   # 问题树执行规则
├── tools/interactive.ts            # 交互式选择工具
├── .gitignore
└── README.md
```

运行时数据存储在 `~/.config/opencode/skills/question-tree/trees/`，不入库。
