## 需求说明

表中有一个 `status` 字段表示状态，存储的码值含义如下：
- 0 - 初始化
- 1 - 待提交
- 2 - 待授权
- 3 - 已授权
- 4 - 已过期

表中还有一个 `update_time` 字段表示更新时间。

**排序需求：**
1. 已过期（status=4）的记录放在最后，并按更新时间倒序排列
2. 其他状态（status=0,1,2,3）按更新时间倒序排列，同一更新时间内按状态正序排列

## 建表语句

```sql
CREATE TABLE `business_records` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `business_no` varchar(100) DEFAULT NULL COMMENT '业务编号',
  `status` tinyint(4) NOT NULL COMMENT '状态：0-初始化，1-待提交，2-待授权，3-已授权，4-已过期',
  `update_time` datetime NOT NULL COMMENT '更新时间',
  `remark` varchar(255) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_status_update_time` (`status`, `update_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='业务记录表';
```

## 测试数据

```sql
INSERT INTO `business_records` (`business_no`, `status`, `update_time`, `remark`) VALUES
('BIZ001', 0, '2024-12-25 10:00:00', '初始化记录1'),
('BIZ002', 1, '2024-12-25 09:30:00', '待提交记录1'),
('BIZ003', 2, '2024-12-25 09:30:00', '待授权记录1'),
('BIZ004', 3, '2024-12-25 09:00:00', '已授权记录1'),
('BIZ005', 4, '2024-12-25 08:30:00', '已过期记录1'),
('BIZ006', 0, '2024-12-25 08:00:00', '初始化记录2'),
('BIZ007', 1, '2024-12-25 09:30:00', '待提交记录2'),
('BIZ008', 2, '2024-12-25 07:30:00', '待授权记录2'),
('BIZ009', 3, '2024-12-25 10:00:00', '已授权记录2'),
('BIZ010', 4, '2024-12-25 10:30:00', '已过期记录2'),
('BIZ011', 1, '2024-12-25 10:00:00', '待提交记录3'),
('BIZ012', 4, '2024-12-25 06:00:00', '已过期记录3');
```

## 实现SQL

### 方案一：使用 CASE WHEN 排序

```sql
SELECT 
    id,
    business_no,
    status,
    CASE status
        WHEN 0 THEN '初始化'
        WHEN 1 THEN '待提交'
        WHEN 2 THEN '待授权'
        WHEN 3 THEN '已授权'
        WHEN 4 THEN '已过期'
    END AS status_name,
    update_time,
    remark
FROM 
    business_records
ORDER BY 
    CASE WHEN status = 4 THEN 1 ELSE 0 END ASC,  -- 已过期放最后
    update_time DESC,                             -- 更新时间倒序
    status ASC;                                   -- 状态正序
```

### 方案二：使用 IF 函数排序

```sql
SELECT 
    id,
    business_no,
    status,
    CASE status
        WHEN 0 THEN '初始化'
        WHEN 1 THEN '待提交'
        WHEN 2 THEN '待授权'
        WHEN 3 THEN '已授权'
        WHEN 4 THEN '已过期'
    END AS status_name,
    update_time,
    remark
FROM 
    business_records
ORDER BY 
    IF(status = 4, 1, 0) ASC,  -- 已过期放最后
    update_time DESC,           -- 更新时间倒序
    status ASC;                 -- 状态正序
```

## 排序逻辑说明

1. **第一排序字段**：`CASE WHEN status = 4 THEN 1 ELSE 0 END ASC`
   - 将 status=4（已过期）的记录标记为 1，其他记录标记为 0
   - 升序排列，使得 0 在前，1 在后，即已过期记录排在最后

2. **第二排序字段**：`update_time DESC`
   - 所有记录按更新时间倒序排列（最新的在前）

3. **第三排序字段**：`status ASC`
   - 在相同更新时间的情况下，按状态码正序排列（0→1→2→3）
   - 已过期（status=4）的记录已经在第一排序字段中被放到最后，不会与其他状态混在一起

## 预期结果顺序

根据测试数据，排序后的结果应该是：

1. BIZ010 (status=4, 2024-12-25 10:30:00) - 已过期，最新
2. BIZ005 (status=4, 2024-12-25 08:30:00) - 已过期
3. BIZ012 (status=4, 2024-12-25 06:00:00) - 已过期，最旧
4. BIZ001 (status=0, 2024-12-25 10:00:00) - 非过期，最新时间，状态最小
5. BIZ009 (status=3, 2024-12-25 10:00:00) - 非过期，最新时间
6. BIZ011 (status=1, 2024-12-25 10:00:00) - 非过期，最新时间
7. BIZ002 (status=1, 2024-12-25 09:30:00) - 非过期
8. BIZ003 (status=2, 2024-12-25 09:30:00) - 非过期
9. BIZ007 (status=1, 2024-12-25 09:30:00) - 非过期
10. BIZ004 (status=3, 2024-12-25 09:00:00) - 非过期
11. BIZ006 (status=0, 2024-12-25 08:00:00) - 非过期
12. BIZ008 (status=2, 2024-12-25 07:30:00) - 非过期
