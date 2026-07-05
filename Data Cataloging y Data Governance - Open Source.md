# Gestión y Gobierno de Metadatos
DataHub, OpenMetadata, Amundsen, Apache Atlas


### El Objetivo General que comparten

Si tuviéramos que resumir el propósito de estas cuatro herramientas (DataHub, OpenMetadata, Amundsen, Apache Atlas) en una sola frase, sería: **Terminar con el "caos de los datos" centralizando los metadatos para que la información sea fácil de encontrar, entender y auditar.**

En las empresas, los datos están dispersos en docenas de herramientas (bases de datos, orquestadores como Airflow, tableros como Superset). El objetivo de estos catálogos es responder a tres preguntas fundamentales sin importar dónde vivan los datos:

1. **¿Qué datos tenemos y dónde están?** (Descubrimiento).
2. **¿De dónde salieron y hacia dónde van?** (Linaje/Trazabilidad).
3. **¿Quién es el responsable y son seguros de usar?** (Gobernanza y Calidad).

---

### El Flujo de Trabajo (Cómo funcionan por dentro)

Aunque cada herramienta tiene su propia arquitectura (DataHub usa eventos en tiempo real, Atlas usa Hadoop, etc.), **todas utilizan el mismo flujo de trabajo lógico de 4 pasos**.

Recuerda siempre la regla de oro: **Estas herramientas NO tocan ni mueven tus datos reales** (no ven el saldo de la cuenta bancaria del cliente), **solo extraen los metadatos** (el nombre de la columna, el tipo de dato, cuándo se actualizó).

#### 1. Extracción (Ingesta de Metadatos)

El catálogo funciona como un pulpo con muchos tentáculos. Envía pequeños programas llamados "conectores" o "crawlers" a todas las herramientas de tu empresa.

* **Se conecta a tu base de datos (ej. PostgreSQL):** Extrae los nombres de las tablas y las columnas.
* **Se conecta a tu orquestador (ej. Airflow / tu DAG):** Extrae cuándo se ejecutan las tareas y si fallaron.
* **Se conecta a tu BI (ej. Metabase/Superset):** Extrae los nombres de los tableros gráficos.

#### 2. Estructuración y Conexión (El Cerebro)

Una vez que el catálogo tiene toda esa información suelta, la junta como si fuera un rompecabezas.

* Crea un mapa visual. Se da cuenta de que la Tarea A de Airflow tomó datos de la Tabla B de PostgreSQL para actualizar el Gráfico C en Superset. A esta conexión de los puntos se le llama **Linaje de Datos**.

#### 3. Enriquecimiento (El Toque Humano)

La herramienta por sí sola solo sabe nombres técnicos. Aquí entran los humanos (o la Inteligencia Artificial) a darle contexto de negocio.

* Un usuario entra a la plataforma y agrega una descripción: *"Esta tabla contiene las ventas de México"*.
* El equipo de seguridad pone una etiqueta (Tag): *"¡Cuidado! Esta columna contiene correos electrónicos (Datos Personales Privados)"*.
* Se asigna un dueño (Owner): *"El equipo de Finanzas es el responsable de mantener esta tabla"*.

#### 4. Consumo (Búsqueda y Uso)

Este es el paso final donde los usuarios interactúan con la herramienta en su día a día a través de una interfaz web (la página principal del catálogo).

* **El Analista:** Escribe "ventas" en el buscador, encuentra la tabla, lee que el dueño es Finanzas y que los datos se actualizaron hace 1 hora. Decide que es seguro usarla para su reporte.
* **El Ingeniero:** Ve que un DAG de Airflow falló. Entra al catálogo, revisa el linaje de datos y avisa de inmediato a los de Negocios: *"Oigan, el tablero de Superset va a estar desactualizado hoy porque falló la base de datos origen"*.

En resumen, el flujo es: **Conectarse a todo $\rightarrow$ Absorber los esquemas $\rightarrow$ Dejar que los humanos pongan reglas $\rightarrow$ Servir como el buscador oficial de la empresa.**

---

## 3. Casos de Uso más Frecuentes

En la práctica, implementamos OpenMetadata cuando las organizaciones enfrentan estos escenarios:

1.  **Reducción del "Time-to-Data":** Los analistas pierden el 40% de su tiempo buscando datos. Con el catálogo, los encuentran en segundos.
2.  **Análisis de Impacto:** Antes de modificar una columna en SQL Server, revisas el linaje en OpenMetadata para ver si vas a romper un tablero en Tableau o Power BI.
3.  **Cumplimiento de Auditorías (GDPR/LGPD):** Etiquetar automáticamente columnas que contienen correos o nombres para asegurar que solo los roles autorizados las vean.
4.  **Observabilidad de Datos:** Monitorear si las tablas se actualizaron a tiempo y si pasaron las pruebas de validación.

---



### La diferencia y cualidad principal de cada uno

Para que veas rápido en qué destaca cada herramienta, aquí tienes su "superpoder":

| Herramienta | Origen | Diferenciador Principal (Superpoder) |
| --- | --- | --- |
| **DataHub** | LinkedIn | **Tiempo real y Escalabilidad.** Usa una arquitectura basada en eventos (Kafka). Cuando un dato cambia, DataHub se entera al instante. |
| **OpenMetadata** | Uber / Comunidad | **El Estándar API.** Está construido pensando primero en los desarrolladores. Define un estándar universal de cómo deben estructurarse los metadatos, con una interfaz increíblemente moderna. |
| **Amundsen** | Lyft | **Simplicidad de búsqueda.** Es el más enfocado en el usuario final. Solo busca ser el "Google" de los datos, sin complicarse con reglas de gobernanza extremas. |
| **Apache Atlas** | Ecosistema Hadoop | **Cumplimiento estricto.** Es un tanque blindado. Está hecho para bancos y empresas muy reguladas que necesitan auditorías de seguridad y control de acceso hiper-detallado. |

---

### ¿Cuándo usar uno u otro?

* **Usa DataHub si:** Tienes un equipo de ingeniería de datos avanzado, usas streaming de datos, quieres ver el linaje de datos en tiempo real y tienes muchos sistemas diferentes interactuando.
* **Usa OpenMetadata si:** Quieres empezar rápido, buscas la mejor experiencia de usuario (UI) moderna, y quieres que todo se conecte mediante APIs limpias y estandarizadas.
* **Usa Amundsen si:** Tu único problema es que tus analistas pierden tiempo buscando tablas. Quieres algo ligero y fácil de entender solo para "descubrimiento", sin meterte en gobernanza compleja.
* **Usa Apache Atlas si:** Trabajas en un banco, una aseguradora, o usas infraestructura antigua (Hadoop). Tu prioridad número uno no es la agilidad, sino que no te demanden por perder datos.

---

### ¿Cuál es el más usado y el más querido?

* **El más usado históricamente:** **Apache Atlas** y **Amundsen**. Atlas porque lleva años en los corporativos, y Amundsen porque fue de los primeros en popularizarse para equipos ágiles.
* **El más usado HOY para proyectos nuevos:** **DataHub**. Es el rey actual del mercado empresarial moderno.
* **El más querido por la comunidad (El "Hype"):** **OpenMetadata**. A los desarrolladores les fascina porque su código es muy limpio, su interfaz es hermosa y es facilísimo integrarlo con otras herramientas modernas (como dbt o Airflow). Le está robando mucho terreno a DataHub.

---

### ¿Qué departamento y puestos usan estas herramientas?

El departamento principal que las administra es el **Equipo de Datos (Data Team)** o **Ingeniería IT**, pero las usan varios perfiles para cosas distintas:

* **Ingenieros de Datos (Data Engineers):** Son los mecánicos. Ellos instalan la herramienta, conectan las tuberías (pipelines/DAGs) para que los metadatos fluyan hacia el catálogo y configuran las alertas de errores.
* **Analistas y Científicos de Datos (Data Analysts / Data Scientists):** Son los consumidores. Entran a la herramienta a buscar tablas, leer descripciones y ver de dónde viene la información antes de crear un reporte o un modelo de IA.
* **Data Stewards / Equipo de Gobernanza:** Son los bibliotecarios y auditores. Entran a poner etiquetas como "Dato Confidencial", a documentar qué significa cada columna y a asegurar que las reglas de la empresa se cumplan.

```
https://www.ovaledge.com/blog/openmetadata-vs-datahub
```
