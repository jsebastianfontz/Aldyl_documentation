# 📌 Vista: public.int_filling_pm2_daily

## 🎯 Objetivo

Estandarizar y **distribuir (“split de los tanques de bombeo”)** la producción asociada a eventos de llenado en **PM-2** (Flow Station) a nivel **diario operativo**, prorrateando el volumen producido por **horas efectivas** dentro de ventanas definidas (12:00–12:00), para integrarlo como dataset diario (compatible con `dash_tank_report`).

---

## 🧷 Fuentes Utilizadas

### Principal
- `daily_report_flow_station_tank` (eventos de llenado: `filling_start_date`, `filling_end_date`, niveles, producción operada)
- `flow_station_tank` (metadatos del tanque + `conversion_factor`)
- `flow_station`
- `field`
- `treatment_plant`
- `lab_report` *(facility_type = 'flow_station_tank')*

---

## 🧠 Lógica Principal

### 1️⃣ Normalización Temporal
Todos los timestamps relevantes se convierten de **UTC → America/Caracas** usando `AT TIME ZONE`:
- `date_created`, `date_updated`
- `filling_start_date`, `filling_end_date`

### 2️⃣ Filtro y Scope
Se filtran únicamente registros con:
- `fst.flow_station_id = 1`
- `drfst.filling_start_level IS NOT NULL`

> Esto limita la vista a eventos de llenado válidos para la estación objetivo.

### 3️⃣ Cálculo base del evento de llenado (CTE: `time_calculations`)
Se calculan variables necesarias para prorrateo:

- **Horas del evento**:
  - `dif_hours = (filling_end_date - filling_start_date) en horas`
- **Volumen bruto del evento**:
  - `gross_volume = (filling_end_level - filling_start_level) * conversion_factor`
- **Ventanas operativas (día 12:00–12:00)**:
  - `modified_start = day_trunc(filling_start_date - 12h) + 12h`
  - `modified_end   = day_trunc(filling_start_date - 12h) + 36h`

Además, se estandarizan campos para compatibilidad:
- `tank_type = 'Flow'`
- `tope = NULL`
- `volume = tank_level * conversion_factor`
- `lag(tank_level)` para referencia histórica

### 4️⃣ Split por ventanas (CTE: `volume_split`)
Se calcula cuántas horas del evento caen en 3 segmentos:

- `hours_previous_window`: horas **antes** de `modified_start` (mapea al día anterior)
- `hours_current_window`: horas entre `modified_start` y `modified_end` (día actual)
- `hours_next_window`: horas **después** de `modified_end` (día siguiente)

Cada una se calcula con `LEAST/GREATEST` + `EXTRACT(epoch)/3600` y se trunca a 0 con `GREATEST(0, ...)`.

### 5️⃣ Producción bruta y neta por día (3 SELECT + UNION ALL)
Se generan hasta **3 filas** por evento (si hay horas en cada segmento), prorrateando:

**Producción bruta diaria:**
- Si `raw_operated_production` existe → `gross_production = 0`
- Si no → `gross_production = gross_volume * (hours_segment / dif_hours)`

**Producción neta diaria:**
- Si `raw_operated_production` o `net_operated_production` existen → `net_production = 0`
- Si no:
  - Si `ays` es NULL o 0 → net = gross prorrateado
  - Else → net = gross prorrateado * (1 - ays/100)

**Asignación del día (`date_created`) por segmento:**
- Segmento previo: `modified_start - 24h`
- Segmento actual: `modified_start`
- Segmento siguiente: `modified_end`

> La vista final ordena por la columna 4 (`date_created`).

---

## 📊 Campos Principales del Resultado

| Campo            | Descripción |
|-----------------|-------------|
| tank_id         | ID del tanque (flow_station_tank_id). |
| tank_name       | Nombre del tanque. |
| status          | Estado del registro. |
| date_created    | Día operativo asignado (ventana 12:00–12:00). |
| field_name      | Campo petrolero asociado. |
| tp_name         | Planta de tratamiento asociada. |
| volume          | Volumen estimado del tanque (tank_level * conversion_factor). |
| level           | Nivel del tanque (`tank_level`). |
| gross_production| Producción bruta prorrateada por horas del segmento. |
| hours           | Horas del evento asignadas a ese día/segmento. |
| ays             | % Agua y Sedimentos (si existe en lab_report). |
| api             | API del crudo (si existe en lab_report). |
| net_production  | Producción neta prorrateada y ajustada por AYS (cuando aplica). |
| tope            | NULL (no aplica para Flow). |
| tank_type       | Constante `'Flow'`. |
| flow_station    | Nombre de la estación de flujo. |
| lag             | Lectura anterior de `tank_level` (referencia). |
| salt_amount     | Cantidad de sal (si aplica). |

---

## ⚠️ Suposiciones y Consideraciones

- La lógica depende de que `filling_end_date` y `filling_start_date` estén correctamente pobladas (para `dif_hours`).
- Se usa `NULLIF(dif_hours, 0)` para evitar división por cero; en esos casos la producción prorrateada resultará NULL.
- Si existen `raw_operated_production` o `net_operated_production`, **la vista pone producción 0** (se asume que esos casos se manejan en otra lógica/tabla).
- Un mismo evento puede generar hasta **3 filas** por el split (día anterior / actual / siguiente).
- Al usar `UNION ALL`, no hay deduplicación.

---

## 🧱 Grano / Nivel de detalle (Grain)

- **1 fila por**: `tank_id` + `date_created` **por evento de llenado segmentado**.
- Un evento puede contribuir a múltiples días (máximo 3 registros).

---

## 📐 Actual SQL Script

```sql
CREATE OR REPLACE VIEW public.int_filling_pm2_daily
AS WITH time_calculations AS (
         SELECT drfst.id,
            ((drfst.date_created AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) AS date_created,
            ((drfst.date_updated AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) AS date_updated,
            drfst.edition_number,
            drfst.activity_id,
            drfst.user_id,
            drfst.flow_station_tank_id,
            drfst.status,
            drfst.current_stock,
            drfst.sample_taken,
            drfst.net_operated_production,
            drfst.tank_level AS level,
            ((drfst.filling_start_date AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) AS filling_start_date,
            drfst.filling_start_level,
            ((drfst.filling_end_date AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) AS filling_end_date,
            drfst.filling_end_level,
            drfst.raw_operated_production,
            NULL::numeric AS tope,
            lag(drfst.tank_level) OVER (PARTITION BY fst.id ORDER BY drfst.date_created) AS lag,
            lr.salt_amount,
            'Flow'::text AS tank_type,
            EXTRACT(epoch FROM drfst.filling_end_date - drfst.filling_start_date) / 3600::numeric AS dif_hours,
            (drfst.filling_end_level - drfst.filling_start_level) * fst.conversion_factor AS gross_volume,
            date_trunc('day'::text, drfst.filling_start_date - '12:00:00'::interval) + '12:00:00'::interval AS modified_start,
            date_trunc('day'::text, drfst.filling_start_date - '12:00:00'::interval) + '36:00:00'::interval AS modified_end,
            fst.name AS tank_name,
            f.name AS field_name,
            fs2.name AS flow_station,
            tp.name AS tp_name,
            round(drfst.tank_level * fst.conversion_factor, 2) AS volume,
            lr.ays,
            lr.api
           FROM daily_report_flow_station_tank drfst
             JOIN flow_station_tank fst ON drfst.flow_station_tank_id = fst.id
             LEFT JOIN flow_station fs2 ON fst.flow_station_id = fs2.id
             LEFT JOIN field f ON fs2.field_id = f.id
             LEFT JOIN treatment_plant tp ON fs2.treatment_plant_id = tp.id
             LEFT JOIN lab_report lr ON drfst.id = lr.daily_report_id AND lr.facility_type::text = 'flow_station_tank'::text
          WHERE fst.flow_station_id = 1 AND drfst.filling_start_level IS NOT NULL
        ), volume_split AS (
         SELECT time_calculations.id,
            time_calculations.date_created,
            time_calculations.date_updated,
            time_calculations.edition_number,
            time_calculations.activity_id,
            time_calculations.user_id,
            time_calculations.flow_station_tank_id,
            time_calculations.status,
            time_calculations.current_stock,
            time_calculations.sample_taken,
            time_calculations.net_operated_production,
            time_calculations.level,
            time_calculations.filling_start_date,
            time_calculations.filling_start_level,
            time_calculations.filling_end_date,
            time_calculations.filling_end_level,
            time_calculations.raw_operated_production,
            time_calculations.tope,
            time_calculations.lag,
            time_calculations.salt_amount,
            time_calculations.tank_type,
            time_calculations.dif_hours,
            time_calculations.gross_volume,
            time_calculations.modified_start,
            time_calculations.modified_end,
            time_calculations.tank_name,
            time_calculations.field_name,
            time_calculations.flow_station,
            time_calculations.tp_name,
            time_calculations.volume,
            time_calculations.ays,
            time_calculations.api,
            GREATEST(0::numeric, EXTRACT(epoch FROM LEAST(time_calculations.filling_end_date, time_calculations.modified_start) - time_calculations.filling_start_date) / 3600::numeric) AS hours_previous_window,
            GREATEST(0::numeric, EXTRACT(epoch FROM LEAST(time_calculations.filling_end_date, time_calculations.modified_end) - GREATEST(time_calculations.filling_start_date, time_calculations.modified_start)) / 3600::numeric) AS hours_current_window,
            GREATEST(0::numeric, EXTRACT(epoch FROM time_calculations.filling_end_date - time_calculations.modified_end) / 3600::numeric) AS hours_next_window
           FROM time_calculations
        )
 SELECT volume_split.flow_station_tank_id AS tank_id,
    volume_split.tank_name,
    volume_split.status,
    volume_split.modified_start - '24:00:00'::interval AS date_created,
    volume_split.field_name,
    volume_split.tp_name,
    volume_split.volume,
    volume_split.level,
    round(
        CASE
            WHEN volume_split.raw_operated_production IS NOT NULL THEN 0::numeric
            ELSE volume_split.gross_volume * volume_split.hours_previous_window / NULLIF(volume_split.dif_hours, 0::numeric)
        END, 2) AS gross_production,
    volume_split.hours_previous_window AS hours,
    volume_split.ays,
    volume_split.api,
    round(
        CASE
            WHEN volume_split.raw_operated_production IS NOT NULL OR volume_split.net_operated_production IS NOT NULL THEN 0::numeric
            WHEN volume_split.ays IS NULL OR volume_split.ays = 0::numeric THEN volume_split.gross_volume * volume_split.hours_previous_window / NULLIF(volume_split.dif_hours, 0::numeric)
            ELSE volume_split.gross_volume * volume_split.hours_previous_window / NULLIF(volume_split.dif_hours, 0::numeric) * (1::numeric - volume_split.ays / 100::numeric)
        END, 2) AS net_production,
    volume_split.tope,
    volume_split.tank_type,
    volume_split.flow_station,
    volume_split.lag,
    volume_split.salt_amount
   FROM volume_split
  WHERE volume_split.hours_previous_window > 0::numeric
UNION ALL
 SELECT volume_split.flow_station_tank_id AS tank_id,
    volume_split.tank_name,
    volume_split.status,
    volume_split.modified_start AS date_created,
    volume_split.field_name,
    volume_split.tp_name,
    volume_split.volume,
    volume_split.level,
    round(
        CASE
            WHEN volume_split.raw_operated_production IS NOT NULL THEN 0::numeric
            ELSE volume_split.gross_volume * volume_split.hours_current_window / NULLIF(volume_split.dif_hours, 0::numeric)
        END, 2) AS gross_production,
    volume_split.hours_current_window AS hours,
    volume_split.ays,
    volume_split.api,
    round(
        CASE
            WHEN volume_split.net_operated_production IS NOT NULL OR volume_split.raw_operated_production IS NOT NULL THEN 0::numeric
            WHEN volume_split.ays IS NULL OR volume_split.ays = 0::numeric THEN volume_split.gross_volume * volume_split.hours_current_window / NULLIF(volume_split.dif_hours, 0::numeric)
            ELSE volume_split.gross_volume * volume_split.hours_current_window / NULLIF(volume_split.dif_hours, 0::numeric) * (1::numeric - volume_split.ays / 100::numeric)
        END, 2) AS net_production,
    volume_split.tope,
    volume_split.tank_type,
    volume_split.flow_station,
    volume_split.lag,
    volume_split.salt_amount
   FROM volume_split
  WHERE volume_split.hours_current_window > 0::numeric
UNION ALL
 SELECT volume_split.flow_station_tank_id AS tank_id,
    volume_split.tank_name,
    volume_split.status,
    volume_split.modified_end AS date_created,
    volume_split.field_name,
    volume_split.tp_name,
    volume_split.volume,
    volume_split.level,
    round(
        CASE
            WHEN volume_split.raw_operated_production IS NOT NULL THEN 0::numeric
            ELSE volume_split.gross_volume * volume_split.hours_next_window / NULLIF(volume_split.dif_hours, 0::numeric)
        END, 2) AS gross_production,
    volume_split.hours_next_window AS hours,
    volume_split.ays,
    volume_split.api,
    round(
        CASE
            WHEN volume_split.net_operated_production IS NOT NULL OR volume_split.raw_operated_production IS NOT NULL THEN 0::numeric
            WHEN volume_split.ays IS NULL OR volume_split.ays = 0::numeric THEN volume_split.gross_volume * volume_split.hours_next_window / NULLIF(volume_split.dif_hours, 0::numeric)
            ELSE volume_split.gross_volume * volume_split.hours_next_window / NULLIF(volume_split.dif_hours, 0::numeric) * (1::numeric - volume_split.ays / 100::numeric)
        END, 2) AS net_production,
    volume_split.tope,
    volume_split.tank_type,
    volume_split.flow_station,
    volume_split.lag,
    volume_split.salt_amount
   FROM volume_split
  WHERE volume_split.hours_next_window > 0::numeric
  ORDER BY 4;
```
