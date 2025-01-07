

### Install extensions: 
```sql
select extname,extversion from pg_extension ;
+--------------------+------------+
|      extname       | extversion |
+--------------------+------------+
| plpgsql            | 1.0        |
| tds_fdw            | 2.0.3      |
| dblink             | 1.2        |
| pgcrypto           | 1.3        |
| pg_stat_statements | 1.10       |
| pg_trgm            | 1.6        |
| pg_cron            | 1.6        |
+--------------------+------------+
```


Estructura Tablas con date_insert,  last_update:
- ctl_users 
- ctl_server
- ctl_querys
- cat_rdbms
- ctl_project_settings
- log_msg_error
- log_exec_querys

  
- funcion de decript , encript: automatico al insertar o hacer update 
