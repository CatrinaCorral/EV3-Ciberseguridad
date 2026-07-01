\## Documentación de revisiones de seguridad



Durante el desarrollo del proyecto se realizaron revisiones de seguridad continuas mediante Jenkins, integrando herramientas automatizadas dentro del pipeline CI/CD. Las revisiones fueron ejecutadas sobre el código vulnerable proporcionado y sobre las dependencias del proyecto.



\### Revisiones realizadas



1\. \*\*Análisis estático de código con Bandit\*\*

&#x20;  - Se ejecutó Bandit dentro de la etapa `Security Scan` del pipeline.

&#x20;  - La herramienta revisó el archivo `vulnerable\_app.py`.

&#x20;  - Se identificaron vulnerabilidades asociadas a construcción insegura de consultas SQL y configuración insegura de Flask.



2\. \*\*Análisis de dependencias con Safety\*\*

&#x20;  - Se ejecutó Safety dentro de la etapa `Security Scan`.

&#x20;  - La herramienta revisó las dependencias declaradas en `requirements.txt`.

&#x20;  - Se identificaron vulnerabilidades asociadas a la versión de Flask utilizada inicialmente.



\### Vulnerabilidades identificadas



\- \*\*SQL Injection\*\*

&#x20; - Herramienta: Bandit.

&#x20; - Identificador: B608 / CWE-89.

&#x20; - Archivo afectado: `vulnerable\_app.py`.

&#x20; - Problema: la consulta SQL era construida mediante `f-string`, mezclando datos ingresados por el usuario con la estructura de la consulta.



\- \*\*Ejecución de Flask con modo debug activado\*\*

&#x20; - Herramienta: Bandit.

&#x20; - Identificador: B201 / CWE-94.

&#x20; - Archivo afectado: `vulnerable\_app.py`.

&#x20; - Problema: la aplicación se ejecutaba con `debug=True`, lo que representa un riesgo en un entorno de producción.



\- \*\*Dependencia vulnerable\*\*

&#x20; - Herramienta: Safety.

&#x20; - Archivo afectado: `requirements.txt`.

&#x20; - Problema: se detectaron vulnerabilidades asociadas a `Flask==2.3.0`, incluyendo CVE-2023-30861 y CVE-2026-27205.



\### Correcciones aplicadas



\- \*\*Mitigación de SQL Injection\*\*

&#x20; - Se eliminó la consulta SQL construida con `f-string`.

&#x20; - Se implementó una consulta parametrizada utilizando `?`.

&#x20; - Esto permite separar los datos ingresados por el usuario de la estructura de la consulta SQL.



\- \*\*Mitigación de configuración insegura de Flask\*\*

&#x20; - Se reemplazó `debug=True` por `debug=False`.

&#x20; - Se ajustó la ejecución de la aplicación para evitar configuraciones inseguras detectadas por Bandit.



\- \*\*Mitigación de dependencia vulnerable\*\*

&#x20; - Se actualizó Flask desde la versión `2.3.0` a `3.1.3`.

&#x20; - Esta actualización permitió corregir las vulnerabilidades detectadas por Safety.



\### Validación posterior



Después de aplicar las correcciones, se ejecutó nuevamente el pipeline en Jenkins. El análisis de Bandit no identificó nuevos problemas en el código fuente y Safety reportó cero vulnerabilidades conocidas en las dependencias del proyecto. Esto evidencia que las vulnerabilidades detectadas anteriormente fueron mitigadas correctamente dentro del flujo CI/CD.

