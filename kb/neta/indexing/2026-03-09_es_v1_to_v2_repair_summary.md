# 2026-03-09 线上索引修复实践总结（v1 → v2）

目标：
- 只读线上 v1：
  - `recsys_remixes_collection_meta_index_v1`
  - `recsys_remixes_tax_index_v1`
- 按“最终定稿规则”修复 taxonomy 相关字段，并补齐缺失字段（来自 MySQL 权威来源）
- 写入 v2：
  - `recsys_remixes_collection_meta_index_v2`
  - `recsys_remixes_tax_index_v2`

## 一、修复范围与关键结论

### 1) v1 的 `url` 字段是错误的
- 线上 v1 meta 的 `url` 实际是作品页面链接（`https://app.nieta.art/collection/interaction?uuid=...`），不是封面图。
- v2 约定 `url` 为封面图，因此必须从 MySQL 重新获取。

### 2) v2 meta 比 v1 meta 多的字段
通过 ES mapping 对比确认：
- v2 meta 新增：`content_tags`, `search_tags`
- 必须从 MySQL `collection_extra_data` 补齐。

### 3) 一级类目强约束
- 一级类目必须严格限制在 9 个大类：
  - 衍生创作 / 互动演绎 / 创意应用 / 视觉设计 / 角色设定 / 情感氛围 / 情景剧情 / 周边衍生 / 自动游玩
- v1 中存在脏值（例如 `衍生创作类` 这类后缀、甚至更异常的值）；
  - 先 `map_primary`（含“xx类”→“xx”）
  - 再校验 `primary in allowed_set`，否则丢弃对应 tax_path。

## 二、数据血缘（权威来源）

### 封面图 url
- 来源：`collections.collection.url`

### content_tags
- 来源：`collections.collection_extra_data`
- 条件：`preset_key = 'latitude://71|live|json_tags'`
- 处理：JSON 解析 → `extract_innermost_tags()` 递归抽取叶子字符串 → normalize → 去重

### search_tags
- 来源：`collections.collection_extra_data`
- 条件：`preset_key = 'latitude://71|live|normal_search'`
- 处理：`dedup_by_base_tags()`（normalize + 去重 + “基础词去派生”包含关系过滤）

## 三、实现方案（为什么这样做）

### 1) 读 v1，但权威字段从 MySQL 覆盖
- ES v1 主要用于：
  - 提供修复对象集合（`id/uuid`）
  - 提供既有 tax 字段（用于重建 tax_paths）
- MySQL 作为权威来源补齐/纠正：
  - `url(content cover)`
  - `content_tags/search_tags`

### 2) tax_v2 不直接修 tax_v1，而是从 meta_v2 的 tax_paths 重建
- 优点：
  - 避免 tax_v1 脏数据继续污染
  - 保证 tax 字典与 meta 的 tax_paths 一致
- 规则：`_id = md5(tax_path)`，`suggest`/`search_keywords_text` 与既有 pipeline 对齐

## 四、核心代码

- 一次性修复脚本（ES v1 只读 + MySQL 补齐 → ES v2 写入）：
  - `scripts/mysql_repair_meta_tax_v1_to_v2.py`

- 最终规则的实现参考（normalize/map_primary/build_tax_fields 等）：
  - `dags/tax_pipeline_dag.py`

## 五、线上执行结果（本次）

脚本输出：
- `processed`: 536
- `cover_missing`: 0
- `dropped_primary_docs~`: 132（至少出现过一次非法 primary 被丢弃的文档数量，近似统计）

## 六、经验/踩坑

1) **不要相信 ES v1 的 `url`**：它可能是页面链接而不是封面。
2) **MySqlHook 不一定可用**：在非 Airflow 环境里更稳妥用 `pymysql` 直连（只读账号）。
3) **ES 客户端依赖尽量少**：本次环境缺 `requests`，因此脚本用 `urllib` 实现 ES scroll/bulk。
4) **严控一级类目**：如果不做 allowlist，会导致 taxonomy 树/筛选维度被污染。
5) **bulk 错误处理要强**：只要 bulk 返回 `errors=true`，应立即 fail-fast，并打印前几条 error 便于排查。
