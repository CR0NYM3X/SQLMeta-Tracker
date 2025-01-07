
# PRIVILEGIOS NECESARIOS

# PSQL 
```
******************************************************************
********************** PSQL  ************************************* 
******************************************************************


echo "DO \$\$
DECLARE 

        var_password_encryption varchar;
        var_current_database varchar := lower(current_database());
        
BEGIN

    IF    var_current_database::varchar   = 'postgres'::varchar THEN
        
        ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO systest; --- permission para tablas futuras
        GRANT select on all tables in schema public   to   systest; 
 
		IF  (select 1 from pg_roles where rolname = 'pg_monitor') THEN		
			GRANT pg_monitor to systest;
		END IF;
		IF (select 1 from pg_roles where rolname = 'pg_stat_scan_tables')  THEN 
			GRANT pg_stat_scan_tables to systest;
		END IF;
		IF (select 1 from pg_roles where rolname = 'pg_read_all_stats')  THEN 
			GRANT pg_read_all_stats to systest;
		END IF;
		IF (select 1 from pg_roles where rolname = 'pg_read_all_settings')  THEN 
			GRANT pg_read_all_settings to systest;
		END IF;
		IF (select 1 from pg_roles where rolname = 'pg_read_server_files')  THEN 
			GRANT pg_read_server_files to systest;
		END IF;
		IF (select 1 from pg_roles where rolname = 'pg_execute_server_program')  THEN 
			GRANT pg_EXECUTE_server_program to systest;  --  must be superuser or a member of the pg_EXECUTE_server_program role to COPY to or from an external program
		END IF;
	
	END IF;
	
    IF var_current_database  ilike  '%plicacione%'::varchar THEN
                ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO systest; --- permission para tablas futuras 
				GRANT select on all tables in schema public   to   systest; 
    END IF;

 

    EXECUTE '    GRANT CONNECT ON DATABASE \"' ||  current_database() || '\" TO systest';
        
    GRANT USAGE ON SCHEMA public TO systest;
    GRANT EXECUTE on all functions  in schema pg_catalog   to   systest; ---- permission denied for function pg_stat_file 
    GRANT select on all tables in schema pg_catalog   to   systest; --- permission denied for view pg_hba_file_rules
         
END \$\$;"  > /tmp/script_02.sql 

 
 # Guarda todas las base de datos en la variable result
result=$(psql -p 5432 -tAX -c "select datname  from pg_database where not datname in('template0') /*and not datname = 'tiendasMueblesTX.30085_ant'*/;" )

# Recorre la lista de base de datos 
for base in $result
do
    # Instala la funcion
    echo   $(psql -p 5432 -tAX -f /tmp/script_02.sql -d $base) " - Base de datos: " $base 
done

# Borra el script
rm  /tmp/script_02.sql

# Ver la version 
psql -V


  
************* <= 8 *******
 
 # Guarda todas las base de datos en la variable result
result=$(psql -p 5432 -tAX -c "select datname  from pg_database where not datname in('template1','template0')  ;" )

# Recorre la lista de base de datos 
for base in $result
do
    # Instala la funcion
    psql -d $base -t   -c "select  'grant select on table ' || table_name || ' to ' || CHR(34) || 'systest' || CHR(34) || ';' from information_schema.tables where table_schema in('pg_catalog','information_schema') ;" | psql -d $base 
	
	psql -d $base -t   -c "SELECT 'GRANT EXECUTE ON FUNCTION '|| proname || '(' || pg_catalog.oidvectortypes(proargtypes) || ')' || ' to ' || CHR(34) || 'systest' || CHR(34) || ';' as qweads FROM pg_proc where proname in(SELECT routine_name FROM information_schema.routines WHERE routine_type = 'FUNCTION' AND specific_schema = 'pg_catalog') ;" | psql -d $base 
	
done

```


# MSSQL 
 ```

******************************************************************
********************** MSSQL  ************************************* 
******************************************************************




-------------------- CREA EL USUARIO EN CADA BASE DE DATOS --------------------

execute SYS.sp_MSforeachdb '

use [?];  


IF NOT EXISTS(SELECT 1 from sys.database_principals where name = ''systest'')
BEGIN
	CREATE USER [systest] FOR LOGIN [systest];
END;

if lower(DB_NAME()) like ''%aplicaciones%'' and lower(DB_NAME()) like ''%db%''
begin 
	EXEC sp_addrolemember db_datareader, systest;
end 
 
' 


---------------------- OTORGA LOS PERMISOS IMPORTANTES EN LA MASTER ----------------------


USE [master]
GO

EXEC sp_addrolemember db_datareader, systest;
GO 
 

SELECT 'GRANT ' + permission_name + ' TO [systest];' FROM fn_builtin_permissions(DEFAULT) where permission_name in(
'SHOWPLAN' --- 
,'VIEW ANY ERROR LOG' -- 
,'VIEW ANY DATABASE'
,'VIEW ANY DEFINITION'
,'VIEW ANY PERFORMANCE DEFINITION'
,'VIEW ANY SECURITY DEFINITION'
,'VIEW SERVER SECURITY AUDIT'
,'VIEW SERVER SECURITY STATE'
,'VIEW SERVER PERFORMANCE STATE'
,'VIEW SERVER STATE'

);
 

GO
grant select  ON DATABASE::master TO [systest];  --- este por permisos de  object sysaltfiles para ver los discos 
GO
GRANT execute on [dbo].[sp_help_revlogin]   to [systest]; --- este es para el permiso del proc sp_help_revlogin para ver los login

GO
-------------------------OTORGA LOS PERMISOS IMPORTANTES EN LA MSDB ----------------------


use msdb
go 
grant select  ON DATABASE::msdb TO [systest]; --- este me pide permiso en la msdb
GO
      GRANT SHOWPLAN TO [systest];  --- este para ver que no marque error al ver los jobs y backups 
GO

 

---------------------------------------


--- Ver si tiene permisos 
select * from sys.server_permissions where  grantee_principal_id in(select principal_id from sys.server_principals  where name = systest)
go

--- veri si existe el usuario 
  select * from sys.syslogins where name = systest 
 
```
