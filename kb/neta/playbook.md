# Neta Playbook (Workspace)

目的：把 `skills/neta` 这套 CLI/能力变成可重复执行的工作流。

> 安全：
> - 不把任何 **NETA_TOKEN / 用户手机号 / 私人信息** 写入仓库或 `pages/`。
> - 你明确要求：**不触发任何飞书通讯录能力**（本 playbook 也不涉及）。

---

## 0. 默认约定（你定的规则）

### 0.1 分享链接（快速浏览）

你确认：以后给你发作品链接 **统一用 interaction**，不区分 profile。

- 作品 UUID → 分享链接：
  - `https://app.nieta.art/collection/interaction?uuid=<COLLECTION_UUID>`

> 注：你也指出“更严谨要用详情接口 shareUrl”。目前按你的指令，先统一拼 interaction 链接。

### 0.2 评论策略（活人反应）

评论生成必须遵守：只基于 **title/body/media_desc/post_tags/cover_url**。

- 规则全文见：`neta-comment-policy.md`

### 0.3 点赞 / 收藏的含义

- 点赞：我觉得内容优质（完成度/有梗/有画面/可互动/信息密度）。
- 收藏：我判断你会喜欢（互动可玩/国风武侠/氛围短篇/你指定的偏好）。

---

## 1. 环境与运行方式

### 1.1 Token 注入

只在命令执行时注入：

```bash
NETA_TOKEN="..." pnpm -s start <command> [args]
```

不写入 `.env`、不写入 `pages/`、不写入 git。

### 1.2 常用命令索引

- 社区信息流：
  - `request_community_feed --theme 最新|关注|热门 --page_index N --page_size M [--biz_trace_id ...]`
- 用户个人主页倒序 feed：
  - `request_interactive_feed --scene personal_feed --target_user_uuid <USER_UUID> --page_index 0 --page_size 20`
- 点赞/收藏/评论：
  - `like_collection --uuid <COLLECTION_UUID> [--is_cancel true]`
  - `favor_collection --uuid <COLLECTION_UUID> [--is_cancel true]`
  - `create_comment --parent_type collection --parent_uuid <COLLECTION_UUID> --content "..."`
- 粉丝/关注列表：
  - `get_fan_list --page_index N --page_size M`
  - `get_subscribe_list --page_index N --page_size M`

---

## 2. 标准工作流（可直接照抄执行）

### 2.1 查某人最近动态（个人主页）

1) 定位 user_uuid（来自粉丝/关注列表，或从作品反查）
2) 拉倒序 feed：

```bash
pnpm -s start request_interactive_feed \
  --page_index 0 \
  --page_size 20 \
  --scene personal_feed \
  --target_user_uuid "<USER_UUID>"
```

3) 选取要处理的作品：
- “优质” → 点赞 +（若适合互动）评论
- “你会喜欢” → 收藏（必要时也可评论）

### 2.2 自动逛最新（有限度、避免风控）

参数：
- 翻页：例如 10 页（每页 40）
- 互动：例如 8 条（点赞+评论）
- 收藏：例如 10 条（国风/武侠/古风优先）

策略：
- 优先挑：描述不为空、互动性强、有清晰锚点（能写出“当下反应”）
- 遇到信息不足：评论退到 reaction_mode C/E

---

## 3. 批量打标（待实现）

目标：把你给的 PROMPT_V2 做成可批处理的 tagging skill。

建议实现：
- 输入：单个作品的结构化信息（title/description/hashtags/cta_info/displayData/cover_url 等）
- 输出：PROMPT_V2 规定的 JSON（primary_tags 1~3，二级 1~3，三级 1，keywords 3~5，技术词转用户体验词）
- 存储：本地私有 `data/tagging/works_tagged.jsonl`
- 可视化：生成脱敏统计页面到 `pages/`（公开，不含 token、不含用户私密信息）

---

## 4. pages 发布规则（公开可访问）

- 任何放进 `pages/` 的东西都视为公开。
- 只放：标签树、公开资料、脱敏统计。
- 禁止：token、手机号、账号隐私、你的个人偏好画像原始日志。

