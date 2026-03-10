# Neta → ES 索引流水线（设计稿）

目标：
- 浏览 Neta 社区最新作品（feed）
- 对每个作品执行“玩法打标”（使用你提供的 PROMPT_V2）
- 写入 / 更新 ES 8.15 两个索引：
  - `recsys_remixes_tax_index_v1`
  - `recsys_remixes_collection_meta_index_v1`

## 数据来源

1) 最新 feed：`pnpm -s start request_community_feed --theme 最新 --page_size N`（从 neta-skill）
2) 作品详情：优先用 feed 返回字段（含 `shareUrl`, `coverUrl`, `hashtag_names`, `user_id`, `user_uuid` 等）；如需补全，再调用：
   - `/v3/story/story-detail?uuids=...`（batch）
   - `/v1/home/feed/interactive?collection_uuid=...`（拿 `cta_info` / `displayData`）

## 打标输出

PROMPT_V2 输出的 JSON 将被映射为：
- collection-meta：`tax_primary/tax_secondary/tax_tertiary/all_tax_nodes/search_keywords/...`
- tax-index：抽取 taxonomy 节点（含 `tax_path`, `tax_primary`, `tax_secondary`, `tax_tertiary`, `search_keywords`, `suggest`）

## 去重与更新策略

- 以 `uuid` 作为唯一键。
- collection-meta 用 `uuid` 做 `_id`。
- tax-index 用 `tax_path + name` 或 `tax_path`（待你确认）作为 `_id`。
- 每次运行只处理最新 feed 中未见过或 mtime 更新的作品。

## 运行方式

- 手动：`node scripts/neta-indexer/run.js --pages 3 --pageSize 40`
- 定时：OpenClaw cron 每 X 分钟/小时跑一次（由你后续配置）。

## 环境变量（写到本地 env，由你来配置真实值）

- `NETA_TOKEN`
- `NETA_API_BASE_URL`（可选）
- `ES_URL`
- `ES_API_KEY`（或 `ES_USERNAME/ES_PASSWORD`）
- `ES_INDEX_TAX=recsys_remixes_tax_index_v1`
- `ES_INDEX_META=recsys_remixes_collection_meta_index_v1`
- `TAGGER_MODEL`（可选）

