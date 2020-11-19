<h1>1.mysql explain介绍</h1>
explain基础字段解析
<ul>
<li>
  <h2>1.1 id: id的值决定表的读取顺序</h2>  
  <h5>1.1.1：</h5>如果id相同，那么sql的执行顺序是按explain执行的结果集由上往下执行的。<br/>
  例如：<br/>
  id  select_type  table<br/>
  1      simple      t1<br/>
  1      simple      t2<br/>
  1      simple      t3<br/>
  那么执行查表的顺序就是t1,t2,t3<br/>
<h5>1.1.2:</h5>
  如果Id不同，id值越大的值执行的优先级越高，越先被执行。
 <h5>1.1.3:</h5>
  如果Id有不相同有相同，那么先走id最大的,其他id相同的，可看成是同一组，由上到下顺序执行。
</li>
  <li>
    <h2>1.2 select_type:数据读取操作的操作类型</h2>   
    <ul>
      <li> simple：简单的查询，查询中不包含子查询和Join</li>
      <li> primary：查询中包含任何一个复杂的子查询，最外层的查询被标记成primary，<br/>即explain select t2.* from t2 where id =(select id from t1),这条sql中,查询t2会被标记为primary</li>
      <li> subquery：在select或where列表中包含了子查询</li>
      <li> derived：在from列表中包含的子查询被标记为derived（衍生），MySQL会递归执行这些子查询，将结果放入临时表中</li>
      <li> union：若第二个select出现在union之后，则被标记为Union，若Union包含在from子句的子查询中，外层selecte被标记为derived</li>
       <li> union result：从Union表中获取结果的select <br/>即select* from t1 a left join t2 b on a.id=b.aid
         union 
         select* from t1 a right join t2 b on a.id=b.aid<br/>
      </li>     
    </ul>
     </li>     
   <li> <h2>1.3 table：这一行数据来自于哪张表</h2>      </li> 
    <li><h2>1.4 type:显示查询使用了何种类型</h2>
    <h5>type的类型从最好到最差的排序是：system>const>eq_ref>ref>fulltext>ref_of_null>index_merge>unique_subquery>index_subquery>range>index>ALL</h5>
      <ul>
        <li>system：表中只有一行记录(相当于是系统表),这是const类型的特例，平时不会出现</li>
        <li>const：表示通过索引一次就找到了，const用于比较primarty key(主键)或者Unique索引,因为只匹配一行数据，所以很快，如果将主键置于where中，MySQL就能将该查询转换成一个常量。</li>
        <li>eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条数据与之匹配，常见于主键或唯一索引扫描。</li>
        <li>ref：非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，它返回的是匹配某个值的行，但是他会找到多个符合条件的行，它属于查找和扫描的混合体。</li>
        <li>range：只检索给定范围的行，使用一个索引来选择行。key列会显示使用了哪个索引，一般就是在where语句中出现了Between,<,>,in等的查询，这种范围扫描索引比全表扫描要好</li>
        <li>index：full index scan,index与all的区别为index类型只遍历索引树，通常会比ALL快，因为索引文件通常比数据文件小，也就是说all和Index虽然都是读全表，但是Index是从索引读，但是all是从硬盘去读。</li>
        <li>ALL：全表扫描 full table scan</li>   
        一般来说,需要将查询优化到range,最好能到ref。
      </ul>
  </li>
   <li> <h2>1.5 possible_keys：显示可能应用在这张表中的索引，一个或多个。查询涉及到的字段中若存在索引，则将索引列出，但查询的时候不一定会用上</h2></li> 
  <li> <h2>1.6 key：实际用到的索引，如果为Null，表示没用到索引（可能没建立索引，或者可能建立了但是该查询导致索引失效了）。查询中若使用了覆盖索引，则该索引仅出现在key列表中。</h2></li>
   <li> <h2>1.7 key_len：表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精度的情况下，长度越短越好。key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，而不是通过表内检索出来的。</h2></li> 
   <li> <h2>1.8 ref：显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或者常量被用于查找索引列上的值。</h2>
  如 select * from t1 where id ='1' 那么ref就是const，表示用到了id索引，并且这个索引的值是一个常量，即const<br/>
     例子2:select * from t1,t2 where t1.clo1=t2.clo1 and t1.clo2='c'，则ref的值为databasename(数据库名称).t2(表名).clo1(字段名),const（常量，对应第二个条件t1.col2='c'，因为指定了值为c则可以将t1.col的值看作为一个常量）。
  </li> 
  <li><h2>1.9 rows 根据表的使用信息和索引的使用情况，大致估算出来找到所需的记录所需要读取多少行。即例如我要查找10条数据，那么我也许需要读取100行数据才能拿到我想要的10行数据。</h2></li>   
  <li> <h2>1.10 Extra 包含不适合在其他列中显示但是十分重要的信息</h2>
    覆盖索引：1.就是select的数据列只需要从索引中就能够获取到，不必读取数据行，MySQL可以利用索引返回select列表中的字段，而不必根据索引再次读取数据文件，换句话说就是查询列要被所建的索引覆盖。<br/>
    理解方式二:索引是高效查找的方法，但是一般数据库也能使用索引找到一个列的数据，因此它不必读取整行数据，毕竟索引叶子节点存储了他们索引的数据，当能通过读取索引就可以得到想要的数据，
    那就不需要读取行了，一个索引包含了或覆盖了满足查询结果的数据就叫做覆盖索引。<br/>
    注意:如果要使用覆盖索引，select列表中尽量只写需要查找的列，尽量避免select *
    <ul>
      <li>Using filesort 说明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取，mysql中无法利用索引完成的排序称为文件排序（该情况很危险,尽量使用索引字段和建立索引的顺序字段进行排序进行优化）。</li>
      <li ><p style="color:red">Using temporary 使用了临时表保存中间结果，Mysql在对查询结果进行排序的时候使用了临时表，常见于order by和分组查询group by（很危险，尽快优化）</p></li>
      <li>Using index 表示相应的select中使用到了覆盖索引，避免了访问表的数据行，效率提升。如果同时出现using where 表明索引被用来执行索引键值的查找（即使用了索引进行查找）。如果没出现using where ，则表示索引用来读取数据而非执行查找动作。</li>
      <li>Using where 表示使用了where进行过滤</li>
      <li>Using join buffer:表示使用了连接缓存，join过多，使用了缓存</li>
      <li>impossible where：where子句的值总是false,不能用来获取元组。例如:where name ="1" and name="2"</li>
      <li>select tables optimized away：在没有groub by 的情况下，基于索引优化min/max操作或者对于myisam存储引擎优化成count(*)操作，不必等到执行阶段再计算，查询执行计划生成的阶段即完成优化。</li>
      <li>distinct：优化distinct操作，在找到第一匹配的元组后即停止查找同样值得动作。</li>
    </ul>
  </li>
</ul>

<h1>2.mysql 索引失效情况</h1>
若建立了复合索引，为name.age,salary三个字段建立了索引，并且索引顺序为name,age,salary<br/>
<h2>2.1 索引的最佳左前缀原则：</h2>
例如:<br/>
1.1 select * from t where name='1' 可用到索引<br/>
1.2 select * from t where name='1' and age='3' 可用到索引<br/>
1.3 select * from t where name='1' and age='3' and salary='4' 可用到索引<br/>
1.4 select * from t where age='3' and salary='4' 不可用到索引<br/>
1.5 select * from t where name='1' and salary='4' 可用到索引,但是只能用到name的索引，用不到salary的索引<br/>
<h5>总结:从最左侧开始，索引的第一个字段不能丢失，不然复合索引会失效，并且索引要连续，中间不能跳过，不然只能用到最左边的第一个索引。</h5>
<h2>2.2 对索引列进行计算等</h2>
 <h5>对建立了索引的列进行操作(函数，计算，类型转换(自动或手动))，会导致索引失效并会让该sql变成全表扫描。</h5>
 <h2>2.3 不能使用索引中范围条件右边的列</h2>
 <h5>索引会变成range,但是会导致第三个字段失效，只会用到前两个索引，而且第二个会变成范围查找</h5>
 <h2>2.4 尽量使用覆盖索引</h2>
 <h5>select后面尽量跟建立了索引的列，少使用select * ,例如上面的可改成select name from t where name='1'</h5>
  <h2>2.5 使用不等于的时候</h2>
  <h5>mysql在使用!=和<>的时候会无法使用索引，导致全表扫描</h5>
  <h2>2.6 使用is null和is not null的时候也无法使用索引</h2>
  <h5>尽量不使用Null值存储，null值用其他固定值替换</h5>
   <h2>2.7 使用like '%a'</h2>
  使用模糊匹配，最好使用右边匹配，不要使用like '%a'，尽量使用like 'a%'。<br/>
  若必须使用左侧%,又不想索引失效,使用复合索引,例如建立name,age索引，但是需要查询的字段要和索引建立的字段有关联或完全吻合，若超出建立索引的字段(见第五条sql)，则也用不到索引<br/>
  select name from t1 where name like '%a%'  可以用到索引<br/>
  select id,name,age from t1 where name like '%a%'  可以用到索引<br/>
  select name,age from t1 where name like '%a%'  可以用到索引<br/>
  select * from t1 where name like '%a%'  不可以用到索引 <br/>
  select id,name,age,email from t1 where name like '%a%'  不可以用到索引<br/>
   <h2>2.8 字符串类型查询不加单引号</h2>
  即你给Name建立了索引，而且你的name类型为varchar,bane你使用select name from t1 where name='a'，可以用到索引<br/>
  但是你是用select name from t1 where name=100，不可以用到索引，因为name本身是varchar的，2000不是varchar类型，但是数据库会在底层隐式的转换你的2000类型，会导致使用不到索引,所以varchar一定要记得带上单引号<br/>
  <h2>2.9 少使用or，用or连接时会使索引失效</h2>
  例如:select nanme from t1 where name='a' or name='b'
<h1>3.sql优化</h1>
  in和exists的写法区别：</br>
  exists里面包括的sql语句只会返回true和false,不会返回具体的结果集</br>
  例如:select * from emp e where e.deptId in (select id from dept)</br>
  select * from emp e where exists(select 1 from dept where dept.id=e.deptId)</br>
  <h3>3.1优化步骤</h3>
  <ul>
    <li>1.开启慢查询日志，并且捕获慢sql</li>
    <li>2.explain+sql进行分析</li>
    <li>3.show profile查询sql在mysql服务器里面的执行细节和生命周期等情况</li>
    <li>4.sql数据库参数的调优</li>
  </ul>
 <h3>3.2优化原则</h3>
   <ul>
    <li>1.小表1驱动大表，即小得数据集驱动大的数据集</li>   
  </ul>
  <h3>3.3order by</h3>
  mysql两种排序方式:文件排序(using filesort)和索引排序(using index)。
  mysql能为排序和查询使用相同的索引。
  使用explain分析时，若出现using filesort则表示order by没用上索引，效率很慢，产生了数据库内排序。</br>
  order by满足两种情况时，会使用到索引，不产生内排序：</br>
  <h4>3.3.1.使用组合索引最前列或单独索引</h4>
  <h4>3.3.2.使用where子句与order by子句条件列满足组合索引的最前列</h4>
  <h4>3.3.3.优化Order by</h4>
  3.3.3.1.尽量使用索引的方式排序，如组合索引c1,c2,那么Order by c1,c2 或者Order by c1是最好的。遵循最佳左前缀原则</br>
  3.3.3.2.如果order by后面的字段没在索引上，那么filesort有两种排序算法，一种双路排序，一种单路排序，4.1之后的版本默认的是单路排序，使用单路排序一定注意要调整Mysql参数，因为单路排序是在sort_buffer中进行的，需要把sort_buffer参数调大一点，还有max_length_for_sort_data的参数也需要调大，不然可能单路排序的效率会更慢。</br>
  <h4>3.3.4提高order by速度</h4>
  3.3.4.1 order by时切忌不使用select * ,若使用select *，，因为*会使用到sort_buffer的容量，那么会影响到排序算法，会使用到双路排序，或者导致多次IO。</br>
  group by和order by优化类似，尽量少用Having就可以，group by是先排序后分组，所以也需要把sort_buffer参数调大一点，还有max_length_for_sort_data的参数也需要调大</br>
  <h1>4.mysql慢查询日志</h1>
  <ul>
    <li>1. mysql默认没开启慢查询，如果不是调优需要，不必开启慢查询日志，会对性能有影响。</li>
    <li> 2.通过show VARIABLES LIKE '%slow_query_log%'命令 查看是否开启，通过set global slow_query_log=1命令开启，重启后会失效，需要的话还需重新通过命令开启。</br>
  需要永久生效，修改my.cnf的配置文件。</li>
    <li>3.通过show VARIABLES LIKE '%long_query_time%'命令查看，通过long_query_time参数控制，超过多少秒的sql会被记录到慢查询日志当中，默认为10，可通过修改配置文件或set global long_query_time=x来设置时间。</br>
    说明：正好等于x不会被记录，记录的只会是大于x的，修改过后，通过show VARIABLES LIKE '%long_query_time%'查询会发现还是10秒，是因为需要重新连接一次或者新开一个会话才会看到改变，或者通过
    show global LIKE '%long_query_time%'查询。
</li>
    <li>4.记录到慢sql后，通过explain进行一步一步的优化和分析。</li>
    <li>5.通过show global status like '%Slow_queries%'查询总共有多少慢sql</li>
  </ul>
 mysql有一个日志分析工具：mysqldumpslow，可通过此工具帮助分析
  <h1>5.mysql show profile</h1>
  可以用来分析当前会话中语句执行的资源消耗情况，用于sql调优。默认关闭状态，并保存最近的15条运行结果。<br/>
  show profile后面可跟参数,cpu（cpu开销相关信息）,block io（io开销相关信息）,content switches（上下文切换开销）,ipc（发送和接受相关开销信息）,memory（内存相关）,page faults（页面错误相关）,source（显示和source_file,source_line,source_function相关的开销信息）,swaps（显示交换次数相关的开销信息）,all（所有）
  <ul>
    <li>1.show variables like 'profiling'查看是否开启，通过set profiling =on;开启</li>
    <li>2.运行sql，执行慢sql,例如:select * from t1</li>
    <li>3.执行show profiles，出现一个列表，三个字段,query_id表示序号，duration表示执行sql所耗费的时间，query表示执行的sql语句。</li>
    <li>4.诊断sql，通过show profile cpu,block io for query [query_id],query_id为show profiles出现列表中对应的那个id,执行完后，会对该sql整个生命周期（从连接到缓存到执行结束）和使用的cpu,io等情况全部展示出来，然后可以看到里面也会有duration字段，会表示你的sql在生命周期的这一步当中使用的时间，从而知道sql问题出在哪，再进行优化和调整。</li>
  <li>5.如果通过第四条执行完成后status字段出现coverting heap to myisam(查询结果太大，内存不够，已开始使用磁盘),create tmp table(创建了临时表),copying to tmp table on disk(吧内存表中的临时表复制到了磁盘),locked表示问题严重，需要及时调优修复。</li>
  </ul>
<h1>6.mysql 锁</h1>
innoDB与myisam的最大不同是：innoDB支持事务，并且支持行级锁。<br/>
<h3>6.1表锁(myisam引擎)</h3>
<h5>6.1.1读锁</h5>
多个session，其中一个session对t1表上读锁（lock table t1 read），其他session包括上锁的session都可以对t1进行读，但是上锁的session不能写t1，也不能去读其他的表，需要释放锁之后才可以，其他session要写t1的话，会进入阻塞，等待t1锁的释放，然后再去执行写t1的命令。<br/>
<h5>6.1.2写锁</h5>
多个session，其中一个session对t1表上读锁（lock table t1 write），上锁的session可以对t1进行读和写，但是不能去读取除了t1之外的表，其他session读和写t1也会进行阻。<br/>
简而言之：读锁不会阻塞读，写锁会阻塞写和读。<br/>
可通过 show open tables查看表是否上锁。<br/>
Myisam不适合做写为主的表，会造成严重堵塞。<br/>
<h3>6.2mysql事务</h3>
innodb并发事务会有以下问题：
1.更新丢失：两个人都在改同一条记录，后面的会将前面的修改给覆盖掉（可以在用户进入修改页面时，对该数据上锁不让其修改可解决此问题）<br/>
2.脏读：事务A读到了事务B已修改但未提交的数据，并且还在这个未提交的数据上进行了修改，此时，若B回滚事务，A读取到的数据无效，不符合一致性。<br/>
3.不可重复：一个事务在读取某些数据后，再次读取那个数据，发现数据已经发生了改变，或者被删掉了，不符合事务的隔离性。<br/>
4.幻读：一个事务按相同的条件去查询，却发现了其他事务插入了满足该条件的数据，不符合事务的隔离性，幻读与脏读有点类似，脏读读到了另一个事务修改了的数据，幻读则是读到了另一个事务新增的数据<br/>
事务隔离级别：mysql默认是可重复读<br/>
1.未提交读（READ UNCOMMITTED）：最低级别，只能保证不读取物理上损坏的数据<br/>
2.已提交读(READ COMMITTED)：语句级别<br/>
3.可重复读(REPEATABLE READ)：事务级别<br/>
4.可序列化(SERIALIZABLE)：最高级别，事务级<br/>
mysql的事务隔离级别越高，并发的副作用越小，但是付出的代价也就越大，大部分应用对于幻读和不可重复读并不是很敏感<br/>
查看当前数据库的事务隔离级别通过“show variables like tx_isolation”<br/>
<h3>6.3行锁(innoDB引擎)</h3>
同时开启多个session,session1修改了id为1的记录的值，在session1未提交事务时，其他session是查不到修改后的信息的，这个就是已提交读，只有在session1执行了commit，提交了事务之后，其他session才能看到session修改后的内容，同时其他session要改id=1的记录的时候，会进入阻塞。session1在修改完成后可以直接查看到修改后的结果，但是在未提交事务前，其他session看不到修改后的结果。<br/>
若where条件后面的字段建立的索引失效，那么会导致行锁变成表锁，切忌用对用好索引，不然会造成很大的性能问题。<br/>
如何锁定一行数据：先begin，开启一个事务，例如：select * from t1 where a=8 for update(其中a=8 for update会对所有a=8的行进行上锁，其他session想要操作a=8的数据，会进入阻塞，直到该session执行完commit)<br/>
可通过 show status like 'innodb_row_lock%'查看行锁情况。 通过查看innodb_row_lock_waits的大小来查看系统启动后行锁等待次数，innodb_row_lock_time_avg查看每次等待时间的平均值，innodb_row_lock_time查看所有的行锁总共等待时长，如果数值过大，则表示行锁等待严重，需要优化。
innodb整体的并发能力和整体性能比myisam高，但是使用不但会造成行锁变表锁的情况。
<h3>6.4间隙锁的危害</h3>
宁可错杀不可放过。<br/>
当我们用范围条件而不是用相等条件检索数据时，对于表中这个范围之内不存在的数据,innodb也会给他加锁，称为间隙锁。<br/>
例如你需要给id从1-5的数据进行修改，但是你的表中只有id为1,2,3,4的数据，并没有id为5的数据，那么在你的执行update t1 set c1="1" where id between 1 and 5。<br/>
然后session2去操作insert into t1(id,name) values ('5','a')，也会进入阻塞，因为5在1-5之中，即使你的表中没有该数据，也会被Innodb上锁，进入阻塞。<br/>
