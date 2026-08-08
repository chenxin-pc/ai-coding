# Codex Workspace 与 Worktree 使用指南

## 1. 背景

在使用 Codex 处理多项目代码时，常见会遇到两类问题：

- 一个需求需要同时查看或修改多个项目，例如 `ka-solution` 依赖 `ka-common`。
- 同一个项目需要同时处理多个任务，例如一个需求开发和一个紧急缺陷修复并行进行。

这两类问题分别对应两个概念：

- `Workspace`：定义 Codex 本次会话能访问和操作哪些目录。
- `Worktree`：为同一个 Git 仓库创建多份独立工作目录，用于隔离不同任务。

简单理解：

```text
Workspace = Codex 本次能操作哪些项目
Worktree  = 同一个项目为不同任务开出的独立工作目录
```

## 2. Workspace 是什么

`Workspace` 解决的是：**这次 Codex 能看到哪些目录、主要在哪个目录工作。**

在 Codex CLI 中，常用两个参数定义工作范围：

```powershell
codex -C 主项目目录 --add-dir 额外项目目录
```

其中：

- `-C`：主工作目录，Codex 默认在这里执行命令、查找文件、修改代码。
- `--add-dir`：额外可访问目录，通常用于加入依赖项目、公共模块或需要联动修改的其他仓库。

示例：

```powershell
codex -C D:\workspace\ka-solution --add-dir D:\workspace\ka-common
```

含义：

- `ka-solution` 是本次任务的主项目。
- `ka-common` 是依赖项目，Codex 可以查看或必要时修改。
- 其他项目如 `ka-order`、`ka-operation` 不会默认进入范围。

## 3. Workspace 适用场景

### 3.1 主项目依赖公共项目

场景：

```text
ka-solution 依赖 ka-common
```

推荐启动方式：

```powershell
codex -C D:\workspace\ka-solution --add-dir D:\workspace\ka-common
```

适用情况：

- 主要修改 `ka-solution`。
- 需要查看 `ka-common` 中的公共类、工具类、DTO、枚举、接口定义。
- 可能需要同步修改 `ka-common`。

注意：

- 如果只查看 `ka-common`，不一定要给 `ka-common` 建需求分支。
- 如果要修改 `ka-common`，则 `ka-common` 也需要单独建分支。

### 3.2 一个需求横跨多个独立项目

场景：

```text
需求 X 同时需要修改 ka-solution 和 ka-order
```

推荐启动方式：

```powershell
codex -C D:\workspace\ka-solution --add-dir D:\workspace\ka-order
```

如果主要逻辑在 `ka-order`，也可以反过来：

```powershell
codex -C D:\workspace\ka-order --add-dir D:\workspace\ka-solution
```

建议：

- 两个项目各自建需求分支。
- 分支名使用同一个需求标识，方便提测、CodeReview 和合并追踪。
- 各项目分别提交、分别发起 MR/PR，并在描述中互相关联。

示例：

```powershell
cd D:\workspace\ka-solution
git checkout test
git pull
git checkout -b ai/202607/20260709_xxx需求

cd D:\workspace\ka-order
git checkout test
git pull
git checkout -b ai/202607/20260709_xxx需求
```

### 3.3 不建议直接把父目录作为主工作目录

如果直接执行：

```powershell
codex -C D:\workspace
```

Codex 可能会把父目录下所有项目都纳入搜索和判断范围，例如：

```text
D:\workspace\
  ka-operation\
  ka-consumer\
  ka-common\
  ka-solution\
  ka-order\
  ka-waybill-router\
```

风险：

- 搜索范围变大，可能读取更多无关文件。
- 多项目中同名类、同名配置、同名接口容易造成误判。
- Token 消耗可能增加。
- Codex 更容易把无关项目卷入任务。

推荐原则：

```text
主项目用 -C
依赖项目或联动项目用 --add-dir
无关项目不要加入
```

## 4. Worktree 是什么

`Worktree` 解决的是：**同一个 Git 仓库同时处理多个任务，不用频繁切分支，也不让改动混在一起。**

没有 worktree 时，一个项目通常只有一份工作目录：

```text
ka-solution\
```

如果正在开发需求 A，突然要修复缺陷 B，就可能遇到：

- 当前改动未提交，不方便切换分支。
- 切换分支后 IDE、依赖、编译缓存可能变化。
- 两个任务的改动容易混在一起。
- Codex 或其他 agent 无法在同一个目录中安全并行处理多个任务。

使用 worktree 后，可以为同一个仓库创建多份工作目录：

```text
ka-solution-main\          -> test
ka-solution-feature-a\     -> ai/202607/feature-a
ka-solution-bugfix-b\      -> ai/202607/bugfix-b
```

每个目录都是同一个 Git 仓库的一个工作树，但可以 checkout 不同分支。

## 5. Worktree 适用场景

### 5.1 同一个项目并行处理多个任务

场景：

```text
ka-solution 同时处理需求 A 和缺陷 B
```

可以创建：

```text
ka-solution-feature-a\
ka-solution-bugfix-b\
```

含义：

- Agent A 在 `ka-solution-feature-a` 中处理需求 A。
- Agent B 在 `ka-solution-bugfix-b` 中处理缺陷 B。
- 本地原始目录仍可保留在 `test` 或其他稳定分支。

### 5.2 Codex 后台任务和本地开发互不影响

可以让 Codex 在一个 worktree 中处理任务，同时开发者在原始目录继续工作。

好处：

- 不需要来回切分支。
- 未提交改动不会互相污染。
- 可以同时打开多个 IDE 窗口。
- 适合并行开发、紧急修复、试验性改动。

## 6. Workspace 与 Worktree 的区别

| 对比项 | Workspace | Worktree |
| --- | --- | --- |
| 解决问题 | Codex 本次能访问哪些项目 | 同一个 Git 仓库如何并行处理多个任务 |
| 面向对象 | 多目录、多项目、多仓库 | 单个 Git 仓库 |
| 典型命令 | `codex -C ... --add-dir ...` | `git worktree add ...` |
| 典型场景 | `ka-solution` 依赖 `ka-common` | `ka-solution` 同时开发需求 A 和缺陷 B |
| 是否合并仓库 | 否 | 否 |
| 是否隔离分支 | 不负责隔离分支 | 负责隔离不同任务的工作目录 |

一句话总结：

```text
Workspace 负责定义 Codex 的工作范围。
Worktree 负责隔离同一仓库的不同任务。
```

## 7. 多仓库场景下的推荐做法

### 7.1 ka-solution 依赖 ka-common

如果只改 `ka-solution`：

```powershell
cd D:\workspace\ka-solution
git checkout test
git pull
git checkout -b ai/202607/20260709_solution需求

codex -C D:\workspace\ka-solution --add-dir D:\workspace\ka-common
```

如果 `ka-solution` 和 `ka-common` 都要改：

```powershell
cd D:\workspace\ka-common
git checkout test
git pull
git checkout -b ai/202607/20260709_common需求

cd D:\workspace\ka-solution
git checkout test
git pull
git checkout -b ai/202607/20260709_solution需求

codex -C D:\workspace\ka-solution --add-dir D:\workspace\ka-common
```

如果 `ka-solution` 本地构建需要引用改后的 `ka-common`，通常需要先安装公共包：

```powershell
cd D:\workspace\ka-common
mvn clean install -DskipTests

cd D:\workspace\ka-solution
mvn clean compile
```

### 7.2 ka-solution 和 ka-order 无依赖，但同一需求都要改

推荐：

```powershell
cd D:\workspace\ka-solution
git checkout test
git pull
git checkout -b ai/202607/20260709_xxx需求

cd D:\workspace\ka-order
git checkout test
git pull
git checkout -b ai/202607/20260709_xxx需求

codex -C D:\workspace\ka-solution --add-dir D:\workspace\ka-order
```

说明：

- 这是 workspace 场景，不是 worktree 必需场景。
- 两个项目独立提交、独立合并。
- 需求、测试、上线需要统一管理。

## 8. 是否需要在任务结束后解除关联

如果只是通过以下方式启动 Codex：

```powershell
codex -C D:\workspace\ka-solution --add-dir D:\workspace\ka-order
```

一般不需要解除关联。

原因：

- `--add-dir` 只是本次 Codex 会话的访问范围。
- 关闭会话后，不会把两个 Git 仓库永久绑定。
- 两个项目仍然是独立仓库。

需要关注的是：

- 各仓库是否还有未提交改动。
- 各仓库分支是否已经提交、推送、合并。
- 如果手动创建过临时 worktree，需求结束后可以按需清理。

检查命令：

```powershell
cd D:\workspace\ka-solution
git status

cd D:\workspace\ka-order
git status
```

如果创建过 Git worktree，可以查看：

```powershell
git worktree list
```

确认没有未提交改动后，再清理不需要的临时 worktree。

## 9. 推荐决策规则

### 9.1 什么时候用 Workspace

使用 workspace 的场景：

- 一个项目依赖另一个项目。
- 一个需求需要同时修改多个独立仓库。
- Codex 需要跨项目查代码、查接口、查枚举、查公共方法。
- 希望控制 Codex 的搜索范围，避免扫描整个父目录。

推荐格式：

```powershell
codex -C 主项目 --add-dir 关联项目
```

### 9.2 什么时候用 Worktree

使用 worktree 的场景：

- 同一个仓库要并行处理多个任务。
- 当前目录有未提交改动，但需要切到另一个分支处理新任务。
- 需要让不同 Codex 线程或 agent 在同一项目的不同分支上并行工作。
- 想把后台任务和本地开发隔离开。

推荐原则：

```text
一个任务，一个分支。
一个分支，一个 worktree。
一个 agent，在自己的 worktree 里工作。
```

## 10. 最佳实践

- 不要把所有同级项目都加入 Codex 工作范围。
- 主项目使用 `-C`，依赖或联动项目使用 `--add-dir`。
- 多仓库联动时，每个仓库都独立建分支、独立提交、独立 MR/PR。
- 同一需求跨多个仓库时，分支名建议使用同一个需求标识。
- 同一个仓库并行多个任务时，使用 worktree 隔离。
- 不同 worktree 不要 checkout 同一个分支。
- 如果两个 agent 修改同一批文件，最终合并仍可能产生冲突。
- 任务完成后，`--add-dir` 不需要清理；临时 worktree 可按需清理。

## 11. 参考资料

- OpenAI Codex App Worktrees：<https://developers.openai.com/codex/app/worktrees>
- Git Worktree 文档：<https://git-scm.com/docs/git-worktree>
