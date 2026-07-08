 
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

 
### 🔐 1. MOTOR CRIPTOGRÁFICO (Zero Trust Encryption)

Para no dejar llaves expuestas en el código fuente, estas funciones utilizan el espacio de memoria de la sesión (`current_setting`) para leer una llave maestra. De esta forma, la llave solo vive en la RAM durante la ejecución y nunca se guarda en texto plano.

```sql
-- ==============================================================================
-- FUNCTION: fdw_conf.fn_encrypt_credentials
-- DESCRIPTION: Encrypts sensitive remote passwords using AES-256 via pgcrypto.
-- Requires a master key injected in the session: SET sqlmeta.master_key = '...';
-- ==============================================================================
CREATE OR REPLACE FUNCTION fdw_conf.fn_encrypt_credentials(p_plain_text TEXT) 
RETURNS BYTEA 
LANGUAGE plpgsql 
SECURITY DEFINER
SET client_min_messages = 'notice'
SET search_path TO fdw_conf, public, pg_temp
AS $$
DECLARE
    v_master_key TEXT;
    v_encrypted BYTEA;
BEGIN
    -- Retrieve the master key from secure volatile memory
    v_master_key := current_setting('sqlmeta.master_key', true);
    
    IF v_master_key IS NULL OR v_master_key = '' THEN
        RAISE EXCEPTION 'SECURITY BREACH: Master key not found in session memory. Set sqlmeta.master_key before execution.';
    END IF;

    -- Encrypt using strong AES-256 cipher
    v_encrypted := pgp_sym_encrypt(p_plain_text, v_master_key, 'cipher-algo=aes256');
    
    -- Scrub memory variable
    v_master_key := NULL;
    
    RETURN v_encrypted;
END;
$$;

REVOKE EXECUTE ON FUNCTION fdw_conf.fn_encrypt_credentials(TEXT) FROM public;

-- ==============================================================================
-- FUNCTION: fdw_conf.fn_decrypt_credentials
-- DESCRIPTION: Decrypts AES-256 passwords exclusively for background workers.
-- ==============================================================================
CREATE OR REPLACE FUNCTION fdw_conf.fn_decrypt_credentials(p_encrypted_data BYTEA) 
RETURNS TEXT 
LANGUAGE plpgsql 
SECURITY DEFINER
SET client_min_messages = 'notice'
SET search_path TO fdw_conf, public, pg_temp
AS $$
DECLARE
    v_master_key TEXT;
    v_plain_text TEXT;
BEGIN
    v_master_key := current_setting('sqlmeta.master_key', true);
    
    IF v_master_key IS NULL OR v_master_key = '' THEN
        RAISE EXCEPTION 'SECURITY BREACH: Master key not found in session memory. Set sqlmeta.master_key before execution.';
    END IF;

    -- Decrypt payload
    v_plain_text := pgp_sym_decrypt(p_encrypted_data, v_master_key);
    
    -- Scrub memory variable
    v_master_key := NULL;
    
    RETURN v_plain_text;
END;
$$;

REVOKE EXECUTE ON FUNCTION fdw_conf.fn_decrypt_credentials(BYTEA) FROM public;

```

---

### ⚙️ 2. MOTOR DE EXTRACCIÓN SEGURO POR BLOQUES (`FETCH`)

Hemos tomado tu lógica del `dblink_fetch_better` y la hemos injertado en la función final. Ahora, abre un cursor remoto, extrae los datos en bloques (chunks) parametrizados, los inserta directamente en la tabla destino y documenta meticulosamente el rendimiento y los fallos en `log_exec_query`.

```sql
-- ==============================================================================
-- FUNCTION: fdw_conf.pgsql_exec_query
-- DESCRIPTION: Core extraction engine for PostgreSQL nodes. Connects securely,
-- decrypts credentials dynamically, and streams data via dblink cursors (FETCH) 
-- to prevent memory exhaustion (OOM Killer).
-- ==============================================================================
CREATE OR REPLACE FUNCTION fdw_conf.pgsql_exec_query(
    p_id_server INT,
    p_id_query INT,
    p_target_db VARCHAR
) RETURNS TABLE (status BOOLEAN, message TEXT) 
LANGUAGE plpgsql 
SECURITY DEFINER
SET client_min_messages = 'notice'
SET search_path TO fdw_conf, public, pg_temp
AS $$
DECLARE
    -- Server & Credential variables
    v_ip INET;
    v_port INT;
    v_user VARCHAR;
    v_pass_enc BYTEA;
    v_pass_plain TEXT;
    
    -- Query & Target variables
    v_source_query TEXT;
    v_columns_type TEXT;
    v_target_schema VARCHAR;
    v_target_table VARCHAR;
    v_target_columns TEXT;
    
    -- Execution control variables
    v_conn_name VARCHAR := 'conn_' || md5(random()::TEXT || clock_timestamp()::TEXT);
    v_cursor_name VARCHAR := 'crs_' || md5(random()::TEXT);
    v_conn_string TEXT;
    v_decrypt_func VARCHAR;
    v_dynamic_sql TEXT;
    v_fetch_sql TEXT;
    
    -- Telemetry & Pagination variables
    v_fetch_size INT := 5000; -- Block size to prevent RAM saturation
    v_batch_rows INT := 0;
    v_total_rows INT := 0;
    v_start_time TIMESTAMPTZ := clock_timestamp();
    v_conn_opened BOOLEAN := FALSE;
BEGIN
    -- [1] Fetch server and encrypted credentials
    SELECT s.ip_address, s.port, u.username, u.password_enc 
    INTO v_ip, v_port, v_user, v_pass_enc
    FROM fdw_conf.cat_server s
    JOIN fdw_conf.ctl_remote_users u ON s.id_user = u.id_user
    WHERE s.id_server = p_id_server;

    -- [2] Fetch query definition and target table configuration
    SELECT q.source_query, q.columns_type, t.target_schema, t.target_table, t.target_columns
    INTO v_source_query, v_columns_type, v_target_schema, v_target_table, v_target_columns
    FROM fdw_conf.ctl_queries q
    JOIN fdw_conf.ctl_table_conf t ON q.id_query = t.id_query
    WHERE q.id_query = p_id_query;

    -- [3] Retrieve the authorized cryptographic function from settings
    SELECT setting_value INTO v_decrypt_func
    FROM fdw_conf.ctl_settings
    WHERE setting_name = 'crypto_decrypt_function';

    IF v_decrypt_func IS NULL THEN
        RAISE EXCEPTION 'CRITICAL: Decryption function not defined in ctl_settings.';
    END IF;

    -- [4] Secure decryption in RAM
    v_dynamic_sql := format('SELECT %s($1)', v_decrypt_func);
    EXECUTE v_dynamic_sql INTO v_pass_plain USING v_pass_enc;

    v_conn_string := format('host=%s port=%s dbname=%s user=%s password=%s connect_timeout=5', 
                            v_ip, v_port, p_target_db, v_user, v_pass_plain);
    v_pass_plain := NULL; -- Scrub plaintext password from memory

    -- [5] Establish dedicated dblink connection
    PERFORM dblink_connect(v_conn_name, v_conn_string);
    v_conn_opened := TRUE;

    -- Set session boundaries on the remote server
    PERFORM dblink_exec(v_conn_name, 'SET statement_timeout = ''15min''; SET log_min_messages = ''panic'';');

    -- [6] Cursor Initiation (FETCH streaming)
    PERFORM dblink_open(v_conn_name, v_cursor_name, v_source_query);

    -- [7] Chunked extraction loop
    LOOP
        v_fetch_sql := format(
            'INSERT INTO %I.%I (%s) SELECT * FROM dblink_fetch(%L, %L, %s) AS data(%s)',
            v_target_schema, v_target_table, v_target_columns, v_conn_name, v_cursor_name, v_fetch_size, v_columns_type
        );
        
        EXECUTE v_fetch_sql;
        GET DIAGNOSTICS v_batch_rows = ROW_COUNT;
        
        v_total_rows := v_total_rows + v_batch_rows;
        
        -- Break loop when no more rows are returned
        EXIT WHEN v_batch_rows = 0;
    END LOOP;

    -- [8] Graceful closure
    PERFORM dblink_close(v_conn_name, v_cursor_name);
    PERFORM dblink_disconnect(v_conn_name);
    v_conn_opened := FALSE;

    -- [9] Register success in audit log
    INSERT INTO fdw_conf.log_exec_query (id_server, id_query, target_db, status, rows_affected, start_time, end_time)
    VALUES (p_id_server, p_id_query, p_target_db, 'SUCCESSFUL', v_total_rows, v_start_time, clock_timestamp());
    
    RETURN QUERY SELECT TRUE, format('Extraction completed. Rows fetched: %s', v_total_rows);

EXCEPTION WHEN OTHERS THEN
    -- [10] Emergency cleanup and error logging
    IF v_conn_opened THEN
        BEGIN
            PERFORM dblink_disconnect(v_conn_name);
        EXCEPTION WHEN OTHERS THEN
            -- Ignore disconnection errors during panic phase
            NULL;
        END;
    END IF;

    INSERT INTO fdw_conf.log_exec_query (id_server, id_query, target_db, status, error_message, start_time, end_time)
    VALUES (p_id_server, p_id_query, p_target_db, 'FAILED', SQLERRM, v_start_time, clock_timestamp());
    
    RETURN QUERY SELECT FALSE, SQLERRM::TEXT;
END;
$$;

-- Perimeter Defense
REVOKE EXECUTE ON FUNCTION fdw_conf.pgsql_exec_query(INT, INT, VARCHAR) FROM public;

```

 
