<h1>1.mysql explain介绍</h1>
explain基础字段解析
<ul>
<li>
  <h2>1.1 </h2>
  <h3>id: id的值决定表的读取顺序</h3>
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
  <h2>1.2<h2>
    <h3>select_type:数据读取操作的操作类型</h3>
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
    <h2>1.3<h2>
    <h3>table：这一行数据来自于哪张表</h3>
  </li>
</ul>
