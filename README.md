# doc-wire-mock

Dockerized WireMock quick-start project. Two independent WireMock setups live side by side under `Docker_Compose/`:

- **TrafficMocker** — stubs a fictional "restaurant_menu" REST API, for local/offline testing against canned responses instead of a real backend.
- **TrafficRecorder** — WireMock as a recording proxy: forwards requests to a real upstream API and records the request/response pairs as reusable mapping + body files.

See [AGENTS.md](AGENTS.md) for full endpoint reference and stub-authoring conventions.

## Prerequisites

- Docker + Docker Compose
- `curl` (or any HTTP client) to exercise the stubs

## Project Structure

```
doc-wire-mock/
└── Docker_Compose/
    ├── TrafficMocker/
    │   ├── docker-compose.yml
    │   └── wm_data/
    │       ├── mappings/       # request matcher -> response
    │       └── __files/        # response bodies
    └── TrafficRecorder/
        ├── docker-compose.yml
        └── wm_recordings/
            ├── mappings/       # auto-generated while recording
            └── __files/
```

## Quick Start — TrafficMocker (stub server)

```bash
docker compose -f Docker_Compose/TrafficMocker/docker-compose.yml up -d
```

WireMock is now listening on `http://localhost:8080` as container `restaurant_menu`.

Try it out:

```bash
# Full menu list
curl http://localhost:8080/api/menu

# Available categories
curl http://localhost:8080/api/menu/categories

# Single item by id (id is echoed back into the response)
curl http://localhost:8080/api/menu/1

# Filtered menu (only this exact category+availability combo is stubbed)
curl "http://localhost:8080/api/menu?category=mains&available=true"

# Not-found case: any numeric id ending in 404
curl http://localhost:8080/api/menu/404

# Create a menu item (echoes the posted fields back)
curl -X POST http://localhost:8080/api/menu \
  -H "Content-Type: application/json" \
  -d '{"name":"Lemonade","category":"drinks","price":4.5,"currency":"USD"}'
```

Stop it with:

```bash
docker compose -f Docker_Compose/TrafficMocker/docker-compose.yml down
```

## Quick Start — TrafficRecorder (recording proxy)

Edit `Docker_Compose/TrafficRecorder/docker-compose.yml` and point `--proxy-all` at the real API you want to capture traffic from, then:

```bash
docker compose -f Docker_Compose/TrafficRecorder/docker-compose.yml up -d
```

Every request sent to `http://localhost:8080` is proxied to that upstream and the real response is recorded as a new mapping + body file under `wm_recordings/`. Copy the recorded files into `TrafficMocker/wm_data/` to turn captured traffic into replayable stubs.

```bash
docker compose -f Docker_Compose/TrafficRecorder/docker-compose.yml down
```

**Note:** both containers bind host port `8080`, so only run one at a time unless you change one of the port mappings.

## Adding or Changing Stubs

1. Add a mapping JSON under `TrafficMocker/wm_data/mappings/` describing the request matcher and response.
2. If the response needs a body, add a file under `TrafficMocker/wm_data/__files/` and reference it via `bodyFileName`.
3. Restart (or re-`up -d`) the container — WireMock loads mappings from disk on startup.

See [AGENTS.md](AGENTS.md#conventions-trafficmocker) for naming conventions, priority rules, and response-templating details.
