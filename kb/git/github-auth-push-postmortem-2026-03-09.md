# GitHub 认证问题复盘（Host key / publickey / 环境变量私钥）- 2026-03-09

> TL;DR：先解决 `known_hosts`（Host key verification failed），再解决 SSH key 授权（Permission denied publickey）。容器环境建议用 **base64 私钥注入 + 显式 ssh config（IdentitiesOnly）**。
>
> 风险与注意事项：
> - 不要把 `GITHUB_SSH_TOKEN`/私钥写入仓库或日志；避免在群里明文发送。
> - `ssh-keyscan` 写入的 host key 适合自动化，但仍要留意被劫持风险（必要时 pin 指纹）。
> - 多 key 场景务必 `IdentitiesOnly yes`，否则会“试错”多个 key 导致鉴权失败或触发风控。

## 背景
在将文档 push 到 `origin openclaw` 时，遇到 git push 失败。

仓库 remote 为 SSH：
- `origin git@github.com:htj-baize/baize-workspace.git`

目标：在不手工介入的情况下恢复 push 能力，并记录排查/修复步骤，方便后续复用。

---

## 问题 1：Host key verification failed
### 现象
执行 `git push origin openclaw` 报错：
- `Host key verification failed.`

### 原因
- `~/.ssh/known_hosts` 中缺少 `github.com` 的主机公钥（首次连接、或环境变更导致 known_hosts 为空）。

### 修复
将 github.com 的主机 key 写入 known_hosts：
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keyscan -t rsa,ecdsa,ed25519 github.com >> ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts
```

> 注：该步骤只解决“信任 github.com 这台主机”的问题，不解决“是否有权限 push”。

---

## 问题 2：Permission denied (publickey)
### 现象
Host key 解决后再次 push：
- `git@github.com: Permission denied (publickey).`

### 原因
- remote 使用 SSH，但当前环境没有可用的 GitHub SSH 私钥（或 ssh-agent 未加载 key）。

---

## 解决方案：从环境变量注入 SSH 私钥并显式配置
### 现状约束
- 用户已在环境变量中配置了 GitHub token，但变量名为 `GITHUB_SSH_TOKEN`，其内容为 **base64 编码的 OpenSSH 私钥**（不是 https PAT）。

### 步骤
1) 从环境变量解码并落盘到 `~/.ssh/github_openclaw`
2) 配置 `~/.ssh/config`，强制 github.com 使用该 key（`IdentitiesOnly yes`）
3) `ssh -T git@github.com` 验证鉴权
4) 再执行 `git push`

示例脚本：
```bash
python3 - <<'PY'
import os, base64
b64 = os.environ['GITHUB_SSH_TOKEN']
key = base64.b64decode(b64).decode('utf-8')
path = os.path.expanduser('~/.ssh/github_openclaw')
os.makedirs(os.path.dirname(path), exist_ok=True)
with open(path, 'w', encoding='utf-8') as f:
    f.write(key)
os.chmod(path, 0o600)
print('wrote', path)
PY

cat > ~/.ssh/config <<'CFG'
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_openclaw
  IdentitiesOnly yes
CFG
chmod 600 ~/.ssh/config

ssh -T git@github.com || true

git push origin openclaw
```

### 验证标准
- `ssh -T git@github.com` 返回：
  - `You've successfully authenticated, but GitHub does not provide shell access.`
- `git push` 成功。

---

## 经验与建议
1) **先分层定位**：
   - host key 问题（known_hosts） vs 认证问题（publickey）。
2) remote 若为 `git@github.com:...`，则必须有 SSH key（PAT 无法直接用于 SSH）。
3) 环境变量命名要清晰：
   - 若是 SSH 私钥，建议命名 `GITHUB_SSH_PRIVATE_KEY_B64`。
   - 若是 https PAT，建议命名 `GITHUB_TOKEN` 并使用 https remote。
4) 对多 key 场景，建议用 `~/.ssh/config + IdentitiesOnly yes`，避免 ssh 在多个 key 间试错导致认证失败。

