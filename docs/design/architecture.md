# Architecture & migration plan

`xrreader` is the **authenticated archive reader** of the geo-ecosystem: it
turns a typed *request* into a service-specific payload and **downloads or
opens** an `xarray.Dataset` ready for `xrtoolz` operators and `geopatcher`
patching. It is being seeded by migrating the `xrtoolz.data` submodule out of
`xrtoolz`.

This page is the living design record for that migration and for the
`xrreader` public API.

## Where xrreader sits

| Package | Role |
|---|---|
| **geocatalog** | Spatiotemporal **index** over files / STAC. Query "what exists & where", persist to GeoParquet/DuckDB, slice with `GeoSlice`, load indexed files. |
| **xrreader** | Authenticated **archive reader**. Typed `Request` → service payload → `download()`/`open()` an `xr.Dataset`. For API archives (CMEMS, CDS, AEMET) that are *not* file/STAC catalogs. |
| **xrtoolz** | Composable `xarray` operators (geo / ocn / atm / rs / metrics / …). |
| **geopatcher** | Four-axis patch → per-patch op → stitch. |
| **pipekit** | Carrier-agnostic `Operator` / `Sequential` / `Graph` core + protocols. |

The end-to-end flow the ecosystem targets:

```
geocatalog.Source.query(STAC) ─▶ GeoCatalog (DuckDB/GeoParquet)      # discover & index
        │  query(GeoSlice) ◀──── request_to_geoslice(Request)
        ▼
   rows ─▶ geocatalog.load_xarray(...)   ── OR ──   xrreader open(Request)   # materialize
        ▼
   xr.Dataset ─▶ xrtoolz operators ─▶ geopatcher patch / stitch
```

`xrreader` fills the *materialize* box for **service archives**; geocatalog
fills it for **file / STAC catalogs**. The shared seam is the
`Request` ↔ `GeoSlice` vocabulary.

### Why a separate package (not folded into geocatalog)

CMEMS / CDS / AEMET are **API services** with their own authentication and
server-side subsetting — you request a subset and receive a NetCDF / lazy
dataset. That is a different model from geocatalog's "index a pile of files /
STAC items, then load slices." Keeping them apart keeps each contract clean:
geocatalog *indexes where data is*; xrreader *fetches the data itself*.

## Dependency topology

```
pipekit ◀── xrtoolz ──▶ xrreader ◀── geocatalog   (optional bridge)
              ▲            ▲
          geopatcher   reader clients (optional extras)
```

Invariants that keep this acyclic:

- **xrreader never imports xrtoolz.** It owns its `types`, optionally uses
  `pipekit` (Operator bridge) and `geocatalog` (catalog bridge) — both behind
  extras.
- **xrtoolz depends on xrreader** for the shared CF `Variable` registry (used
  by `xrtoolz.geo` validation and `xrtoolz.viz` colormaps). This is a *light*
  dependency: `xrreader`'s core needs only `numpy` / `pandas` / `xarray`; the
  heavy reader clients (`copernicusmarine`, `cdsapi`, `httpx`, `geopandas`, …)
  live in `xrreader` **extras**, so `xrtoolz` does not pull them.
- Pre-PyPI, the `xrtoolz → xrreader` and `xrreader[geocatalog] → geocatalog`
  edges are wired with `[tool.uv.sources]` git pins (as `xrtoolz` already pins
  `pipekit`).

## What migrates

`xrtoolz.data` is a near-perfectly severable unit (~5,150 LOC):

- It imports **only** `xrtoolz.types` — nothing else from `xrtoolz`.
- **Nothing** imports it; it is not even re-exported from `xrtoolz/__init__.py`.
- Its tests are isolated under `tests/data/` (15 files).

The CF `Variable` registry inside `xrtoolz.types` is the only shared piece —
`xrtoolz.geo` / `xrtoolz.viz` use it too. **Decision: `xrreader` owns all of
`types`**, and `xrtoolz` imports the registry from `xrreader.types`.

### Locked decisions

| # | Decision | Choice |
|---|---|---|
| 1 | `open`/`download` argument shape | **One canonical `Request`** object (no loose `variables=`/`bbox=`/… kwargs) |
| 2 | `xrtoolz.types` back-compat | **Hard cut** — no deprecating shim; external `xrtoolz.types` imports move to `xrreader.types` |
| 3 | Local archives → geocatalog | **Bridge, don't re-platform** (see [Local archives](#local-archives-aemet-cds-insitu)) |
| 4 | Public namespace | **Flat** — `xrreader.CMEMSSource`, `xrreader.Request`, `xrreader.CATALOG`, … |

## Public API shape (flat)

```python
import xrreader as xr_

src = xr_.CMEMSSource()                       # or CDSSource(), AemetSource()
req = xr_.Request(
    variables=("thetao", "so"),
    bbox=xr_.BBox(-80, -50, 30, 45),
    time=xr_.TimeRange.parse("2020-01-01", "2020-01-31"),
)
ds = src.open("glorys12.daily", req)          # one canonical Request argument
```

Top-level names: `DataSource`, `DatasetInfo`, `DatasetKind`, `CMEMSSource`,
`CDSSource`, `AemetSource`, `CATALOG`, `describe`, `Request`, `BBox`,
`TimeRange`, `DepthRange`, `PressureLevels`, `Station`, `StationCollection`,
`Variable`, `REGISTRY`, the `*Credentials` + `load_*` helpers, and the
archive classes.

## Protocols (`xrreader.protocols`)

Runtime-checkable `Protocol`s (in the `pipekit_cycle.protocols` /
geocatalog `Source` style). Concrete adapters subclass a `BaseDataSource` ABC
for the variable-resolution helpers and satisfy the protocols structurally.

```python
@runtime_checkable
class Describable(Protocol):                 # metadata half ≈ geocatalog.Source
    source_id: str
    def list_datasets(self) -> list[DatasetInfo]: ...
    def describe(self, dataset_id: str) -> DatasetInfo: ...

@runtime_checkable
class Readable(Protocol):                     # materializing half
    def open(self, dataset_id: str, request: Request | None = None) -> xr.Dataset: ...
    def download(self, dataset_id: str, output: Path, request: Request | None = None) -> Path: ...

@runtime_checkable
class DataSource(Describable, Readable, Protocol): ...

@runtime_checkable
class Credentials(Protocol):
    @classmethod
    def from_env(cls) -> "Credentials": ...
    def as_auth(self) -> Mapping[str, str]: ...

@runtime_checkable
class Archive(Protocol):                       # local incremental mirror
    def sync(self, *, time: TimeRange | None = None) -> None: ...
    def load(self, *, bbox: BBox | None = None, time: TimeRange | None = None) -> Any: ...
    def coverage(self) -> Any: ...
```

Splitting `Describable` from `Readable` makes the geocatalog overlap explicit
(geocatalog's `Source` ≈ `Describable` over remote rows) and lets future
readers implement only one half.

## pipekit bridge (`xrreader.ops`, `[pipekit]` extra)

Reading becomes the **source node** of a pipeline:

```python
from xrreader.ops import Open

pipe = (
    Open(CMEMSSource(), "glorys12.daily", Request(...))
    >> xrtoolz.geo.Subset(...)
    >> xrtoolz.ocn.GeostrophicVelocity()
)
ds = pipe(None)
```

- `Download` is **idempotent** via the existing deterministic cache
  (`cache_path(source, dataset_id, request)`) — a cached path is returned
  without re-fetching.
- Ops stay pickleable / `check_pickleable`-clean by **building the service
  client lazily inside `_apply`** and carrying only `source_id` + a frozen
  `Credentials` dataclass — so they do not need `forbid_in_yaml`.

## geocatalog integration (`xrreader.integrations.geocatalog`, `[geocatalog]` extra)

Mirrors `geopatcher.integrations.pipekit` — a soft dependency behind an extra.

### `Request` ↔ `GeoSlice` bridge

`GeoSlice` = bbox + interval + resolution + CRS; `Request` = variables + bbox +
time + depth + levels.

```python
def request_to_geoslice(req: Request, *, crs="EPSG:4326") -> GeoSlice: ...
def geoslice_to_request(gs: GeoSlice, *, variables=()) -> Request: ...
```

The same AOI/time window then drives both a geocatalog `query()` and an
`xrreader` `open()`.

### Local archives (AEMET, CDS-insitu)

`AemetArchive` / `CDSInsituArchive` are **value stores** — they hold the actual
observations in long-format GeoParquet with incremental `sync` and per-station
`coverage`. geocatalog is an **index over files**, not a value store, so the two
serve different jobs.

**Decision: bridge, don't re-platform.** Keep the bespoke value store; add the
`Request ↔ GeoSlice` bridge so archive queries share the ecosystem vocabulary;
*optionally* register the archive's emitted parquet files into a geocatalog as a
thin **discovery** layer (one row per file footprint). Full re-platforming onto
`DuckDBGeoCatalog` is revisited only if in-situ volume outgrows single-file
GeoParquet — at which point geocatalog's DuckDB backend becomes the right home.

## Cleanup folded into the move

`CMEMSSource` / `CDSSource` currently raise
`ImportError("… pip install xrtoolz[data]")`, but **no `data` extra exists** and
`copernicusmarine` is declared nowhere. The migration defines proper
`xrreader[cmems]`, `xrreader[cds]`, `xrreader[cds-insitu]`, `xrreader[aemet]`
extras and fixes the error messages.

## Phasing

| Phase | Work |
|---|---|
| **P0** | Rename template → `xrreader`; move `types/` + `data/_src/**` (flat) + `tests/data/**`; rewrite imports; add reader extras + fix the broken `[data]` ImportError. In `xrtoolz`: repoint `geo`/`viz` to `xrreader.types`, add the `xrreader` dependency (hard cut, no shim). |
| **P1** | `xrreader.protocols`; adapters satisfy them; switch `open`/`download` to the canonical `Request` argument. |
| **P2** | `xrreader.ops` (pipekit) + `integrations.geocatalog` (`Request ↔ GeoSlice`). |
| **P3** | Optional discovery registration of archive files into geocatalog. |
