# 新手作品流推荐（Onboarding → ES → Feed）正式 Spec（v1）

## 0. 目标与范围
- 目标：用户完成 onboarding 后，生成一条“只出作品流”的新手推荐 feed（首屏质量稳定 + 与用户显式偏好相关）。
- 数据源：Elasticsearch
  - `recsys_remixes_collection_meta_index_v2`（必用）
- 特征来源：Onboarding 落库
  - `guide_config.short_tags`（性格 short tags）
  - `themes.short_tags`（主题 short tags）
  - `guide_config.highlight`（用于解释/埋点，不参与召回）
- 非目标：不做复杂画像/协同过滤/长周期个性化；不输出主题卡片；不引入 Airflow/DAG。

---

## 1. 输入 / 输出

### 1.1 输入（服务端接口入参）
- `user_id: int/str`
- `seed_tags_from_db: string[]`
  - 来自落库的 onboarding 特征（性格 short_tags + theme short_tags 合并后的原始数组，可能含大小写/空格/重复）
- 可选：
  - `page_size`（默认 20）
  - `candidate_size`（默认 60，用于召回后再去重）
  - `seen_collection_ids: int[]`（客户端已曝光/已点过，用于过滤，可选）

### 1.2 输出（作品流）
`items: [{id, uuid, name, url, user_id, ctime, mtime, tax_paths, tax_primary, tax_secondary, tax_tertiary}]`
- `url`：封面图 URL（v2 已修复为 MySQL `collection.url`）

---

## 2. 特征预处理（必须做）

### 2.1 normalize 规则（服务端调用 ES 前执行）
使用 `normalize_tax_key`（与 tax pipeline 同规则）：
- strip
- 去掉所有空白字符
- 英文 A-Z 转小写（不影响中文）
- 分隔符变体统一成 `>`

并做去重保序 `uniq_keep`。

得到：
- `seed_tags_all_norm: string[]`

> 说明：不修改落库特征原值，只在服务端发 ES 前做 normalize。

### 2.2 highlights 的使用
- `guide_config.highlight` 不参与召回/排序
- 仅用于：
  - 推荐解释文案（可选）
  - 埋点维度（问题维度效果分析）

---

## 3. ES 召回与排序（DSL）

索引：`recsys_remixes_collection_meta_index_v2`

字段优先级（从高到低）：
1) `search_tags`（normal_search 产物，最贴近检索）
2) `content_tags`（json_tags 叶子，更结构化）
3) `search_keywords`（LLM keywords，兜底）

### 3.1 主查询（推荐默认）
```json
POST recsys_remixes_collection_meta_index_v2/_search
{
  "size": 60,
  "_source": [
    "id","uuid","name","url","user_id","ctime","mtime",
    "tax_paths","tax_primary","tax_secondary","tax_tertiary",
    "search_tags","content_tags","search_keywords"
  ],
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "minimum_should_match": 1,
          "should": [
            { "terms": { "search_tags":     ["<seed_tags_all_norm>"], "boost": 6 } },
            { "terms": { "content_tags":    ["<seed_tags_all_norm>"], "boost": 3 } },
            { "terms": { "search_keywords": ["<seed_tags_all_norm>"], "boost": 1 } }
          ]
        }
      },
      "score_mode": "sum",
      "boost_mode": "sum",
      "functions": [
        {
          "field_value_factor": {
            "field": "mtime",
            "factor": 0.0000008,
            "missing": 0
          },
          "weight": 1
        }
      ]
    }
  }
}
```

#### 参数说明
- `size=60`：召回候选数量（默认 60）。后续通过服务端去重/多样性处理得到 `page_size` 条。
- `boost` 建议（可配置）：
  - `search_tags`: 6
  - `content_tags`: 3
  - `search_keywords`: 1
- 新鲜度：初版用 `field_value_factor(mtime)`；如遇 date 类型不稳定，可改为 `gauss` decay。

### 3.2 过滤（可选增强）
如果要过滤已曝光内容：
- 优先服务端过滤（简单），或
- 在 `bool.must_not` 中加：`{"terms": {"id": seen_collection_ids}}`

---

## 4. 服务端去重与多样性规则（必须做）

ES 返回后，在服务端按顺序处理，直到拿满 `page_size`。

### 4.1 去重策略（顺序执行）
1) **ID 去重**
- key：`id`
- 保留第一个（得分最高的）

2) **作者多样性**
- key：`user_id`
- 限制：同一 `user_id` 最多保留 `MAX_PER_AUTHOR=2`

3) **风格多样性（按 tax_path）**
- 从每个作品的 `tax_paths` 取主路径 `main_tax_path = tax_paths[0]`（若为空则 None）
- 限制：同一 `main_tax_path` 最多保留 `MAX_PER_TAX_PATH=3`

### 4.2 不足补齐（放宽策略）
如果经过去重后数量 < `page_size`：
- 放宽顺序：
  1) `MAX_PER_TAX_PATH: 3 → 5`
  2) `MAX_PER_AUTHOR: 2 → 3`
- 仍不足则走兜底召回（第 5 节）

---

## 5. 兜底策略（召回不足时）

### 5.1 兜底 Query：新鲜度
```json
POST recsys_remixes_collection_meta_index_v2/_search
{
  "size": 60,
  "_source": ["id","uuid","name","url","user_id","ctime","mtime","tax_paths","tax_primary","tax_secondary","tax_tertiary"],
  "sort": [{ "mtime": "desc" }]
}
```

补齐时同样执行第 4 节去重规则（可适当放宽）。

---

## 6. 监控与验收指标（建议上线必备）

### 6.1 线上监控（按天/按版本）
- `coverage`：有 onboarding 特征的用户中，`seed_tags_all_norm` 非空比例
- `recall_hit_rate`：主查询命中数分布（P50/P90）
- `dedup_drop_rate`：
  - 作者去重丢弃比例
  - tax_path 去重丢弃比例
- `fallback_rate`：触发兜底的请求占比

### 6.2 业务指标（AB）
- 首日：点击率、收藏/点赞、停留时长、次日留存
- 对照：不使用 onboarding 特征（只走新鲜/热门）

---

## 7. 实现备注（Python）
- 服务端在调 ES 前做 normalize（不改库里原始特征）：
  - `normalize_tax_key` + `uniq_keep`（与 tax pipeline 一致）
- ES DSL 中的 `<seed_tags_all_norm>` 用数组替换（terms query）。
