# Architecture & migration plan

## Overview

`xrreader` is the **authenticated archive reader** of the geo-ecosystem. Its one
job is to turn a *typed request* — "give me sea-surface height over the Gulf
Stream for January 2020" — into the exact payload a remote data service expects,
hand that payload to the service's client, and return a ready-to-use
`xarray.Dataset`. Everything downstream (`xrtoolz` operators, `geopatcher`
tiling) assumes it is handed clean `xarray` data; `xrreader` is what produces
that data from the messy, per-provider world of credentials, dataset
identifiers, and bespoke subset APIs.

The package is being **seeded by migrating the `xrtoolz.data` submodule** out of
`xrtoolz`. Today that submodule already implements adapters for three services —
Copernicus Marine (CMEMS), the Climate Data Store (CDS), and the Spanish met
agency (AEMET). Pulling it into its own package gives the readers a focused home,
keeps `xrtoolz` about *operators on data* rather than *acquiring data*, and lets
the reader layer grow its own protocols, optional dependencies, and release
cadence.

This page is the living design record for that migration and for the `xrreader`
public API. It is **planning** — at the time of writing no reader code has moved
yet; the sections below describe the target design and the path to it.

!!! note "Why this is a separate package and not part of geocatalog"
    A reasonable question: there is already a `geocatalog` package that talks to
    remote data — why not put the readers there? Because they solve a *different*
    problem. `geocatalog` **indexes files**: point it at a pile of GeoTIFFs / a
    STAC API and it answers "what overlaps this box and time?" without opening
    anything. `xrreader` **fetches arrays**: CMEMS / CDS / AEMET are API services
    with their own auth and server-side subsetting — you ask for a subset and get
    a NetCDF back. One discovers *where data is*; the other *materializes the data
    itself*. Keeping them apart keeps both contracts honest. The two meet at a
    shared query vocabulary (see [geocatalog integration](#geocatalog-integration)).

## The mental model

The ecosystem is a small pipeline of single-responsibility packages. Reading
left to right, data flows from "what exists" to "a patched result":

```text
   DISCOVER &          MATERIALIZE              PROCESS            TILE
   INDEX               (this package)
 ┌───────────┐      ┌──────────────────┐    ┌────────────┐   ┌────────────┐
 │ geocatalog│      │     xrreader     │    │  xrtoolz   │   │ geopatcher │
 │           │      │                  │    │            │   │            │
 │ STAC/CMR  │      │  CMEMS  CDS AEMET│    │ geo  ocn   │   │  split →   │
 │ → index → │─────▶│  DataSource.open │───▶│ atm  rs    │──▶│  op/patch →│
 │  GeoSlice │ rows │   → xr.Dataset   │ ds │ metrics …  │ds │  stitch    │
 └───────────┘      └──────────────────┘    └────────────┘   └────────────┘
       │                    ▲                                 carrier:
       │  query(GeoSlice)   │  open(Request)                  pipekit.Operator
       └────────┬───────────┘
                │  Request ⇄ GeoSlice bridge
                ▼
        same AOI + time window, two query shapes
```

Walking the stages:

1. **Discover & index** (`geocatalog`). Query a STAC API or scan a directory,
   build a spatiotemporal index, persist it to GeoParquet/DuckDB, and slice it
   with a `GeoSlice` (a bounding box + time interval + CRS). Output: rows of
   metadata pointing at files.
2. **Materialize** (`xrreader`, this package). Either load the files a
   `geocatalog` query returned, *or* — for the API services that aren't file
   catalogs — issue an authenticated subset request and receive an
   `xarray.Dataset`. Output: an in-memory or on-disk `xr.Dataset`.
3. **Process** (`xrtoolz`). Run composable operators — regridding, masking,
   geostrophic velocities, spectral metrics — over the dataset.
4. **Tile** (`geopatcher`). Split a field into patches, run an operator per
   patch, stitch the results back.

`xrreader` owns stage 2 for the service archives. The dashed link at the bottom
is the `Request ⇄ GeoSlice` bridge: the same area-of-interest and time window can
drive *both* a `geocatalog` query and an `xrreader` fetch, so the two halves of
"materialize" speak one language.

## Two kinds of "source"

The word *source* shows up in both `geocatalog` and `xrreader`, and it is worth
being precise, because the two are deliberately complementary rather than
competing:

```text
  geocatalog.Source                      xrreader.DataSource
  (metadata, never downloads)            (fetches the actual arrays)
 ┌───────────────────────────┐         ┌────────────────────────────────┐
 │ query(bounds, interval)   │         │ list_datasets() -> [DatasetInfo]│
 │   -> Iterator[SourceRow]  │         │ describe(id)    -> DatasetInfo  │
 │ auth_status()             │         │ open(id, Request)   -> Dataset  │
 │                           │         │ download(id, path,  -> Path     │
 │ STAC · CMR · GEE ·        │         │          Request)               │
 │ EarthAccess               │         │ CMEMS · CDS · AEMET             │
 └───────────────────────────┘         └────────────────────────────────┘
        "where is it?"                          "give it to me"
```

A `geocatalog.Source` answers *where is it?* — it yields lightweight metadata
rows and never pulls bytes. An `xrreader.DataSource` answers *give it to me* — it
authenticates and returns the array. A single future adapter (say, a STAC-backed
reader) could implement both halves; the protocol split in `xrreader`
(`Describable` + `Readable`, below) is designed so that overlap is expressible.

## Anatomy of a read

### The request vocabulary

Every adapter consumes the same typed primitives. These are small frozen
dataclasses that know how to *serialize themselves* into each service's payload
dialect — the per-service knowledge lives on the type, not scattered through the
adapters:

```python
from xrreader import BBox, TimeRange, Request

bbox = BBox(lon_min=-80, lon_max=-50, lat_min=30, lat_max=45)   # Gulf Stream
time = TimeRange.parse("2020-01-01", "2020-01-31")

bbox.as_cmems()
# {'minimum_longitude': -80, 'maximum_longitude': -50,
#  'minimum_latitude': 30, 'maximum_latitude': 45}

bbox.as_cds_area()          # CDS wants [North, West, South, East]
# [45, -80, 30, -50]

time.as_cds_form()          # CDS wants exploded year/month/day lists
# {'year': ['2020'], 'month': ['01'], 'day': ['01', '02', ... '31']}
```

`BBox` understands the antimeridian (a `lon_min > lon_max` box is treated as
wrapping), normalizes between `[-180,180]` and `[0,360]`, and can emit an
`xarray.sel()` selector. `TimeRange` carries an optional sampling frequency and
materializes to a `DatetimeIndex`. `DepthRange` / `PressureLevels` cover the
vertical axis. The whole lot composes into a single `Request`:

```text
                 ┌──────────── Request ────────────┐
                 │ variables : ("adt", "ugos", ...) │
                 │ bbox      : BBox                  │
                 │ time      : TimeRange             │
                 │ depth     : DepthRange | None     │
                 │ levels    : PressureLevels | None │
                 │ extras    : dict | None           │
                 └──────────────┬───────────────────┘
                                │  one object in …
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                    ▼
      .as_cmems()         .as_cds_form()        .as_xarray_sel()
      copernicusmarine    cdsapi year/month     ds.sel(...) for the
      subset kwargs       /day + area form      AEMET / local path
```

One `Request` fans out to whatever dialect the chosen adapter speaks. The catalog
and any future UI build *one* shape; the adapter picks the fields it needs.

### The DataSource lifecycle

Every adapter exposes the same four-method surface. `list_datasets` and
`describe` are discovery (cheap, no download); `download` and `open` materialize:

!!! note "API status — P0 vs P1"
    The snippets below pass a single `Request` to `open`/`download` to
    illustrate the **P1 target** API (locked decision #1). The **P0** code this
    package ships today takes the request fields as keyword-only arguments
    (`open(dataset_id, *, variables=, bbox=, time=, ...)`) and does not resolve
    short names — callers resolve them via `CATALOG[name].dataset_id`. See the
    [README quickstart](https://github.com/jejjohnson/xrreader#usage) for a
    runnable P0 example. P1 introduces the canonical `Request` argument.

```python
import xrreader

src = xrreader.CMEMSSource()                 # resolves creds from env / ~/.cmems

src.list_datasets()                          # -> [DatasetInfo, ...]
src.describe("cmems_mod_glo_phy_my_...")     # -> DatasetInfo (coverage, vars, doi)

req = xrreader.Request(
    variables=("thetao", "so"),
    bbox=xrreader.BBox(-80, -50, 30, 45),
    time=xrreader.TimeRange.parse("2020-01-01", "2020-01-31"),
)

ds = src.open("glorys12.daily", req)         # lazy xr.Dataset (no bytes to disk)
path = src.download("glorys12.daily", "glorys.nc", req)   # NetCDF on disk
```

`"glorys12.daily"` is a **short name** from the bundled `CATALOG` — a convenience
map from memorable labels to the fully-qualified `(source, dataset_id)` the
service actually uses. You can always bypass it and pass a raw `dataset_id`.

```text
  short name              CatalogEntry                  adapter call
 "glorys12.daily" ──▶ source="cmems"            ──▶ CMEMSSource.open(
                      dataset_id=               ──▶   "cmems_mod_glo_phy_my_
                      "cmems_mod_glo_phy_my_…"        0.083deg_P1D-m", req)
```

### Worked examples, one per service

The three adapters share the interface but hide very different machinery behind
it. Lazy imports mean installing `xrreader` core does **not** drag in
`copernicusmarine`, `cdsapi`, or `httpx` — each adapter pulls its client only
when first used, and points you at the right extra if it is missing.

=== "CMEMS (ocean)"

    ```python
    src = xrreader.CMEMSSource()             # pip install 'xrreader[cmems]'
    ds  = src.open("duacs.sla", xrreader.Request(
        variables=("sla", "ugos", "vgos"),
        bbox=xrreader.BBox(-65, -45, 30, 45),
        time=xrreader.TimeRange.parse("2017-01-01", "2017-12-31"),
    ))
    # → copernicusmarine.open_dataset(dataset_id=..., **bbox.as_cmems(),
    #                                 **time.as_cmems(), variables=[...])
    ```

=== "CDS (atmosphere / ERA5)"

    ```python
    src = xrreader.CDSSource()               # pip install 'xrreader[cds]'
    path = src.download("era5.single_levels", "era5.nc", xrreader.Request(
        variables=("t2m", "msl", "u10", "v10"),
        bbox=xrreader.BBox(-10, 5, 35, 45),
        time=xrreader.TimeRange.parse("2021-06-01", "2021-06-30", freq="6h"),
    ))
    # → cdsapi.retrieve(name, {**time.as_cds_form(), 'area': bbox.as_cds_area(),
    #                          'variable': [...]}, path)
    ```

=== "AEMET (stations)"

    ```python
    src = xrreader.AemetSource()             # pip install 'xrreader[aemet]'
    ds  = src.open("aemet.daily", xrreader.Request(
        variables=("tmed", "prec"),
        time=xrreader.TimeRange.parse("2022-01-01", "2022-03-31"),
    ))
    # AEMET's OpenData API is a two-hop fetch (signed-URL envelope → payload)
    # behind a conservative rate limit; the adapter watches the
    # Remaining-request-count header and backs off before HTTP 429.
    ```

Station data is *unstructured* — one record per `(station, time)` — so AEMET (and
CDS in-situ) also ship a local **Archive** that mirrors observations to
long-format GeoParquet with incremental `sync` / `load` / `coverage`. That value
store is discussed under [Local archives](#local-archives).

## The Variable registry

`xrreader` carries a CF-aware variable registry: a `Variable` describes a
canonical quantity (standard name, units, valid range, default colormap) *and*
its per-service aliases. This is the piece that makes "the same variable" mean
the same thing across providers:

```python
from xrreader import resolve

sst = resolve("sea_surface_temperature")
sst.for_source("cmems")     # 'thetao'
sst.for_source("cds")       # 'sea_surface_temperature'
sst.standard_name, sst.units
```

```text
                       Variable("sea_surface_temperature")
                       ┌────────────────────────────────────┐
   CF identity  ◀──────│ standard_name · units · valid_range │──────▶  used by
   (xrtoolz.geo        │ cmap                                │      xrtoolz.viz
    validation,        ├────────────────────────────────────┤      (colormaps)
    CF attrs)          │ aliases = {cmems: "thetao",         │
                       │            cds:   "sea_surface_…",  │──────▶  used by
   source aliases ◀────│            aemet: "ts"}             │      xrreader
   (adapters)          └────────────────────────────────────┘      adapters
```

The registry has a **dual nature**: its CF identity is what `xrtoolz.geo`
(validation) and `xrtoolz.viz` (colormaps) care about, while its source aliases
are what the reader adapters care about. Because both sides need it, the registry
is genuinely shared infrastructure — and that fact drives the central migration
decision below.

## Dependency topology

The registry's dual use means `xrtoolz` and `xrreader` have to share *something*.
The decision (locked) is that **`xrreader` owns the shared `types`** — including
the `Variable` registry — and `xrtoolz` depends on `xrreader` for it:

```text
                         ┌─────────┐
                         │ pipekit │   carrier-agnostic Operator/Graph core
                         └────┬────┘
                  depends on  │  (both sides)
              ┌───────────────┴───────────────┐
              ▼                                ▼
        ┌──────────┐   types (BBox/Time/   ┌──────────┐
        │ xrtoolz  │──  Variable/REGISTRY) ▶│ xrreader │
        └────┬─────┘                        └────┬─────┘
             ▲                                   │ optional, behind extras
             │ depends on                        ├─────────────▶ copernicusmarine
        ┌────┴─────┐                             ├─────────────▶ cdsapi
        │geopatcher│                             ├─────────────▶ httpx/geopandas
        └──────────┘                             └────[geocatalog]──▶ geocatalog
```

The invariants that keep this graph **acyclic**:

- **`xrreader` never imports `xrtoolz`.** It owns its `types`; it *optionally*
  uses `pipekit` (the Operator bridge) and `geocatalog` (the catalog bridge), and
  both of those are gated behind extras.
- **`xrtoolz` depends on `xrreader`**, but only for the lightweight `types`
  surface. `xrreader`'s *core* needs nothing heavier than `numpy` / `pandas` /
  `xarray` (which `xrtoolz` already has). The heavyweight reader clients live in
  extras, so a plain `xrtoolz` install never pulls `copernicusmarine` or `cdsapi`.
- Pre-PyPI, the `xrtoolz → xrreader` and `xrreader[geocatalog] → geocatalog`
  edges are wired with `[tool.uv.sources]` git pins — exactly the mechanism
  `xrtoolz` already uses to pin `pipekit`.

### Optional dependency extras

| Extra | Pulls | Enables |
|---|---|---|
| `xrreader[cmems]` | `copernicusmarine` | `CMEMSSource` |
| `xrreader[cds]` | `cdsapi` | `CDSSource` (gridded reanalysis) |
| `xrreader[cds-insitu]` | `cdsapi`, `geopandas`, `pyarrow`, `loguru` | CDS in-situ + `CDSInsituArchive` |
| `xrreader[aemet]` | `httpx`, `geopandas`, `pyarrow`, `loguru` | `AemetSource` + `AemetArchive` |
| `xrreader[pipekit]` | `pipekit` | `xrreader.ops` Operator bridge |
| `xrreader[geocatalog]` | `geocatalog` | `xrreader.integrations.geocatalog` |

## The types decision (why xrreader owns them)

`xrtoolz.types` today holds two separable things. The **request vocabulary**
(`BBox`, `TimeRange`, `DepthRange`, `PressureLevels`, `Request`, `Station`) is
used *only* by the data readers. The **Variable registry** is used by the readers
*and* by `xrtoolz.geo` / `xrtoolz.viz`. Three homes were considered:

```text
  Option A  (chosen)            Option B                  Option C  (rejected)
  xrreader owns types           new shared package        registry stays in xrtoolz

  xrtoolz ──▶ xrreader          xrtoolz ─▶ geotypes ◀─ x  xrreader ──▶ xrtoolz
              (types)                       ▲  rreader               (Variable)
                                            └── shared base

  light dep; one home;          cleanest seam; but a      reader would drag the
  reader stays slim             7th repo to maintain      ENTIRE heavy geo stack
```

**Option C is the trap**: making the "minimal reader" depend on `xrtoolz` would
pull `rioxarray`, `regionmask`, `xrft`, `cartopy`, `einx`, … into every install —
the opposite of a lean acquisition layer. **Option B** (a neutral `geotypes`
package both depend on) is the cleanest long-term seam but costs a seventh repo;
it stays in reserve for when a third consumer appears. **Option A** — `xrreader`
owns all of `types` — is the pragmatic choice now: `types` has no heavy
dependencies, so `xrtoolz` importing the registry from `xrreader.types` is a cheap
edge, and there is exactly one home for the vocabulary.

The cutover is a **hard cut** — no compatibility shim. The four `xrtoolz` call
sites (`geo/_src/validation.py`, `geo/operators.py`, `viz/_src/cmaps.py`,
`viz/validation/_src/spatial.py`) switch their imports from `xrtoolz.types` to
`xrreader.types`, and external code importing `xrtoolz.types` moves with them.

## Protocols

The adapters are formalized as runtime-checkable `Protocol`s (the
`pipekit_cycle.protocols` / `geocatalog.Source` style), so third parties — and
`geocatalog` — can duck-type against them without subclassing. Concrete adapters
subclass a `BaseDataSource` ABC for the shared variable-resolution helpers and
satisfy the protocols structurally for free.

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Describable(Protocol):                  # the metadata half (≈ geocatalog.Source)
    source_id: str
    def list_datasets(self) -> list[DatasetInfo]: ...
    def describe(self, dataset_id: str) -> DatasetInfo: ...

@runtime_checkable
class Readable(Protocol):                     # the materializing half
    def open(self, dataset_id: str, request: Request | None = None) -> xr.Dataset: ...
    def download(self, dataset_id: str, output: Path,
                 request: Request | None = None) -> Path: ...

@runtime_checkable
class DataSource(Describable, Readable, Protocol):
    """A full archive reader = Describable + Readable."""

@runtime_checkable
class Credentials(Protocol):
    @classmethod
    def from_env(cls) -> "Credentials": ...
    def as_auth(self) -> Mapping[str, str]: ...

@runtime_checkable
class Archive(Protocol):                      # a local incremental mirror
    def sync(self, *, time: TimeRange | None = None) -> None: ...
    def load(self, *, bbox: BBox | None = None,
             time: TimeRange | None = None) -> Any: ...   # GeoDataFrame | xr.Dataset
    def coverage(self) -> Any: ...
```

Splitting `Describable` from `Readable` is deliberate. It makes the `geocatalog`
overlap explicit — a `geocatalog.Source` is essentially a `Describable` over
remote rows — and it lets future adapters implement just one half (a discovery-only
STAC reader; a pure in-memory ingestor). The composite `DataSource` is what the
three current adapters satisfy.

## pipekit bridge

`xrreader.ops` (gated behind `[pipekit]`) wraps a read into a `pipekit.Operator`,
so acquisition becomes the **source node** of a pipeline graph and the whole
read → process → patch flow is one composable object:

```python
from xrreader.ops import Open

pipe = (
    Open(CMEMSSource(), "duacs.sla", Request(variables=("sla", "ugos", "vgos"),
                                             bbox=gulf_stream, time=year_2017))
    >> xrtoolz.geo.Subset(...)
    >> xrtoolz.ocn.GeostrophicVelocity()
    >> xrtoolz.metrics.IsotropicPSD()
)
result = pipe(None)        # the Open node ignores the (empty) carrier
```

```text
       Open(source, id, Request)          a pipekit source node:
              │  _apply(None)             takes no upstream carrier,
              ▼                           emits the fetched dataset
        xr.Dataset ──▶ geo.Subset ──▶ ocn.Geostrophic ──▶ metrics.PSD ──▶ result

   wrap to skip refetch:
        Download(...)  ──▶ writes to cache_path(source, id, Request);
                           a cached path short-circuits the download (idempotent)
        Cache(Open(...)) ─▶ memoizes the in-memory dataset by config hash
```

Two design points carry over from `pipekit` conventions:

- **Idempotent downloads.** `Download` writes to a deterministic
  `cache_path(source, dataset_id, request)`; if the file already exists it is
  returned without re-fetching. Wrapping `Open` in `pipekit.Cache` memoizes the
  in-memory dataset by config hash.
- **Stays serializable.** The ops carry only `source_id` plus a frozen
  `Credentials` dataclass and build the live service client lazily inside
  `_apply`. That keeps them pickleable / `check_pickleable`-clean, so they do not
  need `forbid_in_yaml`.

## geocatalog integration

`xrreader.integrations.geocatalog` (gated behind `[geocatalog]`, mirroring
`geopatcher.integrations.pipekit`) is where the reader and the index meet.

### Request ⇄ GeoSlice

`geocatalog`'s `GeoSlice` is a bounding box + time interval + resolution + CRS;
`xrreader`'s `Request` is variables + bbox + time + depth + levels. The bridge
lets one AOI/time window drive both worlds:

```text
        xrreader.Request                         geocatalog.GeoSlice
   ┌────────────────────────┐   request_to_      ┌──────────────────────┐
   │ bbox      BBox          │──  geoslice() ────▶│ bbox  (minx,miny,    │
   │ time      TimeRange     │                    │        maxx,maxy)    │
   │ variables (…)           │   ◀── geoslice_to_ │ interval (start,end) │
   │ depth/levels            │       request()    │ crs   "EPSG:4326"    │
   └────────────────────────┘                    │ resolution           │
     variables/depth/levels                       └──────────────────────┘
     have no GeoSlice analogue ─┘  (dropped or stashed in extras)
```

```python
from xrreader.integrations.geocatalog import request_to_geoslice

hits = catalog.query(request_to_geoslice(req))   # same req that drives open()
```

### Local archives

`AemetArchive` and `CDSInsituArchive` are **value stores** — they hold the actual
measurements in long-format GeoParquet, with incremental `sync` and per-station
`coverage`. `geocatalog` is an **index over files**, not a value store. The
distinction is the whole decision:

```text
   geocatalog  =  INDEX over files            archive  =  VALUE store
   ┌───────────────────────────┐              ┌──────────────────────────────┐
   │ row = filepath + geometry │              │ row = (station, time) +       │
   │       + interval + crs    │              │       measured values         │
   │ "find files overlapping…" │              │ "the temperature here, then"  │
   └───────────────────────────┘              └──────────────────────────────┘
        knows WHERE data is                        holds the DATA itself
```

**Decision: bridge, don't re-platform.** Keep the bespoke value store; add the
`Request ⇄ GeoSlice` bridge so archive queries share the ecosystem vocabulary;
and *optionally* register the archive's emitted parquet files into a `geocatalog`
as a thin **discovery** layer (one row per file footprint) — without moving value
storage. Folding the values into a `geocatalog` would bend its file-index
contract for marginal code savings, and its row model has no home for the
archive's per-station coverage/gap statistics.

Full re-platforming onto `DuckDBGeoCatalog` is revisited only if in-situ volume
outgrows single-file GeoParquet — at which point geocatalog's DuckDB backend
(millions of rows, remote/S3, predicate pushdown) becomes the right home and the
trade flips.

## Migration mechanics

### Coupling — why this is a clean lift

`xrtoolz.data` is ~5,150 LOC and near-perfectly severable:

```text
   what data/** imports from xrtoolz          what imports data/**
   ┌────────────────────────────┐             ┌──────────────────────────┐
   │ xrtoolz.types     ✓ (only) │             │ (nothing in src/)        │
   │ xrtoolz.utils     ✗        │             │ not re-exported from      │
   │ xrtoolz.geo/ocn   ✗        │             │   xrtoolz/__init__.py     │
   │ pipekit           ✗        │             │ only tests/data/** (×15)  │
   └────────────────────────────┘             └──────────────────────────┘
        one inbound edge (types)                   zero load-bearing
        → travels with the move                    consumers
```

The single coupling is `xrtoolz.types`, which the [types decision](#the-types-decision-why-xrreader-owns-them)
resolves by moving it into `xrreader`. The 15 data tests are isolated under
`tests/data/` and move wholesale.

### Package layout, before and after

```text
   BEFORE (xrtoolz)                      AFTER (xrreader, flat namespace)
   src/xrtoolz/                          src/xrreader/
   ├── types/        ──────────────┐     ├── __init__.py     # flat: CMEMSSource,
   │   └── _src/{geometry,time,     │     │                   #   Request, CATALOG…
   │             levels,request,    ├────▶├── types/          # moved verbatim
   │             station,variable,  │     ├── protocols.py    # new (P1)
   │             validation}        │     ├── _src/
   ├── data/         ──────────────┘     │   ├── base.py catalog.py
   │   └── _src/{base,catalog,            │   │   credentials.py cache.py
   │             credentials,cache,       │   ├── cmems/ cds/ aemet/
   │             cmems/,cds/,aemet/}      │   └── …
   ├── geo/  viz/  ── import types ─┐     ├── ops.py          # new (P2, [pipekit])
   │        (repoint to xrreader)   │     └── integrations/
   └── …                            │         └── geocatalog.py  # new (P2)
                                    │
   xrtoolz keeps geo/ocn/atm/rs/…   └─▶  geo/viz now do:  from xrreader.types import …
```

The public namespace is **flat**: `xrreader.CMEMSSource`, `xrreader.Request`,
`xrreader.BBox`, `xrreader.CATALOG`, `xrreader.describe`, `xrreader.REGISTRY`,
the `*Credentials` + `load_*` helpers, and the archive classes — all at the top
level.

### Cleanup folded into the move

`CMEMSSource` / `CDSSource` currently raise `ImportError("… pip install
xrtoolz[data]")`, but **no `data` extra exists** and `copernicusmarine` is
declared nowhere in `xrtoolz`'s dependencies. The migration defines the proper
`xrreader[cmems]` / `xrreader[cds]` / `xrreader[cds-insitu]` / `xrreader[aemet]`
extras and fixes the error messages to point at them.

### Phasing

| Phase | Work |
|---|---|
| **P0** | Rename template → `xrreader`; move `types/` + `data/_src/**` (flat) + `tests/data/**`; rewrite imports; add reader extras + fix the broken `[data]` ImportError. In `xrtoolz`: repoint `geo`/`viz` to `xrreader.types`, add the `xrreader` dependency (hard cut, no shim). |
| **P1** | `xrreader.protocols`; adapters satisfy them; switch `open` / `download` to the canonical `Request` argument. |
| **P2** | `xrreader.ops` (pipekit) + `integrations.geocatalog` (`Request ⇄ GeoSlice`). |
| **P3** | Optional discovery registration of archive files into `geocatalog`. |

## Locked decisions

| # | Decision | Choice | Rationale |
|---|---|---|---|
| 1 | `open` / `download` argument shape | **One canonical `Request`** | One wire object, no loose-kwarg drift; shares the `Request ⇄ GeoSlice` bridge |
| 2 | `xrtoolz.types` back-compat | **Hard cut** — no shim | Pre-1.0; only four internal call sites + isolated externals |
| 3 | Local archives → geocatalog | **Bridge, don't re-platform** | geocatalog indexes files; archives store values — different jobs |
| 4 | Public namespace | **Flat** (`xrreader.CMEMSSource`) | Short, discoverable top-level API |
| — | Shared `types` home | **Option A** — `xrreader` owns them | Light dep, one home; keeps the reader slim (vs Option C dragging the geo stack) |
