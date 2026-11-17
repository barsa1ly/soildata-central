# soildata-central

A static copy of the USDA-NRCS **Soil Survey Geographic Database (SSURGO)**, fiscal year 2027 (published by NRCS on
1 October 2026), for **9 states**: Iowa, Illinois, Kansas, Minnesota, Missouri, North Dakota, Nebraska, South Dakota and Wisconsin. SSURGO is public domain.

The copy of the 48 states and DC is split by region over four repositories with the same layout, so that each one is a
GitHub Pages site of less than 1 GB:

| Repository | States |
|---|---|
| [soildata](https://github.com/barsa1ly/soildata) | AR LA NM OK TX |
| [soildata-west](https://github.com/barsa1ly/soildata-west) | AZ CA CO ID MT NV OR UT WA WY |
| [soildata-central](https://github.com/barsa1ly/soildata-central) | IA IL KS MN MO ND NE SD WI |
| [soildata-east](https://github.com/barsa1ly/soildata-east) | AL CT DC DE FL GA IN KY MA MD ME MI MS NC NH NJ NY OH PA RI SC TN VA VT WV |

The [SoilGuard](https://barsa1ly.github.io/soilguard/) app reads it as a **backup**: it asks USDA's Soil Data Access
(SDA) first, and reads these files only when SDA does not answer. The copy holds what the app asks SDA for (its
`js/soil.js`, `batch()` and `tablesAt()`): the map unit at a point, and that map unit's rows of eight tables, with the
same columns, in the same order, and every value a string, as SDA's JSON gives them. The app's Basin tab also counts
the copy's cells inside a watershed (`backupIn()`) for its soil loss and runoff when SDA does not answer.

## States in this repository

| Code | State | Survey areas | Map units | Components | Horizons | 30 m tiles | 10 m pixels kept | Size (MB) | Tables from |
|---|---|---|---|---|---|---|---|---|---|
| IA | Iowa | 99 | 11,208 | 29,887 | 126,058 | 675 | 90.1% | 58.7 | state GeoPackage |
| IL | Illinois | 102 | 10,501 | 25,587 | 83,063 | 691 | 92.2% | 49.4 | state GeoPackage |
| KS | Kansas | 105 | 5,780 | 22,481 | 93,461 | 967 | 96.0% | 41.7 | state GeoPackage |
| MN | Minnesota | 92 | 10,762 | 43,592 | 137,288 | 1,033 | 93.8% | 61.3 | national CSV |
| MO | Missouri | 114 | 8,718 | 27,689 | 103,563 | 860 | 94.1% | 51.4 | state GeoPackage |
| ND | North Dakota | 74 | 11,680 | 67,884 | 299,834 | 961 | 94.9% | 65.8 | state GeoPackage |
| NE | Nebraska | 93 | 7,548 | 22,532 | 79,795 | 929 | 95.4% | 43.8 | state GeoPackage |
| SD | South Dakota | 67 | 7,479 | 38,640 | 154,760 | 920 | 95.3% | 55.5 | state GeoPackage |
| WI | Wisconsin | 72 | 8,917 | 32,159 | 120,554 | 696 | 93.1% | 51.2 | state GeoPackage |

479 MB in all. *10 m pixels kept*: the share of the state's 10 m pixels whose map unit is the one their 30 m
cell keeps (see the raster below).

Minnesota: NRCS's FY2027 state package could not be used (Minnesota's GeoPackage is damaged, "database disk image is malformed"), so the tables come from NRCS's national CSV tables of the same fiscal year (`ssurgo_tabular_CONUS_2027.7z`, folder *SSURGO / Open Formatted Files / US / tabular (csv)* of the Box share), the state's survey areas only, through `tools/state_from_csv.py`; the raster comes from the state package as for the other states. Rebuilt this way as a check, Texas's 13,122 records are the same as the ones built from its GeoPackage, value for value, except for trailing spaces that the CSV keeps and the GeoPackage trims (`majcompflag` `"No "`, a few ecological class names).

## Layout

```
<ST>/meta.json              source, fiscal year, survey dates, counts, build date, tool
<ST>/mukeys.json            [null, mukey, mukey, ...]: raster index -> mukey (index 0 = no map unit)
<ST>/mu/<k>.json.gz         map unit records, gzipped, k = mukey % 1024 (1024 files)
<ST>/r30/index.json         the 30 m raster's tile grid
<ST>/r30/<row>/<col>.bin    raster tiles, 512 x 512 cells (a tile with no map unit is not written)
```

The tools that build it are in the [soildata](https://github.com/barsa1ly/soildata/tree/main/tools) repository.

### Map unit records (`<ST>/mu/<k>.json.gz`, gzip: inflate, then parse as JSON)

```
{"<mukey>": {"mukey": 391100, "mu": {...}, "comps": [...], "hz": [...], "areas": [...],
             "eco": [...], "ylds": [...], "plants": [...], "restr": [...]}, ...}
```

The shape `tablesAt()` returns: `mukey` a number, `mu` one row object, the rest arrays of row objects whose keys are
the columns of the app's SELECTs with the table prefix dropped (`mu.mukey` is `mukey`, `tg.texture` is `texture`).

| key | rows | order |
|---|---|---|
| `mu` | `mapunit` with its `legend`, the survey's date (`sacatalog.saverest`) and `muaggatt` | one row |
| `comps` | `component` | `comppct_r` descending, ties by `cokey` |
| `hz` | `chorizon` with its representative texture group (`chtexturegrp`, `rvindicator = 'Yes'`), every component's | `cokey`, then `hzdept_r` |
| `areas` | `muaoverlap` with `laoverlap` (counties, MLRA, USLE climate and rainfall factor areas, ...) | `areaovacres` descending |
| `eco`, `ylds`, `plants`, `restr` | `coecoclass`, `cocropyld`, `coeplants`, `corestrictions` | `cokey`, then the table's own key |

Values are strings the way SDA writes them: whole numbers without `.0` (`"9"`, `"85"`), other numbers as written
(`"0.3"`, `"617.3"`), text as it is (`".37"`, `"Loamy fine sand"`), the survey date as `"9/3/2026 1:37:01 PM"`;
`null` where there is no value. Two horizon columns the apps ask SDA for are not in NRCS's download and are `null`
for every horizon: `fragvoltot_r` (total rock fragment volume) and `esp_r` (exchangeable sodium percentage).

### Map unit raster (`<ST>/r30/`)

`index.json`:

```
{"crs": "EPSG:5070", "res": 30, "tile": 512, "x0": ..., "y0": ..., "rows": ..., "cols": ...,
 "have": ["0/8", "0/9", ...], "format": "...", "resample": "mode"}
```

* The grid is NAD83 / Conus Albers (EPSG:5070, equal area, as NRCS's raster), 30 m cells, in tiles of 512 x 512 cells
  (15,360 m) whose origin (`x0`, `y0`: the left edge of tile column 0 and the top edge of tile row 0) is a multiple of
  15,360 m, so every state's grid lines up with the others'. `have` lists the tiles written, as `"row/col"`.
* Each `.bin` holds 512 x 512 unsigned 16-bit little-endian integers, row by row from the north-west corner,
  compressed with zlib (what `DecompressionStream("deflate")` reads). A value is an index into the state's
  `mukeys.json`; 0 means no map unit.
* **Why 30 m.** NRCS maps the soils at 10 m (`muraster_10m_<ST>.tif`). At 10 m the 48 states would take about 4 GB,
  more than four free GitHub Pages sites hold; at 30 m they take about 1.6 GB, in four sites of less than 1 GB. Each
  30 m cell takes the 9 pixels whose centres lie in it and keeps the map unit that covers most of them (pixels with
  no map unit do not count, so a cell is empty only when all 9 are; a tie goes to the centre pixel's map unit). The
  map unit at a point can therefore differ from NRCS's near the edge of a map unit, and map units narrower than a
  cell (strips along streams, for example) can be missed: see *10 m pixels kept* above.

## Looking up a point

1. Project longitude and latitude to EPSG:5070 (GRS 1980; standard parallels 29.5° N and 45.5° N; origin 23° N,
   96° W; no false easting or northing). Coordinates in WGS 84 can be used as they are (NAD83 differs by about a metre).
2. The tile is column `floor((x - x0) / (res * tile))`, row `floor((y0 - y) / (res * tile))`; if `"row/col"` is not in
   `have`, there is no map unit at the point.
3. The cell is column `floor((x - x0) / res)` and row `floor((y0 - y) / res)`; inside the tile it is at
   `(row mod 512) * 512 + (col mod 512)`, two bytes each.
4. `mukeys.json[index]` is the mukey (`null` for index 0), and `mu/<mukey % 1024>.json.gz` (gzip) holds its record under
   `"<mukey>"`.

In JavaScript (`base` is the state folder, such as `https://barsa1ly.github.io/soildata-central/IA`):

```js
// longitude, latitude (degrees) -> EPSG:5070 x, y (m): Snyder (1987), Map Projections: A Working Manual, eqs. 14-1 to 14-6
function albers5070(lon, lat) {
  const a = 6378137, f = 1 / 298.257222101, e2 = f * (2 - f), e = Math.sqrt(e2), r = Math.PI / 180;
  const m = p => Math.cos(p) / Math.sqrt(1 - e2 * Math.sin(p) ** 2);
  const q = p => { const s = Math.sin(p); return (1 - e2) * (s / (1 - e2 * s * s) - Math.log((1 - e * s) / (1 + e * s)) / (2 * e)); };
  const p1 = 29.5 * r, p2 = 45.5 * r, n = (m(p1) ** 2 - m(p2) ** 2) / (q(p2) - q(p1)), C = m(p1) ** 2 + n * q(p1);
  const rho = a * Math.sqrt(C - n * q(lat * r)) / n, rho0 = a * Math.sqrt(C - n * q(23 * r)) / n, t = n * (lon + 96) * r;
  return [rho * Math.sin(t), rho0 - rho * Math.cos(t)];
}

// the record at a point ({ mukey, mu, comps, hz, areas, eco, ylds, plants, restr }), or null where there is no map unit
async function soilAt(base, lat, lon) {
  const get = async p => { const res = await fetch(`${base}/${p}`); if (!res.ok) throw new Error(`${p}: HTTP ${res.status}`); return res; };
  const ix = await (await get("r30/index.json")).json();
  const [x, y] = albers5070(lon, lat);
  const col = Math.floor((x - ix.x0) / ix.res), row = Math.floor((ix.y0 - y) / ix.res);
  const tr = Math.floor(row / ix.tile), tc = Math.floor(col / ix.tile);
  if (!ix.have.includes(`${tr}/${tc}`)) return null;
  const body = (await get(`r30/${tr}/${tc}.bin`)).body.pipeThrough(new DecompressionStream("deflate"));
  const tile = new DataView(await new Response(body).arrayBuffer());
  const i = tile.getUint16(((row - tr * ix.tile) * ix.tile + (col - tc * ix.tile)) * 2, true);
  const mukey = (await (await get("mukeys.json")).json())[i];
  return mukey ? JSON.parse(await new Response((await get(`mu/${mukey % 1024}.json.gz`)).body.pipeThrough(new DecompressionStream("gzip"))).text())[mukey] || null : null;
}
```

`index.json` and `mukeys.json` are worth keeping once loaded; a point then costs one tile (tens of kB) and one record
file (tens of kB gzipped). In Python: `python tools/lookup_point.py IA <lat> <lon> --root <this repository's folder>`
(`tools/` of the soildata repository).

## Rebuilding

1. `tools/fetch_states.py` (soildata repository) lists NRCS's Box share, https://nrcs.app.box.com/v/soils, folder
   *SSURGO / SQLite SSURGO (Geopackages) / States*, downloads each state's `SQLite_SSURGO_<ST>.7z`, unpacks it with
   7-Zip (`SQLite_SSURGO_<ST>.gpkg` and `muraster_10m_<ST>.tif` with its `.ovr` and value table) and runs
   `tools/build_ssurgo.py <ST> --src <the unpacked folder> --out <this repository's folder>` (Python 3.9 or later,
   numpy and rasterio). It replaces `<ST>/mu` and `<ST>/r30` and rewrites `<ST>/mukeys.json` and `<ST>/meta.json`.
2. A state whose package has no usable GeoPackage takes its tables from the national CSV tables:
   `tools/state_from_csv.py <ST> --csv <unpacked csv_ssurgo folder> --out <folder>`, then
   `tools/build_ssurgo.py <ST> --src <the unpacked state folder> --tables <folder>/SQLite_SSURGO_<ST>.sqlite`.
3. `tools/make_readmes.py` writes this file again from the states' `meta.json`.

## Credit and disclaimer

Soil Survey Staff, Natural Resources Conservation Service, United States Department of Agriculture. Soil Survey
Geographic (SSURGO) Database, FY2027 (SQLite SSURGO state downloads and national CSV tables, published October 1,
2026). Available online at https://nrcs.app.box.com/v/soils. Public domain.

This is not an official NRCS product, and NRCS has not reviewed it. It is a converted copy: the tables are reshaped
and the map unit raster is coarsened from 10 m to 30 m, so the map unit found at a point can differ from the one in
NRCS's data, most often near the edge of a map unit. For official soil information use
[Web Soil Survey](https://websoilsurvey.nrcs.usda.gov/) or [Soil Data Access](https://sdmdataaccess.nrcs.usda.gov/).
