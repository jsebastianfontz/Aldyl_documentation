Repositorio de Documentación – Modelos PostgreSQL y Lógica BI

Este repositorio centraliza la documentación técnica y funcional de los distintos proyectos desarrollados para múltiples clientes. Incluye:

Vistas creadas en PostgreSQL

Lógica de análisis y métricas utilizadas en herramientas BI (Metabase, Looker Studio, Power BI)

Transformaciones clave aplicadas en cada proyecto

Dependencias entre vistas, métricas y dashboards

El objetivo es mantener una estructura clara y escalable que permita realizar handovers rápidos y facilitar la continuidad de los proyectos sin requerir conexiones directas a bases de datos o herramientas BI.

📂 Estructura del Repositorio
/README.md                      ← Este archivo (overview general)
/postgres_documentation/        ← Plantillas y guías base
/clients/                       ← Documentación específica por cliente
    /cliente_x/
        documentation.md        ← Documentación de vistas y lógica BI
        /views_sql/             ← SQL originales de las vistas (opcional)

🧩 Contenido de cada sección
1. /postgres_documentation/

Contiene plantillas, estándares y guías reutilizables para:

Documentar vistas PostgreSQL

Documentar métricas BI

Definir estructura de campos

Crear resúmenes funcionales y técnicos

Sirve como base para todos los proyectos y clientes.

2. /clients/

Cada cliente tiene su propia carpeta aislada con:

documentation.md
Documentación funcional y técnica del proyecto:

Resumen de negocio

Descripción de vistas

Lógica BI

Dependencias con dashboards

Transformaciones aplicadas

/views_sql/ (opcional)
Contiene los SQL originales de las vistas documentadas.

🛠️ Cómo agregar documentación de un nuevo cliente

Crear una carpeta dentro de /clients/ con el nombre del cliente.

Copiar la plantilla desde /postgres_documentation/ a documentation.md.

Documentar:

Vistas PostgreSQL

Campos y transformaciones

Métricas BI

Suposiciones y limitaciones

(Opcional) Agregar los SQL de vistas en /views_sql/.

👤 Autor

Juan Fontalvo
Data Engineer & BI Consultant
