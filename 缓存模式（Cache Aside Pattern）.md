缓存在计算机领域中很常见的应用，用于减少与低效/速率的存储介质的交互次数，提高系统性能

![Using the Cache-Aside pattern to store data in the cache](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/cache-aside-diagram.png)



主要的思想就是，当客户端请求数据时不直接访问数据库，而是先在缓存中查询，如果存在则直接返回，如果不存在再访问数据库，返回数据的同时再将其加载到缓存中，以便下次的访问直接从缓存中取。

使用缓存能够有效的提高系统的数据访问效率和并发性，但也会带来一些问题：如数据的一致性和引入额外模块带来的复杂性等问题。

## 数据一致性

为了保持缓存和底层数据库之间的数据一致性，通常有以下几种数据的读写方式

#### Read-Through Caching（读透、读穿）

当应用从缓存中读取一个键值对，假设key为`X` ，而恰好`X`不在缓存中，那么缓存系统换自动申请去数据库中加载x。如果x在数据库中，那么便将其加载到缓存中以便将来使用。这种方式成为 Read-Through Caching。而Refresh-Ahead Caching能够提供缓存的读性能。

#### Refresh-Ahead Caching（提前刷新）

通过配置，能够在热点key即将过期时重新加载到缓存中，这样做的结果就是能够减少访问频率高的key在缓存中过期时被访问而需重新加载带来的影响（缓存miss重新访问数据库需要时间，这会降低效率）。这种方式也经常被用来解决**缓存击穿**问题。

#### Write-Through Caching（写透、写穿）

当客户端应用对缓存中的数据进行更新时（update），缓存系统将先更新缓存中的值，然后再去更新底层数据库中的数据，在这两步完成之前应用的写操作将不会返回。这种方式丝毫不会提升写数据的性能，因为仍然等待访问数据库的时间。而Write-Behind-Caching能够有效提高写数据的性能。

#### Write-Behind Caching（）

不同于Write-Through，在更新完缓存中的数据后，数据库更新操作是异步进行的，系统会维护一个**write-behind queue**在一定的延迟时间后对需要更新到数据库的数据进行写操作。这种方式有以下四种好处：

1. 对数据库的写操作是由其他线程异步进行的，用户无需过多等待。

	> The application improves in performance, because the user does not have to wait for data to be written to the underlying data source. (The data is written later, and by a different execution thread.)

2. 大幅减少数据库的负载。因为对数据库操作的异步延迟进行可以将多次更新合并成一次，此外，多次的写操作可能会由于执行一次storeAll方法而被合并成一次事务。

	> The application experiences drastically reduced database load: Since the amount of both read and write operations is reduced, so is the database load. The reads are reduced by caching, as with any other caching approach. The writes, which are typically much more expensive operations, are often reduced because multiple changes to the same object within the write-behind interval are "coalesced" and only written once to the underlying data source ("write-coalescing"). Additionally, writes to multiple cache entries may be combined into a single database transaction ("write-combining") by using the [`CacheStore.storeAll()`](https://download.oracle.com/otn_hosted_doc/coherence/340/com/tangosol/net/cache/CacheStore.html#storeAll(java.util.Map)) method.

3. 将客户端应用与数据库的不可用隔离开。

	> The application is somewhat insulated from database failures: the **Write-Behind** feature can be configured in such a way that a write failure will result in the object being re-queued for write. If the data that the application is using is in the Coherence cache, the application can continue operation without the database being up. This is easily attainable when using the Coherence Partitioned Cache, which partitions the entire cache across all participating cluster nodes (with local-storage enabled), thus allowing for enormous caches.

4. 水平可扩展性

	> Linear Scalability: For an application to handle more concurrent users you need only increase the number of nodes in the cluster; the effect on the database in terms of load can be tuned by increasing the write-behind interval.

以上几种的数据读写方式，在高并发系统中均会存在一定程度的数据不一致问题，但可以通过相应的方法，实现数据的最终一致性。

## 使用缓存常见的问题

引入缓存会增加系统的复杂性，同时带来一些问题。

### 缓存穿透（Cache Penetration）

穿透，望文生义，可以简单的理解为缓存像透明地带一样被请求穿过。

请求的数据在缓存中不存在，进而访问数据库，而最后发现DB中也不存在。简言之就是请求一个在数据中根本就不存在的数据。

这样的话整个请求过程中线程白忙活一圈。

如果某个时段有大量的恶意请求集中访问不存在的数据，那么这些请求将全部落在数据库中，导致正常请求无法被处理或者数据库服务不可用。

##### 解决方案

1. 在访问数据库后不存在的key的值在缓存中写为null，那么下次请求来的时候可以直接命中并返回null。但这种方法局限性很大，因为不存在的key会很多，这样做的效率很低且会白白占用大量内存去保存空值。一个缓解的方法是将不存在的key写null的同时设置一个较短的过期时间（不超过5分钟）（能够缓解但不能从根本上解决问题）。

2. 使用布隆过滤器。请求来的时候通过布隆过滤器查询key是否存在，不存在直接返回。布隆过滤器查询的特点是，如果查询结果不存在那么这个key一定不存在， 而判断存在时依然可能不存在。因此存在一定的误判性，且这个比例和布隆过滤器的位数组及哈希函数的个数有关系。但该方法仍然能够有效解决缓存穿透的问题。

	> 如果是使用redis来做数据缓存，可以使用第二种解决方法，因为redis自带的位操作功能天然适合作为布隆过滤器的位数组。

### 缓存击穿（Cache breakdown、Hotspot Invalid）

##### 什么是击穿？

在高并发系统中，某时刻大量请求同时查询一个key，而此时这个key刚好失效，导致请求全部落到数据库上。这种现象即称为缓存击穿。

> Cache breakdown is a scenario where the cached data expires and at the same time there are lots of search on the expired data which suddenly cause the searches to hit DB directly and increase the load to the DB layer dramatically.
>
> This would happen in high concurrency environment. Normally in this case, there needs to be a lock on the searched key so that other threads need to wait when some thread is trying to search the key and update the cache. After the cache is updated and lock is released, other threads will be able to read the newly cached data.
>
> Another feasible method is to asynchronously update the cached data through a worker thread so that the hot data will never expire.

##### 可能带来的问题？

某时刻数据库请求量过大，服务器压力剧增可能带来一系列问题甚至宕机

##### 解决方案

1. 将多个热点数据的过期时间设置随机，尽量分散，减少同时出现cache miss的key 的数量。

2. 加锁，当一个线程发现Cache Miss之后，先加锁然后再去数据库获取内容并写入缓存，这个过程中如果其他线程也请求相同数据则会被阻塞从而不会访问数据库。当能够获取到锁时，通过double check会发现缓存中存在该数据也就不会再访问数据库了。

	> 要注意的是，单机和集群环境要使用不同的锁，
	>
	> 对于单机环境，可以直接在线程内加锁；
	>
	> 而集群中需要使用分布式锁，或者使用queue等方式将相同的key查询路由到同一个机器内，再通过单机加锁保证同一key只访问一次数据库。
	>
	> 另外，由于锁的存在，肯定会影响并发效率。

3. 最后一点就是对业务热点数据使用Refresh-Ahead策略，在热点key即将过期之前，重置其过期时间。

	> 这种方式有两种具体实现方法：
	>
	> 1. 单独开启一个线程，定时检查热点key的过期时间，如果过期时间达到阈值，更新过期时间。（如果直接将热点key设置成永不过期，在也需求改变后需要再手动删除涉及到的key）
	> 2. 在每次对热点key的缓存命中时立即重置其过期时间。



### 缓存雪崩 （Cache Avalanche）

##### 什么是雪崩？

大量key同时过期导致缓存失效、或者缓存服务故障等导致缓存不可用，所有请求直接访问数据库，而数据库服务器无法承受负载而宕机。

> Cache avalanche is a scenario where lots of cached data expire at the same time or the cache service is down and all of a sudden all searches of these data will hit DB and cause high load to the DB layer and impact the performance.
>
> To mitigate the problem, some methods can be adopted.
>
> 1. Using clusters to ensure that some cache server instance is in service at any point of time. If Redis is used, can have redis clusters.
> 2. Some other approaches like hystrix circuit breaker and rate limit can be configured so that the underlying system can still serve traffic and avoid high load
> 3. Can adjust the expiration time for different keys so that they will not expire at the same time.

##### 如何解决

###### 事前：

1. 缓存服务要Highly Available，以Redis为例，使用主从+哨兵或者使用redis cluster实现缓存的高可用，避免全面崩盘。

###### 事中：

1. 本地缓存
2. 服务熔断降级

###### 事后：

通过redis的持久化机制，尽快恢复缓存集群





#### 参考文献

https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside

https://docs.oracle.com/cd/E13924_01/coh.340/e13819/readthrough.htm

https://www.pixelstech.net/article/1586522853-What-is-cache-penetration-cache-breakdown-and-cache-avalanche

https://juejin.im/post/5c9a67ac6fb9a070cb24bf34

https://mp.weixin.qq.com/s/eNdG8gpx6NuffxeACFl-iQ