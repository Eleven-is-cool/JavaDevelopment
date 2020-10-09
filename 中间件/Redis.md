# <span style = 'color:#45B39D'>Redis</span>

## 本地缓存解决方案

常见的单体架构图如下，我们使用 **Nginx** 来做**负载均衡**，部署两个相同的服务到服务器，两个服务使用同一个数据库，并且使用的是本地缓存。

<img src="https://github.com/Eleven-is-cool/img-folder/blob/master/%E6%9C%AC%E5%9C%B0Nginx.png?raw=true" alt="单体架构" style="zoom:50%;" />

一个稍微完善一点的缓存框架至少要提供：**过期时间、淘汰机制、命中率统计**这三点。

**一： `Ehcache` 、 `Guava Cache` 、 `Spring Cache` 这三者是使用的比较多的本地缓存框架。**

`Ehcache` 的话相比于其他两者更加重量。不过，相比于 `Guava Cache` 、 `Spring Cache` 来说， `Ehcache` 支持可以嵌入到 hibernate 和 mybatis 作为多级缓存，并且可以将缓存的数据持久化到本地磁盘中、同时也提供了集群方案。

`Guava Cache` 和 `Spring Cache` 两者的话比较像。

`Guava` 相比于 `Spring Cache` 的话使用的更多一点，它提供了 API 非常方便我们使用，同时也提供了设置缓存有效时间等功能。它的内部实现也比较干净，很多地方都和 `ConcurrentHashMap` 的思想有异曲同工之妙。

使用 `Spring Cache` 的注解实现缓存的话，代码会看着很干净和优雅，但是很容易出现问题比如缓存穿透、内存溢出。

二：**后起之秀 Caffeine。**

相比于 `Guava` 来说 `Caffeine` 在各个方面比如性能要更加优秀，一般建议使用其来替代 `Guava` 。并且， `Guava` 和 `Caffeine` 的使用方式很像！

本地缓存的局限性：

1. **本地缓存对分布式架构支持不友好**，比如同一个相同的服务部署在多台机器上的时候，多个相同服务之间的本地缓存的数据无法共享，因为本地缓存只在当前机器上有。
2. **本地缓存容量受服务部署所在的机器限制明显。** 如果当前系统服务所耗费的内存多，那么本地缓存可用的容量就很少。

## 为什么要有分布式缓存?/为什么不直接用本地缓存?

分布式缓存（Distributed Cache） 看作是一种内存数据库的服务，它的最终作用就是提供缓存数据的服务。一个简单的使用分布式缓存的架构图。我们使用 Nginx 来做负载均衡，部署两个相同的服务到服务器，两个服务使用同一个数据库和缓存。

<img src="https://github.com/Eleven-is-cool/img-folder/blob/master/%E9%9B%86%E4%B8%AD%E5%BC%8F%E7%BC%93%E5%AD%98%E6%9E%B6%E6%9E%84.png?raw=true" alt="集中式缓存架构" style="zoom:50%;" />



## 缓存读写模式/更新策略

:link: [缓存更新的套路](https://coolshell.cn/articles/17416.html)

#### 1. Cache Aside Pattern（旁路缓存模式）

1. 写：更新 DB，然后直接删除 cache 。
2. 读：从 cache 中读取数据，读取到就直接返回，读取不到的话，就从 DB 中取数据返回，然后再把数据放到 cache 中。

Cache Aside Pattern 中服务端需要同时维系 DB 和 cache，并且是以 DB 的结果为准。另外，Cache Aside Pattern 有首次请求数据一定不在 cache 的问题，对于热点数据可以提前放入缓存中。

**Cache Aside Pattern 是我们平时使用比较多的一个缓存读写模式，比较适合读请求比较多的场景。**

#### 2. Read/Write Through Pattern（读写穿透）

Read/Write Through 套路是：服务端把 cache 视为主要数据存储，从中读取数据并将数据写入其中。cache 服务负责将此数据读取和写入 DB，从而减轻了应用程序的职责。

1. 写（Write Through）：先查 cache，cache 中不存在，直接更新 DB。 cache 中存在，则先更新 cache，然后 cache 服务自己更新 DB（**同步更新 cache 和 DB**）。
2. 读(Read Through)： 从 cache 中读取数据，读取到就直接返回 。读取不到的话，先从 DB 加载，写入到 cache 后返回响应。

Read-Through Pattern 实际只是在 Cache-Aside Pattern 之上进行了封装。在 Cache-Aside Pattern 下，发生读请求的时候，如果 cache 中不存在对应的数据，是由客户端自己负责把数据写入 cache，而 Read Through Pattern 则是 cache 服务自己来写入缓存的，这对客户端是透明的。

和 Cache Aside Pattern 一样， Read-Through Pattern 也有首次请求数据一定不在 cache 的问题，对于热点数据可以提前放入缓存中。

#### 3. Write Behind Pattern（异步缓存写入）

Write Behind Pattern 和 Read/Write Through Pattern 很相似，两者都是由 cache 服务来负责 cache 和 DB 的读写。

但是，两个又有很大的不同：**Read/Write Through 是同步更新 cache 和 DB，而 Write Behind Caching 则是只更新缓存，不直接更新 DB，而是改为异步批量的方式来更新 DB。**

**Write Behind Pattern 下 DB 的写性能非常高，尤其适合一些数据经常变化的业务场景比如说一篇文章的点赞数量、阅读数量。** 往常一篇文章被点赞 500 次的话，需要重复修改 500 次 DB，但是在 Write Behind Pattern 下可能只需要修改一次 DB 就可以了。

但是，这种模式同样也给 DB 和 Cache 一致性带来了新的考验，很多时候如果数据还没异步更新到 DB 的话，Cache 服务宕机就 gg 了。

## 介绍一下 Redis

**Redis 就是一个使用 C 语言开发的数据库**，不过与传统数据库不同的是 **Redis 的数据是存在内存中的** ，也就是它是内存数据库，所以读写速度非常快，因此 Redis 被广泛应用于缓存方向。

Redis 除了做缓存之外，Redis 也经常用来做**分布式锁**，甚至是**消息队列**。

Redis 提供了多种数据类型来支持不同的业务场景。Redis 还支持事务 、持久化、Lua 脚本、多种集群方案。

## Redis 和 Memcached 的区别和共同点

**共同点** ：

1. 都是基于内存的数据库，一般都用来当做缓存使用。
2. 都有过期策略。
3. 两者的性能都非常高。

**区别** ：

1. **Redis 支持更丰富的数据类型（支持更复杂的应用场景）**。Redis 不仅仅支持简单的 k/v 类型的数据，同时还提供 list，set，zset，hash 等数据结构的存储。Memcached 只支持最简单的 k/v 数据类型。
2. **Redis 支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用，而 Memecache 把数据全部存在内存之中。**
3. **Redis 有灾难恢复机制。** 因为可以把缓存中的数据持久化到磁盘上。
4. **Redis 在服务器内存使用完之后，可以将不用的数据放到磁盘上。但是，Memcached 在服务器内存使用完之后，就会直接报异常。**
5. **Memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据；但是 Redis 目前是原生支持 cluster 模式的。**
6. **Memcached 是多线程，非阻塞 IO 复用的网络模型；Redis 使用单线程的多路 IO 复用模型。** （Redis 6.0 引入了多线程 IO ）
7. **Redis 支持发布订阅模型、Lua 脚本、事务等功能，而 Memcached 不支持。并且，Redis 支持更多的编程语言。**
8. **Memcached过期数据的删除策略只用了惰性删除，而 Redis 同时使用了惰性删除与定期删除。**

## Redis 有哪些类型

####  一、string

1. **介绍** ：string 数据结构是简单的 key-value 类型。虽然 Redis 是用 C 语言写的，但是 Redis 并没有使用 C 的字符串表示，而是自己构建了一种 **简单动态字符串**（simple dynamic string，**SDS**）。相比于 C 的原生字符串，Redis 的 SDS 不光可以保存文本数据还可以保存二进制数据，并且获取字符串长度复杂度为 O(1)（C 字符串为 O(N)）,除此之外,Redis 的 SDS API 是安全的，不会造成缓冲区溢出。
2. **常用命令:** `set,get,strlen,exists,dect,incr,setex` 等等。
3. **应用场景** ：一般常用在需要计数的场景，比如用户的访问次数、热点文章的点赞转发数量等等。

下面我们简单看看它的使用！

**普通字符串的基本操作：**

```shell
127.0.0.1:6379> set key value #设置 key-value 类型的值
OK
127.0.0.1:6379> get key # 根据 key 获得对应的 value
"value"
127.0.0.1:6379> exists key  # 判断某个 key 是否存在
(integer) 1
127.0.0.1:6379> strlen key # 返回 key 所储存的字符串值的长度。
(integer) 5
127.0.0.1:6379> del key # 删除某个 key 对应的值
(integer) 1
127.0.0.1:6379> get key
(nil)
```

**批量设置** :

```shell
127.0.0.1:6379> mset key1 value1 key2 value2 # 批量设置 key-value 类型的值
OK
127.0.0.1:6379> mget key1 key2 # 批量获取多个 key 对应的 value
1) "value1"
2) "value2"
```

**计数器（字符串的内容为整数的时候可以使用）：**

```shell
127.0.0.1:6379> set number 1
OK
127.0.0.1:6379> incr number # 将 key 中储存的数字值增一
(integer) 2
127.0.0.1:6379> get number
"2"
127.0.0.1:6379> decr number # 将 key 中储存的数字值减一
(integer) 1
127.0.0.1:6379> get number
"1"
```

**过期**：

```shell
127.0.0.1:6379> expire key  60 # 数据在 60s 后过期
(integer) 1
127.0.0.1:6379> setex key 60 value # 数据在 60s 后过期 (setex:[set] + [ex]pire)
OK
127.0.0.1:6379> ttl key # 查看数据还有多久过期
(integer) 56
```

#### 二、 list

1. **介绍：** **list** 即是 **链表**。链表是一种非常常见的数据结构，特点是易于数据元素的插入和删除并且且可以灵活调整链表长度，但是链表的随机访问困难。许多高级编程语言都内置了链表的实现比如 Java 中的 **LinkedList**，但是 C 语言并没有实现链表，所以 Redis 实现了自己的链表数据结构。Redis 的 list 的实现为一个 **双向链表**，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。
2. **常用命令：** `rpush,lpop,lpush,rpop,lrange、llen` 等。
3. **应用场景：** 发布与订阅或者说消息队列、慢查询。

下面我们简单看看它的使用！

**通过 `rpush/lpop` 实现队列：**

```shell
127.0.0.1:6379> rpush myList value1 # 向 list 的头部（右边）添加元素
(integer) 1
127.0.0.1:6379> rpush myList value2 value3 # 向list的头部（最右边）添加多个元素
(integer) 3
127.0.0.1:6379> lpop myList # 将 list的尾部(最左边)元素取出
"value1"
127.0.0.1:6379> lrange myList 0 1 # 查看对应下标的list列表， 0 为 start,1为 end
1) "value2"
2) "value3"
127.0.0.1:6379> lrange myList 0 -1 # 查看列表中的所有元素，-1表示倒数第一
1) "value2"
2) "value3"
```

**通过 `rpush/rpop` 实现栈：**

```shell
127.0.0.1:6379> rpush myList2 value1 value2 value3
(integer) 3
127.0.0.1:6379> rpop myList2 # 将 list 的头部(最右边)元素取出
"value3"
```

**通过 `lrange` 查看对应下标范围的列表元素：**

```shell
127.0.0.1:6379> rpush myList value1 value2 value3
(integer) 3
127.0.0.1:6379> lrange myList 0 1 # 查看对应下标的list列表， 0 为 start,1为 end
1) "value1"
2) "value2"
127.0.0.1:6379> lrange myList 0 -1 # 查看列表中的所有元素，-1表示倒数第一
1) "value1"
2) "value2"
3) "value3"
```

通过 `lrange` 命令，你可以基于 list 实现分页查询，性能非常高！

**通过 `llen` 查看链表长度：**

```shell
127.0.0.1:6379> llen myList
(integer) 3
```

#### 三、hash

1. **介绍** ：hash 类似于 JDK1.8 前的 HashMap，内部实现也差不多(数组 + 链表)。不过，Redis 的 hash 做了更多优化。另外，hash 是一个 string 类型的 field 和 value 的映射表，**特别适合用于存储对象**，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值。 比如我们可以 hash 数据结构来存储用户信息，商品信息等等。
2. **常用命令：** `hset,hmset,hexists,hget,hgetall,hkeys,hvals` 等。
3. **应用场景:** 系统中对象数据的存储。

下面我们简单看看它的使用！

```shell
127.0.0.1:6379> hset userInfoKey name "guide" description "dev" age "24"
OK
127.0.0.1:6379> hexists userInfoKey name # 查看 key 对应的 value中指定的字段是否存在。
(integer) 1
127.0.0.1:6379> hget userInfoKey name # 获取存储在哈希表中指定字段的值。
"guide"
127.0.0.1:6379> hget userInfoKey age
"24"
127.0.0.1:6379> hgetall userInfoKey # 获取在哈希表中指定 key 的所有字段和值
1) "name"
2) "guide"
3) "description"
4) "dev"
5) "age"
6) "24"
127.0.0.1:6379> hkeys userInfoKey # 获取 key 列表
1) "name"
2) "description"
3) "age"
127.0.0.1:6379> hvals userInfoKey # 获取 value 列表
1) "eleven"
2) "dev"
3) "24"
127.0.0.1:6379> hset userInfoKey name "elevenCool" # 修改某个字段对应的值
127.0.0.1:6379> hget userInfoKey name
"elevenCool"
```

#### 四、set

1. **介绍 ：** set 类似于 Java 中的 `HashSet` 。Redis 中的 set 类型是一种无序集合，集合中的元素没有先后顺序。当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的。可以基于 set 轻易实现交集、并集、差集的操作。比如：你可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis 可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程。
2. **常用命令：** `sadd,spop,smembers,sismember,scard,sinterstore,sunion` 等。
3. **应用场景:** 需要存放的数据不能重复以及需要获取多个数据源交集和并集等场景

下面我们简单看看它的使用！

```shell
127.0.0.1:6379> sadd mySet value1 value2 # 添加元素进去
(integer) 2
127.0.0.1:6379> sadd mySet value1 # 不允许有重复元素
(integer) 0
127.0.0.1:6379> smembers mySet # 查看 set 中所有的元素
1) "value1"
2) "value2"
127.0.0.1:6379> scard mySet # 查看 set 的长度
(integer) 2
127.0.0.1:6379> sismember mySet value1 # 检查某个元素是否存在set 中，只能接收单个元素
(integer) 1
127.0.0.1:6379> sadd mySet2 value2 value3
(integer) 2
127.0.0.1:6379> sinterstore mySet3 mySet mySet2 # 获取 mySet 和 mySet2 的交集并存放在 mySet3 中
(integer) 1
127.0.0.1:6379> smembers mySet3
1) "value2"
```

#### 五、sorted set（zset）

:link: [内部实现就依赖了一种叫做 **「跳跃列表」** 的数据结构](https://mp.weixin.qq.com/s/NOsXdrMrWwq4NTm180a6vw)

1. **介绍：** 和 set 相比，sorted set 增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，还可以通过 score 的范围来获取元素的列表。有点像是 Java 中 HashMap 和 TreeSet 的结合体。
2. **常用命令：** `zadd,zcard,zscore,zrange,zrevrange,zrem` 等。
3. **应用场景：** 需要对数据根据某个权重进行排序的场景。比如在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息。

```shell
127.0.0.1:6379> zadd myZset 3.0 value1 # 添加元素到 sorted set 中 3.0 为权重
(integer) 1
127.0.0.1:6379> zadd myZset 2.0 value2 1.0 value3 # 一次添加多个元素
(integer) 2
127.0.0.1:6379> zcard myZset # 查看 sorted set 中的元素数量
(integer) 3
127.0.0.1:6379> zscore myZset value1 # 查看某个 value 的权重
"3"
127.0.0.1:6379> zrange  myZset 0 -1 # 顺序输出某个范围区间的元素，0 -1 表示输出所有元素
1) "value3"
2) "value2"
3) "value1"
127.0.0.1:6379> zrange  myZset 0 1 # 顺序输出某个范围区间的元素，0 为 start  1 为 stop
1) "value3"
2) "value2"
127.0.0.1:6379> zrevrange  myZset 0 1 # 逆序输出某个范围区间的元素，0 为 start  1 为 stop
1) "value1"
2) "value2"
```

## Redis 单线程模型详解

**Redis 基于 Reactor 模式来设计开发了自己的一套高效的事件处理模型** 。Redis 通过 **IO 多路复用程序**来监听来自客户端的大量连接（或者说是监听多个 socket），它会将感兴趣的事件及类型(读、写）注册到内核中并监听每个事件是否发生。

这样的好处非常明显： **I/O 多路复用技术的使用让 Redis 不需要额外创建多余的线程来监听客户端的大量连接，降低了资源的消耗**（和 NIO 中的 `Selector` 组件很像）。

## Redis 没有使用多线程？为什么不使用多线程？

**Redis 在 4.0 之后的版本中就已经加入了对多线程的支持。**

Redis 4.0 增加的多线程主要是针对一些大键值对的删除操作的命令，使用这些命令就会使用主处理之外的其他线程来“异步处理”。

大体上来说，**Redis 6.0 之前主要还是单线程处理。**

**Redis6.0 之前 为什么不使用多线程？**

我觉得主要原因有下面 3 个：

1. 单线程编程容易并且更容易维护；
2. Redis 的性能瓶颈不再 CPU ，主要在内存和网络；
3. 多线程就会存在死锁、线程上下文切换等问题，甚至会影响性能。

**Redis6.0 引入多线程主要是为了提高网络 IO 读写性能**，因为这个算是 Redis 中的一个性能瓶颈（Redis 的瓶颈主要受限于内存和网络）。

虽然，Redis6.0 引入了多线程，但是 Redis 的多线程只是在网络数据的读写这类耗时操作上使用了， 执行命令仍然是单线程顺序执行。

## 过期的数据的删除策略

1. **惰性删除** ：只会在取出key的时候才对数据进行过期检查。这样对CPU最友好，但是可能会造成太多过期 key 没有被删除。
2. **定期删除** ： 每隔一段时间抽取一批 key 执行删除过期key操作。并且，Redis 底层会并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响。

因为还是可能存在定期删除和惰性删除漏掉了很多过期 key 的情况。这样就导致大量过期 key 堆积在内存里，然后就Out of memory了。

怎么解决这个问题呢？答案就是： **Redis 内存淘汰机制。**

## Redis 内存淘汰机制了解么？

Redis 提供 6 种数据淘汰策略：

1. **volatile-lru（least frequently used）**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru（least recently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！

4.0 版本后增加以下两种：

1. **volatile-lfu**：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
2. **allkeys-lfu**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

## Redis 持久化机制

将数据写入内存的同时，异步的慢慢的将数据写入磁盘。大部分原因是为了之后重用数据（比如**重启机器**、**机器故障之后恢复数据**），或者是为了防止系统故障而将数据备份到一个远程位置。

## Redis 如何实现持久化

:link: [Redis常见面试题](https://www.cnblogs.com/jasontec/p/9699242.html)

1. RDB 实现 Redis 数据持久化（默认方式）

   - RDB：将某一时刻的所有数据保存到一个 RDB 文件中。（RDB 文件是一个经过**压缩**的二进制文件）

   - 因为是特定条件下进行一次持久化（每隔一段时间），就可能会导致一旦 redis 崩溃，再次恢复时，可能会导致部分数据丢失。

2. AOF 持久化方案

   - AOF：先把命令追加到操作日志的尾部，保存所有的历史操作。AOF 是通过保存 Redis 服务器所执行的**写命令**来记录数据库的数据的。

   - 相比于 RDB 持久化方案的优点：

     (1) 数据非常完整，故障恢复丢失数据少

     (2) 可对历史操作进行处理

3. RDB 和 AOF 对过期键的策略

   **RDB 持久化对过期键的策略：**

   - 执行 `SAVE` 或者 `BGSAVE` 命令创建出的 RDB 文件，程序会对数据库中的过期键检查，**已过期的键不会保存在 RDB 文件中**。
   - 载入 RDB 文件时，程序同样会对 RDB 文件中的键进行检查，**过期的键会被忽略**。

   **AOF 持久化对过期键的策略：**

   - 如果数据库的键已过期，但还没被惰性/定期删除，AOF 文件不会因为这个过期键产生任何影响(也就说会保留)，当过期的键被删除了以后，会追加一条 DEL 命令来显示记录该键被删除了
   - 重写 AOF 文件时，程序会对 RDB 文件中的键进行检查，**过期的键会被忽略**。

4. RDB和AOF并不互斥，它俩可以**同时使用**。

   - RDB 的优点：载入时**恢复数据快**、文件体积小。
   - RDB 的缺点：会一定程度上**丢失数据**（因为系统一旦在定时持久化之前出现宕机现象，此前没有来得及写入磁盘的数据都将丢失）。
   - AOF 的优点：丢失数据少（默认配置只丢失一秒的数据）。
   - AOF 的缺点：**恢复数据相对较慢**，文件体积大。

## Redis 事务

Redis 可以通过 **MULTI，EXEC，DISCARD 和 WATCH** 等命令来实现事务(transaction)功能。

**Redis事务提供了一种将多个命令请求打包的功能。然后，再按顺序执行打包的所有命令，并且不会被中途打断。**

## 缓存穿透

#### 概念

缓存穿透说简单点就是大量请求的 key 根本不存在于缓存中，导致请求直接到了数据库上，根本没有经过缓存这一层。举个例子：某个黑客故意制造我们缓存中不存在的 key 发起大量请求，导致大量请求落到数据库。

#### 解决方法

最基本的就是首先做好参数校验，一些不合法的参数请求直接抛出异常信息返回给客户端。比如查询的数据库 id 不能小于 0、传入的邮箱格式不对的时候直接返回错误消息给客户端等等。

**1）缓存无效 key**

对于查到DB中不存在的数据，将其key设置过期时间并放到redis中

**2）布隆过滤器**

**流程：**把**所有可能存在的请求的值**都存放在布隆过滤器中，当用户请求过来，先判断用户发来的请求的值是否存在于布隆过滤器中。不存在的话，直接返回请求参数错误信息给客户端，存在的话才会走下面的流程。

**当一个元素加入布隆过滤器中的时候，会进行哪些操作：**

1. 使用布隆过滤器中的哈希函数对元素值进行计算，得到哈希值（有几个哈希函数得到几个哈希值）。

2. 根据得到的哈希值，在位数组中把对应下标的值置为 1。

**当判断一个元素是否存在于布隆过滤器的时候，会进行哪些操作：**

1. 对给定元素再次进行相同的哈希计算；

2. 得到值之后判断位数组中的每个元素是否都为 1，如果值都为 1，那么说明这个值在布隆过滤器中，如果存在一个值不为 1，说明该元素不在布隆过滤器中。

   **不同的字符串可能哈希出来的位置相同。** （可以适当增加位数组大小或者调整我们的哈希函数来降低概率）

## 缓存雪崩

#### 概念

1. 缓存在同一时间大面积的失效，后面的请求都直接落到了数据库上，造成数据库短时间内承受大量请求。

   例子：系统的缓存模块出了问题比如宕机导致不可用。造成系统的所有访问，都要走数据库。 

2. 有一些被大量访问数据（热点缓存）在某一时刻大面积失效，导致对应的请求直接落到了数据库上。

   例子：秒杀开始 12 个小时之前，我们统一存放了一批商品到 Redis 中，设置的缓存过期时间也是 12 个小时，那么秒杀开始的时候，这些秒杀的商品的访问直接就失效了。

#### 解决方法

1. **针对 Redis 服务不可用的情况：**
   - 采用 Redis 集群，避免单机出现问题整个缓存服务都没办法使用。
   - 限流，避免同时处理大量的请求。

2. **针对热点缓存失效的情况：**
   - 设置不同的失效时间比如随机设置缓存的失效时间。
   - 缓存永不失效。

## 缓存击穿

#### 概念

一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发直接请求数据库。

#### 解决方法

- 设置热点数据永不过期

- 增加互斥锁

## 如何保证缓存和数据库数据的一致性？

对 **Cache Aside Pattern（旁路缓存模式）** 来说。

Cache Aside Pattern 中遇到写请求是这样的：更新 DB，然后直接删除 cache 。

如果更新数据库成功，而删除缓存这一步失败的情况的话，简单说两个解决方案：

1. **缓存失效时间变短（不推荐，治标不治本）** ：我们让缓存数据的过期时间变短，这样的话缓存就会从数据库中加载数据。另外，这种解决办法对于先操作缓存后操作数据库的场景不适用。

2. **增加cache更新重试机制（常用）**： 如果 cache 服务当前不可用导致缓存删除失败的话，我们就隔一段时间进行重试，重试次数可以自己定。如果多次重试还是失败的话，我们可以把当前更新失败的 key 存入队列中，等缓存服务可用之后，再将 缓存中对应的 key 删除即可

## 分布式锁

[分布式锁的实现之 redis 篇-小米信息部技术团队](https://xiaomi-info.github.io/2019/12/17/redis-distributed-lock/)

1. 加锁的原子性：通过**加入超时时间释放锁**

2. 防止误删：释放锁的时候**判断**是不是自己的锁

3. 为了使得判断和释放锁是原子操作而不是独立操作，**需要使用Lua脚本**。

4. 为了不让同一时间有多个线程访问代码块（A线程任务未执行完，超时释放锁，B拿到锁执行任务，这时候就是同一时间有多个线程访问代码块的场景），可以让获得锁的线程开启一个**守护线程**，用来给快要过期的锁“续航”。

   当过去了29秒，线程A还没执行完，这时候守护线程会执行expire指令，为这把锁“续命”20秒。守护线程从第29秒开始执行，每20秒执行一次。

   另一种情况，如果节点1 忽然断电，由于线程A和守护线程在同一个进程，守护线程也会停下。这把锁到了超时的时候，没人给它续命，也就自动释放了。

```shell
# 加锁：
String threadId = Thread.currentThread().getId()
set（key, threadId, 30, NX）

# 解锁：
if（threadId.equals(redisClient.get(key))）{
    del(key)
}
```

但是，这样做又隐含了一个新的问题，**判断和释放锁是两个独立操作，不是原子性**。

所以这一块要用Lua脚本来实现：

```shell
String luaScript = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";

redisClient.eval(luaScript , Collections.singletonList(key), Collections.singletonList(threadId));
```

这样一来，验证和删除过程就是原子操作了。

## HyperLoglog

[Reids—神奇的HyperLoglog解决统计问题](https://mp.weixin.qq.com/s/9dtGe3d_mbbxW5FpVPDNow)

## Redis 内部结构

键值对

## 聊聊 Redis 使用场景

1. 秒杀库存扣减
2. APP首页的访问流量高峰 
3. :link: [使用场景](https://blog.csdn.net/csdnlijingran/article/details/88068598)

## BitMap

:link: [详解redis的bitmap在亿级项目中的应用](https://blog.csdn.net/u011957758/article/details/74783347?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)

:link: [Redis中bitmap的妙用](https://segmentfault.com/a/1190000008188655)

:link: [利用redis + bitmap解决统计数据相关问题（也变相解决了大数据量内存占用问题）](https://blog.csdn.net/u012888052/article/details/80380143?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param)

## Rehash机制

### Rehash的简述

随着操作的不断执行， 哈希表保存的键值对会逐渐地增多或者减少， **为了让哈希表的负载因子（load factor）维持在一个合理的范围之内**， **当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。**

**扩展和收缩哈希表的工作可以通过执行 rehash （重新散列）操作来完成。**

**redis内部采用了字典的数据结构实现了键值对的存储**，从结构上看每个字典中都包含了**两个hashtable**。

那么为什么一个字典会需要两个hashtable？

首先redis在**正常读写时会用到一个hashtable**，而**另一个hashtable的作用实际上是作为字典在进行rehash时的一个临时载体**。**redis开始只会用一个hashtable去读写**，如果这个hashtable的数据量**增加或者缩减到某个值**，**到达了rehash的条件**，**redis便会开始根据数据量和链（bucket）的个数初始化那个备用的hashtable**，来使这个hashtable从容量上满足后续的使用，并开始**把之前的hashtable的数据迁移到这个新的hashtable上来**，当然这种迁移是对每个节点值进行一次hash运算。**等到数据全部迁移完成，再进行一次hashtable的地址更名，把这个备用的hashtable为正式的hashtable，同时清空另一个hashtable以供下一次rehash使用。**

### Rehash的过程

我们假设 ht[0] 为正在使用的 hashtable，ht[1] 为 rehash 之后的备用 hashtable

步骤如下：

1. 为字典的备用哈希表分配空间：

   - 如果执行的是扩展操作，那么备用哈希表的大小为第一个大于等于(已用节点个数)*2的2n（2的n次方幂）。

   - 如果执行的是收缩操作，那么备用哈希表的大小为第一个大于等于(已用节点个数)的2n。

2. 在字典中维持一个索引计数器变量 rehashidx，并将它的值设置为0，表示 rehash 工作正式开始（为 -1 时表示没有进行 rehash）。

3. rehash 进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在 rehashidx 索引上的所有键值对 rehash 到ht[1]，当一次rehash 工作完成之后，程序将 rehashidx 属性的值+1。同时在 serverCron 中调用 rehash 相关函数，在1ms的时间内，进行 rehash 处理，每次仅处理少量的转移任务(100个元素)。

4. 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被 rehash 至ht[1]，这时程序将 rehashidx 属性的值设为-1，表示 rehash 操作已完成。

`rehash ` 是以 `bucket(桶)` 为基本单位进行渐进式的数据迁移的，每步完成一个 `bucket` 的迁移，直至所有数据迁移完毕。一个 `bucket` 对应哈希表数组中的一条 `entry` 链表。新版本的`dictRehash()` 还加入了一个最大访问空桶数 `(empty_visits)` 的限制来进一步减小可能引起阻塞的时间。

## redis面试题

:link: [坑人无数的Redis面试题](https://blog.csdn.net/u011405515/article/details/79190652)

:link: ​[妈妈再也不担心我面试被Redis问得脸都绿了](https://mp.weixin.qq.com/s/vXBFscXqDcXS_VaIERplMQ)

![resdis.jpg](https://raw.githubusercontent.com/Eleven-is-cool/img-folder/master/resdis.jpg)

# 