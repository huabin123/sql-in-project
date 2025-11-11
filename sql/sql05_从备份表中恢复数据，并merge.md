### 需求背景
从备份表sys_auth_role_ref_bak中恢复非管理员用户的部分菜单按钮权限数据到正式表sys_auth_role_ref中，要求仅插入备份表中存在且正式表中不存在
的数据

### sql实现
```sql
insert into sys_auth_role_ref select * from sys_auth_role_ref_bak where auth_id IN ( SELECT auth_id FROM sys_auth WHERE menu_id IN ('BOCHM_TRADE_MONITOR', 'BOCHM_TRADE_STATISTIC) )
AND role_id NOT IN ( "BOCWM_admin', 'BOCWM_sysAdmin' ) AND NOT EXISTS ( SELECT 1 FROM sys_auth_role_ref HHERE sys_auth_role_ref_bak.role_id=sys_auth_role_ref.role_id AND sys_auth_role_ref_bak.auth_id=sys_auth_role_ref.auth_id)
```
