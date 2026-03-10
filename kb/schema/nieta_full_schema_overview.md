# Nieta/Neta 全域 Schema 总览（以 neta_data-main.zip 为准）

> 目标：后续任何数据分析/指标需求，都能快速定位“哪张表、怎么 join、时间口径”。
>
> 口径：本总览以 `neta_data-main.zip/docs/mysql/` 为准。
>
> 重要变更：**picture 表已下线**，不要再依赖 picture 作为作品内容来源；优先使用 `content.artifacts` / `content.task` / `collections.collection.display_data` 等。

---

## 0) 主键与 ID 体系（先记住这两套）

- **数值主键（id）**：大多数关系表用 `id int` 和外键 `*_id int` 做 join（如 `collection.id`、`hashtag.id`）。
- **业务 UUID（uuid）**：对外/跨系统常用 `uuid varchar(...)`（如 `collection.uuid`、`comment.uuid`、`artifacts.uuid`）。

常见坑：
- Hashtag 关系表 `hashtag_entity_relation` 关联 collection 用的是 `entity_id = collection.id`（int），不是 uuid。
- CTA/玩法指令产物表 `collection_extra_data` 关联 collection 用的是 `collection_uuid = collection.uuid`（varchar 36）。

时间：大部分表 `ctime/mtime/ptime` 均为 **UTC(+0) 存储**，业务解释经常需要转北京时间(+8)。

---

## 1) Accounts（账户与身份）

用途：用户是谁、会话、设备、权限、主题。

关键表：
- `accounts/user`
- `accounts/session`
- `accounts/device`
- `accounts/user_privilege`
- `accounts/user_theme`

典型 join：
- `session.user_id -> user.id`
- `device.user_id -> user.id`

---

## 2) Collections（作品与世界观/玩法）

这是“作品分析/玩法标签 pipeline”的核心域。

### 2.1 作品主表：`collections/collection`

- PK: `collection.id`
- 业务键: `collection.uuid`（索引 `ix_collection_uuid`）
- 核心字段（打标/检索/统计）：
  - 文本：`title`, `description`
  - JSON：`display_data`（displayData/构图/风格等）、`extra_data`（派生字段/补充元数据）
  - 状态：`status`, `review_status`, `platform`
  - 玩法：`is_interactive`, `has_video`, `video_uuid`, `ratio`
  - 时间：`ctime`, `mtime`, `ptime`
  - 计数：`like_count`, `comment_count`, `view_count`, `shared_count`, `same_style_count`

### 2.2 CTA/玩法指令产物：`collections/collection_extra_data`

- UNIQUE: `(collection_uuid, preset_key)`（`uk_collection_extra_data_collection_preset`）
- 字段：`collection_uuid`, `preset_key`, `result`(text), `ctime/mtime`
- 作用：
  - `result` 通常是 JSON 字符串（不同 preset_key 对应不同结构）。
  - 通过业务侧的 CTA processor（例如你给的 `recsys_cta_processor.py`）解析并标准化为：
    - `cta_info.launch_prompt.core_input / brief_input / ref_image`（ONE_INPUT）
    - 或 `question + choices[]`（QUESTIONS/2selection）

### 2.3 互动配置/状态：`collections/collection_interactive`

- 用途：承载互动玩法相关的 status/config（渲染层的 `interactive_status` / `interactive_config` 往往来自这里）。

### 2.4 世界观/角色/玩法结构

关键表（按需展开）：
- `collections/world_setting`
- `collections/lore`
- `collections/character`
- `collections/original_character`
- `collections/verse`（互动玩法/模板本体：`interactive_config`, `toolset_keys`, `system_root_prompt_key` 等）
- `collections/verse_catalog` / `verse_catalog_entry_relation`

关系/组织：
- `collections/collection_character`
- `collections/collection_relation`
- `collections/collection_theme`

> 注：`verse_*` 系列字段/标签（如 verse_tags/verse_uuids）在你们当前口径中属于**历史逻辑**，后续打标/分析默认不再依赖它，除非你明确要求。

---

## 3) Community（社区与话题）

用途：评论、消息推送、话题（hashtag）体系。

### 3.1 评论：`community/comment`

- 字段重点：
  - 挂载：`parent_id`, `parent_type`
  - 根实体：`root_entity_uuid`, `root_entity_type`
  - 内容：`content`, `like_count`
  - 其他：`hashtags_data`(json), `status`, `review_status`, `platform`, `ctime/mtime/date`

### 3.2 Hashtag 体系

- `community/hashtag`
  - `id`, `name`, `status`, `activity_uuid`, `extra_data`, `heat`, `latest_collection_ptime`, `ctime/mtime`

- `community/hashtag_entity_relation`
  - `hashtag_id -> hashtag.id`
  - `entity_id`（例如 `collection.id`）
  - `entity_type`（例如 `'collection'`）
  - `status`, `activity_result`, `ctime/mtime`

**作品 hashtags 的标准链路**：
- `collection.id` → `hashtag_entity_relation(entity_type='collection', entity_id=collection.id)` → `hashtag.name`

- `community/hashtag_entity_relation_blacklist`：黑名单/屏蔽关系（用于过滤）。

---

## 4) Engagement（用户行为/权益/关系）

用途：点赞/收藏/关注/任务/徽章/糖果/AP 等增长与留存核心。

关键行为表：
- `engagement/user_entity_like`（点赞）
  - UNIQUE: `(user_id, entity_id, entity_type)`
  - 字段：`user_id`, `entity_id`, `entity_type`, `extra_data`, `platform`, `ctime/mtime/date`

- `engagement/user_entity_favor`（收藏）
- `engagement/user_relation`（关注/拉黑等用户关系）
- `engagement/user_hashtag_subscribe`（关注话题）

成长/权益：
- `engagement/assignment` / `user_assignment`
- `engagement/badge` / `user_badge_relation`
- `engagement/user_ap_delta`
- `engagement/user_candy_consumption_log` / `user_candy_inventory`
- `engagement/user_checkin_log` / `user_checkin_state`
- `engagement/rank_list`
- `engagement/user_item`

---

## 5) Content（生成与内容产物）

> picture 已下线后，这一域通常是“还原作品生成输入/产物细节”的主要入口。

关键表：
- `content/artifacts`
  - 产物级：`uuid`, `url`, `modality`, `detail`(json), `input`(json), `raw_prompt`(json), `hashtags`(json), `extra_data`(json)
  - 状态：`status`, `is_published`, `is_starred`, `is_user_upload`, `worker`, `error`
  - 任务：`task_uuid`
  - 文本产物：`text`
  - 时间：`ctime/mtime`

- `content/task`
  - `uuid`, `job`, `entrance`, `params`(json), `raw_prompt`(json), `hashtags`(json), `status`, `worker`, `platform`, `extra_data`, `ctime/mtime/date`

- `content/llm_message`
  - `conversation_uuid`, `preset_key`, `send_payload`(json), `response_package`(json), `status`, `error_code`, `ap_cost`, `ctime/mtime`, `platform`

- 其他：`asset_metadata`, `lora_asset`, `preset`, `scene`, `tag`, `picture_tag`, `audio*`, `artifact_verse_checkpoint*`

---

## 6) Campaigns（活动与专题）

关键表：
- `campaigns/activity`
- travel 系列：`campaigns/travel/travel_*`
- thoumask 系列：`campaigns/thoumask/thoumask_*`

建议：campaigns 表多且关系复杂，通常按具体问题（某活动、某旅行角色、某任务）再展开。

---

## 7) Commerce（交易与订单）

关键表：
- `commerce/spu`、`commerce/sku`
- `commerce/order`、`commerce/payment`
- `commerce/coupon`、`commerce/coupon_redeem`
- `commerce/delivery`

典型 join：
- `order.user_id -> accounts.user.id`
- `payment.order_id -> order.id`
- `coupon_redeem.coupon_id -> coupon.id`

---

## 8) Moderation（审核与风控）

关键表：
- `moderation/moderate`
- `moderation/artifact_moderation_log`

用途：审核状态、违规原因、与分发/推荐/发布状态的关联分析。

---

## 9) 与“玩法标签 pipeline”最相关的字段血缘（速记）

- **displayData/rawPrompt**：`collection.display_data`（必要时结合 `content.artifacts.raw_prompt/input` 做补充）
- **cta_info.launch_prompt**：`collection_extra_data.result`（按 `preset_key` 优先级挑）→ CTA processor 标准化
- **hashtags**：`collection.id` → `hashtag_entity_relation(entity_type='collection')` → `hashtag.name`

### 9.1 content_tags / search_tags（保持结构化的产物口径）

> 约定：`content_tags` / `search_tags` 不用 LLM 直接生成，优先沿用现有产物与清洗规则，作为打标 pipeline 的“强结构化信号/先验”。

- **content_tags**：来自 `collection_extra_data(preset_key='latitude://71|live|json_tags')`，用 `extract_innermost_tags()` 递归抽取最底层字符串列表。
- **search_tags**：来自 `collection_extra_data(preset_key='latitude://71|live|normal_search')`，用 `dedup_by_base_tags()` 做 lower/去重/基础词去派生。


