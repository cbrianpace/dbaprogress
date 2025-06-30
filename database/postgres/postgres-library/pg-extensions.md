# Extensions

## Installed

```sql
SELECT  name,
        default_version,
        installed_version,
        comment
FROM pg_catalog.pg_available_extensions pae
WHERE installed_Version IS NOT NULL
ORDER BY NAME;
```

```sql
SELECT  * 
FROM    pg_catalog.pg_extension;
```

## Upgrade Installed Extension

```sql
DO
$$
DECLARE
  l_sql text;
  l_rec record;
BEGIN
  FOR l_rec IN SELECT extname FROM pg_extension 
  LOOP
    l_sql := format('alter extension %I update', l_rec.extname);
    EXECUTE l_sql;
  END LOOP;
END;
$$;
```
