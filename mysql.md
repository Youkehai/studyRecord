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
  <li> <h2>1.6 key：实际用到的索引，如果为Null，表示没用到索引（可能没建立索引，或者可能建立了但是该查询导致索引失效了）。查询中若使用了覆盖索引(复合索引)，则该索引仅出现在key列表中。</h2></li>
   <li> <h2>1.7 key_len：表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精度的情况下，长度越短越好。key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，而不是通过表内检索出来的。</h2></li> 
   <li> <h2>1.8 ref：显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或者常量被用于查找索引列上的值。</h2>
  如 select * from t1 where id ='1' 那么ref就是const，表示用到了id索引，并且这个索引的值是一个常量，即const<br/>
     例子2:select * from t1,t2 where t1.clo1=t2.clo1 and t1.clo2='c'，则ref的值为databasename(数据库名称).t2(表名).clo1(字段名),const（常量，对应第二个条件t1.col2='c'，因为指定了值为c则可以将t1.col的值看作为一个常量）。
  </li> 
  <li><h2>1.9 rows 根据表的使用信息和索引的使用情况，大致估算出来找到所需的记录所需要读取多少行。即例如我要查找10条数据，那么我也许需要读取100行数据才能拿到我想要的10行数据。</h2></li>   
</ul>
