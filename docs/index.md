# xrreader

> Authenticated archive readers for xarray geodata (CMEMS, CDS, AEMET).

`xrreader` turns a typed **request** — variables, a bounding box, a time
window — into the exact payload a remote data service expects, and returns a
ready-to-use `xarray.Dataset`. It is the data-acquisition layer of the
geo-ecosystem: index with [`geocatalog`](https://github.com/jejjohnson/geocatalog),
**read with `xrreader`**, process with
[`xrtoolz`](https://github.com/jejjohnson/xrtoolz), tile with
[`geopatcher`](https://github.com/jejjohnson/geopatcher).

See the [Architecture & migration plan](design/architecture.md) for the full
design.

## Installation

`xrreader`'s core stays light; each service client is an optional extra:

```bash
pip install 'xrreader[cmems]'        # Copernicus Marine
pip install 'xrreader[cds]'          # Climate Data Store (ERA5)
pip install 'xrreader[cds-insitu]'   # CDS in-situ + archive
pip install 'xrreader[aemet]'        # AEMET OpenData + archive
```

## Quickstart

```python
import xrreader

src = xrreader.CMEMSSource()                  # resolves creds from env / ~/.cmems
req = xrreader.Request(
    variables=("thetao", "so"),
    bbox=xrreader.BBox(-80, -50, 30, 45),     # Gulf Stream
    time=xrreader.TimeRange.parse("2020-01-01", "2020-01-31"),
)

ds = src.open("glorys12.daily", req)          # lazy xr.Dataset
path = src.download("glorys12.daily", "glorys.nc", req)   # NetCDF on disk
```

`"glorys12.daily"` is a short name from the bundled `xrreader.CATALOG`; you can
always pass a fully-qualified `dataset_id` instead.

## Links

- [Architecture & migration plan](design/architecture.md)
- [API Reference](api/reference.md)
- [GitHub](https://github.com/jejjohnson/xrreader)
