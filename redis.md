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
<li>3.从机只能读不能写<br/></li>
几种情况：<br/>
1.主机断线，从机不会去抢夺master的职位，在主机回来后，继续保持主从关系<br/>
2.从机断线，没配置配置文件的replicaof，再次连接，需要重新通过命令和主机绑定，才能将数据同步过来(如果你修改了配置文件中的replicaof属性那么连接回来之后，会自动连上主机)<br/>
<h1>5.redis实现分布式锁</h1>
1.通过setnx命令，如果设置成功则设置过期时间，避免锁长时间不释放
2.也可以通过后面redis升级之后的版本使用set命令 set(key, value, "NX", "PX", time);
SET mykey "1" EX 60 NX  
# 设置mykey并保持60秒。这期间无法再设置mykey（再次 SET mykey 会返回false）。
