# 1.存储结构

![](./images/mysql-innodb存储结构.webp)

- 表空间Tablespace（ibd文件）
- 段Segment（一个索引2个段）
- 区Extent（1MB）：64个Page
- 页Page（16KB）：磁盘管理的最小单位
  - 一个B+树节点就是一个页（16KB）
  - 页的编号可以映射到物理文件偏移
  - B+树叶子节点前后形成双向链表