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

# Short names live in xrreader.CATALOG; resolve to the service dataset id.
entry = xrreader.CATALOG["glorys12.daily"]
sel = dict(
    variables=["thetao", "so"],
    bbox=xrreader.BBox(-80, -50, 30, 45),     # Gulf Stream
    time=xrreader.TimeRange.parse("2020-01-01", "2020-01-31"),
)

ds = src.open(entry.dataset_id, **sel)              # lazy xr.Dataset
path = src.download(entry.dataset_id, "glorys.nc", **sel)   # NetCDF on disk
```

Request fields are keyword-only after `dataset_id`. `"glorys12.daily"` is a
short name from the bundled `xrreader.CATALOG`; a fully-qualified `dataset_id`
works too.

## Links

- [Architecture & migration plan](design/architecture.md)
- [API Reference](api/reference.md)
- [GitHub](https://github.com/jejjohnson/xrreader)
