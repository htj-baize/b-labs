# Remix Tax / Collection Meta 索引结构（定稿）

> 本文件记录 ES 索引 mapping（v1 定稿版），用于 tax pipeline 开发与后续排查。
> 
> 约定：所有 tag/path/keyword 字段均为 **key 版**（normalize：去空白、英文小写、分隔符统一）。

---

## 1) recsys_remixes_tax_index_v1（Tax 字典索引）

```json
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "refresh_interval": "5s"
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "tax_path": {
        "type": "keyword"
      },
      "tax_levels": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "tax_primary": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "tax_secondary": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "tax_tertiary": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "suggest": {
        "type": "completion",
        "analyzer": "ik_max_word",
        "preserve_separators": true,
        "preserve_position_increments": true,
        "max_input_length": 50
      },
      "search_keywords": {
        "type": "keyword",
        "eager_global_ordinals": true,
        "ignore_above": 256
      },
      "search_keywords_text": {
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}
```

写入约定：
- `_id = md5(tax_path)`
- `tax_levels = [tax_primary, tax_secondary, tax_tertiary]`

---

## 2) recsys_remixes_collection_meta_index_v1（作品 Meta 索引）

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "5s"
  },
  "mappings": {
    "properties": {
      "id": {
        "type": "long"
      },
      "uuid": {
        "type": "keyword"
      },
      "url": {
        "type": "keyword"
      },
      "user_id": {
        "type": "long"
      },
      "name": {
        "type": "keyword",
        "ignore_above": 256
      },
      "search_keywords": {
        "type": "keyword",
        "ignore_above": 256
      },
      "tax_paths": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "tax_primary": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "tax_secondary": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "tax_tertiary": {
        "type": "keyword"
      },
      "all_tax_nodes": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "pgc_tags": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "ugc_tags": {
        "type": "keyword",
        "ignore_above": 256
      },
      "highlight_tags": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "content_tags": {
        "type": "keyword",
        "ignore_above": 256
      },
      "search_tags": {
        "type": "keyword",
        "ignore_above": 256
      },
      "ctime": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_second||epoch_millis"
      },
      "mtime": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_second||epoch_millis"
      }
    }
  }
}
```

写入约定：
- `_id = collection.id`
- `all_tax_nodes`：节点序列（`P:`/`S:`/`T:`），字段内去重（保序）
- `tax_paths`：完整路径列表（`P>S>T`），字段内去重（保序）
- 允许：一个作品多 P、多 S、多 T；允许同一 (P,S) 下多个 T

---

## 3) eager_global_ordinals 策略（定稿）

- 开 eager（核心维度/高频聚合）：`tax_paths`, `tax_primary`, `tax_secondary`, `all_tax_nodes`, `pgc_tags`, `highlight_tags`
- 不开 eager（长尾/高基数）：`search_keywords`, `search_tags`, `content_tags`, `ugc_tags`
- `tax_tertiary`：当前 meta 中不开 eager；tax 字典索引中开 eager
