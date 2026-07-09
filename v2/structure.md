 
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


-- Registering the authorized cryptographic functions and the physical key path
INSERT INTO fdw_conf.ctl_settings (setting_name, setting_value, data_type, description) 
VALUES 
('crypto_encrypt_function', 'fdw_conf.fn_encrypt_credentials', 'text', 'Authorized function for encrypting remote passwords'),
('crypto_decrypt_function', 'fdw_conf.fn_decrypt_credentials', 'text', 'Authorized function for decrypting remote passwords'),
('crypto_key_path', '/etc/postgresql/sqlmeta_vault.key', 'text', 'Physical OS path to the master symmetric key');


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


-- 8. Audit Log (Updated with fetch telemetry)
CREATE TABLE fdw_conf.log_exec_query (
    id_log BIGSERIAL PRIMARY KEY,
    id_server INT NOT NULL REFERENCES fdw_conf.cat_server(id_server),
    id_query INT NOT NULL REFERENCES fdw_conf.ctl_queries(id_query),
    target_db VARCHAR(100) NOT NULL,
    status fdw_conf.execution_status NOT NULL,        -- 'SUCCESSFUL', 'FAILED', etc.
    rows_affected INT DEFAULT 0,
    fetch_cycles INT DEFAULT 0,                       -- NEW: Tracks pagination cycles
    error_message TEXT,
    start_time TIMESTAMPTZ DEFAULT clock_timestamp(),
    date_insert TIMESTAMPTZ DEFAULT clock_timestamp()
);
 
```

---

  
### 🔐 1. CONFIGURACIÓN CRIPTOGRÁFICA (Aislamiento Forense)

Registramos las funciones y la **ruta física del archivo** en el catálogo de configuración. Solo el usuario del sistema operativo de PostgreSQL (usualmente `postgres`) tendrá permisos de lectura sobre este archivo en Linux.

```sql

```

---

 
### 🔐 1. FUNCIONES CRIPTOGRÁFICAS (Asymmetric PGCrypto)

```sql
-- ==============================================================================
-- FUNCTION: fdw_conf.fn_encrypt_credentials
-- DESCRIPTION: Encrypts credentials using an OS-level GPG Public Key.
-- SECURITY: Reads file via pg_read_file(). Never stores keys in DB tables.
-- ==============================================================================
CREATE OR REPLACE FUNCTION fdw_conf.fn_encrypt_credentials(p_plain_text TEXT) 
RETURNS BYTEA 
LANGUAGE plpgsql 
SECURITY DEFINER
SET client_min_messages = 'notice'
SET search_path TO fdw_conf, public, pg_temp
AS $$
DECLARE
    v_pub_key_path TEXT;
    v_encrypted_data BYTEA;
BEGIN
    -- Fetch the OS file path from configuration (e.g., '/etc/postgresql/keys/public.key')
    SELECT setting_value INTO v_pub_key_path 
    FROM fdw_conf.ctl_settings 
    WHERE setting_name = 'crypto_pub_key_path';

    IF v_pub_key_path IS NULL THEN
        RAISE EXCEPTION 'SECURITY BREACH: Public key path not defined in ctl_settings.';
    END IF;

    -- Encrypt using the OS-level public key[cite: 37]
    SELECT pgp_pub_encrypt(p_plain_text, dearmor(pg_read_file(v_pub_key_path)), 'cipher-algo=aes256') 
    INTO v_encrypted_data;
    
    RETURN v_encrypted_data;
END;
$$;

-- Perimeter Defense: Strictly revoke public execution
REVOKE EXECUTE ON FUNCTION fdw_conf.fn_encrypt_credentials(TEXT) FROM public;


-- ==============================================================================
-- FUNCTION: fdw_conf.fn_decrypt_credentials
-- DESCRIPTION: Decrypts credentials using an OS-level GPG Private Key & Passphrase.
-- SECURITY: Function owned by postgres. Safe from catalog leaks.
-- ==============================================================================
CREATE OR REPLACE FUNCTION fdw_conf.fn_decrypt_credentials(p_encrypted_data BYTEA) 
RETURNS TEXT 
LANGUAGE plpgsql 
SECURITY DEFINER
SET client_min_messages = 'notice'
SET search_path TO fdw_conf, public, pg_temp
AS $$
DECLARE
    v_priv_key_path TEXT;
    v_passphrase TEXT;
    v_decrypted_data TEXT;
BEGIN
    -- Fetch OS file path and passphrase from secure config
    SELECT setting_value INTO v_priv_key_path FROM fdw_conf.ctl_settings WHERE setting_name = 'crypto_priv_key_path';
    SELECT setting_value INTO v_passphrase FROM fdw_conf.ctl_settings WHERE setting_name = 'crypto_priv_passphrase';

    IF v_priv_key_path IS NULL THEN
        RAISE EXCEPTION 'SECURITY BREACH: Private key path not defined.';
    END IF;

    -- Decrypt using OS-level private key and passphrase[cite: 37]
    SELECT pgp_pub_decrypt(p_encrypted_data, dearmor(pg_read_file(v_priv_key_path)), v_passphrase, 'cipher-algo=aes256') 
    INTO v_decrypted_data;
    
    RETURN v_decrypted_data;
END;
$$;

-- Perimeter Defense: Strictly revoke public execution
REVOKE EXECUTE ON FUNCTION fdw_conf.fn_decrypt_credentials(BYTEA) FROM public;

```

---

### ⚙️ 2. MOTOR DE EXTRACCIÓN (Paginación con Cursores y Auditoría Total)

 
```sql
-- ==============================================================================
-- FUNCTION: fdw_conf.pgsql_exec_query
-- DESCRIPTION: Core extraction engine for PostgreSQL nodes. Uses dblink cursors 
-- (FETCH) to paginate datasets, preventing OOM events, and tracks fetch cycles.
-- SECURITY: Uses OS-level asymmetric decryption and enforces strict search_path.
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
    -- Connection credentials
    v_ip INET;
    v_port INT;
    v_user VARCHAR;
    v_pass_enc BYTEA;
    v_pass_plain TEXT;
    
    -- Query metadata
    v_source_query TEXT;
    v_columns_type TEXT;
    v_target_schema VARCHAR;
    v_target_table VARCHAR;
    v_target_columns TEXT;
    v_statement_timeout VARCHAR;
    
    -- Execution control
    v_conn_name VARCHAR := 'conn_' || md5(random()::TEXT || clock_timestamp()::TEXT);
    v_cursor_name VARCHAR := 'crs_extract';
    v_conn_string TEXT;
    v_fetch_sql TEXT;
    
    -- Telemetry & Pagination
    v_fetch_size INT := 5000; -- RAM protection threshold
    v_batch_rows INT := 0;
    v_total_rows INT := 0;
    v_fetch_cycles INT := 0;  -- Cycle tracker
    v_start_time TIMESTAMPTZ := clock_timestamp();
    v_conn_opened BOOLEAN := FALSE;
BEGIN
    -- [1] Fetch server metadata and encrypted credentials
    SELECT s.ip_address, s.port, u.username, u.password_enc 
    INTO v_ip, v_port, v_user, v_pass_enc
    FROM fdw_conf.cat_server s
    JOIN fdw_conf.ctl_remote_users u ON s.id_user = u.id_user
    WHERE s.id_server = p_id_server;

    -- [2] Fetch query definition and target table structure
    SELECT q.source_query, t.target_schema, t.target_table, t.target_columns, t.target_columns_type
    INTO v_source_query, v_target_schema, v_target_table, v_target_columns, v_columns_type
    FROM fdw_conf.ctl_queries q
    JOIN fdw_conf.ctl_table_conf t ON q.id_query = t.id_query
    WHERE q.id_query = p_id_query;

    -- Fetch global timeout from configuration
    SELECT COALESCE(setting_value, '15min') INTO v_statement_timeout 
    FROM fdw_conf.ctl_settings WHERE setting_name = 'statement_timeout';

    -- [3] Dynamic decryption via OS-level private key
    v_pass_plain := fdw_conf.fn_decrypt_credentials(v_pass_enc);

    -- Build Connection String with network timeout
    v_conn_string := format('host=%s port=%s dbname=%s user=%s password=%s connect_timeout=5', 
                            v_ip, v_port, p_target_db, v_user, v_pass_plain);
    
    -- Memory Hygiene: Erase plaintext password immediately
    v_pass_plain := NULL;

    -- [4] Establish secure DBLINK connection
    PERFORM dblink_connect(v_conn_name, v_conn_string);
    v_conn_opened := TRUE;

    -- Configure remote session parameters
    PERFORM dblink_exec(v_conn_name, format('SET statement_timeout = %L; SET log_min_messages = ''panic'';', v_statement_timeout));

    -- [5] Execute Query and Open Remote Cursor
    PERFORM dblink_open(v_conn_name, v_cursor_name, v_source_query);

    -- [6] Fetch Loop (Pagination)
    LOOP
        -- Build dynamic insert utilizing dblink_fetch
        v_fetch_sql := format(
            'INSERT INTO %I.%I (%s) SELECT * FROM dblink_fetch(%L, %L, %s) AS data(%s)',
            v_target_schema, v_target_table, v_target_columns, v_conn_name, v_cursor_name, v_fetch_size, v_columns_type
        );
        
        EXECUTE v_fetch_sql;
        GET DIAGNOSTICS v_batch_rows = ROW_COUNT;
        
        v_total_rows := v_total_rows + v_batch_rows;
        
        -- Break if the block is empty
        EXIT WHEN v_batch_rows = 0;
        
        -- Increment cycle counter after a successful fetch block
        v_fetch_cycles := v_fetch_cycles + 1;
    END LOOP;

    -- [7] Graceful Teardown
    PERFORM dblink_close(v_conn_name, v_cursor_name);
    PERFORM dblink_disconnect(v_conn_name);
    v_conn_opened := FALSE;

    -- [8] Register Telemetry (Successful execution)
    INSERT INTO fdw_conf.log_exec_query (id_server, id_query, target_db, status, rows_affected, fetch_cycles, start_time)
    VALUES (p_id_server, p_id_query, p_target_db, 'SUCCESSFUL', v_total_rows, v_fetch_cycles, v_start_time);
    
    RETURN QUERY SELECT TRUE, format('Extraction completed. Rows: %s, Cycles: %s', v_total_rows, v_fetch_cycles);

EXCEPTION WHEN OTHERS THEN
    -- [9] Emergency Cleanup & Forensic Logging
    IF v_conn_opened THEN
        BEGIN
            PERFORM dblink_disconnect(v_conn_name);
        EXCEPTION WHEN OTHERS THEN NULL; -- Ignore disconnection errors during panic
        END;
    END IF;

    -- Log the exact failure, keeping the state of v_total_rows and v_fetch_cycles at the moment of the crash
    INSERT INTO fdw_conf.log_exec_query (id_server, id_query, target_db, status, rows_affected, fetch_cycles, error_message, start_time)
    VALUES (p_id_server, p_id_query, p_target_db, 'FAILED', v_total_rows, v_fetch_cycles, SQLERRM, v_start_time);
    
    RETURN QUERY SELECT FALSE, format('Execution failed at cycle %s: %s', v_fetch_cycles, SQLERRM);
END;
$$;

-- Perimeter Defense: Strictly revoke public execution
REVOKE EXECUTE ON FUNCTION fdw_conf.pgsql_exec_query(INT, INT, VARCHAR) FROM public;
```
 
