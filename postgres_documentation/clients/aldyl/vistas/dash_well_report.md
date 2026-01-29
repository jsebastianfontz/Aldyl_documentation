# 📌 Vista: public.dash_well_report

## 🎯 Objetivo

Consolidar en una vista analítica la información diaria **por pozo activo** (well_category_id = 1), incluyendo:
- atributos del pozo y su contexto (Flow Station / Treatment Plant / Field),
- variables de calidad (AYS / API / sal),
- **plan del pozo**,
- variables operativas del `daily_report_well`,
- y una **asignación proporcional de producción** (gross/net) calculada a nivel Flow Station desde `dash_tank_report`, distribuida por pozo según su **potencial**.

---

## 🧷 Fuentes Utilizadas

### Producción y asignación por Flow Station
- `dash_tank_report` (producción agregada por flow station y día)
- `daily_report_well` + `well` + `flow_station` (para calcular potencial total por flow station y día)

### Reporte diario de pozos
- `daily_report_well` (presiones, tasas, parámetros mecánicos, etc.)
- `well` (metadatos del pozo: potencial, categoría, relaciones)

### Dimensiones
- `flow_station` (asignación a estación y tratamiento)
- `treatment_plant`
- `field`

### Calidad
- `lab_report` *(facility_type = 'well')*

### Planificación
- `well_plan` (plan por pozo)
- CTE `ef_plan` (plan agregado por flow station)

---

## 🧠 Lógica Principal

### 1️⃣ CTE `ef_plan` (Plan total por Flow Station)
Calcula el **plan total** por Flow Station para pozos activos:
- Une `flow_station → well → well_plan`
- Filtra `well_category_id = 1`
- `ef_plan = SUM(well_plan.plan)` agrupado por `flow_station.name`

> Sirve para mostrar un “plan total de estación” (no el plan individual del pozo).

### 2️⃣ CTE `ef_production` (Producción total por Flow Station y día)
Construye producción diaria por Flow Station:
- De `dash_tank_report` se agregan:
  - `gross_production = SUM(dtr.gross_production)`
  - `net_production = SUM(dtr.net_production)`
- Se enriquece con `tpf.total_potencial_fs` calculado por día y flow station desde `daily_report_well` + `well`:
  - `total_potencial_fs = SUM(w.potencial)` por `date_day` y `flow_station`

Solo se consideran registros donde:
- `dtr.flow_station IS NOT NULL`
- `total_potencial_fs IS NOT NULL`

### 3️⃣ Dataset final (1 fila por pozo por día)
Base: `daily_report_well drw`
- Join a `well w` **solo pozos activos** (`w.well_category_id = 1`)
- Se convierte fecha: `date_created` UTC → America/Caracas y se castea a `date` (día)
- Calidad:
  - `ays/api/salt_amount = AVG(lab_report...)` por pozo y día (si existen varios)
- Plan:
  - `well_plan = MAX(wp.plan)` por pozo (join por nombre)

### 4️⃣ Asignación proporcional de producción por pozo
Se asigna a cada pozo una fracción de la producción total de su Flow Station (por día), proporcional a su potencial:

- `pctg_prod = potencial / total_potencial_fs`
- `gross_production_pozo = efp.gross_production * pctg_prod`
- `net_production_pozo = efp.net_production * pctg_prod`

La vista usa `MAX(...)` porque `efp` ya viene agregado por FS/día y el group-by está al nivel pozo/día.

---

## 📊 Campos Principales del Resultado

| Campo | Descripción |
|------|-------------|
| well_id | ID del pozo. |
| well_name | Nombre del pozo. |
| tp_name | Planta de tratamiento asociada a la flow station. |
| field_name | Campo asociado al pozo. |
| potencial | Potencial del pozo (input para prorrateo). |
| ays | Promedio de AYS del día (lab_report). |
| api | Promedio de API del día (lab_report). |
| salt_amount | Promedio de sal del día (lab_report). |
| date_created | Día del reporte (America/Caracas). |
| flow_station_name | Estación de flujo del pozo. |
| total_potencial_fs | Potencial total de la flow station para ese día. |
| pctg_prod | Participación del pozo = potencial / total_potencial_fs. |
| well_plan | Plan individual del pozo (`well_plan.plan`). |
| gross_production | Producción bruta asignada al pozo (prorrateo). |
| net_production | Producción neta asignada al pozo (prorrateo). |
| well_category_name | Constante `'Activo'`. |
| ef_plan | Plan total de la flow station (CTE `ef_plan`). |
| presion_cabeza | `heading_pressure` (max). |
| presion_linea | `line_pressure` (max). |
| presion_injeccion | `injection_pressure` (max). |
| inyeccion_gas | `gas_injection_rate` (max). |
| diametro_placa_orificio | `hole_plate_diameter` (max). |
| diametro_reductor | `reducer_diameter` (max). |
| velocidad_bomba | `pump_speed` (max). |
| tasa_inyeccion_diluyente | `diluent_injection_rate` (max). |
| torque_varillas | `rebars_torque` (max). |
| corriente_electrica | `electric_intensity` (max). |
| golpes_por_minuto | `strokes_per_minute` (max). |
| presion_revestidor | `casing_pressure` (max). |
| longitud_carrera | `stroke_length` (max). |

---

## ⚠️ Suposiciones y Consideraciones

- **Prorrateo por potencial**: asume que la producción total de la flow station puede distribuirse proporcionalmente por potencial del pozo.
- `total_potencial_fs` se calcula desde `daily_report_well` (pozos activos) por día; si no existe, el pozo puede quedar sin producción asignada (dependiendo del join con `efp`).
- `well_plan` y `ef_plan` se unen por `well.name = well_plan.well_name` y `flow_station.name`; si hay inconsistencias de nombres, puede quedar NULL.
- Se usan agregaciones `MAX(...)` sobre variables operativas por pozo/día para tolerar múltiples registros en el mismo día.

---

## 🧱 Grano / Nivel de detalle (Grain)

- **1 fila por**: `well_id` + `date_created` (día) para pozos activos (`well_category_id = 1`).

---

## 📐 Actual SQL Script

```sql
CREATE OR REPLACE VIEW public.dash_well_report
AS WITH ef_plan AS (
         SELECT fs2_1.name AS flow_station_name,
            sum(wp_1.plan) AS ef_plan
           FROM flow_station fs2_1
             LEFT JOIN well w_1 ON fs2_1.id = w_1.flow_station_id
             LEFT JOIN well_plan wp_1 ON w_1.name::text = wp_1.well_name::text
          WHERE w_1.well_category_id = 1
          GROUP BY fs2_1.name
        ), ef_production AS (
         SELECT date_trunc('day'::text, dtr.date_created) AS date_day,
            dtr.flow_station::character varying AS flow_station,
            tpf.total_potencial_fs,
            sum(dtr.gross_production) AS gross_production,
            sum(dtr.net_production) AS net_production
           FROM dash_tank_report dtr
             LEFT JOIN ( SELECT date(((dwr.date_created AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text)) AS date_day,
                    fs.name AS flow_station_name,
                    sum(w_1.potencial) AS total_potencial_fs
                   FROM daily_report_well dwr
                     JOIN well w_1 ON dwr.well_id = w_1.id
                     JOIN flow_station fs ON w_1.flow_station_id = fs.id
                  WHERE w_1.well_category_id = 1
                  GROUP BY (date(((dwr.date_created AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text))), fs.name) tpf ON dtr.flow_station = tpf.flow_station_name::text AND date_trunc('day'::text, dtr.date_created) = tpf.date_day
          WHERE dtr.flow_station IS NOT NULL AND tpf.total_potencial_fs IS NOT NULL
          GROUP BY (date_trunc('day'::text, dtr.date_created)), (dtr.flow_station::character varying), tpf.total_potencial_fs
        )
 SELECT w.id AS well_id,
    w.name AS well_name,
    tp.name AS tp_name,
    f.name AS field_name,
    w.potencial,
    avg(lr.ays) AS ays,
    avg(lr.api) AS api,
    avg(lr.salt_amount) AS salt_amount,
    date(((drw.date_created AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text))::timestamp without time zone AS date_created,
    fs2.name AS flow_station_name,
    max(efp.total_potencial_fs) AS total_potencial_fs,
    max(round(COALESCE(w.potencial, 0)::numeric(16,2) / NULLIF(efp.total_potencial_fs::numeric(16,2), 0::numeric), 5)) AS pctg_prod,
    max(wp.plan) AS well_plan,
    max(efp.gross_production * COALESCE(w.potencial, 0)::numeric(16,2) / NULLIF(efp.total_potencial_fs::numeric(16,2), 0::numeric)) AS gross_production,
    max(efp.net_production * COALESCE(w.potencial, 0)::numeric(16,2) / NULLIF(efp.total_potencial_fs::numeric(16,2), 0::numeric)) AS net_production,
    'Activo'::character varying AS well_category_name,
    max(ep.ef_plan) AS ef_plan,
    max(drw.heading_pressure) AS presion_cabeza,
    max(drw.line_pressure) AS presion_linea,
    max(drw.injection_pressure) AS presion_injeccion,
    max(drw.gas_injection_rate) AS inyeccion_gas,
    max(drw.hole_plate_diameter) AS diametro_placa_orificio,
    max(drw.reducer_diameter) AS diametro_reductor,
    max(drw.pump_speed) AS velocidad_bomba,
    max(drw.diluent_injection_rate) AS tasa_inyeccion_diluyente,
    max(drw.rebars_torque) AS torque_varillas,
    max(drw.electric_intensity) AS corriente_electrica,
    max(drw.strokes_per_minute) AS golpes_por_minuto,
    max(drw.casing_pressure) AS presion_revestidor,
    max(drw.stroke_length) AS longitud_carrera
   FROM daily_report_well drw
     JOIN well w ON drw.well_id = w.id AND w.well_category_id = 1
     LEFT JOIN flow_station fs2 ON w.flow_station_id = fs2.id
     LEFT JOIN treatment_plant tp ON fs2.treatment_plant_id = tp.id
     LEFT JOIN field f ON w.field_id = f.id
     LEFT JOIN lab_report lr ON drw.id = lr.daily_report_id AND lr.facility_type::text = 'well'::text
     LEFT JOIN well_plan wp ON w.name::text = wp.well_name::text
     LEFT JOIN ef_plan ep ON fs2.name::text = ep.flow_station_name::text
     LEFT JOIN ef_production efp ON date(((drw.date_created AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text)) = efp.date_day AND fs2.name::text = efp.flow_station::text
  GROUP BY w.id, w.name, tp.name, f.name, w.potencial, (date(((drw.date_created AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text))), fs2.name;
```
