<h1>1.mysql explain介绍</h1>
explain基础字段解析
<ul>
<li>1.1 
<br/>id:
<br/>
1.1.1：如果id相同，那么sql的执行顺序是按explain执行的结果集由上往下执行的。<br/>
例如：<br/>
id  select_type  table<br/>
1      simple      t1<br/>
1      simple      t2<br/>
1      simple      t3<br/>
那么执行查表的顺序就是t1,t2,t3<br/>
1.1.2:如果Id不同，id值越大的值执行的优先级越高，越先被执行。
</li>

</ul>
