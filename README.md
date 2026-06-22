# Contribution [1]: Locations: manage OSM changes (e.g. a shop changes brand)

**Contribution Number:** [1]

**Student:** Hop Le

**Issue:**  https://github.com/openfoodfacts/open-prices/issues/1018

**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it's a real data integrity problem — exactly what I work on as a Cloud/ML/Data/DevOps engineer. The issue is labeled top issue, help wanted and Data quality

My background in ETL pipeline (PySpark, PostgreSQL) and database optimization maps directly to what's needed here: a careful schema migration, a constraint update, and extending an existing Django management command without breaking historical data. I also want to practice making backward-compatible changes to a live open-source codebase, which is a skill I'll carry into production cloud and data engineering work.

Left a comment on the issue introducing myself. The issue is labeld as top issue but inactive since Mar 12.

## Understanding the Issue

### Problem Description

Open Prices stores grocery price data tied to physical store locations. Each location is identified by an OSM ID - a unique number from OpenStreetMap that represents a physical spot on the map (a building, a store front). The problem is that businesses change over time: a Casino supermarket can rebrand to Intermarche, same building, same OSM ID, but a completely different brand. Currently the system has no way to represent this. It stores one location record per OSM ID forever, and when the brand changes in OSM, the old brand data is silently overwritten. Historical prices that were recorded under the old brand now incorrectly show the new brand name.

### Expected Behavior

The system should support multiple versioned Location records for the same OSM ID, each representing a different time period. A price recorded in 2023 at OSM ID 298872652 (when it was Casino) should remain linked to Casino. A price recorded in 2024 at the same OSM ID (now Intermarche) should be linked to Internarche. The system needs an needs an `osm_version_date` field to know when each version became active, and the unique constraint should be relaxed to allow `(osm_id, osm_type, osm_version)` instead of just `(osm_id, osm_type)`.

### Current Behavior

When a second POST is made to '/api/v1/locations' with the same `osm_id` and `osm_type`, the API returns the exisiting record with `"detail":"duplicate"` and silently updates `osm_brand` and `osm_version` to the current OSM values. The old brand is permanently lost. There is no versioned history, no `osm_version_date`, and no way to route historical prices to the correct brand.

### Affected Components

- `open_prices/locations/model.py` - Unique constraint and missing `osm_version_date` field
- `open_prices/locations/migrations/` - needs new migration for constraint + field change
- `open_prices/common/openstreetmap.py` - `get_location_dict()` fetches current OSM data only
- `open_prices/locations/tasks.py` - `fetch_and_save_data_from_openstreetmap()` overwrites instead of versioning
- `open_prices/locations/management/commands/set_location_osm_brand_and_version.py` - backfill command needs to populate `osm_version_date`
- `open_prices/locations/tests.py` - existing unique constraint tests will need updating

---

## Reproduction Process

### Environment Setup

FOUR ISSUES ENCOUNTED DURING LOCAL SETUP:

1. **`TAG=lastest` in `.env` caused Docker to pull a broken pre-built image from `ghcr.io`** instead of building locally. The pre-built image had permission `rwxr-x--xx` on `/docker-entrypoint.sh` executeable but not readable. The container runs as user `off` (UID 1000) who is neither root nor in the root group, so it hit the `--x' slot and bash could not read the script to execute it. Root cause in 'Dockerfile` line 87: `RUN chmod +x /docker-entrypoint.sh` only adds execute permission, not read.
FIX: `chmod 755 /docker-entrypoint.sh` before rebuilding.

2. **`TAG=latest` in `.env` pointed to `ghrc.io/.../api:latest`** which is the production image, not the locally-built dev image.
FIX: run with `TAG=dev` prefix or change `.env` to 'TAG=dev'.

3. **Missing Docker network `po_default`** - had to create manually: `docker network create po_default`.

4. **Scheduler errors on startup** (`ralation "django_q_ormq" does not exist`) - expected before migrations; resolved by running `make migrate-db` after containers were up

**Working dev startup sequence:**
```bash
chmod 755 docker/docker-entrypoint.sh
docker compose -f docker-compose.yml -f docker/dev.yml build --no-cache api
TAG=dev docker compose -f docker-compose.yml -f docker/dev.yml up
### THEN
make migrate-db
```

### Steps to Reproduce

1. Start the dev environment (see above). Confirm API responds:
```bash
    curl http://127.0.0.1:8000/api/v1/locations
    # Return: {"items":[],"page":1,"pages":1,"size":10,"total":0}
```

2. Create a location with a real OSM ID that has a rebrand history:
```bash
   curl -X POST http://127.0.0.1:8000/api/v1/locations \
     -H "Content-Type: application/json" \
     -d '{"type":"OSM","osm_id":298872652,"osm_type":"NODE"}'
   # Returns id:1, osm_brand:null, osm_version:null (OSM fetch is async)
```

3. 3. POST again with the same `osm_id` and `osm_type`:
```bash
   curl -X POST http://127.0.0.1:8000/api/v1/locations \
     -H "Content-Type: application/json" \
     -d '{"type":"OSM","osm_id":298872652,"osm_type":"NODE"}'
```
4. **Observed result:** The API returns the call record (`id:1`) with `"detail":"duplicate"`. By the second call, the async OSM fetch has `osm_brand:"Intermarche"`. No new versioned record was created. Any prices recorded before the rebrand now incorrectly show Intermache. Historical brand data is permanently lost.

### Reproduction Evidence

- **Branch:** https://github.com/Hop-Le133884/open-prices/tree/fix/osm-location-versioning
- **Key finding:** Second POST to same OSM ID returns `"detail":"duplicate"`
  and silently overwrites `osm_brand`/`osm_version` with current OSM data
  instead of creating a new versioned record. There is no `osm_version_date`
  field and no mechanism to route prices to the correct brand by date.
![alt text](<Screenshot From 2026-06-15 11-53-06.png>)
---

## Solution Approach

### Analysis

The root cause is a data model assumption that one physical OSM location equals one database record forever. The specific gaps:

1. **Hard uniqueness constraint** on `(osm_id, osm_type)` in `Location.Meta.constraints` - prevents storing multiple versions of the same location.
2. **No `osm_version_date` field** - `osm_version` exists as an integer but there is no timestamp for when that version became active, making it impossible to route prices to the correct brand by date.
3. **No re-check logic** - `tasks.py -> fetch_and_save_data_from_openstreetmap()` overwrites the current record when OSM data changes instead of creating a new versioned record. The management command `set_location_osm_brand_and_version` only backfills `osm_version=None` records and never detects version increments on already-populated locations.

### Proposed Solution

Relax the unique constraint to `(osm_id, osm_type, osm_version)` and add a `osm_version_date` field to the local model, update the OSM fetch logic to detect version changes and create new records rather than overwrite, and extend the backfill management command to populate `osm_version_date` on existing records. 

### Implementation Plan

**Understand:** The same OSM ID can represent different businesses over time. Prices must be linked to the correct brand based on when they were recorded. The DB currently enforces one record per OSM ID, making versioned history immossible.

**Match:** The existing `osm_version` field and `get_history_location_From_openstreetmap()` in `openstreetmap.py` already fetch versioned OSM data - the infrastructure exists but is not wired into the write path. Migration `0007_alter_location_osm_brand_and_osm_version` shows the team already started thinking about versioning. Maintainer `raphodn` opened draft PR `#1021` with a detection script that compares `response.version() != location.osm_version` and checks whether `name` or `brand` changed, this is the correct singal for a meaningful rebrand vs a minor OSM edit. PHASE III builds directly on his dectetion login by adding record creation after detection, plus the model and constraint changes that make it possible.

**Plan:** [Step-by-step implementation plan]

1. `open_prices/locations/model.py` - Add `osm_version_date` DatetimeField (nullable). Change the unique constraint from `(osm_id, osm_type)` to `(osm_id, osm_type, osm_version)`.
2. open_prices/locations/migrations/` - new migration to alter the constraint and add the new field.
3. `open_prices/common/openstreetmap.py` - Update `get_location_dict()` to also return `osm_version_date` from OSM API response timestamp and add the new field.
4. `open_prices/locations/tasks.py` — Update `fetch_and_save_data_from_openstreetmap()` to compare the fetched `osm_version` against the stored value. If changed, create a new Location record rather than overwriting; if same, update in place.
5. `open_prices/locations/management/commands/set_location_osm_brand_and_version.py` - Extend to also populate `osm_version_date` for existing records.
6. `open_prices/locations/tests.py` — Update the existing unique constraint test (line 77) to reflect the new `(osm_id, osm_type, osm_version)` constraint. Add new tests: same OSM ID with two versions stores two records; prices route to correct brand by date.

**Implement:** https://github.com/Hop-Le133884/open-prices/tree/fix/osm-location-versioning

**Review:** Check `CONTRIBUTING.md` for PR conventions; ensure migration is
backward-compatible (nullable field, no data loss); run full test suite with
`make tests`; verify the 15 existing location tests still pass.

**Evaluate:** Write a test that creates two Location records with the same
`osm_id/osm_type` but different `osm_version`, confirms both are stored
without a constraint violation, and confirms that a price dated before the
rebrand resolves to the old brand record.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week 3 Progress

Built complete OSM location versioning support across 5 files in 5 commits.

**What I built:**
- Added `osm_version_date` DateTimeField (nullable) to the `Location` model so
  the system knows when each OSM version became active
- Updated the unique constraint from `(osm_id, osm_type)` to
  `(osm_id, osm_type, osm_version)` so multiple versions of the same physical
  location can coexist in the database
- Generated and applied migration `0010_add_osm_version_date_and_update_unique_constraint`
- Updated `get_location_from_openstreetmap()` in `openstreetmap.py` to return
  `version_date` from the OSM API timestamp
- Updated `fetch_and_save_data_from_openstreetmap()` in `tasks.py` to detect
  meaningful rebrands (version changed AND name or brand changed) and create a
  new Location record instead of overwriting the existing one — based on the
  detection logic from maintainer raphodn's draft PR #1021
- Extended the backfill management command
  `set_location_osm_brand_and_version.py` to also populate `osm_version_date`
  for existing records
- Updated the existing unique constraint test to reflect new versioning behavior
- Added new `LocationVersioningTest` class with
  `test_same_osm_id_different_version_allowed`

**Key decisions made:**
- Used `version_changed AND identity_changed` as the rebrand signal (same logic
  as raphodn's PR #1021) — this avoids creating new records for minor OSM edits
  like opening hours or phone number changes
- Used `TYPE_OSM_OPTIONAL_FIELDS` to filter which fields get passed to
  `Location.objects.create()` — keeps the create call safe and explicit
- Kept the `else` branch in `tasks.py` to preserve existing update-in-place
  behavior for non-rebrand changes

---

## Challenges Faced

**1. Docker permission bug (`chmod +x` vs `chmod 755`)**
The project's `Dockerfile` uses `RUN chmod +x docker-entrypoint.sh` which sets
world-execute but not world-read. The container runs as user `off` (UID 1000)
who is neither root nor in the root group, so bash could not read the script.
Fixed by running `chmod 755 docker/docker-entrypoint.sh` before rebuilding.
This is a real bug that affects all new contributors and is worth reporting
separately.

**2. `TAG=latest` in `.env` pulled broken production image**
The `.env` file ships with `TAG=latest` which causes Docker to pull the
pre-built `ghcr.io` image instead of building locally. Fixed by running with
`TAG=dev` prefix or changing `.env` to `TAG=dev`.

**3. Pre-commit failing on Python 3.14**
Fedora 44 ships Python 3.14 as the system Python but the project requires 3.11.
`uv run pre-commit install` failed trying to build `pillow` and `pydantic-core`
against 3.14. Fixed with `uv run --python 3.11 pre-commit install`.

**4. Test editing mistakes**
Accidentally edited the `osm_type` loop in `tests.py` when only the unique
constraint block needed changing. Recovered using the `.bak` backup file.
Lesson: always make a backup before editing test files manually.

**Tools that helped:**
- `sed -n 'X,Yp'` to read exact line ranges before editing
- `.bak` backup files before any manual edit
- Running tests after every single change to catch breakage immediately
- raphodn's draft PR #1021 for the change detection logic pattern

---

## Testing Strategy

### Unit Tests Added
- `test_same_osm_id_different_version_allowed` in `LocationVersioningTest`:
  creates two Location records with the same `osm_id + osm_type` but different
  `osm_version` (16 = Casino, 21 = Intermarché), confirms both are stored,
  each has the correct brand, and versions are distinct

### Existing Tests Updated
- `test_location_osm_id_type_validation` in `LocationModelSaveTest`: updated
  the unique constraint assertion to use explicit `osm_version=1` and `osm_version=2`
  to reflect the new `(osm_id, osm_type, osm_version)` constraint

### Validation Performed
- Ran `make migrate-db` successfully after migration
- Confirmed 16/16 tests passing after all changes
- Manually verified via `curl` that the API accepts two locations with the same
  OSM ID when `osm_version` differs

---

## Code Changes

**Files modified:**
- `open_prices/locations/models.py`
- `open_prices/locations/migrations/0010_add_osm_version_date_and_update_unique_constraint.py`
- `open_prices/locations/tests.py`
- `open_prices/common/openstreetmap.py`
- `open_prices/locations/tasks.py`
- `open_prices/locations/management/commands/set_location_osm_brand_and_version.py`

**Key commits:**
- `feat(Locations): add osm_version_date field and update unique constraint to support versioning`
- `feat(Locations): add version_date to OSM API response`
- `feat(Locations): create new location record on OSM version and identity change`
- `feat(Locations): populate osm_version_date in backfill management command`
- `test(Locations): add versioning test for same OSM ID with different versions`

**Branch:**
https://github.com/Hop-Le133884/open-prices/tree/fix/osm-location-versioning

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]

