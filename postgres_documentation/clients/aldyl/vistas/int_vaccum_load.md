# 📌 Vista: public.int_vaccum_load

## 🎯 Objetivo

Estandarizar y **prorratear (“split”)** por día operativo (ventanas **12:00–12:00**) los volúmenes cargados vía **vacuum load** desde tanques de **Flow Station** (flow_station_id = 3), generando producción bruta y neta diaria a partir de `daily_report_vacuum_load`.

Incluye una validación para **evitar doble conteo** cuando ese mismo día ya existe producción operada registrada en `daily_report_flow_station_tank`.

---

## 🧷 Fuentes Utilizadas

### Principal
- `daily_report_vacuum_load` (evento de carga: horas inicio/fin y volumen)
- `flow_station_tank` (origen del facility cuando `origin_facility_type = 'flow_station_tank'`)
- `flow_station`
- `field`
- `treatment_plant`

### Validación anti-duplicados
- `daily_report_flow_station_tank` (para detectar si existe `net_operated_production` o `raw_operated_production` el mismo día en el mismo tanque)

---

## 🧠 Lógica Principal

### 1️⃣ Normalización Temporal
Se convierten timestamps de **UTC → America/Caracas**:
- `loading_start_time`
- `loading_end_time`

Además, el día operativo se define con ventanas:
- `modified_start = day_trunc(start - 12h) + 12h`
- `modified_end   = day_trunc(start - 12h) + 36h`

### 2️⃣ Scope / Filtros
Se consideran únicamente:
- `drvl.activity_id = 1`
- `fst.flow_station_id = 3`
- `origin_facility_type = 'flow_station_tank'` (y join por `origin_facility_id = fst.id`)

### 3️⃣ Cálculo base del evento (CTE: `time_calculations`)
Se calculan métricas del evento:
- `dif_hours = (loading_end_time - loading_start_time) en horas`
- `gross_production = drvl.volume` (volumen del vacuum load)

Se setean campos estándar para compatibilidad con reportes diarios:
- `status = 'active'`
- `tank_type = 'Flow'`
- `volume`, `level`, `ays`, `api`, `tope`, `lag`, `salt_amount` quedan **NULL** (no aplican / no se derivan aquí)

### 4️⃣ Detección de posible duplicidad (CTE: `reports`)
Se construye un listado de (día, tanque) donde ya existe producción operada:
- Desde `daily_report_flow_station_tank`
- Condición: `net_operated_production IS NOT NULL OR raw_operated_production IS NOT NULL`
- Para tanques en `flow_station_id = 3`

Esto se une luego contra el evento por:
- `time_calculations.tank_id = reports.tank_id`
- `date(time_calculations.modified_start) = reports.date_created`

El resultado se guarda como `checkk`:
- **Si `checkk` no es NULL → se fuerza producción neta = 0 (y en un segmento también gross = 0)** para evitar doble conteo.

### 5️⃣ Split por ventanas (CTE: `volume_split`)
El evento se reparte en hasta 3 segmentos:
- `hours_previous_window`
- `hours_current_window`
- `hours_next_window`

Cálculo con `LEAST/GREATEST` + `EXTRACT(epoch)/3600` y truncado con `GREATEST(0, ...)`.

### 6️⃣ Producción bruta y neta por día (3 SELECT + UNION ALL)

Para cada segmento con horas > 0:
- **Producción bruta (gross_production)**:
  - Segmento previo: si `checkk` existe → **0**, si no → `volume * hours_segment / dif_hours`
  - Segmentos actual y siguiente: siempre `volume * hours_segment / dif_hours`

- **Producción neta (net_production)**:
  - Si `checkk` existe → **0**
  - Si no → **igual a bruta** (no hay ajuste por AYS en esta vista)

Asignación de `date_created`:
- previo: `modified_start - 24h`
- actual: `modified_start`
- siguiente: `modified_end`

Orden final: `ORDER BY 1` (por `tank_id`).

---

## 📊 Campos Principales del Resultado

| Campo            | Descripción |
|-----------------|-------------|
| tank_id         | ID del tanque (flow_station_tank). |
| tank_name       | Nombre del tanque. |
| status          | Constante `'active'`. |
| date_created    | Día operativo (ventana 12:00–12:00) asignado al segmento. |
| field_name      | Campo petrolero asociado. |
| tp_name         | Planta de tratamiento asociada. |
| volume          | NULL (no se calcula en esta vista). |
| level           | NULL (no se calcula en esta vista). |
| gross_production| Volumen del vacuum load prorrateado por horas del segmento (con regla anti-duplicado en segmento previo). |
| hours           | Horas del evento asignadas a ese día/segmento. |
| ays             | NULL (no aplica). |
| api             | NULL (no aplica). |
| net_production  | Igual a gross_production, pero **0 si `checkk` detecta producción operada ese día**. |
| tope            | NULL (no aplica). |
| tank_type       | Constante `'Flow'`. |
| flow_station    | Nombre de la estación de flujo. |
| lag             | NULL (no aplica). |
| salt_amount     | NULL (no aplica). |

---

## ⚠️ Suposiciones y Consideraciones

- Se usa `NULLIF(dif_hours, 0)` para evitar división por cero; en esos casos la producción prorrateada resulta NULL.
- Un evento puede generar hasta **3 filas** (día anterior / actual / siguiente).
- La validación `checkk` busca evitar doble conteo cuando ya existe producción operada el mismo día para el mismo tanque.
  - Nota: en el SQL, la regla de `checkk` afecta:
    - `net_production` en los 3 segmentos (fuerza 0)
    - `gross_production` **solo** en el segmento previo (fuerza 0)
- Al usar `UNION ALL`, no hay deduplicación.

---

## 🧱 Grano / Nivel de detalle (Grain)

- **1 fila por**: `tank_id` + `date_created` **por evento de vacuum load segmentado**.
- Un evento puede contribuir a múltiples días (máximo 3 registros).

---

## 📐 Actual SQL Script

```sql
CREATE OR REPLACE VIEW public.int_vaccum_load
AS WITH reports AS (
         SELECT date(((drfst.date_created AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text)) AS date_created,
            drfst.flow_station_tank_id AS tank_id
           FROM daily_report_flow_station_tank drfst
             JOIN flow_station_tank fst ON drfst.flow_station_tank_id = fst.id
          WHERE (drfst.net_operated_production IS NOT NULL OR drfst.raw_operated_production IS NOT NULL) AND fst.flow_station_id = 3
        ), time_calculations AS (
         SELECT ((drvl.loading_start_time AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) AS loading_start_time,
            ((drvl.loading_end_time AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) AS loading_end_time,
            EXTRACT(epoch FROM drvl.loading_end_time - drvl.loading_start_time) / 3600::numeric AS dif_hours,
            drvl.volume AS gross_production,
            fst.id AS tank_id,
            fst.flow_station_id,
            fst.name AS tank_name,
            drvl.activity_id,
            drvl.origin_facility_type,
            drvl.origin_facility_id,
            date_trunc('day'::text, ((drvl.loading_start_time AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) - '12:00:00'::interval) + '12:00:00'::interval AS modified_start,
            date_trunc('day'::text, ((drvl.loading_start_time AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) - '12:00:00'::interval) + '36:00:00'::interval AS modified_end,
            fs2.name AS flow_station,
            tp.name AS tp_name,
            f.name AS field_name,
            'active'::text AS status,
            NULL::numeric AS volume,
            NULL::numeric AS level,
            NULL::numeric AS ays,
            NULL::numeric AS api,
            NULL::numeric AS tope,
            'Flow'::text AS tank_type,
            NULL::numeric AS lag,
            NULL::numeric AS salt_amount
           FROM daily_report_vacuum_load drvl
             JOIN flow_station_tank fst ON drvl.origin_facility_id = fst.id AND drvl.origin_facility_type::text = 'flow_station_tank'::text
             LEFT JOIN flow_station fs2 ON fst.flow_station_id = fs2.id
             LEFT JOIN field f ON fs2.field_id = f.id
             LEFT JOIN treatment_plant tp ON fs2.treatment_plant_id = tp.id
          WHERE drvl.activity_id = 1 AND fst.flow_station_id = 3
        ), volume_split AS (
         SELECT time_calculations.loading_start_time,
            time_calculations.loading_end_time,
            time_calculations.dif_hours,
            time_calculations.gross_production,
            time_calculations.tank_id,
            time_calculations.flow_station_id,
            time_calculations.tank_name,
            time_calculations.activity_id,
            time_calculations.origin_facility_type,
            time_calculations.origin_facility_id,
            time_calculations.modified_start,
            time_calculations.modified_end,
            time_calculations.flow_station,
            time_calculations.tp_name,
            time_calculations.field_name,
            time_calculations.status,
            time_calculations.volume,
            time_calculations.level,
            time_calculations.ays,
            time_calculations.api,
            time_calculations.tope,
            time_calculations.tank_type,
            time_calculations.lag,
            time_calculations.salt_amount,
            GREATEST(0::numeric, EXTRACT(epoch FROM LEAST(time_calculations.loading_end_time, time_calculations.modified_start) - time_calculations.loading_start_time) / 3600::numeric) AS hours_previous_window,
            GREATEST(0::numeric, EXTRACT(epoch FROM LEAST(time_calculations.loading_end_time, time_calculations.modified_end) - GREATEST(time_calculations.loading_start_time, time_calculations.modified_start)) / 3600::numeric) AS hours_current_window,
            GREATEST(0::numeric, EXTRACT(epoch FROM time_calculations.loading_end_time - time_calculations.modified_end) / 3600::numeric) AS hours_next_window,
            reports.tank_id AS checkk
           FROM time_calculations
             LEFT JOIN reports ON time_calculations.tank_id = reports.tank_id AND date(time_calculations.modified_start) = reports.date_created
        )
 SELECT volume_split.tank_id,
    volume_split.tank_name,
    volume_split.status,
    volume_split.modified_start - '24:00:00'::interval AS date_created,
    volume_split.field_name,
    volume_split.tp_name,
    volume_split.volume,
    volume_split.level,
        CASE
            WHEN volume_split.checkk IS NOT NULL THEN 0::numeric
            ELSE volume_split.gross_production::numeric * volume_split.hours_previous_window / NULLIF(volume_split.dif_hours, 0::numeric)
        END AS gross_production,
    volume_split.hours_previous_window AS hours,
    volume_split.ays,
    volume_split.api,
        CASE
            WHEN volume_split.checkk IS NOT NULL THEN 0::numeric
            ELSE volume_split.gross_production::numeric * volume_split.hours_previous_window / NULLIF(volume_split.dif_hours, 0::numeric)
        END AS net_production,
    volume_split.tope,
    volume_split.tank_type,
    volume_split.flow_station,
    volume_split.lag,
    volume_split.salt_amount
   FROM volume_split
  WHERE volume_split.hours_previous_window > 0::numeric
UNION ALL
 SELECT volume_split.tank_id,
    volume_split.tank_name,
    volume_split.status,
    volume_split.modified_start AS date_created,
    volume_split.field_name,
    volume_split.tp_name,
    volume_split.volume,
    volume_split.level,
    volume_split.gross_production::numeric * volume_split.hours_current_window / NULLIF(volume_split.dif_hours, 0::numeric) AS gross_production,
    volume_split.hours_current_window AS hours,
    volume_split.ays,
    volume_split.api,
        CASE
            WHEN volume_split.checkk IS NOT NULL THEN 0::numeric
            ELSE volume_split.gross_production::numeric * volume_split.hours_current_window / NULLIF(volume_split.dif_hours, 0::numeric)
        END AS net_production,
    volume_split.tope,
    volume_split.tank_type,
    volume_split.flow_station,
    volume_split.lag,
    volume_split.salt_amount
   FROM volume_split
  WHERE volume_split.hours_current_window > 0::numeric
UNION ALL
 SELECT volume_split.tank_id,
    volume_split.tank_name,
    volume_split.status,
    volume_split.modified_end AS date_created,
    volume_split.field_name,
    volume_split.tp_name,
    volume_split.volume,
    volume_split.level,
    volume_split.gross_production::numeric * volume_split.hours_next_window / NULLIF(volume_split.dif_hours, 0::numeric) AS gross_production,
    volume_split.hours_next_window AS hours,
    volume_split.ays,
    volume_split.api,
        CASE
            WHEN volume_split.checkk IS NOT NULL THEN 0::numeric
            ELSE volume_split.gross_production::numeric * volume_split.hours_next_window / NULLIF(volume_split.dif_hours, 0::numeric)
        END AS net_production,
    volume_split.tope,
    volume_split.tank_type,
    volume_split.flow_station,
    volume_split.lag,
    volume_split.salt_amount
   FROM volume_split
  WHERE volume_split.hours_next_window > 0::numeric
  ORDER BY 1;
```
