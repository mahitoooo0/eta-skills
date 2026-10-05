# 远端 CI 验证指南

本地装不动 SDK 或构建太慢时，用 GitHub Actions 当验证机。

## 派发前必须确认的三件事

1. **workflow 文件在目标 ref 上存在**：`workflow_dispatch` 要求该 workflow 存在于所指定的分支；只注册在默认分支是不够的。
   ```bash
   gh api "repos/{o}/{r}/actions/workflows" --jq '.workflows[].path'
   gh api "repos/{o}/{r}/contents/.github/workflows/<name>.yml?ref=<branch>" --jq .sha
   ```
2. **该 workflow 有 `workflow_dispatch` 触发器**：只有 `on: push` 的 workflow 无法手动派发。
3. **分支隔离副作用**：Actions 缓存（`actions/cache`）按分支作用域。依赖缓存产物的构建（签名 keystore、依赖包）在其他分支上会 miss，可能产生不同结果——出包类构建固定在专用分支跑。

## 派发与等待

```bash
gh workflow run <file.yml> --repo {o}/{r} --ref <branch> [-f key=value]
# 拿 run id（等几秒让它出现）
gh run list --repo {o}/{r} --branch <branch> --limit 1
# 阻塞等待完成
gh run watch <id> --repo {o}/{r} --exit-status --interval 30
```

## 失败定位

```bash
gh run view <id> --repo {o}/{r} --log-failed    # 只看失败步骤日志
gh run view <id> --repo {o}/{r}                 # 总览每步状态
```

修一次重跑一次，用日志说话，不重复提交「碰运气」的改动。

## 判断「验证通过」的最低标准

- 编译任务：目标模块 compile 成功。
- 测试任务：相关测试类全部 PASSED；新增行为有对应测试。
- 不要把「run 变绿」直接说成「功能正确」——只声明日志里实际验证到的范围。
