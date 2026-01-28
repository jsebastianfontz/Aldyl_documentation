Vista: public.dash_tank_report
Objetivo

Unificar en una sola vista la información diaria de tanques Storage, Settlement y Flow Station, calculando volumen, nivel, producción bruta, producción neta ajustada por AYS, y variables de calidad. Provee un dataset listo para BI, con timestamps normalizados a America/Caracas.

Fuentes
Storage Tanks

daily_report_storage_tank

treatment_plant_dynamic_storage_tank

treatment_plant

field

lab_report (facility_type = 'storage_tank')

Settlement Tanks

daily_report_dynamic_settlement_tank

treatment_plant_dynamic_storage_tank

treatment_plant

field

lab_report (facility_type = 'dynamic_settlement_tank')

Flow Station Tanks

daily_report_flow_station_tank

flow_station_tank

flow_station

field

treatment_plant

lab_report (facility_type = 'flow_station_tank')

Fuentes internas unificadas

int_filling_pm2_daily

int_vaccum_load

int_upt_production

Lógica Principal
1. Normalización temporal

Conversión de date_created desde UTC → America/Caracas.

2. Cálculo de volumen y nivel

Storage: usa volume; si es nulo, deriva de level × conversion_factor.

Settlement: calcula volume usando alturas (ft/in/sixteenths) × conversion_factor.

Flow: nivel = tank_level; volumen = tank_level × conversion_factor.

3. Producción bruta (gross_production)

En Storage/Settlement → delta contra la lectura anterior (lag()), truncado a 0.

En Flow Station:

Si existe raw_operated_production, se usa directamente.

Casos particulares para estaciones PM-2, PC-1 y otras.

En estaciones no PM-2/PC-1: si hay salto de días → delta normalizado / días transcurridos.

4. Producción neta (net_production)

Base: bruta × (1 – AYS/100).

Si ays es 0 o NULL → neta = bruta.

Si existe net_operated_production → tiene prioridad.

Si existe raw_operated_production y no neta → se ajusta con AYS.

5. Unión de fuentes

El resultado final une 6 conjuntos de datos usando UNION ALL:

storage_tank

settlement_tank

flow_station_tank

int_filling_pm2_daily

int_vaccum_load

int_upt_production

Esto permite que el reporte final tenga una estructura uniforme.

Campos Principales
Campo	Descripción
tank_id	Identificador del tanque.
tank_name	Nombre del tanque.
status	Estado operativo reportado.
date_created	Timestamp del reporte (America/Caracas).
field_name	Campo asociado (campo petrolero).
tp_name	Planta de tratamiento asociada.
volume	Volumen calculado o reportado del tanque.
level	Lectura de nivel o interface level.
gross_production	Producción bruta calculada (delta o valor operado).
ays	% Agua y Sedimentos (Calidad).
api	API del crudo.
net_production	Producción neta ajustada por AYS.
tope	Valor de tope (solo Storage; otros orígenes vienen null).
tank_type	Tipo de tanque: Storage, Settlement o Flow.
flow_station	Estación de flujo asociada (solo Flow).
lag	Lectura anterior usada para cálculos de delta.
salt_amount	Cantidad de sal medida (si aplica).
Suposiciones y Consideraciones

Se truncan valores negativos de producción a 0 para evitar falsos negativos en cálculos operativos.

Storage y Settlement aplican filtro: treatment_plant_id = 1.

lab_report puede no estar disponible todos los días, lo que deja AYS/API en null.

Las vistas int_* deben venir ya estandarizadas en su estructura.

Al usar UNION ALL, pueden existir múltiples registros por tanque y día si diferentes fuentes reportan el mismo tanque.

3. Lógica BI (Metabase)

Si me cuentas qué dashboards o tarjetas consumen esta vista, te documento esta sección específica del cliente ALDYL.
Normalmente incluyo:

dashboards → “Tanques diarios”, “Producción total”, “Inventario por campo”, etc.

filtros aplicados

métricas calculadas

columnas usadas en BI

reglas de negocio que Metabase interpreta encima de la vista
