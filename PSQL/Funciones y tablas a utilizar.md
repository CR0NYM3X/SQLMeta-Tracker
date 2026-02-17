
```

-*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-
-*--*--*--*--*--*--*--*--*--*--*--*--*     TESTING           -*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-
-*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-






-*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-
-*--*--*--*--*--*--*--*--*--*--*--*--*     COLLECT DATA        -*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-
-*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-




-*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-
-*--*--*--*--*--*--*--*--*--*--*--*--*     TABLAS        -*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-
-*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*--*-

Esquema: 
	fdw_conf

Tablas:
	
select * from fdw_conf.cat_dbs  limit 10  -- Donde se guarda la informacion de las base de datos
select * from fdw_conf.cat_versions limit 10 -- Donde se guarda la informacion de las versiones

select * from fdw_conf.cat_server limit 10 -- Donde indica que servidor se va escanear si o no
select * from fdw_conf.cat_category_querys limit 10
select * from fdw_conf.cat_blacklist_db limit 10

select * from fdw_conf.cat_lvl_exec_querys limit 10

select * from  fdw_conf.ctl_dbms limit 10-- DBMS
select * from fdw_conf.ctl_ports limit 10 -- Puertos default

select * from fdw_conf.ctl_project_settings limit 10 -- configuraciones que cambian el comportamiento de la herramienta

select * from  fdw_conf.ctl_querys -- donde se guardan 
-------
select * from fdw_conf.cat_version_psql_oficial limit 10
select * from fdw_conf.cat_version_mssql_oficial limit 10


select * from fdw_conf.ctl_remote_users limit 10;
select * from fdw_conf.dataretentionpolicy limit 10;
select * from fdw_conf.details_connection limit 10;
select * from fdw_conf.log_exec_fun limit 10;
select * from fdw_conf.log_exec_query limit 10;
select * from fdw_conf.log_purge_partitions limit 10;
select * from fdw_conf.log_tables_partitioned limit 10;
select * from fdw_conf.log_test_con_dbs limit 10;
select * from fdw_conf.scan_rules_query limit 10;



```

