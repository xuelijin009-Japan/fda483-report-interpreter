# 数据表结构建议：`ysc_fda_483_inspection`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INT PK | 自增主键 |
| remote_id | VARCHAR | 数据源记录唯一 ID（去重依据） |
| fei_number | VARCHAR | FEI 机构编号 |
| record_type | VARCHAR | 记录类型 |
| record_date | DATE | 检查记录日期 |
| company_name | VARCHAR | 企业名称（英文源） |
| establishment_type | VARCHAR | 机构类型 |
| province_or_state | VARCHAR | 州/省 |
| country | VARCHAR | 国家 |
| attachment | VARCHAR | 附件文件名 |
| attachment_url | VARCHAR | 附件下载链接 |
| query_enterprise | VARCHAR | 查询时传入的企业名（用于本地筛选标记） |
| interpretation | TEXT | AI 解读结果（Markdown） |
| interpretation_time | DATETIME | 解读完成时间 |
| interpretation_status | TINYINT | 0=未解读 1=已解读成功 2=解读失败 |
| interpretation_error | VARCHAR | 失败原因（限长 480 字） |

## 关键索引

- `query_enterprise`：按查询企业筛选
- `company_name`：LIKE 模糊匹配
- `remote_id`：去重唯一约束

## 同步去重逻辑

```sql
SELECT id FROM ysc_fda_483_inspection WHERE remote_id = ?;
-- 已存在则跳过，避免重复入库
```
