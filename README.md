
![Logo de GitHub](https://github.com/CR0NYM3X/SQLMeta-Tracker/blob/main/IMG/logo_sqlmeta.jpg)
# SQLMeta-Tracker

In the business environment, where efficiency and precision are key, our SQLMeta-Tracker tool for psql and mssql database servers stands out as an essential solution for the management and maintenance of data infrastructures. Below, we present some of the main advantages of our tool and how it contributes to a more effective business environment:

1. **Centralization of Technical Data:**

   Our tool collects detailed technical information from each database server, centralizing this data in an accessible repository. This facilitates the consultation and analysis of information, allowing IT teams to make quick decisions.

2. **Efficient Monitoring and Maintenance:**

   By providing a unified view of the status of servers and their databases, it significantly improves the ability to monitor and perform preventive maintenance. Administrators can quickly identify potential problems and address maintenance needs before they become critical issues.

3. **Improved Security:**

   The tool allows auditing and reviewing security configurations on each server, ensuring compliance with best practices and regulations. This is crucial for maintaining the integrity and security of business data.

4. **Time and Resource Savings:**

   By automating the collection of technical information and eliminating the need for manual tasks, it frees up time and resources to focus on more strategic and high-value activities.

5. **Advanced Reporting and Analysis:**

   With detailed reporting capabilities, teams can effectively analyze the performance and usage of databases. This facilitates resource optimization and future planning.

6. **Facilitates Technical Documentation:**

   By maintaining an up-to-date technical memory of each server, it simplifies the creation and maintenance of technical documentation. This is especially useful during audits, updates, and personnel transitions.

Our SQLMeta-Tracker tool is a key component for any company looking to improve the efficiency, security, and management of its data infrastructure. By centralizing and automating the collection of technical information, it enables organizations to make more informed decisions, maintain security, and optimize resource usage.

**It is important to note that this tool is designed to centralize technical data and should not be considered a 100% monitoring tool. Its main focus is to provide a detailed inventory and technical profile of each server, allowing for a more organized and efficient management of the database infrastructure.**

**Supported RDBMS:**
- PostgreSQL 100%
- SQL Server 100%
- MySQL 0%
- MongoDB 0%

 
--- 


 
### **Usos principales de la herramienta**

1.  **Memoria técnica / Inventario**  
    En escenarios de contingencia, contar con información completa de la infraestructura de bases de datos es crítico para restaurar rápidamente un entorno. Por ejemplo, al recuperar un backup, es indispensable conocer:
    *   **Sistema Operativo:** tipo, versión, cantidad de CPU, memoria y discos.
    *   **Base de datos:** motor, versión, configuraciones, accesos.  
        Esto permite reconstruir el ambiente tal como estaba antes, evitando inconsistencias o configuraciones nuevas no deseadas.

2.  **Información histórica**  
    Facilita el análisis de tendencias y comportamientos, como:
    *   Crecimiento de datos.
    *   Consumo de disco y memoria.
    *   Creación y modificación de objetos.

3.  **Auditoría**  
    Proporciona evidencia para validar cambios importantes, por ejemplo:
    *   Creación o eliminación de bases de datos.
    *   Alta o baja de usuarios.
    *   Accesos por usuario (ej. mediante `pg_auth_mon`).

4.  **Gestión de incidentes**  
    Permite recuperar información crítica en casos donde se desconoce el estado previo, como:
    *   Restaurar contraseñas modificadas sin exponerlas.
    *   Validar parámetros de configuración en una fecha específica.
    *   Recuperar usuarios eliminados, sus permisos y accesos.

5.  **Reportes de usuarios**  
    Genera informes detallados sobre:
    *   Propietarios de objetos.
    *   Uso y propósito de cada usuario.
    *   Aplicaciones asociadas.

6.  **Reporte de parches**  
    Control y seguimiento de actualizaciones aplicadas al sistema y la base de datos.

7.  **Reporte de mantenimientos**  
    Documentación de tareas programadas y ejecutadas para garantizar la estabilidad del entorno.

8.  **Validación de hardening**  
    Verificación del cumplimiento de políticas de seguridad en servidores y bases de datos.

9.  **Validaciones de reloads**  
    Confirmación de recargas de configuración y su impacto en el sistema.

10. **Programación de ejecuciones**  
    Planificación y control de tareas automatizadas relacionadas con la administración de la base de datos.

 
 
