### 将csv文件的数据导入字典

---

### 本地文件字典使用，需要使用绝对路径，把文件存储在user_files下才能使用字典

```sql
CREATE DICTIONARY default.user_dict
(
    `uid` String,
    `name` String,
    `role` String,
    `department` String,
    `status` String,
    `info` String
)
PRIMARY KEY uid
SOURCE(FILE(PATH '/var/lib/clickhouse/user_files/user_dict.tsv' FORMAT 'TabSeparatedWithNames'))
LIFETIME(MIN 0 MAX 60)
LAYOUT(COMPLEX_KEY_HASHED())
```