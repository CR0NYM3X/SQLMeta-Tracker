 

# SQLMeta Tracker: Arquitectura de Inteligencia Distribuida (Inspirada en Zabbix Proxy)

Para gestionar 2,400 servidores, **SQLMeta Tracker** utiliza "Proxies Recolectores" que actúan como unidades de inteligencia local. Estos proxies eliminan la carga del servidor central y garantizan que la recolección nunca se detenga, incluso si hay fallos de red.

### 1. El Flujo de Trabajo: Recolección Inteligente y Silenciosa

A diferencia de las herramientas que saturan la red, **SQLMeta Tracker** sigue un ciclo de vida de datos optimizado:

* **Paso 1: Recolección Local (El Proxy):** Cada colector/proxy tiene una pequeña base de datos local (SQLite). El proxy consulta los metadatos (permisos, tablas, hashes de contraseñas) de su grupo de servidores asignados.
* **Paso 2: Almacenamiento Temporal (Buffer):** Los datos no se envían inmediatamente. Se guardan en el proxy local. Si el enlace con el corporativo falla, el proxy sigue recolectando. Nada se pierde.
* **Paso 3: Empaquetado y Envío Programado:** En lugar de miles de conexiones pequeñas, el proxy comprime los metadatos en un solo "paquete maestro" y lo envía al **Servidor Central** en intervalos de bajo tráfico (ej. cada hora o en ventanas nocturnas).
* **Paso 4: Consolidación Central:** El Servidor Central recibe el paquete, lo procesa y actualiza el **Inventario Histórico**.

---

### 2. Tres Modalidades de Conexión (Flexibilidad Total)

Para entrar en cada rincón de la infraestructura de OXXO, el Proxy soporta:
1.  **Agente Local:** Instalado en el SO (Windows/Linux), extrae y "empuja" la info al Proxy.
2.  **Agentless (Pull):** El Proxy se conecta por red a las bases de datos (ideal para PaaS y Cloud).
3.  **API Gateway:** El Proxy consume metadatos desde APIs ya existentes en la infraestructura.

---

### 3. ¿Por qué es la herramienta ideal para Grandes Empresas?

#### A. Seguridad Segmentada (Aislamiento de Riesgos)
El Servidor Central no necesita conectarse a los 2,400 servidores. Solo se comunica con los **10 o 15 Proxies**. Esto permite que cada zona (Tiendas, Corporativo, Logística) tenga su propio colector con sus propias llaves de acceso, cumpliendo con el principio de **Privilegio Mínimo**.

#### B. El "Google Search" de sus Bases de Datos
Toda la información reside en la **Base de Datos Central**. Cuando un directivo o auditor necesita saber:
* *"¿En qué servidores existe la tabla 'Pagos_J'?"*
* *"¿Qué usuarios de SQL Server están huérfanos?"*
* *"¿A quién se le vence el password mañana?"*
La consulta se hace al **Servidor Central**, obteniendo resultados instantáneos sin tocar los servidores de producción.

#### C. Auditoría Forense y Recertificación
**SQLMeta Tracker** guarda el historial (Snapshot). Si un usuario cambió su permiso hace 3 meses, usted puede "viajar en el tiempo" y ver exactamente qué permisos tenía ese día. Además, automatiza la **Recertificación**: el sistema detecta cuentas por expirar y envía correos automáticamente para que el dueño del proceso confirme el acceso.

---

### 4. Resumen de Ventajas Estratégicas

| Característica | Beneficio para OXXO |
| :--- | :--- |
| **Arquitectura tipo Proxy** | Escalabilidad ilimitada sin saturar el canal de datos central. |
| **Autonomía Local** | Si la red falla, el Proxy sigue trabajando y sincroniza después. |
| **Cero Impacto (Passive)** | No usa Triggers ni CDC. Es invisible para la operación diaria. |
| **Visibilidad 360°** | Inventario, seguridad, pesos y cambios en una sola pantalla. |

---

### Conclusión para la Dirección
**SQLMeta Tracker** no es una herramienta de monitoreo reactivo; es una plataforma de **Gobernanza Proactiva**. Le da el control total de sus 2,400 activos de información, asegurando que cada cambio de esquema, cada nuevo usuario y cada riesgo de seguridad esté documentado, auditado y bajo control.
 
---



