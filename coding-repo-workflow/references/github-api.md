# GitHub API 操作指南

所有示例中 `{o}/{r}` 表示 owner/repo。认证方式：`gh` CLI（已登录即免 token）或 `curl -H "Authorization: Bearer $GITHUB_TOKEN"`。

## 读

```bash
# 文件树（一次拿全部路径）
gh api "repos/{o}/{r}/git/trees/{branch}?recursive=1" --jq '.tree[].path'

# 读单个文件（base64 输出需解码）
gh api "repos/{o}/{r}/contents/{path}" --jq .content | base64 -d > local_file

# 指定 ref
gh api "repos/{o}/{r}/contents/{path}?ref=<branch>"

# 分支头 commit sha
gh api "repos/{o}/{r}/git/ref/heads/{branch}" --jq .object.sha
```

注意：contents API 单文件上限约 1MB，超限改用 `git/blobs` 端点（先从 tree 里拿 blob sha，再 `gh api repos/{o}/{r}/git/blobs/{sha}`）。

## 写：单文件（Contents API）

```bash
sha=$(gh api "repos/{o}/{r}/contents/{path}" --jq .sha)  # 文件已存在时必填；新文件可省
gh api -X PUT "repos/{o}/{r}/contents/{path}" \
  -f message="commit message" \
  -f content="$(base64 -w0 local_file)" \
  -f branch="{branch}" \
  -f sha="$sha"
```

## 写：多文件单提交（Git Data API）

把多个文件的改动合成一个 commit，避免多次 push 触发多次构建：

```bash
BRANCH=target
BASE=$(gh api "repos/{o}/{r}/git/ref/heads/$BRANCH" --jq .object.sha)

# 1. 为每个改动文件建 blob，收集 (path, blob_sha, mode=100644)
BLOB1=$(gh api -X POST "repos/{o}/{r}/git/blobs" -f content="$(base64 -w0 f1)" | jq -r .sha)

# 2. 基于 base tree 建新 tree
TREE=$(gh api -X POST "repos/{o}/{r}/git/trees" \
  -f base_tree=$(gh api "repos/{o}/{r}/commits/$BASE" --jq .commit.tree.sha) \
  -F 'tree[][path]=path/to/f1' -F 'tree[][mode]=100644' -F 'tree[][type]=blob' -F "tree[][sha]=$BLOB1" \
  --jq .sha)
# 多文件时 tree[] 数组逐条加；gh -F 会把 [] 解析成数组

# 3. 建 commit
COMMIT=$(gh api -X POST "repos/{o}/{r}/git/commits" \
  -f message="one-line summary" -f tree="$TREE" -f "parents[]=$BASE" --jq .sha)

# 4. 移动分支指针
gh api -X PATCH "repos/{o}/{r}/git/refs/heads/$BRANCH" -f sha="$COMMIT"
```

## 大内容：命令行会溢出时

`-f content=` 塞几十 KB 会报 "Argument list too long"。改用 JSON 文件：

```bash
python3 - <<'EOF'
import base64, json
content = open("local_file", "rb").read()
json.dump({
    "message": "commit message",
    "content": base64.b64encode(content).decode(),
    "branch": "target",
    "sha": open(".old_sha").read().strip(),
}, open("payload.json", "w"))
EOF
gh api -X PUT "repos/{o}/{r}/contents/{path}" --input payload.json
```

## 建分支

```bash
BASE=$(gh api "repos/{o}/{r}/git/ref/heads/main" --jq .object.sha)
gh api -X POST "repos/{o}/{r}/git/refs" -f ref="refs/heads/new-branch" -f sha="$BASE"
```

## 常用查询

```bash
gh pr list --repo {o}/{r} --state open
gh pr view <num> --repo {o}/{r} --json title,body,headRefName
gh pr checks <num> --repo {o}/{r}            # PR 的 CI 状态
gh api "repos/{o}/{r}/pulls/<num>/files" --jq '.[].filename'
gh api "repos/{o}/{r}/actions/runs?branch=<b>&per_page=5" --jq '.workflow_runs[] | {id, name, status, conclusion}'
```

## 红线

- 不 push main、不 force push（用户明确要求时除外，且先列出将被覆盖的提交）。
- token 只从环境变量或已登录的 gh 取；不回显、不写入文件。
