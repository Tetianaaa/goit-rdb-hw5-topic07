# goit-rdb-hw5-topic07
# HW #5: Часові ряди та напівструктуровані дані у задачах ML

## Дані

| Частина | Джерело | Як використано |
|---|---|---|
| Time-series | NYC TLC Yellow Taxi Trip Records, лютий–квітень 2024 (<https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page>) | по 150 000 випадкових поїздок з кожного місяця → `nyc_taxi_raw` → денна таблиця `t7_hw_taxi_daily` |
| JSONB | синтетичний event-log застосунку виклику таксі | 100 000 подій п'яти типів у `t7_hw_events` |

Три місяці, щоб lag-30d мав сенс (90 днів історії). Через вибірку `trip_count` — це кількість поїздок у вибірці, форма ряду зберігається.

## Що створює notebook

| Об'єкт | Призначення |
|---|---|
| `nyc_taxi_raw` | сирі поїздки (лише для реальних даних); локальний час NYC як `TIMESTAMP` |
| `t7_hw_taxi_daily` | PK `(pickup_zone_id, ts_day)`, 12 найзавантаженіших зон (щільні ряди без пропусків днів) |
| `v_t7_hw_taxi_features` | усі ознаки, `PARTITION BY pickup_zone_id` у кожному вікні |
| `t7_hw_events` | JSONB event-log |
| `idx_t7_events_payload_gin_pathops` | GIN (`jsonb_path_ops`) для containment |
| `idx_t7_events_device_btree` | expression B-tree по `(payload ->> 'device')` |
| `t7_hw_payments_norm` | нормалізована таблиця платежів |

## Ознаки

Ознаки орієнтовані на прогноз попиту, тож рахуються по `trip_count`:

- **Lag:** `trips_lag_1d/7d/30d` (обов'язкові назви) та `dist_lag_7d` (дистанція тиждень тому).
- **Rolling 7d:** середня кількість поїздок з `RANGE ... INTERVAL '6 days'` (календарні дні, стійко до пропусків) і середнє за попередні 7 днів без поточного (`trips_prev7_avg`, безпечне щодо leakage).
- **Волатильність 30d:** `STDDEV_SAMP(trip_count)` та коефіцієнт варіації попиту.
- **Expanding:** `cumulative_trips`, рекорд попиту `peak_trips` і `days_since_record`.

## Як запустити

1. Відкрийте `hw6_taxi_jsonb.ipynb` у Google Colab.
2. **Runtime → Restart session and run all** (приблизно кілька хвилин: завантаження трьох Parquet-файлів та COPY ~450 000 рядків).
