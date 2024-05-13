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

# 2.PG运维指标

node_exporter 收集的指标的详细信息

## 2.1.文件系统指标

**指标组**：node_filesystem

| 指标         | 类型  | 描述                                           |
| ------------ | ----- | ---------------------------------------------- |
| device       | LABEL | 文件系统的设备                                 |
| mountpoint   | LABEL | 文件系统的挂载点                               |
| fstype       | LABEL | 文件系统的类型                                 |
| size_bytes   | GAUGE | 文件系统大小（以字节为单位）                   |
| avail_bytes  | GAUGE | 非 root 用户可用的文件系统空间（以字节为单位） |
| free_bytes   | GAUGE | 文件系统可用空间（以字节为单位）。             |
| files        | GAUGE | 文件系统文件节点总数                           |
| files_free   | GAUGE | 文件系统可用文件节点总数                       |
| device_error | GAUGE | 获取给定设备的统计信息时是否发生错误           |
| readonly     | GAUGE | 文件系统只读状态。                             |

## 2.2.磁盘指标

**指标组**：node_disk

| 指标                              | 类型    | 描述                           |
| --------------------------------- | ------- | ------------------------------ |
| flush_requests_time_seconds_total | COUNTER | 这是所有刷新请求花费的总秒数。 |
| flush_requests_total              | COUNTER | 成功完成的刷新请求总数。       |
| io_time_seconds_total             | COUNTER | 执行 I/O 所花费的总秒数。      |
| read_bytes_total                  | COUNTER | 成功读取的总字节数。           |
| read_time_seconds_total           | COUNTER | 所有读取所花费的总秒数。       |
| reads_completed_total             | COUNTER | 成功完成的读取总数。           |
| reads_merged_total                | COUNTER | 合并的读取总数。               |
| write_time_seconds_total          | COUNTER | 这是所有写入所花费的总秒数。   |
| writes_completed_total            | COUNTER | 成功完成的写入总数。           |
| writes_merged_total               | COUNTER | 合并的写入次数。               |
| written_bytes_total               | COUNTER | 成功写入的总字节数。           |

## 2.3.内存指标

**指标组**：node_memory

| 指标               | 类型  | 描述                                                         |
| ------------------ | ----- | ------------------------------------------------------------ |
| Active_bytes       | GAUGE | 最近使用过的内存量，除非绝对必要，否则通常不会回收。         |
| Bounce_bytes       | GAUGE | 用于块设备“反弹缓冲区”的内存量。                             |
| Buffers_bytes      | GAUGE | 裸磁盘块的临时存储量。                                       |
| Cached_bytes       | GAUGE | 用作缓存的物理内存量。                                       |
| CommitLimit_bytes  | GAUGE | 根据过量使用率（`vm.overcommit_ratio`）在系统上当前可分配的内存总量。仅当启用了严格的过量使用分配（`vm.overcommit_memory`中的模式 2）时，才会遵守此限制。 |
| Committed_AS_bytes | GAUGE | 要完成工作负载的估计内存总量。此值表示最坏情况的方案值，还包括交换内存。 |
| Dirty_bytes        | GAUGE | 等待写回磁盘的内存总量。                                     |
| HugePages_Free     | GAUGE | 系统可用的大页总数。                                         |
| HugePages_Rsvd     | GAUGE | 为 hugetlbfs 保留的未使用的大页数目。                        |
| HugePages_Surp     | GAUGE | 剩余大页数。                                                 |
| HugePages_Total    | GAUGE | 系统的大页总数。该数目的计算方法是，将`Hugepagesize`除以`/proc/sys/vm/hugetlb_pool`中指定的为大页预留的兆字节数。 |
| Hugepagesize_bytes | GAUGE | 每个大页单位的大小。默认情况下，对于 32 位体系结构，单处理器内核上的值为 4096 KB。对于 SMP、hugemem 内核和 AMD64，默认值为 2048 KB。对于 Itanium 体系结构，默认值为 262144 KB。 |
| Hugetlb_bytes      | GAUGE | 这是各种大小的大页所消耗的内存总量。如果正在使用不同大小的大页，则此数字将超过 HugePages_Total * Hugepagesize。 |
| Inactive_bytes     | GAUGE | 最近使用较少且更适合回收用于其他目的的内存量。               |
| Mapped_bytes       | GAUGE | 用于已被映射的文件（如动态库）的内存。                       |
| MemAvailable_bytes | GAUGE | 在不使用交换内存的情况下，可用于启动新应用程序的内存量的估计值。 |
| MemFree_bytes      | GAUGE | 系统未使用的物理内存量。                                     |
| MemTotal_bytes     | GAUGE | 可用内存的总量，即物理内存减去一些保留位和内核二进制代码。   |
| PageTables_bytes   | GAUGE | 专用于最底层页表的内存总量。                                 |
| Shmem_bytes        | GAUGE | 共享内存（shmem）和 tmpfs 使用的内存总量。                   |
| SwapCached_bytes   | GAUGE | 曾被移动到交换内存中，然后又移回主内存中，但仍保留在交换文件中的内存量。这样可以节省 I/O，因为内存不需要再次移动到交换文件中。 |
| SwapFree_bytes     | GAUGE | 未使用的交换内存量。                                         |
| SwapTotal_bytes    | GAUGE | 可用的交换内存总量。                                         |
| WritebackTmp_bytes | GAUGE | 被 FUSE 用于临时写回缓冲区的内存量。                         |
| Writeback_bytes    | GAUGE | 被主动写回磁盘的内存总量。                                   |

## 2.4.网络指标

**指标组**：node_network

| 指标                      | 类型    | 描述                                                         |
| ------------------------- | ------- | ------------------------------------------------------------ |
| mtu_bytes                 | GAUGE   | 最大传输单元。大于 MTU 字节的 IP 数据报将被分段为多个以太网帧。 |
| receive_bytes_total       | COUNTER | 接口接收的数据的总字节数。                                   |
| receive_compressed_total  | COUNTER | 设备驱动程序接收的压缩数据包数。                             |
| receive_drop_total        | COUNTER | 设备驱动程序丢弃的数据包总数。                               |
| receive_errs_total        | COUNTER | 由设备驱动程序检测到的接收错误的总数。                       |
| receive_fifo_total        | COUNTER | FIFO 缓冲区错误的数量。                                      |
| receive_frame_total       | COUNTER | 数据包成帧错误的数量。                                       |
| receive_multicast_total   | COUNTER | 设备驱动程序接收的多播帧数。                                 |
| receive_packets_total     | COUNTER | 接口接收的数据包总数。                                       |
| speed_bytes               | GAUGE   | 网络设备的速度。                                             |
| transmit_bytes_total      | COUNTER | 接口发送的数据的总字节数。                                   |
| transmit_carrier_total    | COUNTER | 由设备驱动程序检测到的载波损耗的数量。                       |
| transmit_colls_total      | COUNTER | 接口上检测到的冲突数。                                       |
| transmit_compressed_total | COUNTER | 设备驱动程序发送的压缩数据包数。                             |
| transmit_drop_total       | COUNTER | 设备驱动程序丢弃的数据包总数。                               |
| transmit_errs_total       | COUNTER | 由设备驱动程序检测到的发送错误的总数。                       |
| transmit_fifo_total       | COUNTER | FIFO缓冲区错误的数量。                                       |
| transmit_packets_total    | COUNTER | 接口发送的数据包总数。                                       |
| transmit_queue_length     | GAUGE   | 设备传输队列的长度。                                         |
