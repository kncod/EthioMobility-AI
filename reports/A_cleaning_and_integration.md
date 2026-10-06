# A — Data Cleaning & Integration Pipeline

Team: **Addis Demand AI** (Abraham Getachew, Ahmed Hussen, Natanim Masresha, Nigus Shiferaw, Tekilu Asefa)  
Clock used everywhere after cleaning: **Africa/Addis_Ababa (EAT, UTC+3)**.

Code: `src/cleaning.py` · Notebook: `notebooks/01_cleaning_and_integration.ipynb`  
Run: `python src/run_pipeline.py`

---

## A1 Cleaning log

See `reports/A1_cleaning_log.csv` and `data/processed/cleaning_log.csv` (**22 issues** logged across all three tables).

Highlights:

| File | Issue | Fix |
|------|-------|-----|
| trips | 55 zone spellings | Mapped to 12 canonical zones |
| trips | Mixed datetime formats | Parsed ISO+03 / Y-M-D / D/M/Y; floored to hour |
| trips | Negative trips / wait | Set to NaN; trips capped at 99.9th pct |
| weather | UTC `Z` timestamps | Converted to EAT |
| weather | `rain_mm = -9999` | Treated as missing |
| weather | `temp_c > 45` | Set to NaN |
| weather | Duplicate hours | Mean-aggregated to unique hour |
| events | Messy types/status | Normalized; cancelled kept but not used in features |
| events | Multi-zone / citywide | Expanded to long event×zone rows |
| events | Missing/inverted ends | Type-specific defaults |

---

## A2 Time & key standardization

### Zones before → after

- **Before (train):** 55 raw labels (case, spaces, aliases like `Piazza`, `Kazanches`, `Bole Rd`, `C.M.C`, `Kolfe Keranio`, `Mercato`, `Megenaga`).
- **After:** exactly the 12 canonical zones:  
  Arat Kilo, Ayat, Bole, CMC, Gerji, Kazanchis, Kolfe, Lideta, Megenagna, Merkato, Piassa, Sarbet  
  (same list on train, test, and expanded events).

### Event type before → after

| Stage | Unique labels |
|-------|----------------|
| **Before (raw)** | `CONCERT`, `Concert`, `concert`, `Conference`, `conference`, `Exhibition`, `exhibition`, `Football Match`, `football match `, `football_match`, `Public Holiday`, `public holiday`, `public_holiday`, `Road closure`, `road closure`, `road_closure`, `School Break`, `school_break`, `Sports Run`, `sports_run` (**20** spellings) |
| **After** | `concert`, `conference`, `exhibition`, `football_match`, `public_holiday`, `road_closure`, `school_break`, `sports_run` (**8** types) |

### Timestamp formats parsed

| Table | Formats |
|-------|---------|
| trips | `YYYY-MM-DD HH:MM`, `DD/MM/YYYY HH:MM`, ISO with `+03:00` |
| weather | ISO `...Z` (UTC), some `DD/MM/YYYY HH:MM` |
| events | `YYYY-MM-DD HH:MM`, `DD/MM/YYYY HH:MM`, `Mon DD, YYYY h:mm AM/PM` |

Slash dates are parsed **day-first**.

### Timezone proof

1. **Trip evidence:** a large share of `pickup_hour` values carry an explicit `+03:00` offset → EAT.
2. **Weather evidence:** most `timestamp` values end in `Z` → UTC.
3. **Conversion:** `UTC → Africa/Addis_Ababa`, then drop tz for naive EAT wall-clock joins.
4. **Sanity check:** after conversion, mean `temp_c` peaks in mid-afternoon EAT (see notebook figure), which matches local climate expectation. Joining without the +3h shift would misalign rain vs demand.

---

## A3 Join map

```
ride_demand_*  (LEFT, grain = zone × hour)
      |
      | many-to-one: pickup_hour == weather.timestamp (EAT)
      v
weather_hourly (deduped to 1 row/hour)
      |
      | interval join: zone match AND hour ∈ [start−pre, end+post]
      v
events_calendar (confirmed only; citywide → all 12 zones)
      |
      v
master_train / master_test + features
```

**Why trips are left:** every scored zone-hour must survive.  
**Event windows:** football/concert **±2h**; sports_run ±1h; holidays/closures exact interval; **confirmed only**.

---

## A4 Join audit

From `reports/A4_join_audit.json` (re-run pipeline to refresh):

- **Weather:** left row count unchanged (many-to-one). After reindexing weather onto a complete hourly spine + interpolation, join **match rate = 100%** (0 zone-hours without weather).
- **Events:** 159 confirmed event IDs; all matched ≥1 zone-hour under the window rule; 6 cancelled excluded from feature attachment.

---

## A5 Join proof — three concrete zone-hours

Pulled from `master_train` after the pipeline (same logic as notebook 01). Each example shows the left trip key, the attached weather row, event rows in the window, and resulting feature flags.

### Example 1 — Rainy hour (weather join)

| Field | Value |
|-------|-------|
| **Zone × hour** | Arat Kilo · **2025-05-02 17:00** EAT |
| **Trips (target)** | 96 |
| **Weather row** (`weather_clean`, `timestamp` = same hour) | `temp_c=21.5`, `rain_mm=21.0`, `humidity_pct=76` |
| **Derived features** | `rain_class=heavy`, `rain_last_3h=21.85` |

**Proof:** trip `pickup_hour` equals weather `timestamp` after UTC→EAT; heavy rain attaches as 21 mm that hour.

### Example 2 — Football window (event interval join)

| Field | Value |
|-------|-------|
| **Zone × hour** | Kazanchis · **2025-01-26 17:00** EAT |
| **Trips** | 55 |
| **Weather** | `temp_c=21.3`, `rain_mm=0.0` |
| **Events in window** | `EVT-0007` / `EVT-9003` · `football_match` · start **15:00** · end **17:00** · attendance **34,756** (window rule: ±2h → 17:00 is in-window) |
| **Derived features** | `in_event_window=1`, `is_football_window=1`, `hours_since_football=2.0`, `event_attendance_nearby=34756`, `n_events_overlapping=2` |

**Proof:** football features fire only when zone matches and hour ∈ [start−2h, end+2h].

### Example 3 — Public holiday

| Field | Value |
|-------|-------|
| **Zone × hour** | Ayat · **2025-05-01 18:00** EAT |
| **Trips** | 40 |
| **Event** | `EVT-0055` · `public_holiday` · **2025-05-01 00:00 → 23:00** |
| **Derived features** | `in_event_window=1`, `is_public_holiday=1` |

**Proof:** holiday attaches for the calendar day interval on every zone that the expanded event covers (Ayat included).

---

## A6 Feature engineering

≥8 engineered features with forecast-time flags **and why they help** — see `data/processed/data_dictionary_master.csv` (`why_helps` column).

Summary table (model-facing features):

| Feature | Group | Known at forecast? | Why we expect it to help |
|---------|-------|--------------------|--------------------------|
| `hour`, `dow`, `is_weekend`, `month`, `is_payday_window`, `week_index` | calendar | yes | Intra-day peaks, weekend regime, payday, secular trend |
| `temp_c`, `rain_mm`, `humidity_pct`, `wind_kmh`, `rain_class`, `rain_last_3h` | weather | yes | Rain/comfort shift mode choice; dose response via bins + 3h linger |
| `in_event_window`, `is_public_holiday`, `is_football_window`, `is_concert_window`, `is_road_closure`, `hours_to_next_football`, `hours_since_football`, `event_attendance_nearby`, `n_events_overlapping` | events | yes | Venue spikes, holiday suppression, closures |
| `trips_lag_168h`, `trips_roll_mean_24h`, `trips_roll_mean_168h` | lag | yes (history) | Same-hour last week + recent zone level |

**Excluded from model inputs (leakage):** `avg_fare_birr`, `avg_wait_min`, `active_drivers`.

---

## A7 Integrity checks

See `reports/A7_integrity_checks.md`. Latest run: **all PASS** (unique zone-hours, 12 zones, date ranges, no negative trips, weather complete, test=4032 rows, no leaky features in model list).

---

## A8 Master tables

| File | Location |
|------|----------|
| `master_train.csv` | `data/processed/` |
| `master_test.csv` | `data/processed/` (no `trips` / ops outcome columns) |
| `data_dictionary_master.csv` | `data/processed/` |

Test rows use the **same pipeline**; lag medians and imputations are learned from train only.
