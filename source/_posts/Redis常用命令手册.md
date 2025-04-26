---
title: Redis常用命令手册
abbrlink: '92429955'
date: 2024-08-27 10:59:15
tags:
  - Redis
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261101845.png'
toc: true
---

Redis是一款开源的高性能内存数据库，广泛应用于缓存、消息队列、排行榜等场景。它支持丰富的数据类型和多样化的操作命令，满足不同业务需求。为了帮助开发者快速掌握Redis的核心命令，本文汇总了键值操作、字符串、列表、集合、哈希、有序集合等常用命令，并介绍了Bitmaps、HyperLogLog、Geospatial等新特性，涵盖了Redis服务管理的基本指令。无论是入门学习还是日常使用，这份速览都能成为您高效使用Redis的实用参考。

<!-- more -->

## 一、键（Key）

| 命令                 | 说明                              |
|--------------------|---------------------------------|
| `keys *`           | 查看当前库所有key                      |
| `exists <key>`     | 判断某个key是否存在                     |
| `type <key>`       | 查看key是什么类型                      |
| `del <key>`        | 删除指定的key数据                      |
| `unlink <key>`     | 非阻塞删除key（仅将keys从keyspace元数据中删除） |
| `expire <key> 10`  | 为给定的key设置过期时间（10s）              |
| `ttl <key>`        | 查看还有多少秒过期                       |
| `select <dbindex>` | 切换数据库                           |
| `dbsize`           | 查看当前数据库的key数量                   |
| `flushdb`          | 清空当前库                           |
| `flushall`         | 清空所有库                           |

## 二、字符串（String）

| 命令                                         | 说明                            |
|--------------------------------------------|-------------------------------|
| `set <key> <value>`                        | 添加键值对                         |
| `get <key>`                                | 查询对应键值                        |
| `append <key> <value>`                     | 将给定的value追加到原值的末尾             |
| `strlen <key>`                             | 获得值的长度                        |
| `setnx <key> <value>`                      | 只有key不存在时，设置key值              |
| `incr <key>`                               | 将key中储存的数字值增1（为空新增值为1）        |
| `decr <key>`                               | 将key中储存的数字值减1（为空新增值为-1）       |
| `incrby <key> <步长>`                        | 将key中储存的数字值增减，自定义步长           |
| `mset <key1> <value1> <key2> <value2> …`   | 同时设置一个或多个 key-value 对         |
| `mget <key1> <key2> <key3> …`              | 同时获取一个或多个value                |
| `msetnx <key1> <value1> <key2> <value2> …` | 同时设置多个key-value对，当且仅当所有key不存在 |
| `getrange <key> <起始位置> <结束位置>`             | 获得值的范围，类似java中的substring      |
| `setrange <key> <起始位置> <value>`            | 用<value>覆写<key>的值，从起始位置开始     |
| `setex <key> <过期时间> <value>`               | 设置键值的同时，设置过期时间（单位：秒）          |
| `getset <key> <value>`                     | 以新换旧，设置新值的同时获得旧值              |

## 三、列表（List）

| 命令                                         | 说明                            |
|--------------------------------------------|-------------------------------|
| `lpush <key> <value1> <value2> <value3> …` | 从左边插入一个或多个值                   |
| `rpush <key> <value1> <value2> <value3> …` | 从右边插入一个或多个值                   |
| `lpop <key>`                               | 从左边吐出一个值                      |
| `rpop <key>`                               | 从右边吐出一个值                      |
| `rpoplpush <key1> <key2>`                  | 从<key1>列表右边吐出一个值，插到<key2>列表左边 |
| `lrange <key> <start> <stop>`              | 按照索引下标获得元素（从左到右）              |
| `lindex <key> <index>`                     | 按照索引下标获得元素（从左到右）              |
| `llen <key>`                               | 获得列表长度                        |
| `linsert <key> before <value> <newvalue>`  | 在<value>后面插入<newvalue>        |
| `lrem <key> <n> <value>`                   | 从左边删除n个value（从左到右）            |
| `lset <key> <index> <value>`               | 将列表key下标为index的值替换成value      |

## 四、集合（Set）

| 命令                                     | 说明                      |
|----------------------------------------|-------------------------|
| `sadd <key> <value1> <value2> …`       | 将一个或多个member元素加入到集合key中 |
| `smembers <key>`                       | 取出该集合的所有值               |
| `sismember <key> <value>`              | 判断集合<key>是否包含该<value>值  |
| `scard <key>`                          | 返回该集合的元素个数              |
| `srem <key> <value1> <value2> …`       | 删除集合中的某个元素              |
| `spop <key>`                           | 随机从该集合中吐出一个值            |
| `srandmember <key> <n>`                | 随机从该集合中取出n个值，不会从集合中删除   |
| `smove <source> <destination> <value>` | 把集合中的一个值从一个集合移动到另一个集合   |
| `sinter <key1> <key2>`                 | 返回两个集合的交集元素             |
| `sunion <key1> <key2>`                 | 返回两个集合的并集元素             |
| `sdiff <key1> <key2>`                  | 返回两个集合的差集元素             |

## 五、哈希（Hash）

| 命令                                                  | 说明                                      |
|-----------------------------------------------------|-----------------------------------------|
| `hset <key> <field> <value>`                        | 给<key>集合中的<field>键赋值<value>             |
| `hget <key> <field>`                                | 从<key>集合<field>取出value                  |
| `hmset <key> <field1> <value1> <field2> <value2> …` | 批量设置hash的值                              |
| `hexists <key> <field>`                             | 查看哈希表key中，给定域field是否存在                  |
| `hkeys <key>`                                       | 列出该hash集合的所有field                       |
| `hvals <key>`                                       | 列出该hash集合的所有value                       |
| `hincrby <key> <field> <increment>`                 | 为哈希表key中的域field的值加上增量                   |
| `hsetnx <key> <field> <value>`                      | 将哈希表key中的域field的值设置为value，当且仅当域field不存在 |

## 六、有序集合（Zset）

| 命令                                                                         | 说明                                        |
|----------------------------------------------------------------------------|-------------------------------------------|
| `zadd <key> <score1> <value1> <score2> <value2> …`                         | 将一个或多个member元素及其score值加入到有序集key中          |
| `zrange <key> <start> <stop> [WITHSCORES]`                                 | 返回有序集key中，下标在start到stop之间的元素（带WITHSCORES） |
| `zrangebyscore <key> <min> <max> [WITHSCORES] [LIMIT <offset> <count>]`    | 返回有序集key中，所有score值介于min和max之间的成员          |
| `zrevrangebyscore <key> <max> <min> [WITHSCORES] [LIMIT <offset> <count>]` | 同上，从大到小排序                                 |
| `zincrby <key> <increment> <value>`                                        | 为元素的score加上增量                             |
| `zrem <key> <value>`                                                       | 删除该集合下，指定值的元素                             |
| `zcount <key> <min> <max>`                                                 | 统计该集合，分数区间内的元素个数                          |
| `zrank <key> <value>`                                                      | 返回该值在集合中的排名，从0开始                          |

## 七、Bitmaps（新）

| 命令                                  | 说明                           |
|-------------------------------------|------------------------------|
| `setbit <key> <offset> <value>`     | 设置Bitmaps中某个偏移量的值（0或1）       |
| `getbit <key> <offset>`             | 获取Bitmaps中某个偏移量的值            |
| `bitcount <key> [start end]`        | 统计字符串从start字节到end字节比特值为1的量   |
| `bitop <operation> <destkey> [key]` | 复合操作，做多个bitmaps的交集、并集、非、异或操作 |

## 八、HyperLogLog（新）

| 命令                                            | 说明                        |
|-----------------------------------------------|---------------------------|
| `pfadd <key> <element> [element …]`           | 添加指定元素到HyperLogLog中       |
| `pfcount <key> [key …]`                       | 计算HLL的近似基数，可以计算多个HLL      |
| `pfmerge <destkey> <sourcekey> [sourcekey …]` | 将一个或多个HLL合并后的结果存储在另一个HLL中 |

## 九、Geospatial（新）

| 命令                                                                                 | 说明                    |
|------------------------------------------------------------------------------------|-----------------------|
| `geoadd <key> <longitude> <latitude> <member> [<longitude> <latitude> <member> …]` | 添加地理位置（经度、纬度、名称）      |
| `geopos <key> <member> [<member> …]`                                               | 获得指定地区的坐标值            |
| `geodist <key> <member1> <member2> [m\km\ft\mi]`                                   | 获取两个位置之间的直线距离         |
| `georadius <key> <longitude> <latitude> <radius> [m\km\ft\mi]`                     | 以给定的经纬度为中心，找出某一半径内的元素 |

## 十、服务管理

| 命令                                             | 说明                         |
|------------------------------------------------|----------------------------|
| `redis-server`                                 | 启动Redis服务器                 |
| `redis-cli`                                    | 启动Redis命令行客户端              |
| `redis-cli shutdown`                           | 停止Redis服务器                 |
| `redis-cli ping`                               | 检查Redis服务器是否在运行（返回PONG）    |
| `redis-cli info`                               | 获取Redis服务器的各种信息，如内存使用、连接数等 |
| `redis-cli config get <parameter>`             | 获取Redis配置参数的值              |
| `redis-cli config set <parameter> <value>`     | 设置Redis配置参数的值              |
| `redis-cli monitor`                            | 实时监控Redis服务器的命令执行情况        |
| `redis-cli save`                               | 手动触发Redis保存快照到磁盘           |
| `redis-cli bgsave`                             | 在后台异步保存快照到磁盘               |
| `redis-cli flushall`                           | 清空所有数据库（注意：不可恢复）           |
| `redis-cli flushdb`                            | 清空当前数据库（注意：不可恢复）           |
| `redis-cli cluster info`                       | 获取Redis集群的状态信息             |
| `redis-cli cluster nodes`                      | 列出集群中的所有节点信息               |
| `redis-cli cluster meet <ip> <port>`           | 将新节点加入到集群中                 |
| `redis-cli cluster addslots <slot> [<slot> …]` | 为节点添加槽位                    |
| `redis-cli cluster replicate <node_id>`        | 将当前节点设置为指定节点的从节点           |
| `redis-cli cluster forget <node_id>`           | 从集群中移除指定节点                 |

## 结语

Redis强大的功能和灵活的数据结构极大地提升了应用性能和扩展能力。掌握这些常用命令，有助于更好地设计和优化缓存策略，提高系统响应速度。希望本文内容能帮助您深入理解Redis的使用方法和最佳实践，在实际项目中发挥更大价值。持续学习与实践，Redis必将成为您开发中的得力助手。