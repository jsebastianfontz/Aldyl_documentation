Documentación de Vistas en PostgreSQL
# 📌 Vista: public.dash_tank_report
🎯 Objetivo

Unificar en una vista analítica toda la información diaria de tanques Storage, Settlement y Flow Station, calculando volumen, nivel, producción bruta, producción neta ajustada por AYS y variables de calidad.
Incluye estandarización temporal a America/Caracas y unión de fuentes internas adicionales (int_*).

🧷 Fuentes Utilizadas

## Storage Tanks

daily_report_storage_tank

treatment_plant_dynamic_storage_tank

treatment_plant

field

lab_report (facility_type = 'storage_tank')

## Settlement Tanks

daily_report_dynamic_settlement_tank

treatment_plant_dynamic_storage_tank

treatment_plant

field

lab_report (facility_type = 'dynamic_settlement_tank')

## Flow Station Tanks

daily_report_flow_station_tank

flow_station_tank

flow_station

field

treatment_plant

lab_report (facility_type = 'flow_station_tank')

## Fuentes Internas Unificadas

int_filling_pm2_daily

int_vaccum_load

int_upt_production

# 🧠 Lógica Principal
1️⃣ Normalización Temporal

Todos los date_created se convierten de UTC → America/Caracas usando AT TIME ZONE.

2️⃣ Cálculo de Volumen y Nivel

Storage Tanks

Si volume viene nulo → se deriva de level × conversion_factor.

Settlement Tanks

volume se calcula a partir de alturas:
(ft + in/12 + sixteenths/192) × conversion_factor

Flow Station Tanks

level = tank_level

volume = tank_level × conversion_factor

3️⃣ Producción Bruta (gross_production)

Se usa delta contra la lectura anterior:
lag(...) OVER (PARTITION BY tank ORDER BY date_created)

Se aplica GREATEST(0, …) para truncar negativos.

Flow Station tiene reglas especiales:

PM-2 → usa filling_start_level - lag(filling_end_level)

PC-1 → usa delta de tank_level

Otros → normaliza por días transcurridos si existen gaps (delta / days_diff)

4️⃣ Producción Neta (net_production)

Fórmula base:
net = gross_production × (1 – ays/100)

Si ays es NULL o 0 → net = gross

Si net_operated_production existe → prioridad

Si solo hay raw_operated_production → se usa y se ajusta por AYS si corresponde

5️⃣ Unión de Fuentes

La vista final usa UNION ALL para unir 6 subconjuntos:

storage_tank

settlement_tank

flow_station_tank

int_filling_pm2_daily

int_vaccum_load

int_upt_production

Esto permite un dataset “wide” estandarizado.

📊 Campos Principales del Resultado
Campo	Descripción
tank_id	ID del tanque.
tank_name	Nombre del tanque.
status	Estado del tanque en el reporte.
date_created	Timestamp del reporte (America/Caracas).
field_name	Campo petrolero asociado.
tp_name	Planta de tratamiento asociada.
volume	Volumen reportado o calculado.
level	Nivel o interface level del tanque.
gross_production	Producción bruta diaria.
ays	% Agua y Sedimentos.
api	API del crudo.
net_production	Producción neta ajustada por AYS.
tope	Tope del tanque (solo Storage).
tank_type	Tipo de tanque: Storage, Settlement o Flow.
flow_station	Estación de flujo (solo Flow).
lag	Lectura anterior utilizada para calcular deltas.
salt_amount	Cantidad de sal (si aplica).
⚠️ Suposiciones y Consideraciones

La producción negativa siempre se trunca a 0.

Storage y Settlement están filtrados por treatment_plant_id = 1.

lab_report puede no existir para todos los días/tanques → AYS/API pueden quedar null.

Las vistas internas int_* deben venir ya estandarizadas.

Al usar UNION ALL, la vista no deduplica registros.

📐 3. Lógica BI (Metabase)

(Si me dices los dashboards exactos que consumen esta vista, te agrego esta sección con métricas, filtros, cálculos personalizados y dependencias.)
