## Numeric data types support

### Python

```python
get(
    d.data['val'] for d in self.db.Data
    if d.data['val'] < 3.15
)
```

### PostgreSQL

```sql
SELECT DISTINCT ("d"."data"->'val')
FROM "data" "d"
WHERE (("d"."data"->'val'))::text::real < 3.15
LIMIT 2
```

### MySQL

```sql
SELECT DISTINCT json_extract(`d`.`data`, '$."val"')
FROM `data` `d`
WHERE json_extract(`d`.`data`, '$."val"') < 3.15
LIMIT 2
```

### SQLite (with json1)

```sql
SELECT DISTINCT unwrap_extract_json(json_extract("d"."data", null, '$."val"'))
FROM "Data" "d"
WHERE CAST(unwrap_extract_json(json_extract("d"."data", null, '$."val"')) AS real) < 3.15
LIMIT 2
```

### SQLite (without json1)

```sql
SELECT DISTINCT py_json_extract("d"."data", '["val"]', 1)
FROM "Data" "d"
WHERE CAST(py_json_extract("d"."data", '["val"]', 1) AS real) < 3.15
LIMIT 2
```

### Oracle

```sql
SELECT * FROM (
    SELECT DISTINCT REGEXP_REPLACE(JSON_QUERY("d"."DATA", '$."val"' WITH WRAPPER), '(^\[|\]$)', '')
    FROM "DATA" "d"
    WHERE CAST(REGEXP_REPLACE(JSON_QUERY("d"."DATA", '$."val"' WITH WRAPPER), '(^\[|\]$)', '') AS real) < 3.15
) WHERE ROWNUM <= 2
```

## `in` operator support

### Python
```python
select(
    m.info for m in self.M if 'description' in m.info
).first()
```

 ### PostgreSQL
 
```sql
SELECT "m"."info"
FROM "e" "m"
WHERE ("m"."info"#>'{}') ? 'description'
ORDER BY 1
LIMIT 1
```

### MySQL

```sql
SELECT `m`.`info`
FROM `e` `m`
WHERE (json_contains(`m`.`info`, '"description"', '$') OR json_contains_path(`m`.`info`, 'one', '$."description"'))
ORDER BY 1
LIMIT 1
```

### SQLite (with json1)

```sql
SELECT "m"."info"
FROM "E" "m"
WHERE py_json_contains("m"."info", '[]',  'description')
ORDER BY 1
LIMIT 1
```

### SQLite (without json1)

```sql
SELECT "m"."info"
FROM "E" "m"
WHERE py_json_contains("m"."info", '[]',  'description')
ORDER BY 1
LIMIT 1
```

### Oracle

```sql
SELECT * FROM (
    SELECT "m"."INFO"
    FROM "E" "m"
    WHERE (REGEXP_LIKE(JSON_QUERY("m"."INFO", '$'),  '^\[(.+, ?)?"description"(, ?.+)?\]$') OR JSON_EXISTS("m"."INFO", '$."description"'))
) WHERE ROWNUM <= 1
```