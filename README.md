[English](README.md) | **简体中文**

# Engramory_D — 双层记忆模型增强版

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)      [![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/)

一套**有主见、零基础设施、纯文件式**的智能体长期记忆协议,基于原版 Engramory 重新搭建,核心新增:**全局记忆库(两层模型)**——`user` 只进全局库、项目库只放本项目事实,跨项目与项目记忆物理分库。

> **原项目**:[Engramory (tinqiao-oss/engramory)](https://github.com/tinqiao-oss/engramory)
> 本仓库是从原版复制后**重新搭建的独立项目**,在原版 0.5.0 基础上新增 0.6.0 全局记忆库与 0.6.1 两层模型收敛。

- 记忆 = 一堆小小的、人能直接读的 markdown 文件 + 一个每次会话都加载的索引(`MEMORY.md`)。
- 没有数据库、没有向量、没有服务器;真实记忆库保持 git-ignore。
- **状态:0.6.1 —— 实验性。** 索引上限有 `PreToolUse` hook 确定性兜底,但纪律本身靠常驻规则由模型遵守,尽力而为(见 [Skills/engramory/SKILL.md](Skills/engramory/SKILL.md) §8)。假设单写者 / 串行写入。
- **其他宿主(Codex / OpenClaw / 只读读取器等)接入尚未实测**,本 README 暂只描述 Claude Code 部署;后续实测后会更新。

---

## 一、两层记忆模型(核心)

记忆分**两层**,`MEMORY.md` 索引各自独立计数(软提醒 150 行 / 20 KB,硬上限 200 行 / 25 KB),互不挤占:

| 层级 | 位置 | 索引 | 类型 | 谁在读 |
|---|---|---|---|---|
| **全局库** | `~/.engramory/` | `~/.engramory/MEMORY.md` | 四类型:`user` / `feedback` / `project` / `reference` | 每个会话、每个接入的 agent 都读 |
| **项目库** | `<项目根>/memory/` | `memory/MEMORY.md` | 三类型:`project` / `feedback` / `reference`(**永不创建 `user`**) | 只在当前项目 |

**关键约定:`user` 只进全局库。** 谁是用户永远是跨项目事实,放项目库会每项目复制一份然后漂移。全局库保留全部四类型;项目库只留 `feedback` / `project` / `reference`。

**回忆 / 写入纪律**(见 [`rules-snippet.md`](rules-snippet.md)):

- 任务开始读**两个**索引:全局 `~/.engramory/MEMORY.md` + 项目 `memory/MEMORY.md`,只打开 hook 看起来相关的那几个详情文件。
- 写入前**先定库**:跨项目事实 → 全局库;本项目事实 → 项目库;拿不准选项目库(全局索引每会话都加载、放错代价更高)。
- 一条事实 = 一个文件;写前查重、能改就不新增、发现错的就删;git / 项目说明 / 代码里已记录的不再记。
- 一个 `feedback` / `project` 记忆必须带 `Why:` 和 `How to apply:` 两行。

---

## 二、目录结构

```
Engramory_D/
├── CLAUDE.md                    # 本仓库的项目记忆绑定块(memory-init 写入)
├── rules-snippet.md             # 常驻规则片段 —— 部署时贴进全局 CLAUDE.md
├── Skills/
│   ├── engramory/SKILL.md       # 完整协议规范(两层模型、回忆/写入/上限)← skill 一
│   └── memory-init/SKILL.md     # 项目库脚手架 ← skill 二
├── tools/
│   ├── engramory_init.py        # 初始化助手(home / codex / openclaw / <host>-reader)
│   ├── engramory_check.py       # 索引上限检查
│   └── engramory_doctor.py      # 记忆库体检(结构、schema、上限)
├── hooks/
│   ├── engramory_index_guard.py # PreToolUse 硬卡口 hook(拦"变大"的索引编辑)
│   ├── INSTALL.md               # hook 安装 + 双库纪律 + memory-init 说明
│   └── settings.snippet.json    # hook 注册示例
├── templates/                   # 模板:全局/项目索引模板 + 示例(见 §四)
├── adapters/                    # 各宿主接线说明(参考,未实测)
└── tests/                       # test_tools.py(89) + test_index_guard.py(35)
```

---

## 三、Claude Code 部署流程

按顺序执行:**建全局库 → 复制模板 → 常驻规则 → 搭建 hook → 创建两个 skill**。第 3.6 步是每个项目可选的初始化。

### 3.1 建全局记忆库 `~/.engramory/`

```sh
python tools/engramory_init.py home                      # 创建 ~/.engramory/(默认)
python tools/engramory_init.py home --memory-root /path/to/store   # 迁到别处
```

结构:

```
~/.engramory/
├── MEMORY.md        # 索引(每次会话加载)
├── User/            # user 记忆(只进全局库)
├── Feedback/        # 跨项目反馈/纪律
├── Project/         # 跨项目状态
├── Reference/       # 跨项目资源指针
└── templates/       # 模板(第 3.2 步复制)
```

### 3.2 复制模板到 `~/.engramory/templates/`

把仓库 `templates/` 下的模板复制过去(全局/项目索引模板 + 示例):

```sh
mkdir -p ~/.engramory/templates
cp templates/*.md ~/.engramory/templates/
```

### 3.3 常驻规则 → 全局 `~/.claude/CLAUDE.md`

把 [`rules-snippet.md`](rules-snippet.md) 的 Memory 段写进用户级 `~/.claude/CLAUDE.md`,让每个项目每个任务都生效:

```markdown
## Memory (Engramory)

你有一个分**两层**的、基于文件的记忆:
- 全局库 `~/.engramory/`(索引 `MEMORY.md`):跨项目四类型记忆,
  由每个 agent 每个会话加载。
- 项目库 `<项目根>/memory/`(索引 `memory/MEMORY.md`):本项目记忆,
  由 `memory-init` 创建;永不创建 `user` 记忆。

任务开始:读两个索引;写入:先定库(跨项目→全局,本项目→项目)。
```

### 3.4 搭建索引上限 hook(硬卡口)

在 `~/.claude/settings.json` 注册 exec-form hook(片段见 [`hooks/settings.snippet.json`](hooks/settings.snippet.json)):

```jsonc
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "<你的python>",
            "args": ["<本仓库>/hooks/engramory_index_guard.py"],
            "env": {
              "ENGRAMORY_INDEX_IGNORE": "<可省略>"
            }
          }
        ]
      }
    ]
  }
}
```

- 只拦**让索引变大**的编辑;压缩 / 缩小一律放行。
- **豁免 `ENGRAMORY_INDEX_IGNORE`**(逗号分隔):完整路径 → 只按解析后身份匹配,豁免 `.../templates/MEMORY.md` **不会**连累同 basename 的真实索引;bare basename(如 `MEMORY.md`)→ 匹配任意目录下同名文件(**慎用**,会连豁免两层索引)。
- 按文件名 `MEMORY.md` 匹配 → 天然同时守护**两层**索引。

### 3.5 创建两个 skill

把仓库 `Skills/` 下的两个 skill 复制到用户 skill 目录:

```sh
cp -r Skills/engramory   ~/.claude/skills/engramory    # skill 一:完整协议规范
cp -r Skills/memory-init ~/.claude/skills/memory-init  # skill 二:项目库脚手架
```

### 3.6 每个项目:用 `memory-init` 建项目库(可选)

在每个项目根目录触发 `/memory-init`(或按其 [SKILL.md](Skills/memory-init/SKILL.md) 手动建):

```sh
# 在项目里触发 /memory-init
```

它做的事:检测项目根 → 若无 `memory/MEMORY.md` 则从项目模板建库 → 建 `memory/` + `memory/feedback/`、`memory/project/`、`memory/reference/` 三个类型子文件夹 → 在项目 `CLAUDE.md` 追加标记块绑定双库 → `.gitignore` 加 `memory/`。**幂等**,绝不触碰全局库,绝不建 `user` 注释。

## 四、模板路径

| 用途 | 路径 |
|---|---|
| 全局库索引模板(四类型,含 `user`) | `~/.engramory/templates/MEMORY_global_template.md` |
| 项目库索引模板(三类型,无 `user`) | `~/.engramory/templates/MEMORY_project_template.md` |
| 记忆示例(`feedback` / `project`) | `~/.engramory/templates/example-feedback.md`、`example-project.md` |
| 仓库内模板(部署时 3.2 复制到 `~/.engramory/templates/`) | [`templates/`](templates/):`MEMORY_global_template.md`、`MEMORY_project_template.md`、`example-feedback.md`、`example-project.md` |

---

## 五、工具用法

```sh
python tools/engramory_init.py home                    # 建全局库 ~/.engramory/
python tools/engramory_check.py <MEMORY.md>            # 检查索引是否超上限
python tools/engramory_doctor.py <memory-root>         # 记忆库体检
```

> 需要 **Python 3.9+**。其他宿主(codex / openclaw / 只读读取器)接入尚未实测,仅 `adapters/` 下有参考接线说明。

## 六、测试

```sh
python tests/test_tools.py          # 89 全绿(含 5 个 home 模式用例)
python tests/test_index_guard.py    # 35 全绿(含 6 个 IGNORE 豁免用例)
```

## 七、与原项目的关系

- 本仓库由 [原版 Engramory(tinqiao-oss/engramory)](https://github.com/tinqiao-oss/engramory) 复制后重新搭建:**已清除原版 git 历史、无任何远程关联**,是一份全新的独立提交。
- 增强点:全局记忆库(`home` 模式)、两层模型、`memory-init` 项目库脚手架、`ENGRAMORY_INDEX_IGNORE` 豁免、`user` 只进全局库约定。改动详单 `20260806修改.md` 已归档到原仓库目录,不随本仓库分发。
- 原项目仍在上游维护:https://github.com/tinqiao-oss/engramory

## 八、安全与隐私

记忆库是**明文、未加密**的,任何本地进程都能读;`.gitignore` 只挡 git,挡不住云同步 / 系统备份 / 桌面搜索。**永远别把密钥的「值」写进记忆**(只记它「在哪」);尽量少写部分 PII,优先用指针。这条纪律未强制(没有 hook 扫描记忆内容)——当尽力而为。

## 九、许可证

MIT —— 见 [LICENSE](LICENSE)。原项目 [tinqiao-oss/engramory](https://github.com/tinqiao-oss/engramory) 同以 MIT 授权。
