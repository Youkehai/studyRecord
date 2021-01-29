<h1>1.redis的五个内存满了处理策略</h1>
maxmemory 当前已用内存超过maxmemory限定时，触发主动清理策略
<ul>
  <li>volatile-lru：只对设置了过期时间的key进行LRU（默认值）<br/></li>
<li>allkeys-lru ： 删除lru算法的key<br/></li>
<li>volatile-random：随机删除即将过期key<br/></li>
<li>allkeys-random：随机删除<br/></li>
<li>volatile-ttl ： 删除即将过期的<br/></li>
<li>noeviction ： 永不过期，返回错误<br/></li>
  </ul>
当mem_used内存已经超过maxmemory的设定，对于所有的读写请求，都会触发redis.c/freeMemoryIfNeeded(void)函数以清理超出的内存。注意这个清理过程是阻塞的，直到清理出足够的内存空间。所以如果在达到maxmemory并且调用方还在不断写入的情况下，可能会反复触发主动清理策略，导致请求会有一定的延迟。<br/>

清理时会根据用户配置的maxmemory-policy来做适当的清理（一般是LRU或TTL），这里的LRU或TTL策略并不是针对redis的所有key，而是以配置文件中的maxmemory-samples个key作为样本池进行抽样清理。<br/>

maxmemory-samples在redis-3.0.0中的默认配置为5，如果增加，会提高LRU或TTL的精准度，redis作者测试的结果是当这个配置为10时已经非常接近全量LRU的精准度了，并且增加maxmemory-samples会导致在主动清理时消耗更多的CPU时间，建议：<br/>

尽量不要触发maxmemory，最好在mem_used内存占用达到maxmemory的一定比例后，需要考虑调大hz以加快淘汰，或者进行集群扩容。<br/>
如果能够控制住内存，则可以不用修改maxmemory-samples配置；如果Redis本身就作为LRU cache服务（这种服务一般长时间处于maxmemory状态，由Redis自动做LRU淘汰），可以适当调大maxmemory-samples。
这里提一句，实际上redis根本就不会准确的将整个数据库中最久未被使用的键删除，而是每次从数据库中随机取5个键并删除这5个键里最久未被使用的键。上面提到的所有的随机的操作实际上都是这样的，这个5可以用过redis的配置文件中的maxmemeory-samples参数配置。<br/>

<h1>2.redis持久化</h1>
<ul>
  <li>2.1 rdb</li>
   rdb好处，体积小，恢复快，适合大规模的数据恢复。缺点：在没达到备份要求时，可能会丢失数据
  rdb可使用save 或bgsave(后台异步)触发自动备份。
  注意点：SHUTDOWN 和 FLUSHALL 命令都会触发RDB快照，这是一个坑，请大家注意。 
 
  <li>2.2 aof</li> 
   aof好处，数据最多丢失两秒钟，数据的完整性和一致性更高。缺点文件体积大，占用系统内存空间，数据恢复也会越来越慢。
  如果两个持久化策略都开启，那么redis下一次启动，默认会先读取aof的文件。
  但在实际开发中，可能因为某些原因导致appendonly.aof 文件格式异常，从而导致数据还原失败，可以通过命令redis-check-aof --fix appendonly.aof 进行修复 。
  aof默认配置的文件大小为64M,在生产环境中最少应该配置3-5G，在达到这个阈值的时候，会启动优化算法，减少文件体积，在下次达到10G的时候会再优化一次  
  
  若是想使用redis持久化，建议两个都开启
</ul>
<h1>3.redis事务</h1>
redis只支持部分事务。<br/>
步骤：1.通过multi进入事务，再输入需要执行的命令，将命令全部加入到队列，然后通过exec批处理，统一执行。<br/>
例如在通过Multi进入事务模式时，将操作命令加入队列时，命令出错，那么整个事务全部不执行。<br/>
但是要是命令无错误，再加入队列时命令无错误，那么在通过exec执行命令时，正确的命令都会执行，而不会整个事务都不执行。<br/>
watch可watch多个key，例如:watch key1 key2 key3。但是一执行unwatch，那么会直接取消对所有key的监视<br/>
注意：redis在使用事务时，一定要通过watch去监听观察这个key，在通过multi进入事务，如果你watch的这个key被修改过了，那么你的事务将会失败，重新执行unwatch命令，再重新watch，
然后再开启事务再去提交，如果又被修改过，一直重复以上步骤，直到成功。
<h1>4.redis主从复制</h1>

从机通过slaveof ip port进行对主机的绑定(如果主机设置了密码，那么从机里面的配置文件master_auth属性也要设置一样的密码，不然会一直连接不上)<br/>
注意：<br/>
<li>1.一旦建立主从复制关系，那么从机会拿到主机所有数据。<br/></li> 
<li>2.daemonize属性要设置为yes<br/><br/></li>
<li>3.从机只能读不能写，除非你配置了从机配置文件的slave-read-only属性为no<br/></li>
<li>4.从机开启了slave-read-only属性为no，可以修改本机的值，修改的值不会同步到其他主机和从机，当主机又修改了该key，那么该从机会自动同步主机的值<br/>
例如：本来k1=v1,从机set k1 v2, 那么该从机get k1=v2 ,但其他主机和从机get k1=v1 当主机set k1=233.那么所有的机器get k1=233
</li>
<li>5.从机下面还可以挂其他从机，例如三台机器6378是6380和6381的主机，因为就一个主库，复制压力大，那么可以将6381设为6380的从机，但6380的身份照样是从机，但是可以分担主机的复制压力，可以一个一个的按照这种方式往下传，6381也可以发展其他从机。<br/></li>
<h2>4.1redis主从复制主机掉线手动设置新主机版</h2>
几种情况：<br/>
1.主机断线，从机不会去抢夺master的职位，在主机回来后，继续保持主从关系<br/>
2.主机断线，从机通过slaveof no one命令成为新的主机。<br/>
3.从机断线，没配置配置文件的replicaof属性去指定Ip和端口，再次连接，需要重新通过命令和主机绑定，
才能将数据同步过来(如果你修改了配置文件中的replicaof属性（例如replicaof 127.0.0.1 6379）那么连接回来之后，会自动连上(例 127.0.0.1 6379)主机)<br/>
<h2>4.2.redis主从复制哨兵模式</h2>
1.新建sentinel.conf文件，名字不能错<br/>
2.修改配置文件内容为:<br/>
sentinel monitor “name”  ip port num<br/>
<ul>
<li>name为被监控的主机的名称，可自己随意取<br/>  </li>
<li>ip为主机ip<br/>  </li>
<li>port为主机端口<br/>  </li>
<li>num为票数，剩余从机票数大于num那么成为新主机<br/>  </li>
</ul>
3.启动哨兵，输入命令:redis-sentinel sentinel.conf,会看到哨兵巡逻日志。<br/> 
哨兵模式:当主机掉线时，会通过剩余从机进行投票选出新的主机，无人值守，实现高可用，不用手动切换主机。<br/> 
若以前的主机重新连接，被哨兵监控到之后，那么它会自动变成新的主机的从机。<br/>
例如：79之前是80,81的主机，后面79掉线，哨兵选出80为新主机，那么79重新连接之后，在被哨兵监控到之后，会自动变成80的从机。<br/> 
即:之前79管理80,81， <br/> 79掉线后变成80管理81,<br/> 79重新连上之后变成80管理79,81。
<h2>4.3.redis Cluster</h2>
<h3>为什么要用redis Cluster，因为它易扩容，当数据量太多时，可使用redis Cluster，会将数据分片存储，增加节点也很简单，方便扩容<br/> 
使用redis-trib.rb创建集群。使用create命令 --replicas 1 参数表示为每个主节点创建一个从节点，其他参数是实例的地址集合。</h3>
<h4>
例：./redis-trib.rb create --replicas 1 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7000<br/> 
不出意外会打印如下信息：<br/> 
>>> Creating cluster<br/> 
>>> Performing hash slots allocation on 6 nodes...<br/> 
Using 3 masters:<br/> 
127.0.0.1:7001<br/> 
127.0.0.1:7002<br/> 
127.0.0.1:7003<br/> 
Adding replica 127.0.0.1:7004 to 127.0.0.1:7001<br/> 
Adding replica 127.0.0.1:7005 to 127.0.0.1:7002<br/> 
Adding replica 127.0.0.1:7000 to 127.0.0.1:7003<br/> 
</h4>
Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接。<br/> 
1、所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽。<br/> 
2、节点的fail是通过集群中超过半数的节点检测失效时才生效。<br/> 
3、客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可。<br/> 
4、redis-cluster把所有的物理节点映射到[0-16383]slot上（不一定是平均分配）,cluster 负责维护node<->slot<->value。<br/> 
5、Redis集群预分好16384个桶，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中。<br/> 
redis cluster 为了保证数据的高可用性，加入了主从模式，一个主节点对应一个或多个从节点，主节点提供数据存取，从节点则是从主节点拉取数据备份，<br/> 
当这个主节点挂掉后，就会有这个从节点选取一个来充当主节点，从而保证集群不会挂掉


<h1>5.redis实现分布式锁</h1>
首先，为了确保分布式锁可用，我们至少要确保锁的实现同时满足以下四个条件：<br/>

互斥性。在任意时刻，只有一个客户端能持有锁。<br/>
不会发生死锁。即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁。<br/>
具有容错性。只要大部分的Redis节点正常运行，客户端就可以加锁和解锁。旧主机上线后会自己成为主机，但是没有从机<br/>
解铃还须系铃人。加锁和解锁必须是同一个客户端，客户端自己不能把别人加的锁给解了。<br/>
1.通过setnx命令，如果设置成功则设置过期时间，避免锁长时间不释放<br/>
2.也可以通过后面redis升级之后的版本使用set命令 set(key, value, "NX", "PX", time);<br/>
set(String key, String value, String nxxx, String expx, long time)接口详解<br/>
String set(String key, String value, String nxxx, String expx, long time);<br/>
该方法是： 存储数据到缓存中，并制定过期时间和当Key存在时是否覆盖。<br/>

nxxx： 只能取NX或者XX，如果取NX，则只有当key不存在是才进行set，如果取XX，则只有当key已经存在时才进行set<br/>
expx： 只能取EX或者PX，代表数据过期时间的单位，EX代表秒，PX代表毫秒。<br/>
time： 过期时间，单位是expx所代表的单位。<br/>
SET mykey "1" "NX" "PX" 100  <br/>
设置mykey并保持0.1秒。这期间无法再设置mykey（再次 SET mykey 会返回false）。

<h1>6.布隆过滤器</h1>
原理：位数组加一个或多个hash函数实现<br/>
布隆过滤器存在假阳性，即不能百分之百判断是否存在，但是如果元素实际存在，那么一定会判断存在。<br/>
但是布隆过滤器可以百分百判断元素不存在。<br/>
Google的Guava和redis中都有布隆过滤器的实现类。
