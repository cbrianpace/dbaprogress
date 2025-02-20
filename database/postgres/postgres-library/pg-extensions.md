# Extensions

## Installed

```sql
select * from pg_catalog.pg_extension;
```

## Upgrade Installed Extension

```sql
do
$$
declare
  l_sql text;
  l_rec record;
beginfor l_rec in select extname from pg_extension loop
    l_sql := format('alter extension %I update', l_rec.extname);
    execute l_sql;
  end loop;
end;
$$
;
```
