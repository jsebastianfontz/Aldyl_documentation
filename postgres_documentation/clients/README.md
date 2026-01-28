# Documentación por Cliente

Esta carpeta organiza toda la documentación técnica y funcional de cada proyecto, separada por cliente.  
Cada carpeta contiene:

- La documentación completa del proyecto (`documentation.md`)
- Las vistas SQL utilizadas (`/views_sql/`)
- Información de métricas y lógica BI aplicada
- Suposiciones, reglas de negocio y dependencias con dashboards

## Estructura recomendada

/clients/
    /cliente_x/
        documentation.md
        /views_sql/
            vista_1.sql
            vista_2.sql

## Cómo agregar un nuevo cliente

1. Crear una carpeta con el nombre del cliente dentro de `/clients/`.
2. Copiar la plantilla desde `/postgres_documentation/vista_template.md` y `/postgres_documentation/bi_template.md`.
3. Completar `documentation.md` siguiendo las plantillas.
4. (Opcional) Agregar las vistas SQL originales en la carpeta `/views_sql/`.

Cada cliente queda así documentado de manera aislada, organizada y fácil de mantener.
