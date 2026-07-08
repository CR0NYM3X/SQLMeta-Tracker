 
### 🏛️ 1. CREACIÓN DE ESQUEMAS Y EXTENSIONES

```sql
-- Habilitar encriptación para las contraseñas
CREATE EXTENSION IF NOT EXISTS pgcrypto SCHEMA fdw_conf;

-- Creación de las zonas lógicas
CREATE SCHEMA IF NOT EXISTS fdw_conf; -- El Cerebro (Control y Configuración)
CREATE SCHEMA IF NOT EXISTS pgsql;    -- Bóveda PostgreSQL
CREATE SCHEMA IF NOT EXISTS mssql;    -- Bóveda SQL Server
CREATE SCHEMA IF NOT EXISTS mysql;    -- Bóveda MySQL

```

---

### 🗄️ 2. MODELO ENTIDAD-RELACIÓN: EL CEREBRO (`fdw_conf`)

Ejecuta este script estrictamente en este orden para respetar las llaves foráneas.

```sql

-- 1. TYPE para el estado de ejecución (Basado en tus logs reales)
CREATE TYPE fdw_conf.execution_status AS ENUM (
    'SUCCESS', 
    'FAILED', 
    'RUNNING', 
    'ABORTED', 
    'TIMEOUT'
);

-- 2. TYPE para el destino de almacenamiento
CREATE TYPE fdw_conf.table_storage_type AS ENUM (
    'STANDARD',    -- Tablas normales (B-Tree)
    'PRTTB',       -- Particionamiento nativo de PostgreSQL
    'HYPERTABLE'   -- TimescaleDB
);


-- TYPE para definir el alcance o nivel de ejecución de la consulta
CREATE TYPE fdw_conf.execution_level AS ENUM (
    'SERVER',   -- Se ejecuta una sola vez por instancia (Ej. versiones, hardware)
    'DATABASE'  -- Itera y se ejecuta por cada base de datos encontrada (Ej. permisos, tablas)
);


-- 1. Catálogo de Motores Soportados
CREATE TABLE fdw_conf.ctl_dbms (
    id_dbms SERIAL PRIMARY KEY,
    dbms_name VARCHAR(20) NOT NULL UNIQUE, -- 'PSQL', 'MSSQL', 'MYSQL'
    is_enabled BOOLEAN DEFAULT TRUE,
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp()
);

-- 2. Configuraciones Globales
CREATE TABLE fdw_conf.ctl_settings (
    id_setting SERIAL PRIMARY KEY,
    setting_name VARCHAR(100) NOT NULL UNIQUE,
    setting_value TEXT NOT NULL,
    data_type VARCHAR(20) NOT NULL, -- 'integer', 'boolean', 'text'
    description TEXT,
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp()
);


-- Registering the authorized cryptographic functions in the configuration catalog
INSERT INTO fdw_conf.ctl_settings (setting_name, setting_value, data_type, description) 
VALUES 
('crypto_encrypt_function', 'fdw_conf.fn_encrypt_credentials', 'text', 'Authorized function for encrypting remote passwords'),
('crypto_decrypt_function', 'fdw_conf.fn_decrypt_credentials', 'text', 'Authorized function for decrypting remote passwords');


-- 3. Bóveda de Credenciales Encriptadas (Zero Trust)
CREATE TABLE fdw_conf.ctl_remote_users (
    id_user SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    password BYTEA NOT NULL, -- Se guarda encriptado con pgp_sym_encrypt
    description TEXT,
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp()
);

-- 4. Inventario de Servidores Destino
CREATE TABLE fdw_conf.cat_server (
    id_server SERIAL PRIMARY KEY,
    ip_address INET NOT NULL,
    port INT NOT NULL,
    id_dbms INT NOT NULL REFERENCES fdw_conf.ctl_dbms(id_dbms),
    id_user INT REFERENCES fdw_conf.ctl_remote_users(id_user),
    is_active BOOLEAN DEFAULT TRUE,
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp(),
    UNIQUE(ip_address, port)
);

-- 5. Diccionario Core de Consultas (La Lógica Pura)
CREATE TABLE fdw_conf.ctl_queries (
    id_query SERIAL PRIMARY KEY,
    id_dbms INT NOT NULL REFERENCES fdw_conf.ctl_dbms(id_dbms),
    query_code VARCHAR(100) NOT NULL, -- ej. 'cat_versions'
    source_query TEXT NOT NULL,       -- El SELECT que viaja al servidor remoto
    description TEXT,
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp(),
    UNIQUE(id_dbms, query_code)
);

-- 6. Configuración de Tablas Destino (A dónde va la data)
CREATE TABLE fdw_conf.ctl_table_conf (
    id_table_conf SERIAL PRIMARY KEY,
    id_query INT NOT NULL REFERENCES fdw_conf.ctl_queries(id_query) ON DELETE CASCADE,
    target_schema VARCHAR(50) NOT NULL, -- 'pgsql', 'mssql', etc.
    target_table VARCHAR(100) NOT NULL,
    target_columns TEXT NOT NULL,       -- ej. 'version_name'
    insert_statement TEXT NOT NULL,     -- ej. 'INSERT INTO pgsql.versions...'
    table_type fdw_conf.table_storage_type NOT NULL DEFAULT 'STANDARD', -- 'STANDARD', 'PRTTB', 'HYPERTABLE'
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp()
);


-- Catálogo de Frecuencias de Ejecución
CREATE TABLE fdw_conf.cat_exec_categories (
    id_exec_cat SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE, -- 'daily_task', 'weekly_task', 'other'
    description TEXT,
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp()
);

-- Population of the Execution Frequencies Catalog
INSERT INTO fdw_conf.cat_exec_categories (category_name, description) VALUES
('daily_task',   'Daily execution queries (e.g., quick telemetry and live states)'),
('weekly_task',  'Weekly execution queries (e.g., permissions, bulk configurations)'),
('monthly_task', 'Monthly execution queries (e.g., deep audit, unused objects)'),
('biweekly_task','Biweekly execution queries'),
('other',            'Manual or on-demand unscheduled executions');


-- Catálogo de Clasificación de Consultas
CREATE TABLE fdw_conf.cat_query_categories (
    id_query_cat SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE, -- 'VERSION', 'HARDWARE', 'PERMISSIONS'
    description TEXT,
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp()
);

-- Population of the Query Classification Catalog
INSERT INTO fdw_conf.cat_query_categories (category_name, description) VALUES
('VERSION',         'Data regarding the engine version or installed patches'),
('STORAGE',         'Data regarding disk space, database sizes, or physical files'),
('HARDWARE',        'OS-level information, CPU, RAM, Uptime'),
('DATABASE',        'Native database properties (encoding, collation, owner)'),
('SECURITY',        'User audits, granular permissions, HBA, and encryption'),
('CONFIGURATION',   'Engine parameters (pg_settings, postgresql.conf)'),
('OBJECT_META',     'Metadata for tables, schemas, indexes, views, sequences, and columns'),
('PERFORMANCE',     'Usage statistics, unused indexes, autovacuum, and locks'),
('PROGRAMMABILITY', 'Triggers, Functions (funproc), and installed extensions'),
('JOB_AGENT',       'Scheduled jobs, tasks, and agents (e.g., MSSQL Jobs)'),
('BACKUP',          'Information regarding backup retention and execution'),
('OTHER',           'Fallback classification for miscellaneous or unclassified environment metadata queries');



-- Tu tabla de configuración refactorizada (Limpia y a prueba de balas)
-- Tu tabla de configuración refactorizada con el nuevo TYPE
CREATE TABLE fdw_conf.ctl_query_conf (
    id_query_conf SERIAL ,
    id_query INT NOT NULL REFERENCES fdw_conf.ctl_queries(id_query) ON DELETE CASCADE,
    min_version NUMERIC(6,2) NOT NULL,
    max_version NUMERIC(6,2) NOT NULL,
    
    -- Aplicación del nuevo ENUM (Reemplaza al VARCHAR + CHECK)
    lvl_exec fdw_conf.execution_level NOT NULL,
    
    -- Llaves foráneas a las tablas de catálogo
    id_query_cat INT NOT NULL REFERENCES fdw_conf.cat_query_categories(id_query_cat),
    id_exec_cat INT NOT NULL REFERENCES fdw_conf.cat_exec_categories(id_exec_cat),
    
    is_enabled BOOLEAN DEFAULT TRUE,
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp()
);



-- 8. Bitácora de Auditoría (Reemplaza a log_exec_query)
CREATE TABLE fdw_conf.log_exec_query (
    id_log BIGSERIAL PRIMARY KEY,
    id_server INT NOT NULL REFERENCES fdw_conf.cat_server(id_server),
    id_query INT NOT NULL REFERENCES fdw_conf.ctl_queries(id_query),
    target_db VARCHAR(100) NOT NULL,
    status fdw_conf.execution_status NOT NULL,        -- 'SUCCESS', 'FAILED'
    rows_affected INT DEFAULT 0,
    error_message TEXT,
    start_time TIMESTAMPTZ DEFAULT clock_timestamp(),
    date_insert TIMESTAMPTZ
);






```

---

### ⚙️ 3. LOS MOTORES DE EXTRACCIÓN (Funciones Core)

En tu código anterior, el motor de SQL Server creaba y destruía objetos DDL (`CREATE SERVER`, `DROP FOREIGN TABLE`) en cada ejecución, lo cual infla el catálogo del sistema y genera bloqueos.

Aquí están los esqueletos estructurales (Grado Diamante) para las nuevas funciones. Hemos movido el manejo de errores al estándar y hemos preparado el terreno para usar conexiones persistentes o envoltorios seguros.

#### A. Motor de PostgreSQL (Vía `dblink`)

```sql
CREATE OR REPLACE FUNCTION fdw_conf.pgsql_exec_query(
    p_id_server INT,
    p_id_query INT,
    p_target_db VARCHAR
) RETURNS TABLE (status BOOLEAN, message TEXT) AS $$
DECLARE
    v_ip INET;
    v_port INT;
    v_user VARCHAR;
    v_pass BYTEA;
    v_source_query TEXT;
    v_insert_stmt TEXT;
    v_conn_string TEXT;
    v_start_time TIMESTAMPTZ := clock_timestamp();
    v_rows INT := 0;
BEGIN
    -- 1. Obtener credenciales y servidor
    SELECT s.ip_address, s.port, u.username, u.password 
    INTO v_ip, v_port, v_user, v_pass
    FROM fdw_conf.cat_server s
    JOIN fdw_conf.ctl_remote_users u ON s.id_user = u.id_user
    WHERE s.id_server = p_id_server;

    -- 2. Obtener Query y Destino
    SELECT q.source_query, t.insert_statement 
    INTO v_source_query, v_insert_stmt
    FROM fdw_conf.ctl_queries q
    JOIN fdw_conf.ctl_table_conf t ON q.id_query = t.id_query
    WHERE q.id_query = p_id_query;

    -- 3. Construir conexión DBLINK segura (Desencriptando contraseña al vuelo)
    v_conn_string := format('host=%s port=%s dbname=%s user=%s password=%s', 
                            v_ip, v_port, p_target_db, v_user, pgp_sym_decrypt(v_pass, 'TU_LLAVE_MAESTRA'));

    -- 4. Ejecutar vía dblink (Aquí integraremos tu lógica de fetch_size en la versión final)
    -- EXECUTE v_insert_stmt || ' SELECT * FROM dblink(...)';
    
    -- 5. Registrar Éxito
    INSERT INTO fdw_conf.log_exec_query (id_server, id_query, target_db, status, rows_affected, start_time, end_time)
    VALUES (p_id_server, p_id_query, p_target_db, 'SUCCESS', v_rows, v_start_time, clock_timestamp());
    
    RETURN QUERY SELECT TRUE, 'Query executed successfully'::TEXT;

EXCEPTION WHEN OTHERS THEN
    -- 6. Caja Negra de Errores
    INSERT INTO fdw_conf.log_exec_query (id_server, id_query, target_db, status, error_message, start_time, end_time)
    VALUES (p_id_server, p_id_query, p_target_db, 'FAILED', SQLERRM, v_start_time, clock_timestamp());
    
    RETURN QUERY SELECT FALSE, SQLERRM::TEXT;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

```

#### B. Motor de SQL Server (Vía `tds_fdw`)

```sql
CREATE OR REPLACE FUNCTION fdw_conf.mssql_exec_query(
    p_id_server INT,
    p_id_query INT,
    p_target_db VARCHAR
) RETURNS TABLE (status BOOLEAN, message TEXT) AS $$
DECLARE
    v_ip INET;
    v_port INT;
    v_user VARCHAR;
    v_pass BYTEA;
    v_source_query TEXT;
    v_insert_stmt TEXT;
    v_start_time TIMESTAMPTZ := clock_timestamp();
    v_rows INT := 0;
BEGIN
    -- 1. Obtener datos del servidor (Similar a PGSQL)
    -- [...] Lógica de recolección de variables [...]

    -- 2. Ejecución Grado Diamante sin crear objetos DDL
    -- En lugar de crear un CREATE SERVER dinámico, utilizaremos una función 
    -- que altere un servidor TDS_FDW "plantilla" estático, o utilizaremos 
    -- un wrapper en C si está disponible. Por ahora, ajustamos el mapeo persistente:
    
    -- ALTER SERVER mssql_template OPTIONS (SET servername 'ip', SET port 'port');
    -- ALTER USER MAPPING FOR current_user SERVER mssql_template ...
    
    -- 3. Ejecución y volcado de datos
    -- EXECUTE v_insert_stmt || ' SELECT * FROM foreign_table_template';

    -- 4. Registrar Éxito
    INSERT INTO fdw_conf.log_exec_query (id_server, id_query, target_db, status, rows_affected, start_time, end_time)
    VALUES (p_id_server, p_id_query, p_target_db, 'SUCCESS', v_rows, v_start_time, clock_timestamp());

    RETURN QUERY SELECT TRUE, 'Query executed successfully'::TEXT;

EXCEPTION WHEN OTHERS THEN
    INSERT INTO fdw_conf.log_exec_query (id_server, id_query, target_db, status, error_message, start_time, end_time)
    VALUES (p_id_server, p_id_query, p_target_db, 'FAILED', SQLERRM, v_start_time, clock_timestamp());
    
    RETURN QUERY SELECT FALSE, SQLERRM::TEXT;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

```
 
