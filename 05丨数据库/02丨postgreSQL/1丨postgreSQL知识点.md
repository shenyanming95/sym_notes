# 1.查询缓存

PostgreSQL缓存缓冲区大小通过`shared_buffers`参数配置，默认值低，一般是128MB。官方建议，如果使用的是专用数据库服务器，应将其配置为服务器总内存的25%。

## 1.1.索引命中率

如下的查询提供数据库缓存中的索引命中率：

```sql
SELECT 100 * (sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit) AS index_hit_rate
FROM pg_statio_user_indexes;
```

## 1.2.缓存命中率

PostgreSQL在`pg_statio_user_tables`表提供**缓存命中率统计信息**，表中的两列：`heap_blks_read`表示从此表读取的磁盘块数，`heap_blks_hit`表示此表的缓冲区命中数。

postgreSQL将经常访问的数据保存在内存中，可通过pg_statio_user_tables视图查询有关它的统计信息。一个重要的衡量标准是：在工作负载中来自内存缓存与来自磁盘的数据的百分比，若发现命中率明显低于99%，就需要考虑增加数据库可用的缓存

```sql
SELECT
  sum(heap_blks_read) AS heap_read,
  sum(heap_blks_hit)  AS heap_hit,
  100 * sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_rate
FROM
  pg_statio_user_tables;
```

