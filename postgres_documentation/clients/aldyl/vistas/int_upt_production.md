# 📌 Vista: public.int_upt_production

## 🎯 Objetivo

Estandarizar la producción diaria reportada en tanques **UPT** (y su contexto operativo) en un formato compatible con el dataset unificado (`dash_tank_report`), exponiendo volumen, producción bruta/operada, producción neta (raw) y variables de calidad (AYS/API/sal).

---

## 🧷 Fuentes Utilizadas

### Principal
- `daily_report_upt_tank` (mediciones y producción diaria UPT)

### Dimensiones / Catálogos
- `upt_tank` (metadatos del tanque UPT)
- `upt` (relación a campo)
- `field` (campo petrolero)
- `location` *(join presente, no se expone en columnas finales)*

### Calidad
- `lab_report` *(facility_type = 'upt_tank')*

### Enriquecimiento de “tp_name” y “flow_station”
- `flow_station` (cuando el UPT tank está asociado a una flow station directamente)
- `well` (cuando el UPT tank está asociado a un pozo)
- `flow_station` (derivada del pozo vía `well.flow_station_id`)

---

## 🧠 Lógica Principal

### 1️⃣ Normalización Temporal
`date_created` se convierte de **UTC → America/Caracas** con `AT TIME ZONE`.

### 2️⃣ Definición de planta / facility (tp_name)
Se construye `tp_name` como:
- Si existe `fs2.name` (tanque asociado a `flow_station`) → `tp_name = fs2.name`
- Si no → `tp_name = w.name` (tanque asociado a `well`)

> En esta vista, `tp_name` representa el “facility” asociado (flow station o well), no necesariamente una planta de tratamiento.

### 3️⃣ Producción y medidas
- `volume = drut.fluid_volume`
- `level = NULL` *(no aplica / no se reporta)*
- `gross_production = drut.gross_operated_production::integer`
- `net_production = drut.raw_operated_production` *(tal cual viene del reporte)*

### 4️⃣ Tank type y flow station
- `tank_type = 'upt'`
- `flow_station` se define como:
  - Si existe `fs2.name` → `flow_station = fs2.name`
  - Else → `flow_station = fs3.name` (flow station del pozo)

### 5️⃣ Calidad (AYS / API / sal)
Se trae desde `lab_report` por:
- `drut.id = lr.daily_report_id`
- `lr.facility_type = 'upt_tank'`

---

## 📊 Campos Principales del Resultado

| Campo            | Descripción |
|-----------------|-------------|
| tank_id         | ID del tanque UPT (`upt_tank_id`). |
| tank_name       | Nombre del tanque UPT. |
| status          | Estado del registro en el reporte diario. |
| date_created    | Timestamp del reporte (America/Caracas). |
| field_name      | Campo petrolero asociado (vía `upt`). |
| tp_name         | Facility asociado: `flow_station` (si existe) o `well`. |
| volume          | Volumen del fluido (`fluid_volume`). |
| level           | NULL (no aplica). |
| gross_production| Producción bruta/operada (`gross_operated_production`) casteada a entero. |
| ays             | % Agua y Sedimentos (si existe). |
| api             | API del crudo (si existe). |
| net_production  | Producción neta (raw) reportada (`raw_operated_production`). |
| tope            | NULL (no aplica). |
| tank_type       | Constante `'upt'`. |
| flow_station    | Flow station asociada directa (`fs2`) o la del pozo (`fs3`). |
| lag             | NULL (no aplica). |
| salt_amount     | Cantidad de sal (si aplica). |

---

## ⚠️ Suposiciones y Consideraciones

- `gross_production` se fuerza a **entero** (`::integer`); si se requiere precisión decimal, revisar el tipo original.
- `tp_name` puede representar **flow station o well** según `upt_tank.facility_type`.
- `lab_report` puede no existir para todos los días → `ays/api/salt_amount` pueden venir NULL.
- `location` está unido pero no se utiliza en el output (join informativo / legado).

---

## 🧱 Grano / Nivel de detalle (Grain)

- **1 fila por**: `tank_id` + `date_created` (registro diario del tanque UPT).

---

## 📐 Actual SQL Script

```sql
CREATE OR REPLACE VIEW public.int_upt_production
AS SELECT drut.upt_tank_id AS tank_id,
    ut.name AS tank_name,
    drut.status,
    ((drut.date_created AT TIME ZONE 'UTC'::text) AT TIME ZONE 'America/Caracas'::text) AS date_created,
    f.name AS field_name,
        CASE
            WHEN fs2.name IS NULL THEN w.name
            ELSE fs2.name
        END::text AS tp_name,
    drut.fluid_volume AS volume,
    NULL::numeric AS level,
    drut.gross_operated_production::integer AS gross_production,
    lr.ays,
    lr.api,
    drut.raw_operated_production AS net_production,
    NULL::numeric AS tope,
    'upt'::text AS tank_type,
        CASE
            WHEN fs2.name::text IS NOT NULL THEN fs2.name::text
            ELSE fs3.name::text
        END AS flow_station,
    NULL::numeric AS lag,
    lr.salt_amount
   FROM daily_report_upt_tank drut
     LEFT JOIN upt_tank ut ON drut.upt_tank_id = ut.id
     LEFT JOIN upt u ON ut.upt_id = u.id
     LEFT JOIN field f ON u.field_id = f.id
     LEFT JOIN location l ON f.location_id = l.id
     LEFT JOIN lab_report lr ON drut.id = lr.daily_report_id AND lr.facility_type::text = 'upt_tank'::text
     LEFT JOIN flow_station fs2 ON ut.facility_id = fs2.id AND ut.facility_type::text = 'flow_station'::text
     LEFT JOIN well w ON ut.facility_id = w.id AND ut.facility_type::text = 'well'::text
     LEFT JOIN flow_station fs3 ON w.flow_station_id = fs3.id;
```
