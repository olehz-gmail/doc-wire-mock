<!-- claude session id: 73c123aa-8d67-4e08-99d7-d72795fb0edd -->

# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, etc.) when working with this repository.

## Repository Structure

```
doc-wire-mock/                (project root)
└── Docker_Compose/
    ├── TrafficMocker/
    │   ├── docker-compose.yml       # WireMock service definition (stub mode)
    │   └── wm_data/
    │       ├── mappings/            # Stub mapping definitions (request matcher -> response)
    │       └── __files/             # Response body files referenced by mappings via bodyFileName
    └── TrafficRecorder/
        ├── docker-compose.yml       # WireMock service definition (proxy/record mode)
        └── wm_recordings/
            ├── mappings/            # Mappings WireMock auto-generates while recording
            └── __files/             # Response bodies WireMock auto-generates while recording
```

## Purpose

Two independent WireMock setups, each with its own `docker-compose.yml`:

- **TrafficMocker** — stubs a fictional "restaurant_menu" REST API for local/offline testing against a mock backend instead of a real service.
- **TrafficRecorder** — WireMock configured as a recording proxy: it forwards every request to a real upstream API and records the request/response pairs as mapping + body files, so real traffic can be captured once and replayed later (e.g. by copying the recorded output into TrafficMocker's `wm_data/`).

## Running

```bash
# Stub server
docker compose -f Docker_Compose/TrafficMocker/docker-compose.yml up -d
docker compose -f Docker_Compose/TrafficMocker/docker-compose.yml down

# Recording proxy
docker compose -f Docker_Compose/TrafficRecorder/docker-compose.yml up -d
docker compose -f Docker_Compose/TrafficRecorder/docker-compose.yml down
```

Both containers bind host port `8080`, so only run one at a time unless you change one of the port mappings.

### TrafficMocker

Container named `restaurant_menu`, runs `wiremock/wiremock:3.9.1` with `--global-response-templating` enabled, so Handlebars expressions in response bodies (`{{request.path.[n]}}`, `{{jsonPath request.body '$.field'}}`, etc.) are rendered on every response, including file-based ones — no per-mapping `transformers` needed.

### TrafficRecorder

Container named `traffic_recorder`, runs `wiremock/wiremock:3.9.1` with `--proxy-all=https://api.example.com --record-mappings --verbose`. Every request sent to `http://localhost:8080` is proxied through to that upstream URL, and the real response is saved as a new mapping (`wm_recordings/mappings/`) + body file (`wm_recordings/__files/`). Update the `--proxy-all` target to point at whichever real API you want to capture traffic from.

## Stubbed Endpoints (TrafficMocker)

| Method | Path | Mapping file | Notes |
|---|---|---|---|
| GET | `/api/menu` | `get-menu-list.json` | Returns full menu list |
| GET | `/api/menu?category=mains&available=true` | `get-menu-filtered.json` | Matches this exact query combo only (fixed-case stub, not a real filter engine); priority 1 so it beats the plain list stub |
| GET | `/api/menu/categories` | `get-menu-categories.json` | Returns category list |
| GET | `/api/menu/{id}` | `get-menu-item-by-id.json` | `id` is numeric; echoes requested id via `{{request.path.[2]}}`; priority 5 |
| GET | `/api/menu/{id ending in 404}` | `get-menu-item-not-found.json` | Matches any numeric id ending in `404` (e.g. `404`, `1404`); returns 404 with the requested id in the message; priority 1 (must beat the by-id pattern) |
| POST | `/api/menu` | `post-create-menu-item.json` | Requires `name`, `category`, `price`, `currency` in JSON body (`bodyPatterns`); echoes those fields back in the 201 response |

## Conventions (TrafficMocker)

- **Mapping/file naming**: mapping files are named `<method>-<action>.json` (e.g. `get-menu-list.json`); response bodies live separately under `__files/` and are wired up via `bodyFileName`, not inline `body`.
- **Priority**: lower `priority` number wins when multiple stubs could match the same request. Use it whenever a specific-case stub (e.g. an id ending in `404`) must take precedence over a general pattern stub (e.g. any numeric id).
- **Templating requires matching request data to exist**: any `{{jsonPath request.body '$.field'}}` reference assumes that field is present — pair it with a `bodyPatterns`/`matchesJsonPath` requirement on the request so mismatched requests fall through to WireMock's default 404 instead of a broken 201/200 render.
- **Adding a new stubbed endpoint**: add one mapping JSON under `wm_data/mappings/`, and if the response needs a body, add a corresponding file under `wm_data/__files/` referenced via `bodyFileName`.
- **Query-parameter filtering**: WireMock doesn't run real filter logic — each query combo you want to test needs its own stub with `queryParameters` matchers (see `get-menu-filtered.json`). Add one mapping + response file per combo you need, with a `priority` lower than the unfiltered list stub so it's chosen first when the query params match.
