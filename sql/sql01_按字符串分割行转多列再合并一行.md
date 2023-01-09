# sql01_按字符串分割行转多列再合并一行

## 测试数据

```sql
DROP TABLE IF EXISTS `test`;
CREATE TABLE IF NOT EXISTS `test` (
`id` bigint(20) NOT NULL AUTO_INCREMENT ,
`name` varchar(255) DEFAULT NULL,
`prod_code` varchar(255) DEFAULT NULL,
PRIMARY KEY (`id`)
);
INSERT INTO `test`(`name`,`prod_code`) VALUES ('A:2%;B:1%;C:3%', 'D0001');
```

那么,`select * from test;`

![](https://fastly.jsdelivr.net/gh/huabin123/my-picture@main/img/16732519861331673251985167.png)

希望把数据转化成下面这样：

## 方法一：依赖mysql.help_topic表实现

### 第一步：按照分号转多行

![](https://fastly.jsdelivr.net/gh/huabin123/my-picture@main/img/16732345073681673234507321.png)

```sql
SELECT
    SUBSTRING_INDEX( SUBSTRING_INDEX( a.`name`, ';', b.help_topic_id + 1 ), ';',-1 ) name
FROM
    test a
    JOIN mysql.help_topic b ON b.help_topic_id < ( LENGTH( a.`name`) - LENGTH( REPLACE ( a.`name`, ';', '' ) ) + 1 );

```

### 第二步：按冒号分成两个字段

![](https://fastly.jsdelivr.net/gh/huabin123/my-picture@main/img/16732369829231673236982442.png)

```sql
SELECT
    SUBSTRING_INDEX(SUBSTRING_INDEX(SUBSTRING_INDEX( a.`name`, ';', b.help_topic_id + 1 ), ';',-1), ':', 1) as first_name,
SUBSTRING_INDEX(SUBSTRING_INDEX(SUBSTRING_INDEX( a.`name`, ';', b.help_topic_id + 1 ), ';',-1), ':', -1) as second_name
FROM
    test a
    JOIN mysql.help_topic b ON b.help_topic_id < ( LENGTH( a.`name`) - LENGTH( REPLACE ( a.`name`, ';', '' ) ) + 1 );
```

### 第三步：重新拼接字符串

![](https://fastly.jsdelivr.net/gh/huabin123/my-picture@main/img/16732416014711673241601434.png)

```sql
SELECT
	CONCAT( '产品', c.first_name, '的利润为：', c.second_name) AS NAME 
FROM
	(
	SELECT
		SUBSTRING_INDEX( SUBSTRING_INDEX( SUBSTRING_INDEX( a.`name`, ';', b.help_topic_id + 1 ), ';',- 1 ), ':', 1 ) AS first_name,
	SUBSTRING_INDEX( SUBSTRING_INDEX( SUBSTRING_INDEX( a.`name`, ';', b.help_topic_id + 1 ), ';',- 1 ), ':', - 1 ) AS second_name 
	FROM
	test a
	JOIN mysql.help_topic b ON b.help_topic_id < ( LENGTH( a.`name` ) - LENGTH( REPLACE ( a.`name`, ';', '' ) ) + 1 )) AS c;
```

### 第四步：多行转一行

```sql
SELECT e.full_name
FROM
(SELECT
	d.prod_code,
	group_concat( d.NAME SEPARATOR '\\n' ) AS full_name 
FROM
	(
	SELECT
		CONCAT( '产品', c.first_name, '的利润为：', c.second_name ) AS NAME,
		prod_code 
	FROM
		(
		SELECT
		-- 如果第一个是%，则设置first_name位空字符串
		case
		  WHEN SUBSTRING_INDEX( SUBSTRING_INDEX( SUBSTRING_INDEX( a.`name`, ';', b.help_topic_id + 1 ), ';',- 1 ), ':', 1 ) like '%\%%' THEN ''
	    ELSE SUBSTRING_INDEX( SUBSTRING_INDEX( SUBSTRING_INDEX( a.`name`, ';', b.help_topic_id + 1 ), ';',- 1 ), ':', 1 )	END	AS first_name,
		SUBSTRING_INDEX( SUBSTRING_INDEX( SUBSTRING_INDEX( a.`name`, ';', b.help_topic_id + 1 ), ';',- 1 ), ':', - 1 ) AS second_name,
		a.`prod_code` 
		FROM
			test a
		JOIN mysql.help_topic b ON b.help_topic_id < ( LENGTH( a.`name` ) - LENGTH( REPLACE ( a.`name`, ';', '' ) ) + 1 )) AS c 
) AS d 
GROUP BY
	d.prod_code) as e;


```


## 方法二：无mysql.help_topic表权限的情况下，使用临时表

实际上就是要一张有id，且从0开始的表，表的长度要大于按字符串分割的长度
