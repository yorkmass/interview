# Redis相关

## 1、缓存冷启动问题与缓存预热解决方案

### 概述

缓存冷启动就是缓存中没有数据，由于缓存冷启动一点数据都没有，如果直接就对外提供服务了，那么并发量上来mysql就裸奔挂掉了。
 因此需要通过缓存预热的方案，提前给 redis 灌入部分数据后再提供服务。

### 缓存冷启动场景

系统第一次上线启动，或者系统在 redis 故障的情况下重新启动，这时在高并发的场景下就会出现所有的流量 都会打到 mysql（原始数据库） 上去，导致 mysql 崩溃。



![img](https:////upload-images.jianshu.io/upload_images/18924448-eb53b9180090a46a.png?imageMogr2/auto-orient/strip|imageView2/2/w/554/format/webp)

缓存冷启动

- 新系统第一次上线，此时在缓存里是没有数据的。
- 系统在线上稳定运行着，但是突然redis 缓存崩了，而且不幸的是，数据全都无法找回来。

### 缓存预热解决方案

#### 1.缓存预热问题

- 数据量太大的话，无法将所有数据放入 redis 中：耗费时间过长或 redis 根本无法容纳下所有的数据；
- 需要根据当天的具体访问情况，实时统计出访问频率较高的热数据；
- 将访问频率较高的热数据写入 redis 中，肯定数据也比较多， 我们也得多个服务并行读取数据去写，并行的分布式缓存预热。

#### 2.缓存预热大致思路

- nginx +lua 将访问流量上报到 kafka 中，统计出当前最新的实时的热数据，将商品详情页访问的请求对应的流量日志实时上报到 kafka 中；
- storm 从 kafka 中消费数据，实时统计访问次数；
- 访问次数基于 LRU 内存数据结构的存储方案。

由于storm 中读写数据频繁，并且数据量大，需要采用LRU 内存架构：
 这种场景不适合采用 redis 或者 mysql：

- redis 可能会出现故障，会影响 storm 的稳定性；
- mysql扛不住高并发读写；
- hbase：hadoop 生态组合还是不错的，但是维护太重了；
   实际场景就是：统计出最近一段时间访问最频繁的商品，进行访问计数， 同时维护出一个前 N 个访问最多的商品 list 即可。
- 也就是热数据：最近一段时间（如最近 1 小时、5 分钟），1 万个商品请求， - 统计这段时间内每个商品的访问次数，排序后做出一个 top n 列表。
- 计算好每个 task 大致要存放的商品访问次数的数量，计算出大小， 然后构建一个 LRU MAP，它能够给你一个剩下访问次数最多的商品列表，访问高的才能存活。
   LRU MAP 有开源的实现，apach commons collections 中有提供，设置好 map 的最大大小， 就会自动根据 LRU 算法去剔除多余的数据，保证内存使用限制， 即时有部分数据被干掉了，下次会从 0 开始统计，也没有关系，因为被 LRU 算法干掉了， 就表示它不是热数据，说明最近一段时间都很少访问了，热度下降了。

#### 3.分布式并行缓存预热解决方案

- 每个 Storm task 启动时，基于 zk 分布式锁，将自己的 ID 写入 zk 同一个节点中：
   这个 id 写到一个固定节点中，形成一个 task id 列表， 后续可以通过这个 id 列表去拿到对于 task 存储在 zk node 上的 topn 列表。
- 每个 Storm task 负责完成自己这里的热数据统计。
   比如每隔一段时间，就遍历下这个 map，维护并更新一个前 n 个商品的 list。
- 定时同步到 zk 中去。
   写一个后台线程，每隔一段时间，比如 1 分钟，将这个 task 所有的商品排名算一次 将排名前 n 的热数据 list 同步到 zk 中去。
- 需要一个服务，根据 top n 列表在 mysql 中获取数据往 redis 中存
   这个服务有会部署多个实例，在启动时会拉取 storm task id 列表， 然后通过 zk 分布式锁，基于 id 去加锁，获取到这个 task id 节点中存储的 topn 列表， 然后读取 mysql 中的数据，存储在 redis 中。
   这个服务可以是单独的服务，也可以放在缓存服务中。

### 总结

冷启动是说缓存中没有数据但是缓存短时间又恢复正常后的流量被大量打到 mysql。
 那么通过缓存预热来解决缓存冷启动问题：

- 使用 stom 实时计算出最近一段时间内的 n 个 topn 列表，并存储在 zk task id 节点上。
- 多服务通过 task id 进行分布式锁，获取 topn 列表，去 mysql 拉取数据放入 redis 中。
   利用storm 创建大量并行的 task 和数据分组策略， 让大量的访问日志分发到 n 个 task 中，让 storm 这种抗住大量并发访问量的计算能力， 这里是计算出 n 个 topn 列表，也就是大量的热数据。而不是唯一的一份 topn 列表， 而且是最近一段时间内的（通过这种分而治之方式 + 分段时间来重复计算自己负责的部分结果数据实现的）。

## 2、缓存雪崩

缓存雪崩我们可以简单的理解为：由于原有缓存失效，新缓存未到期间(例如：我们设置缓存时采用了相同的过期时间，在同一时刻出现大面积的缓存过期)，所有原本应该访问缓存的请求都去查询数据库了，而对数据库CPU和内存造成巨大压力，严重的会造成数据库宕机。从而形成一系列连锁反应，造成整个系统崩溃。

缓存正常从Redis中获取，示意图如下：

![阿里P8技术专家细究分布式缓存问题](http://p1.pstatp.com/large/pgc-image/1521269040783ad8b0b3415)

缓存失效瞬间示意图如下：

![阿里P8技术专家细究分布式缓存问题](http://p3.pstatp.com/large/pgc-image/15212690408812aad2545bb)

缓存雪崩的解决方案：

（1）碰到这种情况，一般并发量不是特别多的时候，使用最多的解决方案是加锁排队，伪代码如下：

![img](https://images2018.cnblogs.com/blog/1227483/201803/1227483-20180318100156533-1480686317.png)

加锁排队只是为了减轻数据库的压力，并没有提高系统吞吐量。假设在高并发下，缓存重建期间key是锁着的，这是过来1000个请求999个都在阻塞的。同样会导致用户等待超时，这是个治标不治本的方法！

注意：加锁排队的解决方式分布式环境的并发问题，有可能还要解决分布式锁的问题；线程还会被阻塞，用户体验很差！因此，在真正的高并发场景下很少使用！

（2）给每一个缓存数据增加相应的缓存标记，记录缓存的是否失效，如果缓存标记失效，则更新数据缓存，实例伪代码如下：

![img](https://images2018.cnblogs.com/blog/1227483/201803/1227483-20180318100506324-706047489.png)

解释说明：

1、缓存标记：记录缓存数据是否过期，如果过期会触发通知另外的线程在后台去更新实际key的缓存；

2、缓存数据：它的过期时间比缓存标记的时间延长1倍，例：标记缓存时间30分钟，数据缓存设置为60分钟。 这样，当缓存标记key过期后，实际缓存还能把旧数据返回给调用端，直到另外的线程在后台更新完成后，才会返回新缓存。

关于缓存崩溃的解决方法，这里提出了三种方案：使用锁或队列、设置过期标志更新缓存、为key设置不同的缓存失效时间，还有一各被称为“二级缓存”的解决方法，有兴趣的读者可以自行研究。

## 3、缓存穿透

缓存穿透是指用户查询数据，在数据库没有，自然在缓存中也不会有。这样就导致用户查询的时候，在缓存中找不到，每次都要去数据库再查询一遍，然后返回空（相当于进行了两次无用的查询）。这样请求就绕过缓存直接查数据库，这也是经常提的缓存命中率问题。

 缓存穿透解决方案：

（1）采用布隆过滤器，将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。

（2）如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。通过这个直接设置的默认值存放到缓存，这样第二次到缓存中获取就有值了，而不会继续访问数据库，这种办法最简单粗暴！

 ![img](https://images2018.cnblogs.com/blog/1227483/201803/1227483-20180318101409345-2084104244.png)

 把空结果也给缓存起来，这样下次同样的请求就可以直接返回空了，即可以避免当查询的值为空时引起的缓存穿透。同时也可以单独设置个缓存区域存储空值，对要查询的key进行预先校验，然后再放行给后面的正常缓存处理逻辑。

## 4、缓存预热

 缓存预热就是系统上线后，提前将相关的缓存数据直接加载到缓存系统。避免在用户请求的时候，先查询数据库，然后再将数据缓存的问题！用户直接查询事先被预热的缓存数据！

 缓存预热解决方案：

（1）直接写个缓存刷新页面，上线时手工操作下；

（2）数据量不大，可以在项目启动的时候自动进行加载；

（3）定时刷新缓存；

## 5、缓存更新

除了缓存服务器自带的缓存失效策略之外（Redis默认的有6中策略可供选择），我们还可以根据具体的业务需求进行自定义的缓存淘汰，常见的策略有两种：

（1）定时去清理过期的缓存；

（2）当有用户请求过来时，再判断这个请求所用到的缓存是否过期，过期的话就去底层系统得到新数据并更新缓存。

两者各有优劣，第一种的缺点是维护大量缓存的key是比较麻烦的，第二种的缺点就是每次用户请求过来都要判断缓存失效，逻辑相对比较复杂！具体用哪种方案，大家可以根据自己的应用场景来权衡。

## 6、缓存降级

当访问量剧增、服务出现问题（如响应时间慢或不响应）或非核心服务影响到核心流程的性能时，仍然需要保证服务还是可用的，即使是有损服务。系统可以根据一些关键数据进行自动降级，也可以配置开关实现人工降级。

降级的最终目的是保证核心服务可用，即使是有损的。而且有些服务是无法降级的（如加入购物车、结算）。

在进行降级之前要对系统进行梳理，看看系统是不是可以丢卒保帅；从而梳理出哪些必须誓死保护，哪些可降级；比如可以参考日志级别设置预案：

（1）一般：比如有些服务偶尔因为网络抖动或者服务正在上线而超时，可以自动降级；

（2）警告：有些服务在一段时间内成功率有波动（如在95~100%之间），可以自动降级或人工降级，并发送告警；

（3）错误：比如可用率低于90%，或者数据库连接池被打爆了，或者访问量突然猛增到系统能承受的最大阀值，此时可以根据情况自动降级或者人工降级；

（4）严重错误：比如因为特殊原因数据错误了，此时需要紧急人工降级。