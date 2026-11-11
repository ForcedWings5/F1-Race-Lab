# Race Summary: design spec

- **Date:** 2026-11-11
- **Status:** awaiting review
- **Scope:** sub-project 1 of 3 (race summary). The race replay and the what-if simulator each get their own spec later.

## 1. Purpose

This project is a portfolio piece for software engineer applications and a way to learn by writing the code by hand. The author mainly uses Python, is still building fluency in it, and is new to React and TypeScript.

The finished project has three parts, built in order:

1. **Race summary (this spec):** load one Formula 1 race from a public data source, store it, serve it through an API, and show it on a deployed web page.
2. **Race replay:** an animated top-down recreation of the race.
3. **What-if simulator:** change an event in the race and see a modelled outcome.

Each part must be showable on its own.

### Success criteria for this spec

- One command loads a race from 2023 or later into Postgres, and refuses to load data that fails its checks.
- The API serves the race list, race details, results, laps and tyre stints, with generated documentation at `/docs`.
- A public web page shows the race summary for at least one race. Nothing in the design limits it to one.
- Tests and linters pass on every push in GitHub Actions.
- The author wrote the product code and can explain every part of it.

### Out of scope for this spec

- The replay and the simulator.
- Car location and telemetry data, pit stop durations, weather, race control messages and championship points.
- Lap time charts and driver comparison.
- Live timing, user accounts, and any endpoint that changes data.
- Async Python, caching headers, pagination, API versioning, a state management library, a chart library, a CSS framework, browser-driven end-to-end tests, a database migration tool, and a custom domain.

### One constraint carried forward

Later specs will add tables that hang off the same keys used here: a race is identified by `race_id`, and a driver within a race by `(race_id, driver_number)`. The replay will store car positions as distance along the track over time, so that real and simulated races can share one renderer. Nothing in this spec implements that; it only must not block it.

## 2. Architecture

```
OpenF1 API (internet)
   |  HTTPS, only when the pipeline is run by hand
Pipeline (Python command-line tool, runs on the author's Mac)
   |  writes
Database (Postgres)
   |  reads only
API (FastAPI, read-only JSON)
   |  HTTP
Web app (React + TypeScript)
```

Rules:

- Each piece talks only to its neighbour. The API and the web app never call OpenF1.
- The data source sits behind one interface in the pipeline, so a second source (FastF1, for races before 2023) can be added later without changing the database, API or web app.
- Everything lives in one repository.

### Repository layout

```
backend/
  pyproject.toml
  schema.sql
  src/f1app/
    config.py           settings read from environment variables
    db.py               database connections
    sources/
      base.py           the RaceSource interface
      openf1.py         the OpenF1 implementation
    pipeline/
      models.py         Pydantic models for cleaned data
      transform.py      raw JSON to models, including position per lap
      checks.py         data checks run before loading
      load.py           writes a race to Postgres
      cli.py            the command-line entry point
    api/
      main.py           app setup, cross-origin settings, error handling
      routes.py         HTTP only
      queries.py        SQL only
      schemas.py        response shapes
  tests/
    fixtures/           small saved OpenF1 responses
  data/raw/             raw responses from real runs (not in git)
web/
  src/
docs/
  specs/                design specs (in git)
  plans/                working plans (not in git)
.github/workflows/ci.yml
```

`f1app` is the working name of the Python package.

### Tools

- **Backend:** Python 3.12 or newer, `httpx`, Pydantic 2, FastAPI, `psycopg` 3 with its connection pool, pytest, `ruff`, `mypy`. Dependencies are declared in `pyproject.toml` and pinned with `pip-tools`.
- **Database:** Postgres, installed locally with Postgres.app or Homebrew.
- **Web:** React, TypeScript, Vite, React Router, Vitest, React Testing Library, ESLint, Prettier.

## 3. Data source

OpenF1 is the source for this spec. These facts were read from openf1.org on 2026-11-11.

- **Base URL:** `https://api.openf1.org/v1`
- **Coverage:** sessions from the 2023 season onward. Past sessions are free and need no key or signup.
- **Rate limit (free):** 3 requests per second and 30 per minute.
- **Licence:** CC BY-NC-SA 4.0, for non-commercial use. OpenF1 is unofficial and community-run.
- **Filtering:** query parameters on any attribute, for example `?session_key=9158&driver_number=1`.

Endpoints used:

| Endpoint | Used for | Fields used |
|---|---|---|
| `sessions` | Finding the race | `session_key`, `meeting_key`, `year`, `country_name`, `circuit_short_name`, `session_name`, `date_start` |
| `meetings` | The race's display name | `meeting_name` |
| `drivers` | Driver and team details | `driver_number`, `full_name`, `name_acronym`, `team_name`, `team_colour` |
| `session_result` | Final classification | `position`, `number_of_laps`, `duration`, `gap_to_leader`, `dnf`, `dns`, `dsq` |
| `starting_grid` | Grid positions | `position` |
| `laps` | Lap and sector times | `lap_number`, `lap_duration`, `duration_sector_1` to `3`, `is_pit_out_lap`, `date_start` |
| `stints` | Tyre stints | `stint_number`, `compound`, `lap_start`, `lap_end`, `tyre_age_at_start` |
| `position` | Position changes by timestamp | `date`, `position` |

FastF1 is not used in this spec. It covers 2018 onward and can be added later as a second `RaceSource`.

## 4. Pipeline

### Command

```
python -m f1app.pipeline.cli ingest --year 2024 --country Italy
```

Options:

- `--refresh` fetches from OpenF1 again even when raw files already exist.
- `--allow-result-mismatch` loads a race even when check 1 below fails.

The first race to load is the 2024 Italian Grand Prix at Monza, chosen as a dry race without a red flag. Any race from 2023 onward can be substituted.

### Steps

1. **Find the race.** Query `sessions` for the year, country and session name "Race". Exactly one match is required; none or several is an error that lists what was found.
2. **Fetch.** Request the other seven endpoints in the table above for that session.
3. **Save raw responses** to `backend/data/raw/{race_id}/{endpoint}.json`. Later runs read these files unless `--refresh` is given.
4. **Validate and shape** the raw JSON into Pydantic models. A response that does not match the expected shape is an error naming the endpoint and field.
5. **Derive position per lap** (see below).
6. **Run the checks** (see below).
7. **Load** into Postgres in one transaction: delete any existing rows for that `race_id`, then insert. Running the command again for the same race gives the same result.

### OpenF1 client behaviour

- Leaves at least 2 seconds between requests, which stays under 30 per minute.
- On HTTP 429, waits and retries up to 3 times, with a longer wait each time.
- Uses a 30 second timeout per request.
- Sends a generic User-Agent naming the project, with no personal details.
- Raises a clear error on a timeout, a non-200 status after retries, or a body that is not JSON.

### The RaceSource interface

`sources/base.py` defines one interface with a method per kind of data: find the race, drivers, results, starting grid, laps, stints and position changes. Each method returns raw records. `sources/openf1.py` is the only implementation in this spec. The transform, checks and load code depend on the interface, not on OpenF1.

### Race ID

`race_id` is the year and the circuit's short name, lower-cased with spaces replaced by hyphens, for example `2024-monza`.

### Position per lap

OpenF1 reports position changes by timestamp, not by lap. For each driver and lap:

1. The lap's end time is the start time of the driver's next lap. For the driver's last lap it is the lap's start time plus its lap time.
2. The position at the end of the lap is the position in the driver's most recent position record at or before that end time.
3. If a lap has no usable end time, its position is left empty.

OpenF1 describes lap start times as approximate, so this rule can be wrong near a position change. The checks below measure that. If the rule proves unreliable on real races, the fallback is to order drivers by the time they complete each lap.

### Checks before loading

| # | Check | On failure |
|---|---|---|
| 1 | For each classified finisher, the position at the end of their final lap equals their official finishing position. | Nothing is written, unless `--allow-result-mismatch` is given. Post-race penalties can legitimately reorder the result, so this check can be overridden; the mismatches are always printed. |
| 2 | For each driver in the starting grid, their earliest position record equals their grid position. | Nothing is written. |
| 3 | For each driver, every lap from 1 to their laps completed falls inside exactly one tyre stint. | Nothing is written. |
| 4 | Every driver in the results, laps and stints also appears in the drivers list. | Nothing is written. |

A failed check prints which check failed and the rows involved.

### To confirm against real data before the schema is fixed

The first task is to fetch one real race and inspect it. These points are unverified and may change the details above:

- How often first-lap times and sector times are empty.
- How far lap start times drift from the real start of the lap.
- Whether `position` holds the race position throughout, which the docs' examples suggest but do not state.
- What `session_result` returns for drivers who retired: a position, or none.
- Whether drivers who start from the pit lane appear in `starting_grid`.

## 5. Data model

`backend/schema.sql` creates these tables. Times are stored in seconds as `double precision`; timestamps as `timestamptz`.

### `races`

| Column | Type | Notes |
|---|---|---|
| `race_id` | `text` | Primary key, for example `2024-monza` |
| `source` | `text` | `openf1` in this spec |
| `source_session_key` | `integer` | The source's own ID for the session |
| `year` | `integer` | |
| `name` | `text` | For example "Italian Grand Prix" |
| `circuit` | `text` | |
| `country` | `text` | |
| `started_at` | `timestamptz` | |

### `drivers`

Primary key `(race_id, driver_number)`. Drivers are stored per race because teams and numbers change between seasons.

| Column | Type | Notes |
|---|---|---|
| `race_id` | `text` | References `races` |
| `driver_number` | `integer` | |
| `full_name` | `text` | |
| `code` | `text` | Three-letter code, for example `LEC` |
| `team_name` | `text` | |
| `team_colour` | `text` | Six hex digits without `#` |

### `results`

Primary key `(race_id, driver_number)`, which references `drivers`.

| Column | Type | Notes |
|---|---|---|
| `position` | `integer` | Empty when the source gives none |
| `grid_position` | `integer` | Empty when the driver is not in the starting grid |
| `laps_completed` | `integer` | |
| `race_time_seconds` | `double precision` | Empty when not given |
| `gap_to_leader_seconds` | `double precision` | Empty for the winner and for lapped cars |
| `laps_behind` | `integer` | Set when the source reports a gap in laps |
| `status` | `text` | One of `finished`, `dnf`, `dns`, `dsq` |

### `laps`

Primary key `(race_id, driver_number, lap_number)`.

| Column | Type | Notes |
|---|---|---|
| `lap_time_seconds` | `double precision` | Can be empty |
| `sector_1_seconds`, `sector_2_seconds`, `sector_3_seconds` | `double precision` | Can be empty |
| `is_pit_out_lap` | `boolean` | |
| `started_at` | `timestamptz` | Approximate, per OpenF1 |
| `position_at_end` | `integer` | Derived; can be empty |

### `stints`

Primary key `(race_id, driver_number, stint_number)`.

| Column | Type | Notes |
|---|---|---|
| `compound` | `text` | As given by the source, upper case, for example `SOFT` |
| `lap_start` | `integer` | |
| `lap_end` | `integer` | |
| `tyre_age_at_start` | `integer` | Laps already run on the tyres |

Deleting a race deletes its rows in every other table.

## 6. API

A FastAPI service with `GET` endpoints only, written with ordinary (non-async) functions.

| Request | Returns |
|---|---|
| `GET /api/races` | A list of `{race_id, year, name, circuit, country, started_at}`, newest first |
| `GET /api/races/{race_id}` | The same fields plus `total_laps` (the most laps completed by any driver) and `drivers`: a list of `{driver_number, code, full_name, team_name, team_colour}` |
| `GET /api/races/{race_id}/results` | A list of `{driver_number, position, grid_position, laps_completed, race_time_seconds, gap_to_leader_seconds, laps_behind, status}`, ordered by position with unclassified drivers last |
| `GET /api/races/{race_id}/laps` | A list of `{driver_number, lap_number, lap_time_seconds, position}`, ordered by driver then lap. `position` is the position at the end of the lap. |
| `GET /api/races/{race_id}/stints` | A list of `{driver_number, stint_number, compound, lap_start, lap_end, tyre_age_at_start}` |
| `GET /api/health` | `{"status": "ok"}` when the database answers; HTTP 503 when it does not |

Sector times are stored but not served in this spec.

### Errors

An unknown `race_id` returns HTTP 404. Every error the API raises itself has this shape:

```json
{"error": {"code": "race_not_found", "message": "No race with id 2024-nowhere"}}
```

### Structure

- `routes.py` handles HTTP only and calls functions in `queries.py`, which hold the SQL and return plain data.
- `schemas.py` declares each response shape with Pydantic; FastAPI builds `/docs` from them.
- Database connections come from a pool and are handed to routes per request.

### Configuration

| Variable | Used by | Meaning |
|---|---|---|
| `DATABASE_URL` | API | Connection string for a read-only database user |
| `PIPELINE_DATABASE_URL` | Pipeline | Connection string for a user that can write |
| `ALLOWED_ORIGIN` | API | The one web address allowed to call the API from a browser |
| `VITE_API_BASE_URL` | Web app | Where the API lives |

Local values live in a `.env` file that git ignores.

## 7. Web app

A React and TypeScript app built with Vite.

### Pages

- **`/`** lists the loaded races, each linking to its summary.
- **`/races/{race_id}`** is the race summary.

```
+------------------------------------------------------+
| 2024 Italian Grand Prix · Monza          [all races] |
+----------------------+-------------------------------+
| Results              | Position by lap               |
| 1  LEC  Ferrari      |  (one line per driver)        |
| 2  PIA  McLaren      |                               |
| 3  NOR  McLaren      +-------------------------------+
| ...                  | Tyre stints                   |
|                      |  (one bar per driver)         |
+----------------------+-------------------------------+
```

- The position chart and the tyre stints share one lap axis.
- At phone width the three panels stack, with no sideways scrolling of the page.
- Hovering or tapping a driver in any panel highlights that driver in all three. The highlighted driver is held in the summary page's state and passed down.

### Choices

- **Charts are hand-drawn SVG.** The functions that map laps and positions to screen coordinates are plain functions, separate from the components, so they can be tested alone.
- **Data fetching is written by hand** with `useState` and `useEffect`. Each panel fetches its own endpoint.
- **Every panel has three states:** loading, error and loaded. A slow first response shows a "waking the server" message.
- **Styling is plain CSS** in per-component files.
- **Types are written by hand** in one file that mirrors the API's response shapes.
- **Team-mates are distinguishable.** They share a team colour, so one has a solid line and the other a dashed line, and each line is labelled with the driver's code.
- **Tyre compounds have fixed colours:** soft red, medium yellow, hard white with an outline, intermediate green, wet blue, and grey for anything else.
- **The footer** credits OpenF1 and says the site is unofficial and not associated with Formula 1. The site uses no Formula 1 or team logos.

## 8. Testing

### Backend (pytest)

| What | How |
|---|---|
| Transform functions | Small saved OpenF1 responses in `tests/fixtures/`; no network |
| Position per lap | Hand-built cases: a normal lap, a pit stop, a retirement, a lapped car, a lap with no end time |
| Checks | Deliberately broken data, proving each check stops the load |
| OpenF1 client | Fake HTTP responses through `httpx`'s mock transport: success, 429 then success, timeout, malformed JSON |
| SQL queries and loading | A separate Postgres test database, with each test rolled back |
| API endpoints | FastAPI's test client against a small seeded race: response shapes and 404s |

No automated test calls the real OpenF1 service. One check that does is run by hand.

### Web (Vitest and React Testing Library)

- The coordinate-mapping functions.
- The results table renders the right rows in the right order.
- Each panel shows its loading and error states.
- Highlighting a driver in one panel highlights them in the others.

### Automatic checks

- **Python:** `ruff` for linting and formatting, `mypy` for types.
- **Web:** the TypeScript compiler, ESLint and Prettier.
- **GitHub Actions** runs both test suites and all linters on every push, with a Postgres service for the database tests.

### Practices

- The author writes the tests.
- For data logic, the test cases are agreed in words first, then the tests are written, then the code.
- Every bug gets a test before its fix.
- There is no coverage percentage target.

## 9. Deployment

Hosting facts were read from the providers' pages on 2026-11-11.

| Piece | Host | Free tier | Catch |
|---|---|---|---|
| Database | Neon | 1 GB, no card, not deleted for inactivity | Sleeps after 5 minutes idle; wakes in a few hundred milliseconds |
| API | Render | 750 hours a month | Sleeps after 15 minutes without traffic; about a minute to wake |
| Web app | Vercel (Hobby) | 100 GB of transfer a month | Personal, non-commercial use only |

- **The API is packaged as a Docker container,** so it can move to an always-on host if the wake-up delay becomes a problem.
- **Code:** a push to `main` runs the tests; Render and Vercel deploy from `main`.
- **Data:** the author runs the pipeline locally with `PIPELINE_DATABASE_URL` pointing at Neon. Production data changes only then.
- **Secrets:** connection strings are set in Render's settings and the local `.env`, never in git. The author creates the accounts and enters the values.
- **Database users:** the API connects as a user that can only read. The pipeline connects as a user that can write.
- **Cross-origin:** `ALLOWED_ORIGIN` is the Vercel address.
- **Health:** Render checks `/api/health`.
- **Timing:** the first deployment happens as soon as one endpoint and one page exist.

## 10. Build order

1. **Setup:** repository, Python environment, local Postgres.
2. **Explore:** fetch one real race and inspect it; settle the open points in section 4.
3. **Pipeline:** client, transform, checks, load.
4. **API:** the races endpoint first.
5. **First deployment:** one endpoint and one minimal page, live.
6. **Rest of the API.**
7. **Web app:** a short modern JavaScript and TypeScript refresher using the race data, then the race list, results table, tyre stints and position chart.
8. **Finish:** README, footer, a second and third race loaded.

## 11. How the work is done

- **The author writes the product code.** Claude explains the concept, sets one small task with a clear definition of done, and reviews the result.
- **Help comes in steps:** a hint, then a pointer to documentation, then an example in a different context, then the answer if asked for.
- **Claude writes** setup files with little to learn in them, explaining each line, and anything the author explicitly asks for.
- **Each task is a branch and a pull request,** reviewed before merging.
- **After each piece, the author explains it back** in a few sentences.
- **The author keeps a decisions log** in their own words.
- **AI autocomplete stays off** in the editor for at least the pipeline and API stages.

### Documents

- **This spec is committed.** It records the design as agreed on its date. A later change of direction is recorded in the decisions log and, where it matters, in an updated spec.
- **The working plan is not committed.** It lives in `docs/plans/`, which git ignores. It details only the current stage; later stages stay as one-line outlines until they start.
- **The README describes what is true now,** and git history and pull requests record what was done.
