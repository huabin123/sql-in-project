### 准备数据

```sql
CREATE TABLE `prices` (
  `pid` int(11) NOT NULL AUTO_INCREMENT,
  `category` varchar(100) NOT NULL,
  `price` float NOT NULL,
  PRIMARY KEY (`pid`)
);

INSERT INTO prices
    (pid, category, price)
VALUES
    (1, 'A', 2),
    (2, 'A', 1),
    (3, 'A', 5),
    (4, 'A', 4),
    (5, 'A', 3),
    (6, 'B', 6),
    (7, 'B', 4),
    (8, 'B', 3),
    (9, 'B', 5),
    (10, 'B', 2),
    (11, 'B', 1)
;
```

### 查询语句

```sql
select 
    category, 
    avg(case when 2*rn in (cnt,cnt+1,cnt+2) then price end ) as median_val
from (
select 
    category,
    price, 
    row_number() over(partition by category order by price) rn,
    count(*) over(partition by category) cnt
  from prices
) as dd
group by category
```
