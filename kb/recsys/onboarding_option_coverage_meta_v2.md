# Onboarding options/themes 覆盖度检查（meta_v2）

索引：`recsys_remixes_collection_meta_index_v2`

查询：terms on `search_tags/content_tags/search_keywords`，`minimum_should_match=1`，`size=0` 只看覆盖数量。

## 总览

| kind | name | norm_tags | matched_works |
|---|---|---|---:|
| theme | #角色养成 | oc, 养成 | 31 |
| theme | #百变影棚 | 服饰, 风格 | 2 |
| theme | #伪人大本营 | 伪人, 槐安, 里界, 伪人大本营 | 91 |
| theme | #武侠江湖 | 武侠, 江湖, 国风 | 29 |
| theme | #捏Ta学院 | 学院, 萌新指引, 教程 | 1 |
| theme | #现实世界 | 现实, 次元 | 1 |
| personality | 温柔治愈 | 温柔, 治愈, 体贴, 暖心, 细腻, 关怀, 安抚 | 168 |
| personality | 冷酷无情 | 冷酷, 无情, 冰山, 淡漠, 理性, 决绝, 冷漠 | 8 |
| personality | 沉稳可靠 | 沉稳, 可靠, 稳重, 成熟, 冷静, 负责, 周全 | 3 |
| personality | 傲娇毒舌 | 傲娇, 毒舌, 别扭, 嘴硬, 吐槽, 心软, 害羞 | 11 |
| personality | 开朗活力 | 开朗, 活力, 阳光, 热情, 乐观, 元气, 积极 | 82 |
| personality | 友好随和 | 友好, 随和, 亲和, 温和, 包容, 好相处, 友善 | 5 |
| personality | 腹黑心机 | 腹黑, 心机, 算计, 伪装, 城府, 深藏, 谋略 | 1 |
| personality | 内向软萌 | 内向, 软萌, 害羞, 胆小, 纯真, 呆萌, 文静 | 15 |
| personality | 搞怪幽默 | 搞怪, 幽默, 搞笑, 风趣, 逗比, 欢乐, 脑洞 | 67 |
| personality | 霸气强势 | 霸气, 强势, 气场, 掌控, 领导, 果断, 霸道 | 2 |

## 结论

- 所有 bucket 均能在 meta_v2 召回到作品（matched_works > 0）。


## 备注

- 这份覆盖度只验证‘按 short tags 关键词召回是否能命中作品’，不代表 tax 字典映射一定命中。
- 若 matched_works 很大，建议在推荐侧加多样性去重（author/tax_path）。
