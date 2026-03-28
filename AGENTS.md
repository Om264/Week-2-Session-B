# AGENTS.md — Rainfall Monitoring System
> **Purpose:** This file provides structured context for AI coding assistants (e.g., Claude, Copilot, Cursor) working on this codebase. Read this file in full before generating, modifying, or reviewing any code.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Directory Structure](#4-directory-structure)
5. [Key Functions](#5-key-functions)
6. [Data Flow](#6-data-flow)
7. [Data Formats](#7-data-formats)
8. [API Details](#8-api-details)
9. [Coding Guidelines](#9-coding-guidelines)
10. [Assumptions and Constraints](#10-assumptions-and-constraints)
11. [Future Extensions](#11-future-extensions)

---

## 1. Project Overview

### Purpose
The **Rainfall Monitoring System** is a Python-based web application that fetches real-time and forecast weather data from the OpenWeatherMap API, processes it using pandas, and presents actionable rainfall insights via an interactive Streamlit dashboard.

### Goals
- Provide real-time rainfall data for one or more geographic locations.
- Generate tiered alerts (INFO, WARNING, CRITICAL) based on configurable rainfall thresholds.
- Visualize historical and forecast rainfall trends using interactive charts.
- Enable non-technical users to monitor rainfall conditions without writing code.
- Be extensible to support additional weather metrics (wind, humidity, UV index) in the future.

### Intended Users
- Hydrologists and environmental scientists
- Municipal infrastructure and flood-management teams
- Agricultural operations monitoring irrigation and crop risk
- General-purpose weather monitoring for any location

### Non-Goals
- This system does **not** replace professional meteorological forecasting services.
- This system does **not** persist data to a production database (a local SQLite cache is used for rate-limit management only).
- This system is **not** a real-time streaming application; data is polled on a configurable interval.

---

## 2. System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User (Browser)                           │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HTTP (localhost)
┌─────────────────────────▼───────────────────────────────────────┐
│                   Streamlit UI Layer                             │
│   dashboard.py  ·  components/charts.py  ·  components/alerts.py│
└──────────┬──────────────────────────────────┬───────────────────┘
           │ calls                            │ calls
┌──────────▼──────────┐            ┌──────────▼──────────────────┐
│   Data Processing   │            │      Alert Engine            │
│   processor.py      │            │      alerts.py               │
└──────────┬──────────┘            └──────────┬───────────────────┘
           │ consumes                         │ consumes
┌──────────▼──────────────────────────────────▼───────────────────┐
│                     API Client Layer                             │
│                     api/client.py                                │
│              (handles requests, retries, caching)                │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS REST
┌──────────────────────────▼──────────────────────────────────────┐
│                  OpenWeatherMap API                              │
│    /data/2.5/weather  ·  /data/2.5/forecast  ·  /geo/1.0/direct │
└─────────────────────────────────────────────────────────────────┘
           │
┌──────────▼──────────┐
│  Local SQLite Cache │  (optional — stores last N API responses)
│  cache/weather.db   │
└─────────────────────┘
```

### Component Responsibilities

| Component | File(s) | Responsibility |
|---|---|---|
| **UI Layer** | `dashboard.py` | Renders Streamlit pages, manages session state, orchestrates calls |
| **Chart Components** | `components/charts.py` | Builds Plotly/Altair figures from processed DataFrames |
| **Alert Components** | `components/alerts.py` | Renders alert banners and notification badges in UI |
| **API Client** | `api/client.py` | Executes HTTP requests, handles retries, applies caching |
| **Data Processor** | `processor.py` | Transforms raw API JSON into structured pandas DataFrames |
| **Alert Engine** | `alerts.py` | Evaluates rainfall thresholds and generates alert objects |
| **Config** | `config.py` | Centralizes all environment variables and constants |
| **Cache** | `cache/db.py` | Manages SQLite read/write for API response caching |
| **Utils** | `utils/` | Shared helpers: logging, date formatting, unit conversion |

---

## 3. Tech Stack

| Technology | Version (min) | Role |
|---|---|---|
| **Python** | 3.10+ | Core runtime language |
| **Streamlit** | 1.32+ | Web UI framework; handles layout, widgets, and reactivity |
| **pandas** | 2.0+ | Data wrangling: reshaping API JSON, resampling time series, computing aggregates |
| **requests** | 2.31+ | HTTP client for OpenWeatherMap REST API calls |
| **plotly** | 5.20+ | Interactive charts (bar, line, area) embedded in Streamlit |
| **python-dotenv** | 1.0+ | Loads secrets and config from `.env` file at runtime |
| **SQLite3** | stdlib | Local cache database (no external DB dependency) |
| **pytest** | 7.4+ | Unit and integration test runner |
| **loguru** | 0.7+ | Structured logging with automatic rotation |
| **pydantic** | 2.0+ | Runtime data validation for API responses and config values |

### Why These Choices
- **Streamlit** was chosen over Flask/Django because this is a data-science dashboard, not a full web app. Streamlit's reactive model is a natural fit.
- **pandas** is used instead of raw Python dicts because rainfall data is inherently time-series and benefits from vectorized operations and resampling.
- **SQLite** is used for caching to avoid hammering the API rate limit (60 calls/min on the free tier) and to allow offline development.
- **Pydantic** is used to validate API responses so that unexpected schema changes fail loudly rather than silently corrupting downstream DataFrames.

---

## 4. Directory Structure

```
rainfall-monitor/
│
├── AGENTS.md                    # ← You are here. AI context file.
├── README.md                    # Human-facing project documentation
├── requirements.txt             # Pinned Python dependencies
├── .env.example                 # Template for environment variables (never commit .env)
├── .env                         # Local secrets — NEVER commit this file
├── .gitignore
│
├── dashboard.py                 # Main Streamlit entry point (run with `streamlit run dashboard.py`)
├── processor.py                 # Data transformation logic (API JSON → DataFrames)
├── alerts.py                    # Alert generation engine
├── config.py                    # Centralized config using pydantic-settings
│
├── api/
│   ├── __init__.py
│   ├── client.py                # OpenWeatherMap HTTP client (requests + retry logic)
│   ├── endpoints.py             # URL constants and endpoint builders
│   └── models.py                # Pydantic models for API request params and responses
│
├── components/
│   ├── __init__.py
│   ├── charts.py                # Plotly chart factory functions
│   ├── alerts.py                # Streamlit alert/badge rendering components
│   └── sidebar.py               # Sidebar widgets (location picker, threshold sliders)
│
├── cache/
│   ├── __init__.py
│   └── db.py                    # SQLite cache: read, write, expire
│
├── utils/
│   ├── __init__.py
│   ├── logger.py                # Loguru setup and log formatter
│   ├── converters.py            # Unit conversion: mm→inches, Kelvin→Celsius/Fahrenheit
│   └── time_utils.py            # UTC↔local timezone conversion, ISO 8601 helpers
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py              # Shared pytest fixtures (mock API responses, sample DataFrames)
│   ├── test_processor.py        # Unit tests for processor.py
│   ├── test_alerts.py           # Unit tests for alerts.py
│   ├── test_client.py           # Integration tests for API client (uses VCR cassettes or mocks)
│   └── fixtures/
│       ├── sample_current.json  # Fixture: sample /weather API response
│       └── sample_forecast.json # Fixture: sample /forecast API response
│
└── assets/
    └── logo.png                 # Optional branding asset for dashboard header
```

---

## 5. Key Functions

### 5.1 `api/client.py`

#### `get_current_weather(city: str, units: str = "metric") -> dict`
- **Purpose:** Fetches real-time weather data for a given city.
- **Parameters:**
  - `city` — City name string (e.g., `"London"`) or `"lat,lon"` format.
  - `units` — Unit system: `"metric"` (mm, °C), `"imperial"` (inches, °F), `"standard"` (Kelvin).
- **Returns:** Raw parsed JSON dict from the `/data/2.5/weather` endpoint.
- **Side Effects:** Writes response to SQLite cache with a TTL of 10 minutes.
- **Raises:** `APIConnectionError`, `APIRateLimitError`, `CityNotFoundError`.

#### `get_forecast(city: str, cnt: int = 40, units: str = "metric") -> dict`
- **Purpose:** Fetches a 5-day / 3-hour interval forecast.
- **Parameters:**
  - `city` — Same as above.
  - `cnt` — Number of time steps to return (max 40, each step = 3 hours → 5 days total).
  - `units` — Same as above.
- **Returns:** Raw parsed JSON dict from the `/data/2.5/forecast` endpoint.
- **Side Effects:** Writes to cache with a TTL of 30 minutes.
- **Raises:** Same as above.

#### `geocode_location(query: str) -> tuple[float, float]`
- **Purpose:** Converts a free-text location query to `(lat, lon)` coordinates.
- **Uses:** `/geo/1.0/direct` endpoint.
- **Returns:** `(latitude, longitude)` tuple. Returns the first result only.
- **Raises:** `LocationNotFoundError` if no results are returned.

---

### 5.2 `processor.py`

#### `parse_current_rainfall(data: dict) -> pd.Series`
- **Purpose:** Extracts rainfall metrics from a `/weather` response dict.
- **Returns:** A `pd.Series` with index `["rain_1h", "rain_3h", "timestamp", "city", "humidity", "description"]`.
- **Notes:** If `rain` key is absent (no rain reported), fills `rain_1h` and `rain_3h` with `0.0`. Never raises on missing rain data.

#### `parse_forecast_to_dataframe(data: dict) -> pd.DataFrame`
- **Purpose:** Transforms a `/forecast` response into a tidy time-series DataFrame.
- **Returns:** DataFrame with columns:
  ```
  timestamp (datetime64[UTC]), city (str), rain_3h (float64),
  temp (float64), humidity (int64), description (str), icon (str)
  ```
- **Notes:** Timestamps are always stored as UTC. Conversion to local time happens in the UI layer via `utils/time_utils.py`.

#### `resample_rainfall(df: pd.DataFrame, freq: str = "D") -> pd.DataFrame`
- **Purpose:** Resamples 3-hourly rainfall into daily (or other frequency) totals.
- **Parameters:**
  - `df` — Output of `parse_forecast_to_dataframe`.
  - `freq` — Pandas offset alias: `"D"` (daily), `"6H"` (6-hourly), `"W"` (weekly).
- **Returns:** DataFrame with `timestamp` as index and `rain_total` (sum), `rain_max` (max 3h reading) as columns.

#### `compute_rolling_average(df: pd.DataFrame, window: int = 3) -> pd.DataFrame`
- **Purpose:** Adds a `rain_rolling_avg` column — the rolling mean of `rain_3h` over `window` periods.
- **Returns:** Input DataFrame with new `rain_rolling_avg` column appended.

---

### 5.3 `alerts.py`

#### `evaluate_alerts(series: pd.Series, thresholds: dict) -> list[Alert]`
- **Purpose:** Compares current rainfall readings against configured thresholds and returns a list of `Alert` objects.
- **Parameters:**
  - `series` — Output of `parse_current_rainfall`.
  - `thresholds` — Dict with keys `"info"`, `"warning"`, `"critical"` mapping to mm values (e.g., `{"info": 2.5, "warning": 10.0, "critical": 25.0}`).
- **Returns:** List of `Alert` dataclass instances (may be empty if no thresholds are breached).

#### `Alert` dataclass
```python
@dataclass
class Alert:
    level: Literal["INFO", "WARNING", "CRITICAL"]
    message: str
    value: float          # The rainfall reading that triggered the alert (mm)
    threshold: float      # The threshold that was breached
    timestamp: datetime   # UTC time of the alert
    city: str
```

#### `generate_forecast_alerts(df: pd.DataFrame, thresholds: dict) -> list[Alert]`
- **Purpose:** Scans the forecast DataFrame for any future time steps that are projected to exceed thresholds.
- **Returns:** List of `Alert` objects, each timestamped to the projected breach window.

---

### 5.4 `dashboard.py`

#### `main()`
- **Purpose:** Streamlit entry point. Orchestrates sidebar inputs, data fetching, processing, and rendering.
- **Session State Keys Used:**
  - `st.session_state.location` — Currently selected city string.
  - `st.session_state.units` — `"metric"` or `"imperial"`.
  - `st.session_state.thresholds` — Dict of alert thresholds from sidebar sliders.
  - `st.session_state.last_refresh` — UTC datetime of last successful API fetch.

#### `refresh_data(location: str, units: str) -> tuple[pd.Series, pd.DataFrame]`
- **Purpose:** Coordinates calls to `api/client.py` and `processor.py`. Handles errors gracefully and surfaces them as Streamlit warnings rather than crashing.
- **Returns:** `(current_series, forecast_df)` tuple ready for the UI layer.

---

### 5.5 `components/charts.py`

#### `build_forecast_bar_chart(df: pd.DataFrame, units: str) -> plotly.graph_objects.Figure`
- **Purpose:** Renders a grouped bar chart of rainfall per forecast period, color-coded by severity.
- **X-axis:** Timestamp (formatted to local timezone).
- **Y-axis:** Rainfall in mm or inches depending on `units`.
- **Color:** Green → Yellow → Orange → Red based on threshold proximity.

#### `build_daily_totals_chart(df: pd.DataFrame) -> plotly.graph_objects.Figure`
- **Purpose:** Renders an area chart of daily rainfall totals resampled via `resample_rainfall`.

---

### 5.6 `cache/db.py`

#### `get_cached_response(cache_key: str) -> dict | None`
- **Purpose:** Returns a cached API response if it exists and has not expired. Returns `None` on miss or expiry.

#### `set_cached_response(cache_key: str, data: dict, ttl_seconds: int) -> None`
- **Purpose:** Serializes `data` as JSON and writes it to the SQLite cache table with an expiry timestamp.

---

## 6. Data Flow

```
User Input (City / Location)
        │
        ▼
[sidebar.py] — Collects city name, unit preference, threshold sliders
        │
        ▼
[api/client.py: geocode_location()] — Resolves city → (lat, lon)
        │
        ├──► [cache/db.py: get_cached_response()] ──► Cache HIT → return cached dict
        │                                                              │
        │    Cache MISS ──────────────────────────────────────────────┘
        │
        ▼
[api/client.py: get_current_weather() + get_forecast()]
        │ — HTTPS GET to OpenWeatherMap
        │ — Validates response with Pydantic models (api/models.py)
        │ — Writes to cache on success
        │
        ▼
[processor.py: parse_current_rainfall()] ──► pd.Series (current conditions)
[processor.py: parse_forecast_to_dataframe()] ──► pd.DataFrame (5-day forecast)
        │
        ├──► [alerts.py: evaluate_alerts()] ──► list[Alert] (current)
        ├──► [alerts.py: generate_forecast_alerts()] ──► list[Alert] (forecast)
        │
        ▼
[processor.py: resample_rainfall() + compute_rolling_average()]
        │ (enriches the forecast DataFrame with daily totals and rolling averages)
        │
        ▼
[dashboard.py] — Distributes data to rendering components:
        ├──► [components/alerts.py] — Renders alert banners
        ├──► [components/charts.py: build_forecast_bar_chart()] — Plotly bar chart
        └──► [components/charts.py: build_daily_totals_chart()] — Plotly area chart
```

---

## 7. Data Formats

### 7.1 OpenWeatherMap `/weather` Response (Current)
```json
{
  "coord": { "lon": -0.1257, "lat": 51.5085 },
  "weather": [
    { "id": 501, "main": "Rain", "description": "moderate rain", "icon": "10d" }
  ],
  "base": "stations",
  "main": {
    "temp": 12.4,
    "feels_like": 10.8,
    "temp_min": 11.2,
    "temp_max": 13.5,
    "pressure": 1012,
    "humidity": 87
  },
  "visibility": 9000,
  "wind": { "speed": 5.1, "deg": 220 },
  "rain": {
    "1h": 3.2,
    "3h": 6.5
  },
  "clouds": { "all": 90 },
  "dt": 1711612800,
  "sys": {
    "type": 2, "id": 2019646, "country": "GB",
    "sunrise": 1711595200, "sunset": 1711641600
  },
  "timezone": 3600,
  "id": 2643743,
  "name": "London",
  "cod": 200
}
```
**Key fields used by this application:**
- `rain.1h` → Rainfall in the last 1 hour (mm). May be absent if no rain.
- `rain.3h` → Rainfall in the last 3 hours (mm). May be absent if no rain.
- `main.humidity` → Relative humidity (%).
- `dt` → Unix timestamp (UTC) of the observation.
- `name` → City name string.
- `weather[0].description` → Human-readable condition string.

---

### 7.2 OpenWeatherMap `/forecast` Response (5-Day, 3-Hour)
```json
{
  "cod": "200",
  "message": 0,
  "cnt": 40,
  "list": [
    {
      "dt": 1711620000,
      "main": {
        "temp": 11.9,
        "feels_like": 10.1,
        "temp_min": 11.5,
        "temp_max": 11.9,
        "pressure": 1011,
        "humidity": 90
      },
      "weather": [
        { "id": 500, "main": "Rain", "description": "light rain", "icon": "10d" }
      ],
      "clouds": { "all": 95 },
      "wind": { "speed": 4.8, "deg": 215, "gust": 7.2 },
      "visibility": 10000,
      "pop": 0.85,
      "rain": { "3h": 2.1 },
      "sys": { "pod": "d" },
      "dt_txt": "2024-03-28 12:00:00"
    }
  ],
  "city": {
    "id": 2643743,
    "name": "London",
    "coord": { "lat": 51.5085, "lon": -0.1257 },
    "country": "GB",
    "population": 1000000,
    "timezone": 3600,
    "sunrise": 1711595200,
    "sunset": 1711641600
  }
}
```
**Key fields per list item:**
- `dt` → Unix timestamp (UTC) for the forecast window start.
- `rain.3h` → Forecasted rainfall for the 3-hour window (mm). Absent if no rain expected.
- `pop` → Probability of precipitation (0.0 – 1.0). Used as a confidence indicator.
- `main.humidity` → Forecast humidity (%).
- `dt_txt` → ISO 8601 string (UTC). Used as a human-readable label fallback.

---

### 7.3 Internal `Alert` Object (Python Dataclass)
```json
{
  "level": "WARNING",
  "message": "Rainfall of 11.3 mm/h exceeds WARNING threshold of 10.0 mm/h.",
  "value": 11.3,
  "threshold": 10.0,
  "timestamp": "2024-03-28T14:00:00+00:00",
  "city": "London"
}
```

---

### 7.4 Processed Forecast DataFrame Schema
```
Column          dtype           Description
─────────────── ─────────────── ──────────────────────────────────────────
timestamp       datetime64[UTC] Start of the 3-hour forecast window (UTC)
city            object (str)    City name from API response
rain_3h         float64         Rainfall for the 3h window (mm). 0.0 if absent.
temp            float64         Temperature in selected unit system
humidity        int64           Relative humidity (%)
description     object (str)    Weather condition description
icon            object (str)    OpenWeatherMap icon code (e.g., "10d")
pop             float64         Probability of precipitation (0.0 – 1.0)
rain_rolling_avg float64        Rolling mean of rain_3h over N periods (added by processor)
```

---

## 8. API Details

### 8.1 Base URL
```
https://api.openweathermap.org
```

### 8.2 Authentication
All requests require the `appid` query parameter. The API key is loaded from the `.env` file via `config.py`. **Never hardcode the API key.**

```python
# .env
OPENWEATHER_API_KEY=your_api_key_here
```

### 8.3 Endpoints

#### Current Weather
```
GET /data/2.5/weather
```
| Parameter | Type | Required | Description |
|---|---|---|---|
| `q` | string | Yes (or lat/lon) | City name. E.g., `London,GB` |
| `lat` | float | Yes (or q) | Latitude |
| `lon` | float | Yes (or q) | Longitude |
| `appid` | string | Yes | API key |
| `units` | string | No | `metric`, `imperial`, `standard` (default) |
| `lang` | string | No | Language for description field. Default `en`. |

**Sample Request:**
```
GET https://api.openweathermap.org/data/2.5/weather?q=London,GB&units=metric&appid=YOUR_KEY
```

#### 5-Day Forecast
```
GET /data/2.5/forecast
```
| Parameter | Type | Required | Description |
|---|---|---|---|
| `q` | string | Yes (or lat/lon) | City name |
| `lat` | float | Yes (or q) | Latitude |
| `lon` | float | Yes (or q) | Longitude |
| `cnt` | int | No | Number of 3h steps to return. Max 40. |
| `appid` | string | Yes | API key |
| `units` | string | No | `metric`, `imperial`, `standard` |

**Sample Request:**
```
GET https://api.openweathermap.org/data/2.5/forecast?q=London,GB&cnt=40&units=metric&appid=YOUR_KEY
```

#### Geocoding
```
GET /geo/1.0/direct
```
| Parameter | Type | Required | Description |
|---|---|---|---|
| `q` | string | Yes | Location name (city, state, country) |
| `limit` | int | No | Max results. Default 5. Use `1` for this app. |
| `appid` | string | Yes | API key |

**Sample Response:**
```json
[
  {
    "name": "London",
    "local_names": { "en": "London" },
    "lat": 51.5085,
    "lon": -0.1257,
    "country": "GB",
    "state": "England"
  }
]
```

### 8.4 Rate Limits (Free Tier)
| Limit | Value |
|---|---|
| Calls per minute | 60 |
| Calls per day | 1,000 |
| Forecast depth | 5 days |
| Historical data | Not available on free tier |

### 8.5 Error Codes
| HTTP Code | Meaning | Handling |
|---|---|---|
| `200` | Success | Process normally |
| `401` | Invalid API key | Raise `APIAuthError`; surface to UI |
| `404` | City not found | Raise `CityNotFoundError`; prompt user to revise input |
| `429` | Rate limit exceeded | Raise `APIRateLimitError`; serve from cache if available |
| `5xx` | Server error | Retry up to 3 times with exponential backoff; log error |

---

## 9. Coding Guidelines

### 9.1 Naming Conventions
- **Files:** `snake_case.py`
- **Classes:** `PascalCase`
- **Functions:** `snake_case`
- **Constants:** `UPPER_SNAKE_CASE` (defined in `config.py`)
- **DataFrame columns:** `snake_case` (e.g., `rain_3h`, not `Rain3H` or `rain3h`)
- **Pydantic models:** `PascalCase` with a `Model` suffix where ambiguous (e.g., `ForecastResponseModel`)

### 9.2 Type Hints
All functions **must** include type hints for parameters and return values. Use `from __future__ import annotations` for forward references. Use `typing` or built-in generics (`list[str]`, `dict[str, float]`).

```python
# GOOD
def parse_current_rainfall(data: dict) -> pd.Series:
    ...

# BAD
def parse_current_rainfall(data):
    ...
```

### 9.3 Error Handling
- Define custom exceptions in `api/client.py`:
  ```python
  class APIConnectionError(Exception): ...
  class APIRateLimitError(Exception): ...
  class CityNotFoundError(Exception): ...
  class APIAuthError(Exception): ...
  ```
- Never let raw `requests.exceptions` propagate to the UI layer — catch and wrap them.
- In `dashboard.py`, always use `st.warning()` or `st.error()` to surface errors. Never call `st.stop()` unless data is completely unrecoverable.
- Log all caught exceptions with `logger.exception()` (includes stack trace automatically via Loguru).

### 9.4 API Response Validation
- Every raw API response dict must be validated through the corresponding Pydantic model before being passed to `processor.py`.
- If validation fails (unexpected schema), log the raw payload at `DEBUG` level and raise a descriptive `ValueError`.

### 9.5 DataFrame Conventions
- Always set `timestamp` as a `DatetimeTZDtype` column in UTC. Never store naive datetimes.
- Never mutate a DataFrame in-place inside a function — return a new DataFrame or explicit copy.
- Use `.assign()` over `df[col] = ...` to maintain immutability within pipelines.
- After any `resample()` or `groupby()`, always `.reset_index()` to avoid confusing multi-index bugs.

### 9.6 Streamlit Session State
- All stateful values must be initialized in a single `_init_session_state()` function called at the top of `main()`.
- Never write to `st.session_state` from within a component function. Components should be pure rendering functions that accept data as arguments.
- Use `st.cache_data` with appropriate `ttl` for functions that fetch or process data.

### 9.7 Configuration Management
- All constants, thresholds, and API parameters are defined in `config.py` and loaded via `pydantic-settings`.
- No magic numbers in business logic. Reference named constants from `config.py`.
```python
# BAD
if rain_1h > 25.0:
    ...

# GOOD
from config import settings
if rain_1h > settings.ALERT_THRESHOLD_CRITICAL:
    ...
```

### 9.8 Testing Requirements
- Every public function in `processor.py` and `alerts.py` must have a corresponding unit test.
- API calls in tests must be mocked using `unittest.mock.patch` or `pytest-mock`. Never make real HTTP calls in tests.
- Fixtures (sample JSON responses) must be stored in `tests/fixtures/` and loaded via `conftest.py`.
- Minimum test coverage target: **80%** across `processor.py`, `alerts.py`, and `api/client.py`.

### 9.9 Logging
- Use `loguru` exclusively. Do not use `print()` statements in production code.
- Log levels:
  - `DEBUG` — Raw API payloads, cache hits/misses.
  - `INFO` — Successful fetches, user location changes, alert triggers.
  - `WARNING` — Degraded state: serving from cache due to API failure.
  - `ERROR` / `CRITICAL` — Unrecoverable failures.

### 9.10 Security
- `.env` is listed in `.gitignore`. The `OPENWEATHER_API_KEY` must never appear in source code or logs.
- Sanitize all user-supplied location strings before embedding in API query parameters (strip special characters, limit length to 100 chars).

---

## 10. Assumptions and Constraints

### Assumptions
1. The application targets the **free tier** of OpenWeatherMap. Paid-tier features (historical data, minute-by-minute forecast, air quality) are not used.
2. Users have a stable internet connection. Offline mode relies only on the SQLite cache (no guarantees of data freshness).
3. A single deployment serves one user or a small team (not a multi-tenant SaaS product).
4. Rainfall data is measured in **millimeters (mm)** internally. Unit conversion to inches is performed only at the display layer.
5. The application is deployed locally via `streamlit run` or on a single-server environment (not Kubernetes or serverless).
6. Python 3.10+ is available on the host machine.

### Constraints
1. **API Rate Limit:** 60 requests/minute on the free tier. The SQLite cache is mandatory to prevent exceeding this limit during development and in production with frequent page refreshes.
2. **Forecast Horizon:** Maximum 5 days ahead, in 3-hour intervals. No sub-hourly forecasts are available.
3. **No WebSocket/Push:** Streamlit does not natively support push notifications. Alerts are evaluated on each data refresh (manual or auto-refresh via `st.rerun()`).
4. **Single Location per Session:** The current architecture processes one city at a time. Multi-location comparison requires a refactor of the data layer (see Section 11).
5. **No Authentication Layer:** The Streamlit app does not implement user authentication. If deployed publicly, it must sit behind a reverse proxy with basic auth (e.g., Nginx).

---

## 11. Future Extensions

The following extensions are planned or recommended. When implementing any of these, update this `AGENTS.md` accordingly.

### 11.1 Multi-Location Comparison
- Refactor `processor.py` to accept a list of cities and return a dictionary of DataFrames keyed by city name.
- Add a `st.tabs()` layout in `dashboard.py` to display per-city panels.

### 11.2 PostgreSQL Persistence
- Replace the SQLite cache with a full PostgreSQL database using `psycopg2` or `SQLAlchemy`.
- Store processed DataFrames as time-series records to support historical trend analysis beyond the 5-day API window.
- Consider `TimescaleDB` extension for efficient time-series queries.

### 11.3 Email / SMS Alert Notifications
- Integrate `smtplib` (email) or `twilio` (SMS) to push CRITICAL alerts to configured recipients.
- Add a `notifications.py` module with a `NotificationDispatcher` class.
- Store recipient configuration in `.env`.

### 11.4 Expanded Weather Metrics
- Add processing for `wind.speed`, `main.pressure`, and `uvi` (UV index) fields from the API.
- Extend `processor.py` to output these columns in the forecast DataFrame.
- Add new chart components in `components/charts.py`.

### 11.5 Containerization
- Add a `Dockerfile` and `docker-compose.yml` for reproducible deployments.
- The SQLite database volume should be mounted externally to persist cache across container restarts.

### 11.6 Scheduled Auto-Refresh
- Implement a background scheduler using `APScheduler` to refresh data at a configurable interval without requiring user interaction.
- Store the latest data in `st.session_state` and trigger `st.rerun()` on a timer.

### 11.7 Unit Test Coverage Expansion
- Add integration tests using `responses` or `vcrpy` to replay real API HTTP interactions.
- Add end-to-end Streamlit UI tests using `streamlit-testing` or Playwright.

---

*This file is maintained as living documentation. Update it whenever the architecture, data formats, or key functions change. AI coding assistants must treat this file as the single source of truth for context about this codebase.*

---

**Last Updated:** 2024-03-28  
**Maintained By:** Project Engineering Team  
**AI Context Version:** 1.0.0
