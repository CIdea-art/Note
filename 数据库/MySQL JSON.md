## json

5.7+开始支持json格式，但8.0+才有比较多的函数支持和优化

json类型适合

- 无主键的值对象、快照
- 静态数据，从不修改

取值

```sql
select json ->> '$.key1.key2' from tablename;
```

设值

```mysql
update dept set json = JSON_SET(json,'$.key1','新值');
```



虚拟列

虚拟列可正常查询和添加索引

```mysql
ALTER TABLE ${tablename} ADD COLUMN keyname VARCHAR(255) AS (${columnname}->>'$.keyname');
```

## jsonarray

### 索引

```mysql
ALTER TABLE ${tablename}
ADD INDEX idx_jsonarray ((cast((jsonarray->"$") as unsigned array)));
```

### 查询

MEMBER OF

包含成员

```mysql
SELECT * FROM ${tablename}
WHERE ${val} MEMBER OF(${columnname}->"$");
```

JSON_CONTAINS

包含子集

```mysql
SELECT * FROM ${tablename}
WHERE JSON_CONTAINS(${columnname}->"$", '[2,10]');
```

JSON_OVERLAP

```mysql
SELECT * FROM ${tablename}
WHERE JSON_OVERLAPS(${columnname}->"$", '[2,3,10]')
```

> ？？？JSON_CONTAINS和JSON_OVERLAP有什么区别

JSON_REMOVE移除

```mysql
UPDATE 
${tablename}
SET ${columnname} = JSON_REMOVE(${columnname}, JSON_UNQUOTE(JSON_SEARCH(${columnname}, 'one', '${val}')))
-- 包含时才set，否则会set null
WHERE JSON_SEARCH(${columnname}, 'one', '${val}') IS NOT NULL
```

