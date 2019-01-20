### SQLite数据库操作

### SQLite数据库

```java
SQLiteQueryBuilder queryBuilder = new SQLiteQueryBuilder();
queryBuilder.setProjectionMap(mConfigMap);//用户自定义列表与数据库列表的映射
```

Sqlite3数据库操作命令：

- 打开数据库：`sqlite3 test.db`

- 查看数据表：`.tables`

- 查看某个表总共有多少条目数：`select count(*) from events;` 注意要以分号结尾，其中 *events* 为数据表的名字。

  只查看前10行：`select * from events limit 10;`

  只查看后10行：`select * from events order by rowid desc limit 10 offset 0;`

  > offset表示偏移量，表示从第几行开始的10行

- 插入一条数据：`insert into events values(1234,'EVENT','Param');` 其中 *events* 为数据表的名字

  在sqlite命令行中批量插入数据，可以先将SQL语句写到一个文本文件中：test.sql，然后在命令行中执行：

  `.read test.sql`，完成批量SQL语句的执行

  只插入其中的某一列：`INSERT INTO COMPANY (ID) VALUES (1);`

- 删除数据库中某个表的最后几行：

  `delete from events where id in (select id from events order by id desc limit 0,10);  `

  其中*events* 为数据表的名字，根据id这一列，按照降序排列：`order by id desc` ，删除10行。

  > 这里要注意，如果你的数据库中，没有id这一列，可以用**rowid** 替换掉上述语句的**id**，**rowid** 是数据库自动生成的。

- 修改表结构，在已有的数据表中，新增一列url：（不支持批量增加列）

  `alter table events add column url text;`

- 退出sqlite命令行模式：`.quit` 

> SQL语句是不区分大小写的

#### 通过bat脚本实现批量插入多条数据

**sqlite.bat** 脚本，循环1万次执行**insertdb.bat** 脚本，向数据库**statistics.db**插入一万条数据：

```bash
@ECHO OFF 
For /L %%i in (1,1,10000) do (sqlite3.exe statistics.db<insertdb.bat) 
pause
```

**insertdb.bat** 内容如下：

```bash
insert into events values(1234,'EVENT','Param');
```

其中，确保：**sqlite.bat** 、**insertdb.bat**、**statistics.db** 放到同一目录下面、sqlite3.exe已加入到环境变量中。





