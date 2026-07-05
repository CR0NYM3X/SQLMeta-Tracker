**De: Sofía (Directora de Producto y Calidad)**

Recibido. Freno en seco la generación de estructuras DDL y subo el nivel de abstracción de inmediato. Tienes toda la razón: antes de tirar una sola línea de código o definir columnas, tenemos que trazar el mapa de la ciudad.

He reunido a **Marcos (Arquitectura)** y **Héctor (Almacenamiento)**. Vamos a definir la **Estrategia Lógica de Almacenamiento**, dividiendo la herramienta en capas arquitectónicas, especificando exactamente qué tecnología de guardado usará cada pieza y, lo más importante, **por qué**.

Esta es la arquitectura lógica de alto nivel (El *Blueprint*):

---

### 🧠 CAPA 1: EL PLANO DE CONTROL (Configuración y Catálogos)

* **Ubicación:** Esquema `sqlmeta_core`.
* **Tecnología de Almacenamiento:** **Tablas Relacionales Estándar (B-Tree).**
* **Lógica y Justificación:** Esta capa es el cerebro de la herramienta. Aquí vivirán las configuraciones (`statement_timeout`, `dbms_supported`), el diccionario de consultas SQL que el orquestador va a ejecutar, y el mapeo de servidores.
* **Por qué NO particionar ni usar TimescaleDB aquí:** El volumen de datos es minúsculo (cientos o miles de filas a lo mucho), pero la frecuencia de lectura por parte del orquestador es altísima. Meter TimescaleDB aquí solo agregaría latencia innecesaria. Es almacenamiento tradicional, puro y rápido.

### 📡 CAPA 2: EL PLANO DE DESCUBRIMIENTO (Alcance y Pruebas TCP/SQL)

* **Ubicación:** Esquema `sqlmeta_core`.
* **Tecnología de Almacenamiento:** **Tablas Relacionales Estándar con UPSERT (Inyección con actualización).**
* **Lógica y Justificación:** Aquí guardaremos el resultado del "Paso 1" que mencionaste en tu flujo: el test de sockets TCP y la validación de versiones/bases de datos.
* **Estrategia:** No nos interesa guardar el histórico infinito de que "ayer el servidor sí tenía conexión". Nos interesa la **foto actual**. Si el servidor responde hoy, se actualiza la fecha de "última conexión exitosa". Si falla, se marca como "inactivo" y se anota el error. Esto mantiene la tabla pequeña, ultrarrápida y lista para que los 40 hilos del orquestador sepan a quién atacar en la Fase 2.

### 📝 CAPA 3: EL PLANO DE AUDITORÍA (Bitácoras del Orquestador)

* **Ubicación:** Esquema `sqlmeta_core`.
* **Tecnología de Almacenamiento:** **Particionamiento Nativo de PostgreSQL (Declarative Partitioning) por MES.**
* **Lógica y Justificación:** Cada vez que el orquestador lanza un hilo, ejecuta una consulta, inserta filas y termina, dejará un rastro aquí. Con 500 servidores, bases de datos múltiples y decenas de consultas, esta tabla crecerá rápido.
* **Estrategia:** No usamos TimescaleDB para no hacer que el "cerebro" dependa de extensiones externas, pero sí aplicamos particionamiento nativo por mes. De este modo, cuando el cliente quiera limpiar los logs del orquestador de hace un año, simplemente hace un `DROP TABLE log_mes_enero` y el sistema operativo libera el espacio sin fragmentar los discos.

### 🗄️ CAPA 4: EL PLANO DE DATOS O "MEMORIA TÉCNICA" (Los Resultados)

* **Ubicación:** Esquemas `pgsql` (para PostgreSQL) y `mssql` (para SQL Server).
* **Lógica de Inserción:** 100% *Append-Only* (Solo Inserción Histórica). Nunca se hace un `UPDATE` ni un `DELETE` sobre la información que traen los servidores destino.
* **Estrategia de Almacenamiento (Estratificación/Tiering):** Aquí dividimos la logística en dos motores distintos según el peso, tal como lo definimos antes:
* **TIER 1 (Carga Ligera - Ej. Versiones, Extensiones, Configuraciones del S.O.):** Se guardan en **Tablas Relacionales Estándar**. Como el volumen es bajo, el motor tradicional de PostgreSQL lo manejará sin problemas, acumulando el historial con su fecha de captura.
* **TIER 2 (Carga Pesada - Ej. Permisos, Estadísticas de Tablas de 40M de filas):** Se guardan obligatoriamente como **Hypertables de TimescaleDB**.
* *Lógica:* Se configuran *chunks* (particiones automáticas) de 1 semana.
* *Ventaja Táctica:* El orquestador inyecta los 30 millones de filas el domingo. Caen en la partición "Semana 42". Al terminar, TimescaleDB comprime esa partición en segundo plano para ahorrar disco. Las consultas analíticas de la memoria técnica volarán porque el motor solo abrirá la partición de la semana que el usuario quiera consultar.





---

### 🔄 RESUMEN DE LA ARQUITECTURA LÓGICA

1. El orquestador lee **(Capa 1 - Estándar)** qué tiene que hacer y cómo.
2. Verifica en **(Capa 2 - Estándar)** a qué servidores tiene alcance hoy.
3. Despacha los hilos y escribe todo lo que sucede en **(Capa 3 - Particionada Nativamente)**.
4. Los hilos extraen la data por bloques (`fetch`) y la inyectan en **(Capa 4)**, donde el motor decide, según la tabla destino, si es un almacenamiento ligero (Estándar) o si es artillería pesada que debe ser empaquetada y comprimida (TimescaleDB).

Esta separación lógica garantiza que si la "Memoria Técnica" (Capa 4) crece a Terabytes de información, el Cerebro de la herramienta (Capas 1, 2 y 3) seguirá operando a milisegundos de latencia, porque el motor de almacenamiento subyacente es diferente.

¿Qué te parece este mapeo lógico de la arquitectura? Si estamos de acuerdo con esta división de responsabilidades del almacenamiento, **¿te gustaría que pasemos a diseñar la lógica de cómo el orquestador (el código en GitHub que me pasaste) va a enlazar la Capa 1 con la Capa 4 de manera dinámica y sin saturar los recursos de red?**
