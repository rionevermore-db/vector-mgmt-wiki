# Lance Geospatial Support — captured content (WebFetch, 2026-05-29)

> 来源：https://www.lancedb.com/blog/geo-support,发布 2026-02-25。Layer-1 citation 锚点。

## 层级

**Lance(列式格式 / 引擎)层**新增地理空间支持;**LanceDB(vector DB)继承**(底层用 Lance 存储)。

## 1. R-Tree 空间索引（真索引,production-ready）

- **static / immutable 2D 空间索引**,基于 bounding box,**多层 hierarchical 结构**:
  - leaf page 存 `(bbox, rowid)` 元组
  - branch page 存子 bbox 聚合 + page ID
  - 单 root page 包住整棵树
- **build 策略**:packed-build + **Hilbert 曲线排序**最大化空间局部性
- **剪枝**:对 `ST_Intersects(geometry, query_bbox)`,从 root 检查 query_bbox 是否相交 → descend 或整 subtree 剪掉;候选定位后用 Lance random access 高效取行
- **需显式建索引**(不像 Parquet 自动 bbox 统计)
- **由 ByteDance 的 Xin Sun 贡献**
- blog 未加"实验性"限定词 → 视为 shipped / production-ready

## 2. GeoArrow 扩展类型（"with no new code" 的部分）

- 支持 Point / LineString / Polygon / MultiPoint / MultiLineString / MultiPolygon / GeometryCollection + **CRS(坐标参考系)**
- 全部按 **GeoArrow extension type 规范**存储
- **"no new code" 仅指存储层**:Arrow 的 extension type 机制是原生的,Lance 忠实存储所有 Arrow extension metadata → geo 类型存储白嫖,不改 Lance core format

## 3. GeoDataFusion 空间函数（新写的集成代码）

- 集成 GeoDataFusion(扩展 Apache DataFusion),遵循 **OGC Simple Feature Access** 标准
- 函数:`ST_Distance / ST_Intersects / ST_Contains / ST_Within / ST_Touches / ST_Crosses / ST_Overlaps / ST_Covers / ST_CoveredBy / ST_GeomFromText`
- **这部分需要新实现**:把 GeoDataFusion 的 function registry 接进 Lance 的 DataFusion context

## 4. 查询范式

- bbox 过滤先剪候选,精确几何验证由 execution engine 在剩余行上做 → "orders of magnitude faster spatial queries on large datasets"(blog 用 billion-row 量级举例)

## 5. 限制 / future work

- R-Tree 需显式建索引(非自动)
- future:HuggingFace 上的 geospatial Lance 数据集;Spark / Trino / DuckDB / Ray 引擎集成仍 pending;pluggable secondary index system(让不需要 geo 的用户不必引入 geo 依赖)
- 无明确 Lance / LanceDB 版本号
