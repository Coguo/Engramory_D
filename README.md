# Engramory_D — 双层记忆模型增强版

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)      [![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/)

一套**有主见、零基础设施、纯文件式**的智能体长期记忆协议,基于原版 Engramory 重新搭建,核心新增:**全局记忆库(两层模型)**——`user` 只进全局库、项目库只放本项目事实,跨项目与项目记忆物理分库。

> **原项目**:[Engramory (tinqiao-oss/engramory)](https://github.com/tinqiao-oss/engramory)
> 本仓库是从原版复制后**重新搭建的独立项目**,在原版 0.5.0 基础上新增 0.6.0 全局记忆库与 0.6.1 两层模型收敛。

> **⚠️ 声明:本项目仅供个人学习、研究使用，非商业项目，不用于任何商业用途。**

- 记忆 = 一堆小小的、人能直接读的 markdown 文件 + 一个每次会话都加载的索引(`MEMORY.md`)。
- 没有数据库、没有向量、没有服务器;真实记忆库保持 git-ignore。
- **状态:0.6.2 —— 实验性。** 索引上限有 `PreToolUse` hook 确定性兜底,但纪律本身靠常驻规则由模型遵守,尽力而为(见 [Skills/engramory/SKILL.md](Skills/engramory/SKILL.md) §8)。假设单写者 / 串行写入。
- **平台状态**:macOS 已实测部署验证(2026-08-24);Linux 与 macOS 同为 POSIX、逻辑一致(未单独实测);Windows 代码已按文档适配、未实测。三平台操作方案见 [§十](#十平台支持与操作方案claude-code)。
- **其他宿主(Codex / OpenClaw / 只读读取器等)接入尚未实测**,本 README 暂只描述 Claude Code 部署;后续实测后会更新。

---

## 一、两层记忆模型(核心)

**一句话:** 全局库是你的「个人档案」——跨项目通用,每个 agent 每个会话都读;项目库是当前项目的「工作笔记」——只属于本项目,只在项目里读。两者**物理分库**,`MEMORY.md` 索引各自独立计数(软提醒 150 行 / 20 KB,硬上限 200 行 / 25 KB),互不挤占。

### 1.1 各管什么

| | **全局库** `~/.engramory/` | **项目库** `<项目根>/memory/` |
|---|---|---|
| 类比 | 个人档案(简历 + 习惯 + 常用资源) | 当前项目的笔记本 |
| 放什么 | 你是谁、你跨项目怎么工作、跨项目常用资源 | 这个项目做到哪了、项目内的约定、项目内资源 |
| 类型 | 四类:`user` / `feedback` / `project` / `reference` | 三类:`project` / `feedback` / `reference`(**永不 `user`**) |
| 谁在读 | 每个会话、每个接入的 agent | 只当前项目 |
| 改一处会怎样 | 所有项目一起生效 | 只影响本项目 |

### 1.2 一条记忆该进哪个库——举例

| 这条记忆 | 进 |
|---|---|
| "用户是 Python 开发者、偏好中文交流" | 全局库 `user` |
| "用户写记忆前习惯先定库(跨项目→全局,本项目→项目)" | 全局库 `feedback` |
| "用户常用某个跨项目的工具 / 文档链接" | 全局库 `reference` |
| "本项目正在重构,当前做到目录结构调整" | 项目库 `project` |
| "本项目 hook 豁免了 `templates/MEMORY.md`" | 项目库 `reference` |
| "这个项目要用中文写文档" | 项目库 `feedback` |

### 1.3 为什么分两层

跨项目的事实如果塞进项目库,每个项目就要各复制一份,改一处别的项目还留着旧版本——这就是**漂移**。物理分库后:全局事实只写一处、处处生效;项目事实各归各,互不串味。

**`user` 只进全局库**正是这条原则的直接结果:谁是用户永远是跨项目事实,放项目库会每项目复制一份然后漂移。

### 1.4 回忆 / 写入纪律(见 [`rules-snippet.md`](rules-snippet.md))

- 任务开始读**两个**索引:全局 `~/.engramory/MEMORY.md` + 项目 `memory/MEMORY.md`,只打开看起来相关的那几个详情文件。
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
# macOS / Linux(macOS 已实测;Linux 逻辑一致)
python3 tools/engramory_init.py home                      # 创建 ~/.engramory/(默认)
python3 tools/engramory_init.py home --memory-root /path/to/store   # 迁到别处
```

```powershell
# Windows(命令按文档给出,未实测)
python tools\engramory_init.py home                       # 创建 %USERPROFILE%\.engramory\
python tools\engramory_init.py home --memory-root D:\path\to\store   # 迁到别处
```

> `python3` / `python` 均可换成绝对解释器路径(`python3 -c "import sys; print(sys.executable)"` 的输出),三平台最稳。
> 0.6.2 起 `home` 模式按本 fork 布局自检,索引模板用 `templates/MEMORY_global_template.md`(四类型,含 `user`)。

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
# macOS / Linux
mkdir -p ~/.engramory/templates
cp templates/*.md ~/.engramory/templates/
```

```powershell
# Windows
New-Item -ItemType Directory -Force $env:USERPROFILE\.engramory\templates | Out-Null
Copy-Item templates\*.md $env:USERPROFILE\.engramory\templates\
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
            "command": "<python 解释器,按平台见下表>",
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

**`command` 按平台选择**(hook 由 Claude Code 直接 spawn、**不经 shell**,shell 别名不生效):

| 平台 | `command` | 说明 |
|---|---|---|
| macOS | `python3` | 裸 `python` 常不存在或只是 zsh 别名,hook 里会静默失效 |
| Linux | `python3` | 同 macOS |
| Windows | `python` | 或 `py -3`;`args` 里的路径反斜杠在 JSON 中要写 `\\` |
| 三平台最稳 | 绝对路径 | `python3 -c "import sys; print(sys.executable)"` 的输出,绕开别名 / 版本差异 |

- **先备份**:改 `~/.claude/settings.json` 前先 `cp ~/.claude/settings.json ~/.claude/settings.json.bak`。
- **hook 在会话启动时读取** —— 注册后重启 Claude Code 会话才生效。
- **冒烟测试**(不依赖 Claude Code,直接喂 hook JSON):

  ```sh
  # 超限写入 → 应输出 permissionDecision: deny
  echo '{"tool_name":"Write","tool_input":{"file_path":"/绝对/路径/MEMORY.md","content":"# Index\n...(300 行)"}}' \
    | python3 hooks/engramory_index_guard.py
  # 非索引文件 → 应无输出(静默放行)
  echo '{"tool_name":"Edit","tool_input":{"file_path":"/tmp/foo.txt","content":"x"}}' \
    | python3 hooks/engramory_index_guard.py
  ```

  > 注意:payload 字段按工具区分 —— `Write` 用 `content`,`Edit` 用 `old_string` / `new_string`,
  > 否则 guard 视为"无编辑"而放行(拦截信号是输出 JSON 里的 `permissionDecision: "deny"`,不是退出码)。

### 3.5 创建两个 skill

把仓库 `Skills/` 下的两个 skill 复制到用户 skill 目录:

```sh
# macOS / Linux
cp -r Skills/engramory   ~/.claude/skills/engramory    # skill 一:完整协议规范
cp -r Skills/memory-init ~/.claude/skills/memory-init  # skill 二:项目库脚手架
```

```powershell
# Windows
Copy-Item -Recurse Skills\engramory   $env:USERPROFILE\.claude\skills\engramory
Copy-Item -Recurse Skills\memory-init $env:USERPROFILE\.claude\skills\memory-init
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
python3 tools/engramory_init.py home                    # 建全局库 ~/.engramory/(Windows: python)
python3 tools/engramory_check.py <MEMORY.md>            # 检查索引是否超上限
python3 tools/engramory_doctor.py <memory-root>         # 记忆库体检
```

> 需要 **Python 3.9+**。macOS / Linux 用 `python3`;Windows 用 `python`;最稳是两者都换成绝对解释器路径。其他宿主(codex / openclaw / 只读读取器)接入尚未实测,仅 `adapters/` 下有参考接线说明。

## 六、测试

```sh
python3 tests/test_tools.py          # 89 全绿(含 5 个 home 模式用例)
python3 tests/test_index_guard.py    # 35 全绿(含 6 个 IGNORE 豁免用例)
```

> macOS / Linux 用 `python3`;Windows 用 `python`。

## 七、与原项目的关系

- 本仓库由 [原版 Engramory(tinqiao-oss/engramory)](https://github.com/tinqiao-oss/engramory) 复制后重新搭建:**已清除原版 git 历史、无任何远程关联**,是一份全新的独立提交。
- 增强点:全局记忆库(`home` 模式)、两层模型、`memory-init` 项目库脚手架、`ENGRAMORY_INDEX_IGNORE` 豁免、`user` 只进全局库约定。改动详单 `20260806修改.md` 已归档到原仓库目录,不随本仓库分发。
- 原项目仍在上游维护:https://github.com/tinqiao-oss/engramory

## 八、安全与隐私

记忆库是**明文、未加密**的,任何本地进程都能读;`.gitignore` 只挡 git,挡不住云同步 / 系统备份 / 桌面搜索。**永远别把密钥的「值」写进记忆**(只记它「在哪」);尽量少写部分 PII,优先用指针。这条纪律未强制(没有 hook 扫描记忆内容)——当尽力而为。

## 九、许可证

MIT —— 见 [LICENSE](LICENSE)。原项目 [tinqiao-oss/engramory](https://github.com/tinqiao-oss/engramory) 同以 MIT 授权。

---

## 十、平台支持与操作方案(Claude Code)

| 平台 | 状态 | 说明 |
|---|---|---|
| **macOS** | ✅ **已实测**(2026-08-24) | 全流程部署验证:全局库初始化、hook 注册与冒烟(超限 deny / 压缩放行)、89 + 35 测试全绿 |
| **Linux** | ⚠️ 未单独实测 | 与 macOS 同为 POSIX,脚本逻辑一致(`_display_path` 走同一 else 分支);按 macOS 步骤操作即可 |
| **Windows** | ⚠️ 已适配、未实测 | 代码已按 Windows 适配;操作上有 4 个差异点,请在本机验证后更新本文档 |

### 三平台操作差异点

| 步骤 | macOS / Linux | Windows |
|---|---|---|
| Python 解释器 | `python3`(裸 `python` 常不存在) | `python` 或 `py -3` |
| 全局库路径 | `~/.engramory/` | `%USERPROFILE%\.engramory\` |
| settings.json | `~/.claude/settings.json` | `%USERPROFILE%\.claude\settings.json` |
| JSON 中的路径 | 正斜杠,无需转义 | 反斜杠要写 `\\` |
| `ENGRAMORY_INDEX_IGNORE` 完整路径 | `/Users/...` | `C:\Users\...`(JSON 中 `\\`) |

三平台最稳的做法:hook 的 `command` 与所有命令行一律用**绝对解释器路径**
(`python3 -c "import sys; print(sys.executable)"` 的输出),绕开 `python` / `python3`
差异与 shell 别名问题。

### 0.6.2 跨平台修复记录

- `engramory_init.py` 按本 fork 布局修正自检清单:模板改名(`MEMORY_global_template.md` /
  `MEMORY_project_template.md`)、`SKILL.md` 移入 `Skills/engramory/` 后,原自检找不到源文件会直接报错。
- `home` 模式索引改用 `MEMORY_global_template.md`(四类型含 `user`),项目库沿用项目模板(三类型)。
- 路径比较按平台显式分支归一化(`_display_path`):Windows `normcase` + resolve(大小写折叠、junction);
  macOS resolve(`/tmp → /private/tmp` 符号链接);Linux resolve。`home` 的 `~/` 显示同样先 resolve 家目录。
- 修复 win32 分支 `os.path.normcase()` 返回 `str` 导致 `relative_to` 崩溃的问题(包回 `Path(...)`);
  该分支已按 WindowsPath 语义验证,但**未在真实 Windows 机器上实测**(Windows 上 `Path` 才是 WindowsPath)。
- 审计结论:三个工具 + hook 均为纯 stdlib Python,无 shell 调用(`subprocess` 均传参数列表)、
  路径匹配用 `normcase` / `realpath`、写文件统一 `newline="\n"` —— 跨平台安全。

### macOS 部署验证记录(2026-08-24)

1. `python3 tools/engramory_init.py home` → 全局库就绪,显示 `memory root: ~/.engramory` ✓
2. hook 注册到 `~/.claude/settings.json`(`command` 用绝对路径 `/opt/anaconda3/bin/python3`,
   已先备份 `settings.json.bak-20260824`)✓
3. hook 冒烟:超限 Write / Edit → `permissionDecision: deny`;压缩编辑 / 非索引文件 → 静默放行 ✓
4. `python3 tests/test_tools.py` → 89 全绿;`python3 tests/test_index_guard.py` → 35 全绿 ✓
5. 注意:该机器 `python` 是 zsh 别名,hook 无 shell 环境不识别 —— 必须 `python3` 或绝对路径
