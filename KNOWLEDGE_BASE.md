# B-Labs Knowledge Base（工作空间沉淀）

> 目标：把我在 OpenClaw 工作空间里“最近踩过的坑 / 形成的工作流 / 规格文档”沉淀到可共享的知识库，供 B-Labs 团队成员复用。

## 目录

### GitHub / DevOps
- [Git Push（OpenClaw 容器）Deploy Key 实战与常见报错](./git/github-push-deploy-key-runbook.md)
- [GitHub 认证问题复盘（Host key / publickey / 环境变量私钥）2026-03-09](./git/github-auth-push-postmortem-2026-03-09.md)

### Neta（业务工作流与规范）
- [Neta Playbook（可重复执行工作流）](./neta/playbook.md)
- [Neta 评论策略（只基于可见信息）](./neta/comment-policy.md)
- [Neta Tax search workflow（流程记录）](./neta/neta-tax-search.md)

### Neta（索引/数据工程）
- [ES indexing README](./neta/indexing/README.md)
- [2026-03-09 ES v1→v2 repair summary](./neta/indexing/2026-03-09_es_v1_to_v2_repair_summary.md)

### Recsys / Onboarding specs
- [New user feed spec](./recsys/new_user_feed_spec.md)
- [Onboarding meta v2：coverage/theme/personality 等规范（合集）](./recsys/)

### Schema / Mappings
- [Nieta full schema overview](./schema/nieta_full_schema_overview.md)
- [Remixes tax & meta index mappings](./schema/remixes_tax_and_meta_index_mappings.md)

---

## 贡献规则（重要）

- **禁止提交任何密钥/Token/私钥/手机号/私人信息**。
- `pages/` 里的内容默认视为公开；知识库同样按“可公开分享”标准脱敏。
- 每篇文档尽量包含：背景 → 结论/做法 → 命令/示例 → 常见坑。
