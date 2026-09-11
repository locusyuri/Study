---
name: smart-commit
description: >
  智能 Git 提交助手。自动检测所有未提交文件，按类型/模块智能分组，
  生成符合规范的中文提交信息，分组提交。适合项目代码批量提交场景。
license: MIT
compatibility: git 2.0+
metadata:
  author: CatMono
  version: "1.0.0"
allowed-tools: Bash(git:*) Read Write Edit Glob Grep
user_invocable: true
disable_model_invocation: false
---

# Smart Commit — 智能 Git 提交

自动检测未提交文件 → 智能分组 → 生成中文提交信息 → 分组提交。

## 工作流程

### Step 1：检测未提交文件

运行以下命令获取所有未跟踪和已修改文件：

```bash
git status --porcelain
```

### Step 2：读取改动详情

为每个有改动的文件读取具体 diff：

```bash
# 工作区改动
git diff <file>
# 暂存区改动（如果有）
git diff --cached <file>
```

### Step 3：智能分组

根据以下规则将文件分组（按优先级匹配，匹配即归入该组）：

#### 分组优先级（从高到低）

| 优先级 | 组名 | 匹配规则 |
|--------|------|----------|
| 1 | **feat** (新功能) | 模式匹配：`src/*.rs` `crates/*/src/**/*.rs` 中新增的函数/模块/命令 |
| 2 | **fix** (修复) | 文件名含 `fix`，或 diff 中出现 `bug` `fix` `patch` `crash` `leak` |
| 3 | **docs** (文档) | 路径含 `docs/`，或后缀为 `.md` `.txt` `.rst` |
| 4 | **config** (配置) | 路径含 `.toml` `.json` `.yaml` `.yml` `.ini` `.conf` `.gitignore` `.env*` |
| 5 | **refactor** (重构) | diff 行数 > 文件总行数 30%（大规模重排），且非新增文件 |
| 6 | **test** (测试) | 路径含 `tests/` `test/` `*_test.rs` `*.spec.*` `*_test.go` |
| 7 | **style** (样式) | 仅含空格/缩进/空行/注释修改，无逻辑变更 |
| 8 | **chore** (杂务) | 以上都不匹配的剩余文件 |

**合并规则**：如果某组仅 1-2 个小文件（<20 行改动），尝试合并到最近的同类型组。

#### 附加分组信号（根据 diff 内容检测）：

**文件变更类型判定（每个分组额外输出）：**
- `new file` → 标记为 `(新建)`
- `deleted`  → 标记为 `(删除)`
- `renamed`  → 标记为 `(重命名)`
- `modified` → 标记为 `(修改)`

### Step 4：生成中文提交信息

每组生成一条规范的提交信息：

#### 提交信息格式

```
<type>(<scope>): <简短中文描述>

<详细说明（可选，仅当改动复杂时）>
```

#### type 取值

| 值 | 中文含义 | 适用场景 |
|----|---------|---------|
| feat | 新功能 | 新增功能、命令、模块 |
| fix | 修复 | 修复 bug、崩溃、性能问题 |
| docs | 文档 | 文档、注释、README |
| refactor | 重构 | 代码重构、重命名、模块重组 |
| test | 测试 | 新增或修改测试 |
| style | 样式 | 格式化、代码风格（非语义修改） |
| chore | 杂务 | 依赖、构建、CI、配置、其他 |

#### scope 取值

从文件路径自动推断：

| 路径模式 | scope |
|----------|-------|
| `crates/hit-cli/**` | cli |
| `crates/hit-core/**` | core |
| `crates/hit-shim/**` | shim |
| `crates/hit-bucket/**` | bucket |
| `crates/hit-common/**` | common |
| `crates/hit-uninstaller/**` | uninstaller |
| `crates/hit-plugin/**` | plugin |
| `docs/**` | docs |
| `src/**` (单 crate) | app |
| 其他/混合 | general |

#### 描述规范

- 中文，简洁明了（不超过 50 字）
- 动词开头：`添加` `修复` `更新` `重构` `优化` `移除` `迁移`
- 不结尾句号

**示例：**
```
feat(cli): 添加 search 命令支持正则搜索

fix(core): 修复安装路径包含空格时崩溃的问题

docs: 更新项目结构文档和 API 参考

refactor(bucket): 重构 bucket 索引逻辑，拆分全局索引模块

chore: 更新 Cargo.toml 依赖版本
```

### Step 5：分组提交

对每个分组依次执行：

```bash
# 暂存该组所有文件
git add <file1> <file2> ...

# 提交（两种方式任选）
# 方式一：直接使用 -m（适合简短提交信息）
# ⚠️ 注意：Windows cmd 下 git commit -m "message" 时，cmd 会剥除外层双引号
#   并按空格拆分参数，导致后续单词被 git 解析为 pathspec 而报错。
#   解决方案：推荐使用方式二（临时文件 + git commit -F）代替。
git commit -m "<type>(<scope>): <中文描述>"

# 方式二：使用临时文件（适合含详细说明的多行提交信息）
# 临时文件应放在项目目录下（如 .git/ 目录），避免跨分区权限问题
# ⚠️ 注意：Windows cmd 下 echo "text" > file 会把双引号原样写入文件，
#   导致 commit message 包含字面双引号（git log 显示为 \"text\"）。
#   务必用不带引号的 echo 命令：
echo <type>(<scope>): <中文描述> > .git/SMART_COMMIT_MSG.txt
echo. >> .git/SMART_COMMIT_MSG.txt
echo <详细说明> >> .git/SMART_COMMIT_MSG.txt
git commit -F .git/SMART_COMMIT_MSG.txt
# 使用后删除临时文件
del .git\SMART_COMMIT_MSG.txt
```

**规则**：
- 每个分组独立提交，不要一次提交所有文件
- 按照分组优先级顺序提交（feat → fix → docs → config → refactor → test → style → chore）
- 如果某组只有一个文件且改动很小（<5 行），可以与其逻辑相邻的组合并提交
- 若使用临时文件，必须放在项目目录内（如 `.git/`），且提交后立即删除

### Step 6：验证

提交后运行 `git log --oneline -5` 确认提交记录正确。

## 错误处理

- 若无未提交文件：提示"没有未提交的更改，无需提交"
- 若 git 命令失败：显示错误并中止，不进行部分提交
- 若暂存区已有内容（`git diff --cached` 有输出）：先提示用户，按用户确认继续
- 若某组文件在 add 时冲突：跳过该组并报告，继续处理其他组
