## 数据准备

产品表建表语句

```sql
CREATE TABLE `comprehensive_info` (
  `prod_code` varchar(100) NOT NULL,
  `prod_name` varchar(255) DEFAULT NULL,
  `data_date` datetime DEFAULT NULL,
  `prod_cls` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`prod_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

指标表建表语句

~~~sql
CREATE TABLE `sys_idx` (
  `prod_code` varchar(100) NOT NULL,
  `wk` decimal(10,8) DEFAULT NULL,
  `year` decimal(10,8) DEFAULT NULL,
  `biz_date` date DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
~~~

产品表数据

```sql
INSERT INTO `sqlinproject`.`comprehensive_info`(`prod_code`, `prod_name`, `data_date`, `prod_cls`) VALUES ('1', '产品1', '2022-08-03 00:00:00', '分类1');
INSERT INTO `sqlinproject`.`comprehensive_info`(`prod_code`, `prod_name`, `data_date`, `prod_cls`) VALUES ('2', '产品2', '2022-08-03 00:00:00', '分类1');
INSERT INTO `sqlinproject`.`comprehensive_info`(`prod_code`, `prod_name`, `data_date`, `prod_cls`) VALUES ('3', '产品3', '2022-08-03 00:00:00', '分类2');
INSERT INTO `sqlinproject`.`comprehensive_info`(`prod_code`, `prod_name`, `data_date`, `prod_cls`) VALUES ('4', '产品4', '2022-08-03 00:00:00', '分类3');
```

指标表数据

```sql
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('1', 0.1, 0.11, '2022-08-01');
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('1', 0.1, 0.11, '2022-08-02');
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('1', 0.1, 0.11, '2022-08-03');

INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('2', 0.2, 0.22, '2022-08-01');
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('2', 0.2, 0.22, '2022-08-02');
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('2', 0.2, 0.22, '2022-08-03');

INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('3', 0.3, 0.33, '2022-08-01');
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('3', 0.3, 0.33, '2022-08-02');
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('3', 0.3, 0.33, '2022-08-03');

INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('4', 0.4, 0.44, '2022-08-01');
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('4', 0.4, 0.44, '2022-08-02');
INSERT INTO `sqlinproject`.`sys_idx`(`prod_code`, `wk`, `year`, `biz_date`) VALUES ('4', 0.4, 0.44, '2022-08-03');
```

## 查询

### 原sql

```sql
SELECT
	t1.prod_code as prod_code,
	t1.prod_name as prod_name,
	i1.wk as wk,
	t1.prod_cls as prod_cls,
	DATE_FORMAT(i1.biz_date,'%Y/%m/%d') as data_date
FROM
	comprehensive_info AS t1
	LEFT JOIN sys_idx AS i1 ON t1.prod_code = i1.prod_code 
	AND i1.biz_date >= '2022-08-01' 
	AND i1.biz_date <= '2022-08-03' 
WHERE
	t1.data_date = '2022-08-03' 
	and t1.prod_cls in ('分类1','分类2','分类3')
	AND i1.wk > 0.1
ORDER BY
	t1.data_date,
	t1.data_date,
	i1.biz_date
```

### 不分组-前（后）几

```sql
select * from
(
SELECT
	t1.prod_code as prod_code,
	t1.prod_name as prod_name,
	i1.wk as wk,
	t1.prod_cls as prod_cls,
	DATE_FORMAT(i1.biz_date,'%Y/%m/%d') as data_date,
	ROW_NUMBER() over (PARTITION by i1.biz_date ORDER BY i1.`year` ) as rk
FROM
	comprehensive_info AS t1
	LEFT JOIN sys_idx AS i1 ON t1.prod_code = i1.prod_code 
	AND i1.biz_date >= '2022-08-01' 
	AND i1.biz_date <= '2022-08-03' 
WHERE
	t1.data_date = '2022-08-03' 
	and t1.prod_cls in ('分类1','分类2','分类3')
	AND i1.wk > 0.1
ORDER BY
	t1.prod_code,
	t1.prod_name,
	i1.biz_date
) tt
WHERE tt.rk<=2
```

### 不分组-前（后）%

```sql
SELECT
	* 
FROM
	(
	SELECT
		t1.prod_code AS prod_code,
		t1.prod_name AS prod_name,
		i1.wk AS wk,
		t1.prod_cls AS prod_cls,
		DATE_FORMAT( i1.biz_date, '%Y/%m/%d' ) AS data_date,
		ROW_NUMBER () OVER ( PARTITION BY i1.biz_date ORDER BY i1.`year` ) AS rk,
		COUNT(*) OVER ( PARTITION BY i1.biz_date ) AS total_count 
	FROM
		comprehensive_info AS t1
		LEFT JOIN sys_idx AS i1 ON t1.prod_code = i1.prod_code 
		AND i1.biz_date >= '2022-08-01' 
		AND i1.biz_date <= '2022-08-03' 
	WHERE
		t1.data_date = '2022-08-03' 
		AND t1.prod_cls IN ( '分类1', '分类2', '分类3' ) 
		AND i1.wk > 0.1 
	ORDER BY
		t1.prod_code,
		t1.prod_name,
		i1.biz_date 
	) tt 
WHERE
	tt.rk <= ROUND(
	tt.total_count * 0.2)
```
