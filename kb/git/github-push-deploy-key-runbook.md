# Git Push（OpenClaw 容器）Deploy Key Runbook

> TL;DR：在容器里 `git push` 最稳的方式是 **Deploy Key（允许写） + base64 私钥注入 + ssh-keyscan 生成 known_hosts**。
>
> 风险与注意事项：
> - **不要把私钥/Token 写进仓库**；只用环境变量/密钥管理注入。
> - 不建议关闭 `StrictHostKeyChecking`（除非你明确接受风险）。
> - Deploy Key 记得勾选 **Allow write access**，否则只能读不能推。

场景：OpenClaw 运行在容器中，需要从容器内 `git push` 到 GitHub（SSH remote）。

## 1) 常见报错与含义

### 1.1 `Host key verification failed.`
- 原因：容器内 `~/.ssh/known_hosts` 不包含 GitHub 主机指纹，且非交互环境无法手动确认。
- 影响：SSH 连接在握手阶段直接失败，无法拉取/推送。

### 1.2 `Load key ".../id_ed25519": error in libcrypto`
- 原因：写入的私钥内容不符合 OpenSSH 私钥格式，常见情况：
  - 把公钥 `.pub` 当私钥用
  - 私钥内容复制时丢失换行，变成一行
  - 环境变量带了多余前缀（例如把 `KEY=...` 整行塞进了 value）
  - 私钥被转义/污染（不可见字符）
- 影响：SSH 无法加载 key，随后会 `Permission denied (publickey)`。

### 1.3 `Permission denied (publickey).`
- 原因：GitHub 侧未授权该 key（Deploy key 未添加/未允许写权限），或 SSH 客户端未使用正确 key。

### 1.4 `base64: invalid input`
- 原因：base64 字符串不纯（包含空格/换行/引号/非 base64 字符），或拷贝不完整。

---

## 2) 推荐方案：Deploy Key + 每次 push 前自动 keyscan（更稳一点）

目标：不依赖固定 known_hosts 注入；每次 push 前通过 `ssh-keyscan` 动态生成 `known_hosts`。

### 2.1 GitHub 侧配置（一次性）
1) 在本机生成 deploy key（ed25519）：
```bash
ssh-keygen -t ed25519 -C "openclaw-deploy-key" -f openclaw_deploy_ed25519 -N ""
```
2) GitHub 仓库 Settings → Deploy keys → Add deploy key
- Key：`openclaw_deploy_ed25519.pub` 内容
- 勾选：**Allow write access**

### 2.2 私钥注入方式（推荐 base64，避免换行丢失）
macOS base64：
```bash
base64 -i openclaw_deploy_ed25519 | tr -d '\n'
```
将输出配置到环境变量（推荐）：
- `GITHUB_SSH_KEY_B64`（或兼容名 `GITHUB_SSH_TOKEN`）

> 关键点：环境变量 value 只放“base64 内容本身”，不要包含 `KEY=` 这类键名。

### 2.3 容器内 push 前的 SSH 初始化流程（脚本化）
1) 写入私钥：
- 从 `GITHUB_SSH_KEY_B64`（或 `GITHUB_SSH_TOKEN`）读取
- **先去掉所有空白字符**再 decode（容错拷贝时的换行/空格）

2) 自动生成 known_hosts（只取 ssh-ed25519，更稳）：
```sh
ssh-keyscan -p 22 -T 5 github.com | grep 'ssh-ed25519' >> ~/.ssh/known_hosts
ssh-keyscan -p 443 -T 5 ssh.github.com | grep 'ssh-ed25519' >> ~/.ssh/known_hosts
sort -u ~/.ssh/known_hosts -o ~/.ssh/known_hosts
```

3) 强制 SSH 使用该 key：
```sshconfig
Host github.com
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  StrictHostKeyChecking yes

Host ssh.github.com
  HostName ssh.github.com
  Port 443
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  StrictHostKeyChecking yes
```

4) 推送：`git push`

---

## 3) 额外补充：为什么不能只配私钥、不配 known_hosts
- 若没有 known_hosts，SSH 默认会拒绝连接（非交互环境直接失败），对应报错就是 `Host key verification failed.`。
- 可以关闭 StrictHostKeyChecking，但不推荐（安全性差）。

---

## 4) 这次实际落地结果
- 通过 **公网白名单** 放通 ES 联通后，ES ping 200。
- 通过 **base64 私钥 + 自动 keyscan** 生成 known_hosts，成功完成 `git push`。
