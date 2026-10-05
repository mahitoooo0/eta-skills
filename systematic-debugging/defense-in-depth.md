# Defense-in-Depth Validation（纵深防御式校验）

## Overview（概述）

当你修一个由非法数据引起的 bug 时，在一处加校验会感觉很足够。但那个单一检查会被不同的代码路径、重构或 mock 绕过。

**核心原则：** 在数据经过的每一层都校验。让 bug 在结构上不可能发生。

## Why Multiple Layers（为什么多层）

单层校验："我们修好了这个 bug"
多层校验："我们让这个 bug 不可能发生"

不同的层抓不同的情况：
- 入口校验抓住大多数 bug
- 业务逻辑抓住边界情况
- 环境守卫防止特定上下文的危险
- 调试日志在其它层失效时帮忙

## The Four Layers（四层）

### Layer 1: Entry Point Validation（入口点校验）
**目的：** 在 API 边界拒绝明显非法的输入

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... 继续
}
```

### Layer 2: Business Logic Validation（业务逻辑校验）
**目的：** 确保数据对这个操作来说是合理的

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... 继续
}
```

### Layer 3: Environment Guards（环境守卫）
**目的：** 在特定上下文中阻止危险操作

```typescript
function gitInit(directory) {
  // 测试环境里，拒绝在临时目录之外 git init
  if (运行环境标记 NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... 继续
}
```

### Layer 4: Debug Instrumentation（调试埋点）
**目的：** 为事后取证捕获上下文

```typescript
function gitInit(directory) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... 继续
}
```

## Applying the Pattern（套用这个模式）

当你发现一个 bug：

1. **追踪数据流** —— 坏值从哪来？在哪用？
2. **标出所有检查点** —— 列出数据经过的每一个点
3. **在每一层加校验** —— 入口、业务、环境、调试
4. **逐层测试** —— 试着绕过第 1 层，验证第 2 层能抓住它

## Example from Session（来自会话的例子）

Bug：空的 `projectDir` 导致在源码目录里 `git init`

**数据流：**
1. 测试 setup → 空字符串
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` 在 `process.cwd()` 里运行

**加的四层：**
- Layer 1: `Project.create()` 校验非空/存在/可写
- Layer 2: `WorkspaceManager` 校验 projectDir 非空
- Layer 3: `WorktreeManager` 在测试中拒绝在 tmpdir 之外 git init
- Layer 4: git init 前的栈追踪日志

**结果：** 1847 个测试全部通过，bug 无法再复现

## Key Insight（关键洞见）

四层都必要。测试期间，每一层都抓住了其它层漏掉的 bug：
- 不同的代码路径绕过了入口校验
- mock 绕过了业务逻辑检查
- 不同平台上的边界情况需要环境守卫
- 调试日志识别出了结构性误用

**不要只停在一个校验点。** 在每一层都加检查。
