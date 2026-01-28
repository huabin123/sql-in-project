# sql07_递归查询菜单并批量更新权限URL

## 需求说明

有两张表：
- `sys_menu`：菜单表，包含字段 `menu_id`（菜单ID）、`menu_pid`（父菜单ID）
- `sys_auth`：权限表，包含字段 `menu_id`（菜单ID）、`auth_url`（权限URL）

**需求：**
将 `menu_id='BOCWM_PRODUCT_SYSTEM_EVALUATION'` 的菜单及其所有子菜单（递归）关联的 `sys_auth` 表中的 `auth_url` 字段，将其中的 `/bocwm/` 替换为 `/metric/`。

## 测试数据

### 创建菜单表

```sql
DROP TABLE IF EXISTS `sys_menu`;
CREATE TABLE `sys_menu` (
  `menu_id` varchar(100) NOT NULL COMMENT '菜单ID',
  `menu_pid` varchar(100) DEFAULT NULL COMMENT '父菜单ID',
  `menu_name` varchar(255) DEFAULT NULL COMMENT '菜单名称',
  PRIMARY KEY (`menu_id`),
  KEY `idx_menu_pid` (`menu_pid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='菜单表';
```

### 创建权限表

```sql
DROP TABLE IF EXISTS `sys_auth`;
CREATE TABLE `sys_auth` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `menu_id` varchar(100) NOT NULL COMMENT '菜单ID',
  `auth_url` varchar(500) DEFAULT NULL COMMENT '权限URL',
  `auth_name` varchar(255) DEFAULT NULL COMMENT '权限名称',
  PRIMARY KEY (`id`),
  KEY `idx_menu_id` (`menu_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限表';
```

### 插入测试数据

```sql
-- 插入菜单数据（树形结构）
INSERT INTO `sys_menu` (`menu_id`, `menu_pid`, `menu_name`) VALUES
('BOCWM_PRODUCT_SYSTEM_EVALUATION', NULL, '产品系统评估'),
('BOCWM_PSE_OVERVIEW', 'BOCWM_PRODUCT_SYSTEM_EVALUATION', '评估总览'),
('BOCWM_PSE_DETAIL', 'BOCWM_PRODUCT_SYSTEM_EVALUATION', '评估明细'),
('BOCWM_PSE_DETAIL_SUB1', 'BOCWM_PSE_DETAIL', '明细子菜单1'),
('BOCWM_PSE_DETAIL_SUB2', 'BOCWM_PSE_DETAIL', '明细子菜单2'),
('BOCWM_PSE_REPORT', 'BOCWM_PRODUCT_SYSTEM_EVALUATION', '评估报告'),
('OTHER_MENU', NULL, '其他菜单'),
('OTHER_MENU_SUB', 'OTHER_MENU', '其他子菜单');

-- 插入权限数据
INSERT INTO `sys_auth` (`menu_id`, `auth_url`, `auth_name`) VALUES
('BOCWM_PRODUCT_SYSTEM_EVALUATION', '/bocwm/evaluation/query', '查询评估'),
('BOCWM_PRODUCT_SYSTEM_EVALUATION', '/bocwm/evaluation/export', '导出评估'),
('BOCWM_PSE_OVERVIEW', '/bocwm/overview/list', '总览列表'),
('BOCWM_PSE_OVERVIEW', '/bocwm/overview/detail', '总览详情'),
('BOCWM_PSE_DETAIL', '/bocwm/detail/query', '明细查询'),
('BOCWM_PSE_DETAIL_SUB1', '/bocwm/detail/sub1/view', '子菜单1查看'),
('BOCWM_PSE_DETAIL_SUB2', '/bocwm/detail/sub2/edit', '子菜单2编辑'),
('BOCWM_PSE_REPORT', '/bocwm/report/generate', '生成报告'),
('OTHER_MENU', '/bocwm/other/query', '其他查询'),
('OTHER_MENU_SUB', '/bocwm/other/sub/view', '其他子查询');
```

## 解决方案

### 方案一：使用递归CTE（MySQL 8.0+）

#### 第一步：递归查询所有子菜单

```sql
-- 查询指定菜单及其所有子菜单
WITH RECURSIVE menu_tree AS (
    -- 锚点成员：查询根菜单
    SELECT menu_id, menu_pid, menu_name, 1 AS level
    FROM sys_menu
    WHERE menu_id = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    
    UNION ALL
    
    -- 递归成员：查询子菜单
    SELECT m.menu_id, m.menu_pid, m.menu_name, mt.level + 1
    FROM sys_menu m
    INNER JOIN menu_tree mt ON m.menu_pid = mt.menu_id
)
SELECT * FROM menu_tree
ORDER BY level, menu_id;
```

#### 第二步：查询需要更新的权限记录

```sql
-- 查询所有需要更新的权限URL
WITH RECURSIVE menu_tree AS (
    SELECT menu_id, menu_pid, menu_name
    FROM sys_menu
    WHERE menu_id = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    
    UNION ALL
    
    SELECT m.menu_id, m.menu_pid, m.menu_name
    FROM sys_menu m
    INNER JOIN menu_tree mt ON m.menu_pid = mt.menu_id
)
SELECT 
    a.id,
    a.menu_id,
    a.auth_url AS old_url,
    REPLACE(a.auth_url, '/bocwm/', '/metric/') AS new_url,
    a.auth_name
FROM sys_auth a
INNER JOIN menu_tree mt ON a.menu_id = mt.menu_id
WHERE a.auth_url LIKE '%/bocwm/%'
ORDER BY a.menu_id, a.id;
```

#### 第三步：执行批量更新

```sql
-- 批量更新权限URL
UPDATE sys_auth a
INNER JOIN (
    WITH RECURSIVE menu_tree AS (
        SELECT menu_id
        FROM sys_menu
        WHERE menu_id = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        
        UNION ALL
        
        SELECT m.menu_id
        FROM sys_menu m
        INNER JOIN menu_tree mt ON m.menu_pid = mt.menu_id
    )
    SELECT menu_id FROM menu_tree
) mt ON a.menu_id = mt.menu_id
SET a.auth_url = REPLACE(a.auth_url, '/bocwm/', '/metric/')
WHERE a.auth_url LIKE '%/bocwm/%';
```

### 方案二：使用存储过程（MySQL 5.7推荐）

适用于MySQL 5.7，不使用临时表，通过存储过程实现递归查询。

```sql
-- 删除已存在的存储过程
DROP PROCEDURE IF EXISTS update_auth_url_recursive;

DELIMITER $$

CREATE PROCEDURE update_auth_url_recursive(
    IN root_menu_id VARCHAR(100),
    IN old_path VARCHAR(100),
    IN new_path VARCHAR(100)
)
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE current_menu_id VARCHAR(100);
    DECLARE current_level INT DEFAULT 1;
    DECLARE max_level INT DEFAULT 10; -- 最大递归层级，根据实际情况调整
    
    -- 创建会话变量存储所有需要更新的menu_id
    SET @menu_ids = root_menu_id;
    SET @current_level_ids = root_menu_id;
    
    -- 递归查找所有子菜单
    WHILE current_level <= max_level AND @current_level_ids IS NOT NULL AND @current_level_ids != '' DO
        -- 查找下一层的menu_id
        SELECT GROUP_CONCAT(menu_id SEPARATOR ',') INTO @next_level_ids
        FROM sys_menu
        WHERE FIND_IN_SET(menu_pid, @current_level_ids) > 0;
        
        -- 如果找到子菜单，添加到总列表
        IF @next_level_ids IS NOT NULL AND @next_level_ids != '' THEN
            SET @menu_ids = CONCAT(@menu_ids, ',', @next_level_ids);
            SET @current_level_ids = @next_level_ids;
        ELSE
            SET @current_level_ids = NULL;
        END IF;
        
        SET current_level = current_level + 1;
    END WHILE;
    
    -- 执行更新操作
    SET @update_sql = CONCAT(
        'UPDATE sys_auth SET auth_url = REPLACE(auth_url, ''', old_path, ''', ''', new_path, ''') ',
        'WHERE FIND_IN_SET(menu_id, ''', @menu_ids, ''') > 0 ',
        'AND auth_url LIKE ''%', old_path, '%'''
    );
    
    PREPARE stmt FROM @update_sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 返回影响的记录数
    SELECT ROW_COUNT() AS affected_rows, @menu_ids AS updated_menu_ids;
END$$

DELIMITER ;

-- 调用存储过程执行更新
CALL update_auth_url_recursive('BOCWM_PRODUCT_SYSTEM_EVALUATION', '/bocwm/', '/metric/');
```

### 方案三：使用多层自连接（MySQL 5.7，适用于层级较少的情况）

不使用临时表和存储过程，直接通过多层LEFT JOIN实现。适用于菜单层级不超过5层的场景。

```sql
-- 查询所有需要更新的权限记录（最多支持5层菜单）
SELECT 
    a.id,
    a.menu_id,
    a.auth_url AS old_url,
    REPLACE(a.auth_url, '/bocwm/', '/metric/') AS new_url,
    a.auth_name
FROM sys_auth a
WHERE a.auth_url LIKE '%/bocwm/%'
AND a.menu_id IN (
    -- 第1层：根菜单
    SELECT m1.menu_id FROM sys_menu m1 WHERE m1.menu_id = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    UNION
    -- 第2层：子菜单
    SELECT m2.menu_id FROM sys_menu m2 WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    UNION
    -- 第3层：孙菜单
    SELECT m3.menu_id 
    FROM sys_menu m3
    INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
    WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    UNION
    -- 第4层
    SELECT m4.menu_id 
    FROM sys_menu m4
    INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
    INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
    WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    UNION
    -- 第5层
    SELECT m5.menu_id 
    FROM sys_menu m5
    INNER JOIN sys_menu m4 ON m5.menu_pid = m4.menu_id
    INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
    INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
    WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
);

-- 执行批量更新
UPDATE sys_auth a
SET a.auth_url = REPLACE(a.auth_url, '/bocwm/', '/metric/')
WHERE a.auth_url LIKE '%/bocwm/%'
AND a.menu_id IN (
    SELECT menu_id FROM (
        -- 第1层：根菜单
        SELECT m1.menu_id FROM sys_menu m1 WHERE m1.menu_id = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        UNION
        -- 第2层：子菜单
        SELECT m2.menu_id FROM sys_menu m2 WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        UNION
        -- 第3层：孙菜单
        SELECT m3.menu_id 
        FROM sys_menu m3
        INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
        WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        UNION
        -- 第4层
        SELECT m4.menu_id 
        FROM sys_menu m4
        INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
        INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
        WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        UNION
        -- 第5层
        SELECT m5.menu_id 
        FROM sys_menu m5
        INNER JOIN sys_menu m4 ON m5.menu_pid = m4.menu_id
        INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
        INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
        WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    ) AS menu_list
);
```

## 验证结果

### 更新前查询（MySQL 5.7版本）

```sql
-- 查询所有受影响的权限记录
SELECT 
    a.id,
    a.menu_id,
    a.auth_url,
    a.auth_name
FROM sys_auth a
WHERE a.menu_id IN (
    SELECT menu_id FROM (
        -- 第1层：根菜单
        SELECT m1.menu_id FROM sys_menu m1 WHERE m1.menu_id = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        UNION
        -- 第2层：子菜单
        SELECT m2.menu_id FROM sys_menu m2 WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        UNION
        -- 第3层：孙菜单
        SELECT m3.menu_id 
        FROM sys_menu m3
        INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
        WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        UNION
        -- 第4层
        SELECT m4.menu_id 
        FROM sys_menu m4
        INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
        INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
        WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
        UNION
        -- 第5层
        SELECT m5.menu_id 
        FROM sys_menu m5
        INNER JOIN sys_menu m4 ON m5.menu_pid = m4.menu_id
        INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
        INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
        WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    ) AS menu_list
)
ORDER BY a.menu_id, a.id;
```

### 更新后验证

执行更新后，再次运行上述查询，应该看到所有 `/bocwm/` 都已替换为 `/metric/`。

## 注意事项

1. **备份数据**：执行更新操作前，务必备份相关表数据
   ```sql
   CREATE TABLE sys_auth_backup AS SELECT * FROM sys_auth;
   ```

2. **事务控制**：建议在事务中执行更新操作
   ```sql
   START TRANSACTION;
   -- 执行更新SQL
   -- 验证结果
   COMMIT;  -- 或 ROLLBACK; 回滚
   ```

3. **MySQL版本**：
   - MySQL 8.0+ 推荐使用方案一（递归CTE）
   - MySQL 5.7 推荐使用方案二（存储过程）或方案三（多层自连接）
   - 方案二适合层级深度不确定的情况
   - 方案三适合层级深度确定且不超过5层的情况

4. **性能优化**：
   - 确保 `sys_menu.menu_pid` 和 `sys_auth.menu_id` 有索引
   - 对于大数据量，可以分批更新

5. **URL匹配**：
   - 使用 `LIKE '%/bocwm/%'` 确保只更新包含该路径的URL
   - 如果需要精确匹配，可以调整匹配条件

## 扩展应用

### 统计影响范围（MySQL 5.7版本）

```sql
-- 统计受影响的菜单和权限数量
SELECT 
    COUNT(DISTINCT menu_list.menu_id) AS menu_count,
    COUNT(a.id) AS auth_count,
    SUM(CASE WHEN a.auth_url LIKE '%/bocwm/%' THEN 1 ELSE 0 END) AS need_update_count
FROM (
    -- 第1层：根菜单
    SELECT m1.menu_id FROM sys_menu m1 WHERE m1.menu_id = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    UNION
    -- 第2层：子菜单
    SELECT m2.menu_id FROM sys_menu m2 WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    UNION
    -- 第3层：孙菜单
    SELECT m3.menu_id 
    FROM sys_menu m3
    INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
    WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    UNION
    -- 第4层
    SELECT m4.menu_id 
    FROM sys_menu m4
    INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
    INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
    WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
    UNION
    -- 第5层
    SELECT m5.menu_id 
    FROM sys_menu m5
    INNER JOIN sys_menu m4 ON m5.menu_pid = m4.menu_id
    INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
    INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
    WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'
) AS menu_list
LEFT JOIN sys_auth a ON menu_list.menu_id = a.menu_id;
```

### 查看菜单树结构（MySQL 5.7版本）

```sql
-- 查看菜单层级结构
SELECT 
    1 AS level,
    m1.menu_id,
    m1.menu_name,
    m1.menu_name AS menu_path
FROM sys_menu m1
WHERE m1.menu_id = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'

UNION ALL

SELECT 
    2 AS level,
    m2.menu_id,
    m2.menu_name,
    CONCAT('  └─ ', m2.menu_name) AS menu_path
FROM sys_menu m2
WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'

UNION ALL

SELECT 
    3 AS level,
    m3.menu_id,
    m3.menu_name,
    CONCAT('    └─ ', m3.menu_name) AS menu_path
FROM sys_menu m3
INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'

UNION ALL

SELECT 
    4 AS level,
    m4.menu_id,
    m4.menu_name,
    CONCAT('      └─ ', m4.menu_name) AS menu_path
FROM sys_menu m4
INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'

UNION ALL

SELECT 
    5 AS level,
    m5.menu_id,
    m5.menu_name,
    CONCAT('        └─ ', m5.menu_name) AS menu_path
FROM sys_menu m5
INNER JOIN sys_menu m4 ON m5.menu_pid = m4.menu_id
INNER JOIN sys_menu m3 ON m4.menu_pid = m3.menu_id
INNER JOIN sys_menu m2 ON m3.menu_pid = m2.menu_id
WHERE m2.menu_pid = 'BOCWM_PRODUCT_SYSTEM_EVALUATION'

ORDER BY level, menu_id;
```

## 方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 方案一：递归CTE | 代码简洁，性能好，支持任意层级 | 仅MySQL 8.0+支持 | MySQL 8.0及以上版本 |
| 方案二：存储过程 | 支持任意层级，不需要临时表 | 需要创建存储过程，代码较复杂 | MySQL 5.7，层级深度不确定 |
| 方案三：多层自连接 | 不需要存储过程，代码直观 | 需要预知层级深度，层级多时代码冗长 | MySQL 5.7，层级≤5层 |
