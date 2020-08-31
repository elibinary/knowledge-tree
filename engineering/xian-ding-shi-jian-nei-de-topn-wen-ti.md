# 限定时间内的 top n 问题

#### 问题

需求：获取 m 天内 feeds （或者说动态）点赞数的 top n

功能点剖析：

* 创建时间在 m 天内的动态
* 因为是对点赞数排序，点赞数会实时变动（增/减）

#### 思路一

因为涉及到频繁变动的数据排序，并且在限定时间内，数据量可控。优先想到使用 `Redis#SortedSet`

首先使用一个 `SortedSet` 结构存储点赞数据

```text
{ source_id: likes_count }
```

主要涉及到的操作有

```text
# zincrby
# zrevrange
# zrem
```

再使用一个 `SortedSet` 结构存储过期数据

```text
{ source_id: created_at }
```

主要涉及到的操作有

```text
# zadd
# zrangebyscore
# zremrangebyscore
```

这种方法有个很大的问题是，需要定期执行过期操作（cron），执行过期时对 redis 的操作比较重

#### 思路二

按创建时间存储，每小时一个 `SortedSet` 并设置 TTL，过期时间为 24 \* m 小时，存储结构为

```text
SortedSet Key: created_at.beginning_of_hour.to_i

content: { source_id: likes_count }
```

对应创建时间的动态点赞数变动操作对应时间的 key

主要涉及到的操作有

```text
# zincrby
# zrevrange
```

然后在获取 top n 的时候

1. 对最近 `24 * m` 小时的 `24 * m` 个 `SortedSet` 分别取 top n
2. 合并排序再取 top n

