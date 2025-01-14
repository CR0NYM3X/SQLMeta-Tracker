

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

# Database :
- centraldata

# Schema : 
- fdw: Objetos importantes de la herramienta
- cdc: controles de cambios 
- prttb: tablas particionadas 
- secdb: proporcionan seguridad en la db.

# Basic connect 
### Structure table  date_insert,  last_update:
- ctl_users 
- ctl_servers
- ctl_querys: 
- ctl_config_querys: enabled_fetch, limit_fetch, lvl_exec_query, db_name_default, parameter_set 
- cat_rdbms
- ctl_settings , puedes configurar a nivel global en caso de que las querys no tengan configurado  se colocan unas por default 
- log_error
- log_exec_querys

### Funcion  
- decript , encript: automatico al insertar o hacer update 
- auto_create_tables_partitioned: crea automaticamente nueva particion del siguiente mes. opcion por parámetro de guardar los dias que no se encontraron particiones. usa tabla: log_tables_partitioned 
- purge_partitions: purga/depura  tablas con un tiempo limite de retencion de info. usa tabla : dataretentionpolicy y log_purge_partitions
- test_telnet

### test connections TELNET and SQL 

- details_connection 
- report_connection
- cat_versions
- cat_dbs
- fn_test_connection

# test connections DB
- fn_test_con_dbs


# Automatic exec 
- ctl_exec_time_querys
- gather_server_data
- fn_exec_query
- fn_add_query



### Mejoras 

Agregar templete_rdbms parametrizadas
