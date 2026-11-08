# weather-otel


Reads live observations from the Cumbuco OceanView weather station and exports
them as OpenTelemetry metrics over OTLP/HTTP to an existing collector.

- **Primary source:** `clientraw.txt` (Weather Display text feed, refreshed about
  once a minute).
- **Fallback source:** fixed-layout OCR of `index.gif` when the text feed is
  unavailable. No CGO, no Tesseract, no external binaries.
- **Transport:** OTLP/HTTP, configured entirely through the standard
  `OTEL_EXPORTER_OTLP_*` environment variables.

## How it works

```
scrape loop (60s) → clientraw provider ──┐
                                        ├─→ Reading snapshot (atomic.Pointer)
                  GIF OCR provider ─────┘            │
                                                     ▼
                                  OTel observable gauges → periodic reader
                                                     │
                                                     ▼
                                       OTLP/HTTP → your collector
```

The last successful reading is retained when a scrape fails, so dashboards keep
the previous values while `weather.station.up` drops to `0` and
`weather.station.data_age` keeps rising. Every data point carries a `source`
attribute (`clientraw` or `gif`).

## Quick start

```bash
go build ./...
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
  go run ./cmd/weather-otel
```

To verify the fallback path, point the primary URL at something that 404s:

```bash
go run ./cmd/weather-otel \
  -url https://www.cumbuco-oceanview.com/weather/does-not-exist.txt
```

To compare the text feed with the OCR fallback continuously, add
`-verify-fallback`; the absolute difference per field is exported as
`weather.fallback.delta`.

## Docker Compose

[`docker-compose.yaml`](docker-compose.yaml) starts the exporter together with a
collector that uses the debug exporter:

```bash
docker-compose up --build
docker-compose logs -f otel-collector
```

The compose configuration configures `dns: 10.43.0.10` so containers can resolve
in-cluster service domains. Every setting from the table below can be overridden
through the environment:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://observability-daemon-collector-headless.observability.svc.cluster.local:4318 \
OTEL_EXPORTER_OTLP_INSECURE=true \
docker-compose up -d weather-otel
```

If you already operate a collector, remove the `otel-collector` service and keep
only `weather-otel` with `OTEL_EXPORTER_OTLP_ENDPOINT` pointing at it.

## Configuration

Flags take precedence over environment variables.

| Flag | Environment variable | Default | Meaning |
|---|---|---|---|
| `-url` | `WEATHER_CLIENTRAW_URL` | `https://www.cumbuco-oceanview.com/weather/clientraw.txt` | primary text feed |
| `-fallback-url` | `WEATHER_GIF_URLS` | `https://www.cumbuco-oceanview.com/weather/index.gif` | comma-separated GIF URLs |
| `-gif-fallback` | `WEATHER_GIF_FALLBACK` | `true` | enable the OCR fallback |
| `-verify-fallback` | `WEATHER_VERIFY_FALLBACK` | `false` | also scrape the GIF and report deltas |
| `-interval` | `WEATHER_INTERVAL` | `1m` | scrape interval (minimum 10s) |
| `-export-interval` | `WEATHER_EXPORT_INTERVAL` | `30s` | OTLP export interval (minimum 5s) |
| `-timeout` | `WEATHER_HTTP_TIMEOUT` | `10s` | per-request HTTP timeout |
| `-max-body-bytes` | `WEATHER_MAX_BODY_BYTES` | `2097152` | maximum accepted response size |
| `-station` | `WEATHER_STATION_NAME` | `Cumbuco` | station name resource attribute |
| `-timezone` | `WEATHER_STATION_TIMEZONE` | `America/Fortaleza` | IANA zone of the station clock |
| `-log-level` | `WEATHER_LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error` |

Collector connection is handled by the OpenTelemetry SDK:

| Variable | Default | Meaning |
|---|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4318` | collector OTLP/HTTP endpoint |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | (same as above) | metrics-specific endpoint |
| `OTEL_EXPORTER_OTLP_HEADERS` | – | e.g. `Authorization=Bearer …` |
| `OTEL_EXPORTER_OTLP_COMPRESSION` | `none` | `gzip` to compress |
| `OTEL_EXPORTER_OTLP_INSECURE` | `false` | plaintext for local collectors |
| `OTEL_SERVICE_NAME` | `cumbuco-weather-otel` | `service.name` resource attribute |

## Metric catalog

Wind speeds are in knots. Everything else uses SI units and UCUM codes.

| Metric | Unit | Attributes |
|---|---|---|
| `weather.wind.speed` | `[kn_i]` | `source`, `aggregation=current\|average_1m` |
| `weather.wind.gust` | `[kn_i]` | `source`, `window=max_today\|max_last_hour` |
| `weather.wind.direction` | `deg` | `source`, `aggregation=current\|average` |
| `weather.temperature` | `Cel` | `source`, `aggregation=current\|max_today\|min_today`, `sensor=indoor` |
| `weather.temperature.feels_like` | `Cel` | `source`, `kind=dew_point\|heat_index\|wind_chill\|humidex\|apparent` |
| `weather.humidity` | `%` | `source`, `sensor=outdoor\|indoor` |
| `weather.pressure` | `hPa` | `source`, `aggregation=current\|max_today\|min_today` |
| `weather.pressure.trend` | `hPa/h` | `source` |
| `weather.precipitation` | `mm` | `source`, `period=today\|yesterday\|month\|year` |
| `weather.precipitation.rate` | `mm/h` | `source`, `aggregation=current\|max` |
| `weather.solar.radiation` | `W/m2` | `source` |
| `weather.uv.index` | `1` | `source` |
| `weather.station.data_age` | `s` | `source` |
| `weather.station.up` | `1` | `source` |
| `weather.fetch.duration` | `s` | `source` |
| `weather.fetch.errors` | `1` | `reason` (counter) |
| `weather.fallback.delta` | `1` | `field`, `unit` (only with `-verify-fallback`) |

Unavailable sensors (Weather Display sentinels such as `-100`, `255`, `-50`) are
omitted rather than exported. The GIF fallback only reports the fields the
rendered panel contains, so UV, solar radiation, wind chill, humidex and apparent
temperature are absent while it is active.

## Fallback OCR

`internal/gifocr` recognises the station's fixed bitmap font. Only cyan value
text is read; white labels and headers are ignored. The glyph atlas
(`internal/gifocr/glyphs.json`) is embedded in the binary and was trained from
`internal/gifocr/testdata/reference.gif` plus an expected-value manifest.

Regenerate it after a layout or font change:

```bash
go run ./cmd/atlasgen
```

`atlasgen` verifies that every region in the manifest round-trips to its expected
value and fails if a region is misaligned, which prevents unit letters from
being learned as digits. If the station changes the image size or the panel
layout, recognition fails closed and the exporter keeps the last good reading.

## Collector example

A collector configuration with debug logging and in-cluster forwarding to the
daemon collector is provided in [`examples/otelcol-config.yaml`](examples/otelcol-config.yaml):

```bash
docker run --rm -p 4318:4318 --dns 10.43.0.10 \
  -v "$PWD/examples/otelcol-config.yaml:/etc/otelcol/config.yaml" \
  otel/opentelemetry-collector:latest \
  --config /etc/otelcol/config.yaml
```

Forwarding endpoint configured in the example:
- `http://observability-daemon-collector-headless.observability.svc.cluster.local:4318` (via `otlphttp` exporter)

## Grafana Dashboard

A complete Grafana dashboard matching all exported metrics is provided in:
- [`deploy/grafana/weather-dashboard.json`](deploy/grafana/weather-dashboard.json)
- [`examples/grafana/weather-dashboard.json`](examples/grafana/weather-dashboard.json)

Features:
- **Overview & Live Status**: Station up/down status, active scrape source (`clientraw` vs `gif`), station data age, current wind speed, compass direction, temperature, outdoor humidity, today's rainfall.
- **Wind Conditions**: 1-minute rolling average, current wind speed, max gust today, max gust last hour, and wind direction history.
- **Temperature & Comfort**: Outdoor temperature with daily min/max, indoor temperature, plus derived indices (dew point, heat index, wind chill, humidex, apparent temperature) and relative humidity.
- **Atmosphere & Precipitation**: Barometric pressure (current, min, max today), 3-hour pressure trend, accumulated rain (today, yesterday, month, year), and rain rate (current vs max today).
- **Solar & Radiation**: Solar radiation ($W/m^2$) and color-coded UV index scale.
- **Scraper Operations & Health**: Scrape duration, error rates by failure reason (`timeout`, `layout`, `parse`, etc.), and OCR verification deltas when `-verify-fallback` is active.

### Importing the Dashboard

#### Option A: Grafana UI (Instant)
1. Go to `https://eftech.grafana.net/dashboard/import`.
2. Click **Upload dashboard JSON file** and select `deploy/grafana/weather-dashboard.json`.
3. Select the Prometheus datasource (`grafanacloud-prom`) and click **Import**.

#### Option B: Automated Script / API
Run the import script with a token that has the `Editor` role or `dashboards:create` permission:
```bash
GRAFANA_TOKEN="<your-grafana-token>" ./deploy/grafana/import-dashboard.sh
```

Useful alerts:

- `weather.station.up == 0` for more than a few minutes.
- `weather.station.data_age > 600` (station clock is stale).
- `weather.fallback.delta` above the precision of the panel while
  `-verify-fallback` is enabled (OCR drift).

## Development

```bash
go test ./...          # unit and golden tests
go test -race ./...    # before handing off concurrent changes
go vet ./...
gofmt -l .
make build docker      # static binary and distroless image
```

Tests never contact the live station; HTTP behaviour is exercised with
`httptest.Server` and parser/OCR changes use fixtures in `testdata/`.
