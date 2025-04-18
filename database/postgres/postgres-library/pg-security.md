# PostgreSQL Security

## Queries

```sql
SELECT coalesce(role.rolname, 'database wide') as role, 
       coalesce(db.datname, 'cluster wide') as database, 
       setconfig as what_changed
FROM pg_db_role_setting role_setting
LEFT JOIN pg_roles role ON role.oid = role_setting.setrole
LEFT JOIN pg_database db ON db.oid = role_setting.setdatabase;

SELECT rolname, unnest(rolconfig) AS role_setting 
FROM pg_roles
WHERE rolconfig IS NOT NULL;

SELECT datname, unnest(datconfig) AS database_setting 
FROM pg_database
WHERE datconfig IS NOT NULL;

SELECT d.datname, s.setconfig AS database_setting
FROM pg_db_role_setting s
JOIN pg_database d ON s.setdatabase = d.oid
WHERE s.setrole = 0;
```