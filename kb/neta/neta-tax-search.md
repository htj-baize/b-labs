# Neta Tax → 搜索/筛选（不公开元数据）

目标：
- `pages/htj/neta-community/taxonomy.html` 只公开 **标签树**；
- **不**把 `collection-meta` / works 元数据全集放到 pages；
- 当你需要“按标签筛选作品”时，由 OpenClaw/Neta CLI 在本地执行查询，再导出**结果页**到 pages（只含本次结果的少量字段）。

## 公开入口（只含标签树）
- taxonomy 页：/htj/neta-community/taxonomy.html
- taxonomy 数据：/htj/neta-community/taxonomy.json

## 使用流程（推荐）
1) 你在 taxonomy.html 里确定想筛的 tag / tax path（例如：`衍生创作类>同人二创`）
2) 发我：
   - tax path
   - 页数/条数（比如 20/40）
3) 我会在本地调用 Neta：
   - `pnpm start suggest_content --intent exact --tax_paths "..." ...`
4) 我把结果导出为公开 results 页：
   - `pages/htj/neta-community/results/<timestamp>.html`
5) 回你一个链接，你点开就能浏览，并且每个卡片点开走：
   - `https://app.nieta.art/collection/interaction?uuid=...`

## 为什么这样不会“公开元数据全集”
- pages 上只有本次结果子集（例如 20~40 条），不会暴露 10k/100k 规模的全集。
- 真正的搜索/筛选逻辑在本地执行，不在浏览器端跑。

