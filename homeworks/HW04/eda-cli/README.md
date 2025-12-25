# S03 – eda_cli: мини-EDA для CSV

Небольшое CLI-приложение для базового анализа CSV-файлов.
Используется в рамках Семинара 03 курса «Инженерия ИИ».

## Требования

- Python 3.11+
- [uv](https://docs.astral.sh/uv/) установлен в систему
- (Опционально) uvicorn для запуска API: `pip install uvicorn[standard]`

## Инициализация проекта

В корне проекта (S03):

```bash
uv sync
```

Эта команда:

- создаст виртуальное окружение `.venv`;
- установит зависимости из `pyproject.toml`;
- установит сам проект `eda-cli` в окружение.

## Запуск CLI

### Краткий обзор

```bash
uv run eda-cli overview data/example.csv
```

Параметры:

- `--sep` – разделитель (по умолчанию `,`);
- `--encoding` – кодировка (по умолчанию `utf-8`).

### Полный EDA-отчёт

```bash
uv run eda-cli report data/example.csv --out-dir reports
```

В результате в каталоге `reports/` появятся:

- `report.md` – основной отчёт в Markdown;
- `summary.csv` – таблица по колонкам;
- `missing.csv` – пропуски по колонкам;
- `correlation.csv` – корреляционная матрица (если есть числовые признаки);
- `top_categories/*.csv` – top-k категорий по строковым признакам;
- `hist_*.png` – гистограммы числовых колонок;
- `missing_matrix.png` – визуализация пропусков;
- `correlation_heatmap.png` – тепловая карта корреляций.

## Тесты

```bash
uv run pytest -q
```

## Запуск API (FastAPI)

Проект предоставляет HTTP API (FastAPI). Ниже — явные примеры запуска сервера с указанием `uvicorn` и `eda_cli.api:app`.

### Рекомендуемый запуск через `uv` (использует окружение проекта)

```bash
uv run uvicorn eda_cli.api:app --reload --host 127.0.0.1 --port 8000
```

> Обратите внимание, что команда содержит точную строку `uvicorn eda_cli.api:app` — это модуль и объект приложения FastAPI.

### Прямой запуск (если `uvicorn` установлен в текущем окружении)

```bash
uvicorn eda_cli.api:app --reload --host 127.0.0.1 --port 8000
```

После запуска:

- API доступно по `http://127.0.0.1:8000`;
- Swagger UI доступен по `http://127.0.0.1:8000/docs`.

## Список эндпоинтов

1. `GET /health` — системный health-check
2. `POST /quality` — заглушка оценки качества по агрегированным признакам (JSON)
3. `POST /quality-from-csv` — оценка качества по реальному CSV-файлу (multipart/form-data)
4. `POST /quality-flags-from-csv` — возвращает полный набор флагов качества из CSV (multipart/form-data)

---

## 1) GET `/health`

- **Тег:** `system`
- **Описание:** Простейший health-check сервиса.
- **Ответ (200):**

```json
{
  "status": "ok",
  "service": "dataset-quality",
  "version": "0.2.0"
}
```

---

## 2) POST `/quality`

- **Тег:** `quality`
- **Описание:** Эндпоинт-заглушка — принимает агрегированные признаки датасета и возвращает эвристическую оценку качества (без чтения CSV).
- **Request Content-Type:** `application/json`
- **Request model:** `QualityRequest`

### `QualityRequest` (JSON)

| Поле | Тип | Описание |
|---|---:|---|
| `n_rows` | `int` (>=0) | Число строк в датасете |
| `n_cols` | `int` (>=0) | Число колонок |
| `max_missing_share` | `float` (0..1) | Максимальная доля пропусков среди колонок |
| `numeric_cols` | `int` (>=0) | Количество числовых колонок |
| `categorical_cols` | `int` (>=0) | Количество категориальных колонок |

### Пример запроса

```bash
curl -X POST "http://127.0.0.1:8000/quality" \
  -H "Content-Type: application/json" \
  -d '{
    "n_rows": 1500,
    "n_cols": 12,
    "max_missing_share": 0.02,
    "numeric_cols": 8,
    "categorical_cols": 4
  }'
```

### Response model: `QualityResponse`

| Поле | Тип | Описание |
|---|---:|---|
| `ok_for_model` | `bool` | True — датасет годен для обучения по эвристике |
| `quality_score` | `float` (0..1) | Интегральная оценка |
| `message` | `string` | Читаемое пояснение |
| `latency_ms` | `float` | Время обработки в мс |
| `flags` | `dict[str,bool] | null` | Булевые флаги (например `too_few_rows`) |
| `dataset_shape` | `dict` или `null` | `{"n_rows":..., "n_cols":...}` |

### Пример успешного ответа

```json
{
  "ok_for_model": true,
  "quality_score": 0.85,
  "message": "Данных достаточно, модель можно обучать (по текущим эвристикам).",
  "latency_ms": 3.2,
  "flags": {
    "too_few_rows": false,
    "too_many_columns": false,
    "too_many_missing": false,
    "no_numeric_columns": false,
    "no_categorical_columns": false
  },
  "dataset_shape": {"n_rows": 1500, "n_cols": 12}
}
```

---

## 3) POST `/quality-from-csv`

- **Тег:** `quality`
- **Описание:** Принимает CSV, запускает EDA-ядро (`summarize_dataset`, `missing_table`, `compute_quality_flags`) и возвращает `QualityResponse` с оценкой и булевыми флагами.
- **Request Content-Type:** `multipart/form-data` (поле `file` — файл CSV)
- **Поддерживаемые `Content-Type` загружаемого файла:** `text/csv`, `application/vnd.ms-excel`, `application/octet-stream`

### Валидация и ошибки

- Если `file.content_type` не в списке — `400 Bad Request` с `{"detail": "Ожидается CSV-файл (content-type text/csv)."}`
- Если `pandas.read_csv` выбросил исключение — `400 Bad Request` с `{"detail": "Не удалось прочитать CSV: <описание>"}`
- Если CSV пуст (пустой DataFrame) — `400 Bad Request` с `{"detail": "CSV-файл не содержит данных (пустой DataFrame)."}`

### Пример `curl`

```bash
curl -X POST "http://127.0.0.1:8000/quality-from-csv" \
  -H "accept: application/json" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@data/example.csv;type=text/csv"
```

### Успешный ответ — тот же `QualityResponse` (см. выше)

Примечание: в `flags` возвращаются только булевы значения из `compute_quality_flags`.

---

## 4) POST `/quality-flags-from-csv`

- **Тег:** `quality`
- **Описание:** Возвращает **полный** набор флагов и метрик, вычисленных `compute_quality_flags` (без отбора/упрощения). Полезно для отладки и подробного логирования.
- **Request Content-Type:** `multipart/form-data` (поле `file` — CSV)
- **Response model:** `FlagsResponse` — просто объект с полем `flags`, где хранится результат `compute_quality_flags`.

### Пример `curl`

```bash
curl -X POST "http://127.0.0.1:8000/quality-flags-from-csv" \
  -H "accept: application/json" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@data/example.csv;type=text/csv"
```

### Пример ответа (примерная структура)

```json
{
  "flags": {
    "quality_score": 0.62,
    "too_few_rows": true,
    "too_many_missing": false,
    "high_cardinality_cols": ["user_id", "session_id"],
    "numeric_cols": 5,
    "categorical_cols": 6,
    "missing_per_col": {
      "age": 0.12,
      "name": 0.0
    }
    // ...другие метрики/флаги, возвращаемые compute_quality_flags
  }
}
```

> Важно: точный набор ключей в `flags` зависит от реализации `compute_quality_flags` в `core.py`.