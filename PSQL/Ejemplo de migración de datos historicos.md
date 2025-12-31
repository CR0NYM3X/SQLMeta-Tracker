 
## 1) ¿Para qué sirve `synchronous_commit`?

`**synchronous_commit**` controla **cuándo** PostgreSQL considera “**confirmada**” una transacción respecto al **WAL** (log de escritura):

*   **ON (por defecto)**: el commit **espera** a que los registros del WAL se **flusheen a disco** antes de responder “OK”. Máxima durabilidad; si el servidor se cae justo después, **no** pierdes commits confirmados.
*   **OFF**: el commit **no espera** el flush; se delega al SO de forma asíncrona. **Sube el throughput** (más commits/segundo) a costa de que, ante caída brusca, **algunos commits “confirmados” podrían perderse**. Útil en **cargas masivas controladas** (ventanas batch) donde puedes **rehacer** la carga si algo falla. [\[postgresql.org\]](https://www.postgresql.org/docs/current/performance-tips.html), [\[dev.to\]](https://dev.to/mateus-rauli/how-fsync-and-synchronouscommit-affect-postgresql-performance-22di)

> En pruebas con `pgbench`, desactivar `synchronous_commit` **mejora significativamente el TPS** y reduce latencia; pero **no** es apropiado para transacciones críticas que no puedas volver a cargar. [\[dev.to\]](https://dev.to/mateus-rauli/how-fsync-and-synchronouscommit-affect-postgresql-performance-22di)

***

## 2) Tu idea de **dos tablas padre**: `psql.tables` (día) y `psql.his_tables` (histórico)

Sí, puedes tener **dos tablas padre particionadas** con **la misma estructura** y **mover millones de filas por día sin INSERT/DELETE** usando **DETACH/ATTACH** de particiones. El flujo correcto sería:

1.  **Ingesta diaria**:
    *   Insertas directamente en **`psql.tables`** (padre particionado por rango de `fecha` a **diario**).
    *   Postgres enruta a la **partición del día** (p.ej. `psql.tables_2025_12_31`). [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html)

2.  **Cierre del día** (mover al histórico):
    *   **`ALTER TABLE psql.tables DETACH PARTITION psql.tables_2025_12_31;`**
        > Tras **DETACH**, esa partición queda como **tabla normal** independiente con todos sus datos (no se copian filas). [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html)
    *   **`ALTER TABLE psql.his_tables ATTACH PARTITION psql.tables_2025_12_31 FOR VALUES FROM ('2025-12-31') TO ('2026-01-01');`**
        > El **ATTACH** convierte esa tabla independiente en **partición hija** del padre histórico **sin reinsertar filas**. Es una operación de **metadatos** muy rápida si preparas bien las **CHECK constraints**. [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html), [\[stackoverflow.com\]](https://stackoverflow.com/questions/75065147/postgres-how-to-add-a-partition-to-a-table-that-has-a-default-partition-witho)

3.  (Opcional) Para evitar bloqueos/lentos escaneos durante **ATTACH**:
    *   **Asegura un `CHECK`** en la tabla/partición que **coincida exactamente** con el rango (FROM…TO). Así **evitas** que Postgres tenga que **escanear** la tabla para validar que todas las filas caen en ese rango **mientras sostiene AccessExclusiveLock**.
    *   Si tu padre tiene **partición `DEFAULT`**, añade un **`CHECK` que la excluya** del rango que vas a adjuntar, o Postgres intentará **escanear** `DEFAULT` para verificar que no contiene filas del nuevo rango. [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html), [\[stackoverflow.com\]](https://stackoverflow.com/questions/75065147/postgres-how-to-add-a-partition-to-a-table-that-has-a-default-partition-witho)

> Este patrón “**detach de A** → **attach a B**” es la forma recomendada de mover grandes volúmenes entre conjuntos lógicos (día vs histórico) **sin** operaciones de datos fila a fila. Es equivalente a un “partition switch” y **evita** `DELETE`/`TRUNCATE` sobre la tabla diaria. [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html)

***

## 3) Ejemplo completo (DDL + operación diaria)

> Supongamos que particionas por `fecha TIMESTAMPTZ` (rango diario).

```sql
-- 1) Padres
CREATE TABLE psql.tables (
  id BIGSERIAL,
  fecha TIMESTAMPTZ NOT NULL,
  fuente TEXT,
  payload JSONB,
  PRIMARY KEY (id, fecha)
) PARTITION BY RANGE (fecha);

CREATE TABLE psql.his_tables (
  id BIGSERIAL,
  fecha TIMESTAMPTZ NOT NULL,
  fuente TEXT,
  payload JSONB,
  PRIMARY KEY (id, fecha)
) PARTITION BY RANGE (fecha);

-- 2) Preparas la partición del día en psql.tables (o la creará pg_partman)
CREATE TABLE psql.tables_20251231
  PARTITION OF psql.tables
  FOR VALUES FROM ('2025-12-31 00:00:00+00') TO ('2026-01-01 00:00:00+00');

-- índices locales (solo lo necesario)
CREATE INDEX ON psql.tables_20251231 (fecha);

-- 3) Carga masiva del día (alto rendimiento)
BEGIN;
SET LOCAL synchronous_commit = OFF;  -- si aceptas el trade-off durante la carga

COPY psql.tables (fecha, fuente, payload)
FROM '/data/ingesta_2025-12-31.csv' WITH (FORMAT csv, HEADER true);

COMMIT;
ANALYZE psql.tables_20251231;  -- actualizar estadísticas
```

**Al cierre del día** (mover al histórico):

```sql
-- 4) DETACH del padre "día"
ALTER TABLE psql.tables DETACH PARTITION psql.tables_20251231;

-- (Opcional) valida/ajusta la CHECK del rango en la tabla ahora independiente
ALTER TABLE psql.tables_20251231
  ADD CONSTRAINT chk_dia_20251231
  CHECK (fecha >= '2025-12-31 00:00:00+00' AND fecha < '2026-01-01 00:00:00+00')
  NOT VALID;     -- si la tabla proviene de la partición, ya cumple; VALID si quieres forzar chequeo

-- 5) ATTACH al padre "histórico"
ALTER TABLE psql.his_tables
  ATTACH PARTITION psql.tables_20251231
  FOR VALUES FROM ('2025-12-31 00:00:00+00') TO ('2026-01-01 00:00:00+00');
```

> Con el `CHECK` alineado al rango y sin conflictos con `DEFAULT`, el **ATTACH** es **rápido** y evita un escaneo completo con bloqueo. [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html), [\[stackoverflow.com\]](https://stackoverflow.com/questions/75065147/postgres-how-to-add-a-partition-to-a-table-that-has-a-default-partition-witho)

***

## 4) Consideraciones clave y “gotchas”

*   **Unicidad/PK**: si quieres `UNIQUE`/`PRIMARY KEY` **global** sin incluir la columna de partición, hay restricciones en tablas particionadas. En general, define la PK como `(id, fecha)` o mecanismos alternos (secuencias por partición, etc.). Revisa tu necesidad real de unicidad **global**. [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html)
*   **Índices**: crea **índices locales** solo donde aporten; cada índice cuesta en carga masiva. Puedes añadirlos a los históricos fuera de la ventana crítica. [\[postgresql.org\]](https://www.postgresql.org/docs/current/performance-tips.html)
*   **FK entre particiones**: las FK a tablas muy grandes y particionadas requieren planificación (a veces se evita, o se usan FK hacia tablas pequeñas “reference”). [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html)
*   **Bloqueos**: `DETACH/ATTACH` toma **locks DDL** (rápidos si no hay escaneos). Programa esta operación en tu ventana nocturna o usa **pg\_partman** para automatizar y minimizar tiempos. [\[github.com\]](https://github.com/pgpartman/pg_partman)
*   **Default partition**: si usas una partición **DEFAULT**, **prepara `CHECK`** para que `ATTACH` **no** escanee DEFAULT (ver nota en docs y discusión técnica). [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html), [\[stackoverflow.com\]](https://stackoverflow.com/questions/75065147/postgres-how-to-add-a-partition-to-a-table-that-has-a-default-partition-witho)

***

## 5) Automatización recomendada

*   **pg\_partman**: crea **particiones futuras**, rota y **borra** según retención con su **BGW**; también soporta “migrar” juegos existentes. [\[github.com\]](https://github.com/pgpartman/pg_partman), [\[pgxn.org\]](https://pgxn.org/dist/pg_partman/doc/pg_partman.html)
*   **pg\_cron**: agenda `run_maintenance_proc()` o tus `DETACH/ATTACH`/`DROP` en la **hora exacta** que cierre el día. [\[docs.aws.amazon.com\]](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL_pg_cron.html)

***

## 6) ¿Responde tu duda?

**Sí**: tu proceso de **detachar** del padre diario y **atachar** al padre histórico **sirve exactamente** para **mover millones de registros rápidamente** **sin** `INSERT`/`DELETE`/`TRUNCATE` en la tabla original. Es **el patrón correcto** para tu volumen nocturno.  
Y si además usas `COPY` y (en la ventana de carga) `synchronous_commit=OFF`, **aceleras mucho** la ingesta del día. [\[postgresql.org\]](https://www.postgresql.org/docs/17//ddl-partitioning.html), [\[cybertec-p...gresql.com\]](https://www.cybertec-postgresql.com/en/bulk-load-performance-in-postgresql/), [\[dev.to\]](https://dev.to/mateus-rauli/how-fsync-and-synchronouscommit-affect-postgresql-performance-22di)



### ¿Para qué sirve `NOT VALID`?

En PostgreSQL, `NOT VALID` es un **modificador de una restricción (CONSTRAINT)** que indica que:

*   **No se verifica inmediatamente** que **las filas ya existentes** cumplan la restricción al momento de crearla.
*   **Sí se aplica a partir de ese momento** para **todas las filas nuevas o actualizadas** (INSERT/UPDATE).
*   Puedes **validarla después** con `ALTER TABLE ... VALIDATE CONSTRAINT ...`, lo que hace un escaneo para confirmar que todas las filas existentes cumplen la condición. Mientras la restricción esté sin validar, el optimizador no la usa para ciertas optimizaciones (por ejemplo, eliminar particiones en queries), y pueden seguir existiendo datos históricos que la incumplan.
