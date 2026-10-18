# End-to-End Project Brief: Tactical Resource Allocation for Madrid Road Safety and Emergency Response

## 1. Dataset Selection and Source Map

This project uses the official road traffic accident records published by the Madrid City Council (*Ayuntamiento de Madrid*), combining the complete annual files for 2024 and 2025. To address missing weather records and measure actual rainfall intensity, the primary dataset will be joined with historical hourly meteorological records from Open-Meteo (sourced from AEMET station data in Madrid).

### Dataset Card

| Field | Project Details |
| :--- | :--- |
| **Name** | Accidentes de tráfico de la Ciudad de Madrid (2024–2025 combined) |
| **Publisher and URL** | Ayuntamiento de Madrid — Open Data Portal (https://datos.madrid.es/dataset/300228-0-accidentes-trafico-detalle) |
| **Licence** | Condiciones generales para la modalidad general de puesta a disposición de los documentos reutilizables del Ayuntamiento de Madrid (Open reuse allowed for commercial and non-commercial purposes with attribution: https://datos.madrid.es/egob/catalogo/aviso-legal) |
| **Download date and version** | Downloaded on 18/10/2026 (Complete consolidated files for 2024 and 2025) |
| **Grain: what is one row?** | One person involved in a traffic accident (driver, passenger, or pedestrian). Multiple rows share the same `num_expediente` (case ID) when several people or vehicles are involved in a single crash. |
| **Period covered** | 24 months: `01/01/2024` to `31/12/2025` |
| **Size: rows × columns, MB** | `100,407` rows × `19` columns (`49,340` rows in 2024 + `51,067` rows in 2025), representing `42,386` unique accidents (`num_expediente`). Disk size: `18.45 MB`. |
| **Key columns** | **Dates/Time:** `fecha`, `hora`. **Dimensions:** `distrito` (21 districts), `tipo_accidente`, `estado_meteorológico`, `tipo_vehiculo`, `tipo_persona`, `rango_edad`, `sexo`. **Measures / Target inputs:** `cod_lesividad` (injury severity code), `positiva_alcohol`, `positiva_droga`, `coordenada_x_utm`, `coordenada_y_utm`, plus derived accident counts and severe injury rates per district-shift. |
| **Known quality issues spotted** | 1. **High missingness in specific columns:** `positiva_droga` is `99.66%` null (only positive `"S"` results are recorded; negatives/untested are left blank), `cod_lesividad` / `lesividad` are `44.44%` null (corresponding to property-damage-only crashes without medical attendance), `estado_meteorológico` is `11.92%` null, `tipo_vehiculo` is `0.82%` null, `positiva_alcohol` is `0.37%` null, and UTM coordinates are `0.02%` null.<br>2. **Exact duplicate rows (`4,118` rows):** Because personal identifiers are stripped for privacy, two passengers of the same age group and sex in the same vehicle produce identical rows under the same `num_expediente`.<br>3. **Unreported categories:** Several categorical columns contain literal strings such as `"Desconocido"` or `"Sin especificar"` alongside actual `NaN` values.<br>4. **Type formatting:** `fecha` and `hora` are stored as plain strings; UTM coordinates require numeric parsing and projection conversion (`EPSG:25830` to `WGS84`) for map plotting. |
| **Other sources to join** | 1. **Open-Meteo Historical Weather API (AEMET Madrid stations):** Hourly precipitation (`mm`), temperature (`°C`), and wind gusts (`km/h`) joined by `fecha` and `hora` to fix the `11.92%` missing weather values and replace subjective labels with numerical rain volume.<br>2. **Madrid Official Public Holiday Calendar (`datos.madrid.es`):** Joined by `fecha` to distinguish regular weekdays from bank holidays and bridge days (*puentes*). |

### Data Limitations, Coverage Bias, and Operational Impact

Relying on police-reported records introduces **reporting bias**: minor collisions resolved amicably between drivers via insurance forms (*parte amistoso*) without calling the Municipal Police or SAMUR are absent from the dataset. Consequently, property-damage-only accidents are under-represented compared to crashes involving injuries (`cod_lesividad` codes 01 to 07). Furthermore, because the raw grain is at the person level (`100,407` rows) rather than the crash level (`42,386` unique `num_expediente`), aggregating metrics without first grouping by `num_expediente` would artificially inflate crash counts in districts with higher public transport or high-occupancy vehicle traffic. In Block 1 and Block 2, we will maintain two linked tables: a person-level table for demographic/vulnerability analysis and a crash-level fact table (`42,386` rows) for spatial-temporal demand forecasting.

---

## 2. Business Problem Framing

### 2.1 Organisation, Department, and Stakeholder

- **Organisation:** The Madrid City Council (*Ayuntamiento de Madrid*), specifically the General Directorate of Municipal Police (*Policía Municipal de Madrid*) in operational coordination with the municipal emergency medical service (*SAMUR-Protección Civil*). Together, they manage urban traffic enforcement, crash investigation (*Atestados*), and pre-hospital emergency response across Madrid's 21 administrative districts.
- **Department:** The Central Traffic Operations and Emergency Dispatch Unit (*Unidad Central de Tráfico y Centro Integrado de Señales y Emergencias*), consisting of patrol shift planners and ambulance positioning coordinators who assign mobile traffic units and standby emergency vehicles across the 21 districts across three daily 8-hour shifts (Morning: `06:00–14:00`, Afternoon: `14:00–22:00`, Night: `22:00–06:00`).
- **Stakeholder:** The **Shift Operations Commander (*Jefe de Turno de Operaciones de Movilidad y Emergencias*)**. They review the operational dashboard daily at `05:30` and every Friday at `12:00` to lock in the weekly district allocation roster. They have direct authority to reassign preventive patrol checkpoints (such as speed and sobriety controls) and pre-position standby ambulances across district base stations.

### 2.2 The Decision to Support

> *"With this solution, the Shift Operations Commander will decide how many preventive traffic patrols and standby emergency units to pre-position in each of Madrid's 21 districts per 8-hour shift, and where to schedule weekend alcohol/speed enforcement checkpoints."*

**Operational Constraints and Alternatives:**

- **Resource and budget limits:** The unit operates with a fixed fleet per shift; moving a patrol car or ambulance to reinforce high-risk districts reduces coverage elsewhere, and overtime shifts are capped by the municipal budget.
- **Minimum coverage rule:** By municipal mandate, none of the 21 districts can be left with zero coverage, meaning optimization only applies to the flexible pool of tactical reinforcement units.
- **Legal and privacy boundaries (GDPR / LOPDGDD):** No license plates, driver IDs, or exact residential addresses are available. Predictive profiling must operate strictly at the district and time-window level, never targeting individuals.
- **Alternatives considered:** Currently, allocation relies on static historical averages (assigning units purely by district population or last year's annual totals) or reactive dispatching after a 112 call arrives. Static rules ignore weather shocks and holiday shifts, while reactive dispatching increases response times during peak traffic congestion.

### 2.3 Hypotheses

| Field | H1 Details |
| :--- | :--- |
| **H1 Statement** | Moderate-to-heavy rainfall (`> 2.0 mm/hour`) increases hourly crash frequency across inner-ring commuter districts (*Chamartín, Salamanca, Chamberí, Tetuán*) by more than `25%` during weekday morning (`07:00–10:00`) and afternoon (`17:00–20:00`) rush hours compared to dry days. |
| **Metric** | **Rain Crash Lift (%)**: `((Mean crashes per rush-hour on rainy days - Mean crashes per rush-hour on dry days) / Mean crashes per rush-hour on dry days) * 100`, grouped by district and weekday. |
| **Expected direction** | Significantly higher (`> +25%`) in high-density commuter districts, particularly involving two-wheeled vehicles (motorcycles/mopeds). |
| **How we test it** | Join crash timestamps (`num_expediente`) with Open-Meteo hourly precipitation; compare medians and IQRs (Lesson 03/04) using grouped boxplots and a two-sample Mann-Whitney U test (given right-skewed hourly crash counts). |
| **If true →** | Trigger an automated "Wet Rush-Hour Protocol" whenever AEMET forecasts `> 2.0 mm` of rain, pre-positioning traffic units at major arterial entry corridors 30 minutes before peak traffic starts. |
| **If false →** | Keep standard dry-weather patrol routes on rainy days and avoid unnecessary pre-deployment reallocations. |

| Field | H2 Details |
| :--- | :--- |
| **H2 Statement** | Crashes occurring during weekend night shifts (Friday and Saturday, `22:00–06:00`) involving positive alcohol tests (`positiva_alcohol == 'S'`) are spatially concentrated (`> 50%` of cases) in just 5 of the 21 districts (*Centro, Salamanca, Chamberí, Moncloa-Aravaca, Tetuán*) and exhibit at least double the severe injury rate of daytime crashes. |
| **Metric** | 1. **District Share of Alcohol Crashes (%)**: `(Alcohol-positive crashes in top 5 districts / Total alcohol-positive crashes in Madrid) * 100`.<br>2. **Severe Injury Ratio**: Share of crashes with `cod_lesividad` in `{03, 04}` (hospitalisation >24h or death within 24h). |
| **Expected direction** | Highly concentrated (`> 50%` in 5 districts) with a severe injury rate at least `2.0x` higher than weekday daytime shifts. |
| **How we test it** | Cross-tabulation and Pareto/sorted bar charts (Lesson 04) of unique crashes by `distrito`, `shift`, and `positiva_alcohol`, paired with relative risk ratios on cleaned `cod_lesividad` categories. |
| **If true →** | Concentrate 65% of mobile sobriety checkpoints (*controles de alcoholemia*) at the exit arteries of those 5 nightlife districts between `00:30` and `04:30` on Friday and Saturday nights. |
| **If false →** | Distribute weekend night checkpoints proportionally to total road network mileage across all 21 districts. |

| Field | H3 Details |
| :--- | :--- |
| **H3 Statement** | Vulnerable road users (motorcycles, mopeds, bicycles, and personal mobility vehicles/VMPs) account for less than `25%` of total involved vehicles (`tipo_vehiculo`), but represent over `55%` of all severe injuries and fatalities (`cod_lesividad` in `{03, 04}`). |
| **Metric** | **Vulnerability Disparity Index**: Share of severe/fatal victims riding two-wheeled/VMP vehicles divided by their share in total vehicle involvement. |
| **Expected direction** | Strongly disproportionate concentration (`> 55%` of severe casualties). |
| **How we test it** | Group person-level records (excluding exact passenger duplicates) by standardised `tipo_vehiculo` families and severity brackets; visualize via stacked 100% bar charts and contingency tables (Lesson 04). |
| **If true →** | Prioritise standby SAMUR Advanced Life Support (ALS) ambulances near corridors with high two-wheel crash density rather than corridors with high volumes of minor bumper-to-bumper car collisions. |
| **If false →** | Position ALS ambulances strictly according to total raw crash volume regardless of vehicle mix. |

### 2.4 Key Performance Indicators (KPIs)

1. **Crash Frequency per Shift (`CFS`):**
   - **Definition & Formula:** Total unique accidents per shift: `COUNT(DISTINCT num_expediente)`.
   - **Grain & Comparison:** Per District × 8-hour Shift × Day; compared against the 4-week rolling median for the same district and shift (`units_roll4` baseline from Lesson 05).
   - **Target:** Keep peak-shift unassisted backlog below threshold by pre-allocating units where forecasted `CFS` is in the top quartile.
2. **Severe Casualty Rate (`SCR`) — *Connected to H2 & H3*:**
   - **Definition & Formula:** Percentage of unique crashes resulting in serious hospitalisation (`03`) or death (`04`): `(Crashes with min(cod_lesividad) in {03, 04} / Total unique crashes) * 100`.
   - **Grain & Comparison:** Per District × Month × Vehicle Family.
   - **Target:** Reduce `SCR` by `10%` year-over-year in high-risk corridors through targeted ambulance pre-positioning.
3. **Alcohol-Related Night Crash Rate (`ANCR`) — *Connected to H2*:**
   - **Definition & Formula:** Share of night-shift crashes (`22:00–06:00`) where at least one driver tested positive (`positiva_alcohol == 'S'`): `(Night crashes with alcohol positive / Total night crashes) * 100`.
   - **Grain & Comparison:** Per District × Weekend/Weekday.
   - **Target:** Lower `ANCR` below `8.0%` in nightlife districts via proactive checkpoint placement.
4. **Weather Sensitivity Ratio (`WSR`) — *Connected to H1*:**
   - **Definition & Formula:** Ratio of hourly crash rate during rainy hours (`precip > 1.0 mm`) to dry hours (`precip == 0 mm`): `(Crashes / Rainy Hours) / (Crashes / Dry Hours)`.
   - **Grain & Comparison:** Per District × Rush-Hour Window.
   - **Target:** Identify all districts where `WSR > 1.25` to automate wet-weather patrol alerts.
5. **Vulnerable User Involvement Rate (`VUIR`) — *Connected to H3*:**
   - **Definition & Formula:** Share of accidents involving at least one pedestrian, cyclist, e-scooter (VMP), or motorcyclist.
   - **Grain & Comparison:** Per District × Time of Day.
   - **Target:** Flag any district exceeding a `35%` `VUIR` for targeted municipal speed-calming enforcement.

---

## 3. Solution Design and General Architecture

### 3.1 Solution Type

We will build an **Analytical Engine (Primary)** integrated into a **Management & Operational Dashboard (Secondary)**:

- **Inputs:** Historical police crash records (`datos.madrid.es` CSVs), Madrid official holiday calendar, and 48-hour weather forecasts / historical hourly logs from the Open-Meteo API.
- **Logic:** A data pipeline aggregates person-level records into two clean analytical tables (crash-level and victim-level) and engineers lagged temporal features (`shift(1)`, 4-week rolling averages, cyclical hour/month `sin/cos` encodings, and rain interaction terms, avoiding any future data leakage). In Block 3, a supervised machine learning model will predict two outputs per District × Shift: **(a)** expected crash count (regression) and **(b)** probability of a severe/high-casualty shift (classification).
- **Outputs:** A daily risk-ranked table of the 21 districts for the upcoming three shifts, flagging "High Alert" districts when rain or holiday patterns spike risk, displayed directly on an interactive Streamlit/Plotly dashboard for the Shift Commander.

### 3.2 Architecture Diagram and Layer Specification

```mermaid
flowchart LR
    A[Source<br/>datos.madrid.es CSVs<br/>+ Open-Meteo REST API] --> B[Ingestion<br/>Python requests & pandas<br/>schema validation]
    B --> C[Cleaning & Quality<br/>Deduplication, null handling,<br/>UTM to Lat/Lon, text norm]
    C --> D[Data Model<br/>Parquet Star Schema:<br/>fact_crashes, fact_victims, dims]
    D --> E[Analytics & ML Engine<br/>KPI calculation, lag features,<br/>scikit-learn risk model]
    E --> F[Dashboard & App<br/>Streamlit + Plotly<br/>Interactive Shift Planner]
    F --> G((Stakeholder<br/>Shift Commander<br/>Unit Allocation))
