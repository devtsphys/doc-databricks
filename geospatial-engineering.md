# Geospatial Engineering, Indexing & H3 — Reference Card and Cheat Sheet
### Focus: Azure Databricks, Unity Catalog, large-scale and operationalized geospatial services

> **Version note:** Written for Databricks Runtime 15.x–17.x era, `h3` Python lib v4.x, Apache Sedona 1.6+, Databricks Labs Mosaic (maintenance mode). Databricks spatial features evolve quickly (native `GEOMETRY`/`GEOGRAPHY` types and `ST_` functions are rolling out). **Always verify function names/availability against current Databricks docs for your runtime.** Items marked 🔎 are especially worth re-checking.

---

## Table of Contents
1. [Geospatial Fundamentals](#1-geospatial-fundamentals)
2. [Data Formats & Storage](#2-data-formats--storage)
3. [Spatial Indexing — Concepts & Comparison](#3-spatial-indexing--concepts--comparison)
4. [H3 In Depth](#4-h3-in-depth)
5. [H3 Function Reference (Databricks SQL / PySpark / Python)](#5-h3-function-reference)
6. [Other Spatial Tooling on Databricks (native ST, Sedona, Mosaic, GeoPandas)](#6-other-spatial-tooling-on-databricks)
7. [Core Processing Patterns](#7-core-processing-patterns)
8. [Sample Project Code](#8-sample-project-code)
9. [Delta Lake Layout & Performance Tuning](#9-delta-lake-layout--performance-tuning)
10. [Operationalizing on Azure & Azure Databricks](#10-operationalizing-on-azure--azure-databricks)
11. [Serving Geospatial Data to Other Users & Groups](#11-serving-geospatial-data-to-other-users--groups)
12. [Governance, Security, Cost, Monitoring](#12-governance-security-cost-monitoring)
13. [Best Practices Checklist](#13-best-practices-checklist)
14. [Use Cases & Reference Architectures](#14-use-cases--reference-architectures)
15. [Pitfalls & Troubleshooting](#15-pitfalls--troubleshooting)
16. [Quick Reference Tables](#16-quick-reference-tables)

---

## 1. Geospatial Fundamentals

### 1.1 Core vocabulary

| Term | Meaning |
|---|---|
| **Geometry** | Planar (Cartesian) shapes: Point, LineString, Polygon, Multi*, GeometryCollection. Distances in CRS units. |
| **Geography** | Shapes on an ellipsoid/sphere. Distances in meters, geodesic math. Better for global data. |
| **CRS / SRS / SRID** | Coordinate Reference System. Identified by EPSG code. |
| **EPSG:4326** | WGS 84 lon/lat degrees (GPS, GeoJSON default). **Axis order trap:** EPSG defines lat,lon; most software (GeoJSON, WKT in practice, Sedona, H3 SQL `longlat`) uses **lon,lat**. |
| **EPSG:3857** | Web Mercator (meters; web tiles; area/distance distorted far from equator). |
| **UTM zones** (e.g. EPSG:32632 for Frankfurt/Germany region, 25832 ETRS89/UTM32N) | Accurate local metric CRS for buffers/distances. |
| **EPSG:3035** | ETRS89-LAEA Europe — equal-area, good for area stats across Europe. |
| **Datum** | Reference model of the Earth (WGS84, ETRS89, NAD83…). |
| **Projection** | Maps ellipsoid to plane; always trades off area / angle / distance. |
| **Topology predicates** | `Contains`, `Within`, `Intersects`, `Touches`, `Crosses`, `Overlaps`, `Disjoint`, `Covers`, `Equals`. |
| **Spatial join** | Join two datasets on a spatial predicate instead of key equality. |
| **Buffer** | Polygon at distance *d* around geometry. |
| **Tessellation** | Cover an area with cells (grid / hexagons) and optionally clip to the original shape. |
| **Dissolve / Union** | Merge geometries into one. |
| **Simplify** | Reduce vertices (Douglas–Peucker, Visvalingam). |
| **Raster vs Vector** | Raster = gridded values (imagery, DEM); Vector = discrete features. |
| **Map matching** | Snap GPS traces to road network. |
| **Routing / Isochrone** | Shortest path / reachable area within time budget. |

### 1.2 Golden rules
1. **Know your CRS for every dataset.** Store SRID/CRS as metadata (column comment, table property, or column).
2. **lon/lat order**: `(x = longitude, y = latitude)`. Mixed-up axes are bug #1.
3. **Use geography/geodesic or a local projected CRS for distance/area** — never compute meters from degrees.
4. **Index early, join on indices** (cell IDs) rather than on raw geometry wherever accuracy allows.
5. **Validate geometries** (self-intersections, unclosed rings, NaN, 0/0 coordinates, out-of-range lat ±90 / lon ±180).
6. **Keep raw (bronze) coordinates + derived index columns**; never replace the raw geometry with just a cell.

### 1.3 Typical geospatial workload types
| Workload | Typical operation | Scale pattern |
|---|---|---|
| Point enrichment (POIs, GPS pings, IoT) | point-in-polygon, nearest neighbor | billions of points × thousands of polygons |
| Density / heatmaps / hotspots | aggregate by cell | H3 res 6–10 |
| Trajectory analytics | line building, stay-point, map matching | sessionization + window functions |
| Proximity / geofencing | within-distance, k-ring | streaming + stateful |
| Network analysis | routing, isochrones | graph engines (OSRM, Valhalla, GraphHopper, pgRouting) |
| Raster analytics | zonal stats, NDVI | Sedona raster / rasterio / Mosaic |
| Site selection / catchment | buffers, Voronoi, drive-time polygons | batch |

---

## 2. Data Formats & Storage

### 2.1 Formats

| Format | Type | Notes / When to use |
|---|---|---|
| **WKT** | text | Human readable: `POINT(8.68 50.11)`; interop, debugging. Large. |
| **WKB / EWKB** | binary | Compact, fast; store in Delta as `BINARY`. Preferred interchange for Spark. |
| **GeoJSON** | text | Web/API payloads; always EPSG:4326; RFC 7946 (lon,lat). |
| **Shapefile** | multi-file | Legacy; 2 GB limit, 10-char field names. Read via Sedona/GeoPandas/pyogrio. |
| **GeoPackage (.gpkg)** | SQLite | Modern single-file replacement for shapefile. |
| **GeoParquet** | columnar | Parquet + geometry metadata (WKB column, CRS, bbox covering). Best lake format for vector data. |
| **FlatGeobuf** | binary, streaming | Efficient with spatial index for HTTP range reads. |
| **PMTiles / MBTiles / MVT** | tiles | Vector tiles for web maps. |
| **GeoTIFF / COG** | raster | Cloud-Optimized GeoTIFF for range reads from ADLS. |
| **Zarr / NetCDF / GRIB** | n-D arrays | Climate/weather. |
| **Delta Lake** | table format | The default analytics store; geometry as WKB `BINARY` or native `GEOMETRY`/`GEOGRAPHY` 🔎. |
| **KML/GPX** | XML | Consumer GPS / Google Earth. |

### 2.2 Recommended Delta column conventions

```text
geom_wkb        BINARY      -- raw geometry, WKB, EPSG:4326 (or native GEOMETRY/GEOGRAPHY)
srid            INT         -- if not constant
lon, lat        DOUBLE      -- for points: explicit columns help filtering & BI tools
h3_r6, h3_r8, h3_r10  BIGINT   -- H3 cell ids stored as BIGINT (smaller, faster than STRING)
ingest_ts       TIMESTAMP
source_system   STRING
```
- **Store H3 as `BIGINT`** (8 bytes) for joins/Z-order; provide `STRING` hex only on the serving/API layer (`h3_h3tostring`).
- Use **column comments** to document CRS and resolution: `ALTER TABLE t ALTER COLUMN h3_r8 COMMENT 'H3 res 8 cell id (BIGINT)'`.

### 2.3 Azure storage layout (medallion for geospatial)

```text
abfss://lake@<account>.dfs.core.windows.net/
  landing/geo/<source>/<yyyy>/<mm>/<dd>/      # raw files (GeoJSON, SHP zips, GPX, CSV)
  bronze/geo_<source>                          # Delta, raw WKB/WKT + minimal parse
  silver/geo_<entity>_clean                    # validated geometry, standardized CRS, H3 columns
  gold/geo_<product>_h3_r8_agg                 # aggregates, serving tables, features
  reference/geo_admin_boundaries               # polygons, road nets, grids (slowly changing)
```
Use **Unity Catalog** managed tables (or external locations with storage credentials) — `catalog.geo.silver_pois`.

---

## 3. Spatial Indexing — Concepts & Comparison

### 3.1 Why index?
Exact spatial predicates (`ST_Contains`) are CPU-heavy and do not map to equi-joins, so Spark would otherwise run a **cartesian / nested loop** join. An index converts spatial proximity into **keys** so Spark can use hash/sort-merge joins, partition pruning and file skipping.

### 3.2 Families of spatial indexes

| Family | Examples | Idea | Best for |
|---|---|---|---|
| **Discrete global grid systems (DGGS)** | **H3**, S2, Geohash, Uber H3, A5 🔎, ISEA3H | Hierarchical cells on sphere with ID | Distributed joins, aggregation, analytics at scale |
| **Tree-based (in-memory)** | R-tree, quadtree, KD-tree, STR-tree | Bounding-box hierarchy | Single-node / per-partition lookups (JTS STRtree, Sedona) |
| **Space-filling curves** | Z-order (Morton), Hilbert | Map 2-D → 1-D preserving locality | File clustering in Delta (`ZORDER`), Liquid Clustering |
| **Planar tiling** | XYZ/TMS tiles, Quadkey (Bing), MGRS | Square tiles at zoom levels | Web maps, raster |
| **Administrative** | Postal code, NUTS, census blocks | Human boundaries | Reporting; not uniform |

### 3.3 Comparison of grid systems

| Property | **H3** (Uber) | **S2** (Google) | **Geohash** | **Quadkey/XYZ** |
|---|---|---|---|---|
| Cell shape | Hexagon (+12 pentagons) | Quadrilateral (cube-face projection) | Rectangle (lat/lon) | Square (Mercator) |
| Resolutions | 0–15 | 0–30 | 1–12 chars | 0–~23+ |
| Neighbors | 6, all equidistant | 8 (edge vs corner differ) | 8 (distorted) | 8 |
| Hierarchy | Approx. (aperture 7; children not perfectly contained) | Exact (aperture 4) | Exact (prefix) | Exact |
| Area uniformity | Good | Very good | Poor near poles | Poor away from equator |
| ID | 64-bit int / 15-char hex | 64-bit int | string | int/string |
| Strength | Neighbor analysis, smooth gradients, ML features, built-in Databricks support | Exact containment/covering | Simple, prefix search | Web tiles |
| Weakness | Parent/child not exact; pentagons | Less intuitive; fewer analytics libs | Edge effects, distortion | Distortion |

### 3.4 Choosing index vs exact geometry
- **Cell join only** → approximate, very fast; good for density, coarse enrichment, features.
- **Cell join + exact refine** (the *"index then verify"* pattern) → exact results, 10–100× faster than naive.
- **Core vs border cells** (tessellation/Mosaic pattern): cells fully inside the polygon need **no geometry test**; only border cells need `ST_Contains`.

---

## 4. H3 In Depth

### 4.1 What is H3?
H3 is a hierarchical **hexagonal geospatial indexing system** (Uber, open source, Apache-2.0). Earth surface → icosahedron projection → 122 base cells → recursive refinement by **aperture 7** (each cell has ~7 children, rotated ≈19.1° per resolution).

### 4.2 Key facts
- **16 resolutions: 0 (coarsest) … 15 (finest, ≈0.9 m²)**.
- **122 base cells** (res 0); **12 pentagons per resolution** (at icosahedron vertices, placed over ocean mostly). Pentagons have 5 neighbors and break some assumptions (e.g. `hexring`/k-ring may fail or distort near pentagons; some functions error out).
- **Hexagons have one neighbor distance** → clean gradients, smooth kernel smoothing.
- **Hierarchy is approximate** — children are not geometrically contained in the parent; they are *logically* children (the parent of a cell is always well-defined). Don't assume polygon containment of children's boundaries.
- Cell ID = 64-bit unsigned integer, usually displayed as 15-hex-char string (e.g. `8928308280fffff`). In Spark/Delta it is stored as signed `BIGINT`.

### 4.3 64-bit index layout

```text
bit  63   : reserved (0)
bits 59-62: mode (1 = cell, 2 = directed edge, 3 = undirected edge, 4 = vertex)
bits 56-58: mode-dependent (reserved / edge or vertex number)
bits 52-55: resolution (0-15)
bits 45-51: base cell (0-121)
bits 0-44 : 15 x 3-bit digits (child position per resolution, 7 = unused)
```

### 4.4 Resolution table (averages)

| Res | Avg hex edge length | Avg hex area | # cells (globally) | Typical use |
|---:|---:|---:|---:|---|
| 0 | 1107.71 km | 4,357,449 km² | 122 | continents |
| 1 | 418.68 km | 609,788 km² | 842 | |
| 2 | 158.24 km | 86,801 km² | 5,882 | large countries/regions |
| 3 | 59.81 km | 12,393 km² | 41,162 | |
| 4 | 22.61 km | 1,770 km² | 288,122 | states / provinces |
| 5 | 8.54 km | 252.9 km² | 2,016,842 | metro regions |
| 6 | 3.23 km | 36.13 km² | 14,117,882 | cities / districts |
| 7 | 1.22 km | 5.16 km² | 98,825,162 | districts / large neighborhoods |
| 8 | 461 m | 0.737 km² | 691,776,122 | neighborhoods |
| 9 | 174 m | 0.105 km² | 4,842,432,842 | city blocks clusters |
| 10 | 65.9 m | 15,047 m² | 33,897,029,882 | blocks |
| 11 | 24.9 m | 2,150 m² | 237,279,209,162 | |
| 12 | 9.42 m | 307 m² | 1,660,954,464,122 | buildings/parcels |
| 13 | 3.56 m | 43.9 m² | 11,626,681,248,842 | |
| 14 | 1.35 m | 6.3 m² | 81,386,768,741,882 | |
| 15 | 0.51 m | 0.9 m² | 569,707,381,193,162 | |

**Rule of thumb:** pick the resolution so a cell is **≈ the scale of the question** and yields **≥ 30–50 observations per cell on average** (avoid sparse cells *and* giant hot cells). Typical: **res 7–9 for city analytics, res 5–6 regional/country, res 10–12 for building-level**. Each +1 resolution ≈ 7× more cells / ÷7 area.

**Count math:** cells ≈ 2 + 120 × 7^res.  Area ≈ Earth surface (510 M km²) / cell count.

### 4.5 H3 concepts and terms

| Concept | Description |
|---|---|
| **Cell** | The hexagon/pentagon with an index. |
| **Parent / child / center child** | `toparent(res)`, `tochildren(res)`, `centerchild`. Children of a hex: 7 (6 for pentagon children variants → pentagon has 6). |
| **Grid disk / k-ring** | All cells within *k* steps (including origin). Size for hex: 3k(k+1)+1. |
| **Grid ring / hexring** | Only the cells at exactly distance *k* (hollow ring). Size 6k. |
| **Grid distance** | Number of steps between cells (not meters). |
| **Grid path** | Shortest line of cells between two cells. |
| **Compaction** | Replace sets of 7 siblings with parent (reduces count). `h3_compact`. |
| **Uncompaction** | Expand a compacted set back to a single resolution. `h3_uncompact`. |
| **Polyfill** | Cells whose **center** lies in polygon (`polygon_to_cells`). Border may be missed/extra. |
| **Cover** | Cells that **cover** a geometry (minimum covering set; `h3_coverash3`). |
| **Tessellate** | Cells + clipped geometry chips per cell (Mosaic / `h3_tessellateaswkb`). |
| **Directed edge** | Edge between two neighboring cells (mode 2). Used for flows/OD graphs. |
| **Vertex** | Corner of a cell (mode 4). Used for dissolving shared borders. |
| **Local IJ coordinates** | Local 2-D coordinate system around an origin cell (for algorithms); no global validity. |
| **Pentagon distortion** | Pentagons: 5 neighbors, some algorithms undefined or error. |

### 4.6 When to use H3
✅ Aggregating/point density, hotspot analysis, ML features (neighbors' features), joins at scale, uniform comparison units, dispatch/demand-supply, geo-sharding keys, privacy-preserving coarse location, API response keys.
❌ Exact area/length computations on small shapes, legal boundaries, high-accuracy topology/overlay, anything requiring exact hierarchy containment (use S2/quadkeys), cadastral precision.

### 4.7 Choosing which H3 representation to carry

| Representation | Type | Pro | Con |
|---|---|---|---|
| Integer (`BIGINT`) | 8 B | Smallest, fast join/Z-order | Not human readable; unsigned→signed on JVM |
| Hex string | 15 chars | Readable, JSON-safe, JS-safe | 2–4× bigger, slower join |
| Multiple resolutions as columns | `h3_r6, h3_r8, h3_r10` | Fast multi-scale roll-ups without recomputing | Storage overhead |

**Best practice:** compute `h3_r<max_res>` once; derive coarser levels with `h3_toparent` (cheap, no geometry math) or materialize columns for hot paths.

---

## 5. H3 Function Reference

### 5.1 Databricks SQL / PySpark built-in H3 functions (Photon-accelerated; no library install)
Available in Databricks SQL and Databricks Runtime (Photon recommended). Functions accept/return `BIGINT` or `STRING` cells depending on the variant. 🔎 Check current docs for the full list/behaviors in your runtime.

#### Import / conversion (coordinates → cells)

| Function | Returns | Description / Example |
|---|---|---|
| `h3_longlatash3(lon, lat, res)` | BIGINT | Point (lon,lat) → cell. `SELECT h3_longlatash3(8.6821, 50.1109, 9)` |
| `h3_longlatash3string(lon, lat, res)` | STRING | Same, hex string. |
| `h3_pointash3(geog_or_wkb_geom, res)` | BIGINT | Point (WKB/EWKB/WKT/GeoJSON accepted) → cell. |
| `h3_pointash3string(geom, res)` | STRING | |
| `h3_polyfillash3(geom, res)` | ARRAY<BIGINT> | Cells whose **center** is inside polygon/multipolygon (WKT/WKB/GeoJSON). |
| `h3_polyfillash3string(geom, res)` | ARRAY<STRING> | |
| `h3_try_polyfillash3(…)` / `…string` | array or NULL | Same, returns NULL instead of error for invalid input. |
| `h3_coverash3(geom, res)` | ARRAY<BIGINT> | Minimal set of cells **covering** the geometry (any geometry type, incl. lines/points). |
| `h3_coverash3string(geom, res)` | ARRAY<STRING> | |
| `h3_try_coverash3(…)` / `…string` | | Safe variants. |
| `h3_tessellateaswkb(geom, res)` | ARRAY<STRUCT<cellid, core, chip>> | Cells plus clipped geometry chips (`core` = fully inside, chip = WKB of intersection). 🔎 Newer runtimes. |

#### Export (cells → geometry)

| Function | Returns | Description |
|---|---|---|
| `h3_boundaryaswkt(cell)` | STRING | Polygon boundary WKT. |
| `h3_boundaryaswkb(cell)` | BINARY | Boundary WKB (preferred for Spark/Delta). |
| `h3_boundaryasgeojson(cell)` | STRING | Boundary GeoJSON. |
| `h3_centeraswkt(cell)` / `h3_centeraswkb` / `h3_centerasgeojson` | | Cell center point. |

#### Hierarchy

| Function | Returns | Description |
|---|---|---|
| `h3_resolution(cell)` | INT | Resolution 0–15. |
| `h3_toparent(cell, res)` | BIGINT/STRING | Ancestor at coarser res. |
| `h3_tochildren(cell, res)` | ARRAY | All descendants at finer res. |
| `h3_minchild(cell, res)` / `h3_maxchild(cell, res)` | | Min / max child ID (use for range predicates on sorted data!). |
| `h3_ischildof(cell, parent)` | BOOLEAN | |
| `h3_compact(array_of_cells)` | ARRAY | Compact. |
| `h3_uncompact(array_of_cells, res)` | ARRAY | Expand to res. |

> **Performance trick:** `WHERE cell BETWEEN h3_minchild(p, r) AND h3_maxchild(p, r)` becomes a min/max range scan on Delta stats, ideal for parent-level partition/Z-order pruning.

#### Traversal

| Function | Returns | Description |
|---|---|---|
| `h3_kring(cell, k)` | ARRAY | All cells within k steps (filled disk). |
| `h3_kringdistances(cell, k)` | ARRAY<STRUCT<cellid, distance>> | Disk with distances. |
| `h3_hexring(cell, k)` | ARRAY | Hollow ring at exactly k. |
| `h3_distance(c1, c2)` | INT | Grid distance (steps). `h3_try_distance` returns NULL on failure (e.g. too far / pentagon). |

#### Conversion & validation

| Function | Description |
|---|---|
| `h3_h3tostring(bigint)` / `h3_stringtoh3(str)` | BIGINT ⇄ hex string. |
| `h3_isvalid(x)` | BOOLEAN — valid cell (BIGINT or STRING)? |
| `h3_validate(x)` | Returns the cell or **throws** if invalid. |
| `h3_try_validate(x)` | Returns the cell or **NULL** if invalid (use in pipelines). |

#### SQL snippets
```sql
-- Point → H3 at three resolutions
SELECT lon, lat,
       h3_longlatash3(lon, lat, 6)  AS h3_r6,
       h3_longlatash3(lon, lat, 8)  AS h3_r8,
       h3_longlatash3(lon, lat, 10) AS h3_r10
FROM   bronze.gps_pings;

-- Explode polygon to cells
SELECT zone_id, explode(h3_polyfillash3(geom_wkt, 9)) AS h3_r9
FROM   ref.zones;

-- Neighbor aggregation (smoothing, k=2)
SELECT c.h3_r8 AS center, SUM(a.cnt) AS cnt_smoothed
FROM  (SELECT h3_r8, explode(h3_kring(h3_r8, 2)) AS nb FROM agg.cells) c
JOIN   agg.cells a ON a.h3_r8 = c.nb
GROUP BY c.h3_r8;

-- Cell geometry for BI / maps
SELECT h3_r8, h3_boundaryaswkt(h3_r8) AS wkt, h3_h3tostring(h3_r8) AS h3_hex FROM agg.cells;

-- Parent roll-up
SELECT h3_toparent(h3_r10, 7) AS h3_r7, COUNT(*) FROM silver.pings GROUP BY 1;
```

#### PySpark equivalents
```python
from pyspark.sql import functions as F

df = (spark.table("bronze.gps_pings")
        .withColumn("h3_r8", F.expr("h3_longlatash3(lon, lat, 8)"))
        .withColumn("h3_hex", F.expr("h3_h3tostring(h3_r8)"))
        .withColumn("nbrs", F.expr("h3_kring(h3_r8, 1)")))
```
(In Databricks, H3 expressions are called via `F.expr` / `selectExpr`/SQL; there are also DataFrame-API wrappers in recent `pyspark.databricks.sql.functions` 🔎 — e.g. `h3_longlatash3`.)

---

### 5.2 Open-source `h3` Python library (v4.x) — single-node / UDF use

```bash
%pip install h3          # v4.x API (v3 names differ!)
```

| v4 function | Purpose | v3 name (legacy) |
|---|---|---|
| `h3.latlng_to_cell(lat, lng, res)` | point → cell (**lat first!**) | `geo_to_h3` |
| `h3.cell_to_latlng(cell)` | center | `h3_to_geo` |
| `h3.cell_to_boundary(cell)` | boundary tuple of (lat,lng) | `h3_to_geo_boundary` |
| `h3.str_to_int(s)` / `h3.int_to_str(i)` | hex ⇄ int | — |
| `h3.is_valid_cell(c)` | validation | `h3_is_valid` |
| `h3.is_pentagon(c)` | | `h3_is_pentagon` |
| `h3.is_res_class_III(c)` | orientation class | |
| `h3.get_resolution(c)` | | `h3_get_resolution` |
| `h3.get_base_cell_number(c)` | | |
| `h3.cell_to_parent(c, res)` | | `h3_to_parent` |
| `h3.cell_to_children(c, res)` | | `h3_to_children` |
| `h3.cell_to_center_child(c, res)` | | `h3_to_center_child` |
| `h3.cell_to_children_size(c, res)` | count | |
| `h3.cell_to_child_pos(c, res)` / `h3.child_pos_to_cell(parent, res, pos)` | position in sibling order | |
| `h3.grid_disk(c, k)` | filled disk | `k_ring` |
| `h3.grid_ring(c, k)` | hollow ring | `hex_ring` |
| `h3.grid_distance(a, b)` | steps | `h3_distance` |
| `h3.grid_path_cells(a, b)` | cell path | `h3_line` |
| `h3.are_neighbor_cells(a, b)` | | `h3_indexes_are_neighbors` |
| `h3.compact_cells(cells)` / `h3.uncompact_cells(cells, res)` | | `compact`/`uncompact` |
| `h3.polygon_to_cells(h3shape, res)` (alias `h3shape_to_cells`) | polyfill (center-in-polygon) | `polyfill` |
| `h3.cells_to_h3shape(cells)` / `h3.cells_to_geo(cells)` | cells → polygon(s)/GeoJSON-like | `h3_set_to_multi_polygon` |
| `h3.LatLngPoly(outer, *holes)` / `h3.LatLngMultiPoly(...)` | shape objects (lat,lng order) | |
| `h3.cell_area(c, unit='km^2')` | exact cell area | `cell_area` |
| `h3.average_hexagon_area(res, unit)` / `h3.average_hexagon_edge_length(res, unit)` | averages | `hex_area` |
| `h3.edge_length(edge, unit)` / `h3.great_circle_distance(p1, p2, unit)` | | |
| `h3.get_num_cells(res)` / `h3.get_res0_cells()` / `h3.get_pentagons(res)` | globals | |
| `h3.origin_to_directed_edges(c)`, `h3.cells_to_directed_edge(a, b)`, `h3.directed_edge_to_cells(e)`, `h3.get_directed_edge_origin/destination(e)`, `h3.is_valid_directed_edge(e)` | directed edges | |
| `h3.cell_to_vertexes(c)`, `h3.cell_to_vertex(c, n)`, `h3.vertex_to_latlng(v)`, `h3.is_valid_vertex(v)` | vertices | |
| `h3.cell_to_local_ij(origin, c)` / `h3.local_ij_to_cell(origin, i, j)` | local IJ | |

```python
import h3
cell = h3.latlng_to_cell(50.1109, 8.6821, 9)        # Frankfurt, res 9   (LAT, LNG!)
h3.cell_to_latlng(cell)
h3.grid_disk(cell, 2)                                # 19 cells
h3.cell_to_parent(cell, 7)
poly = h3.LatLngPoly([(50.12,8.65),(50.12,8.72),(50.08,8.72),(50.08,8.65)])
cells = h3.polygon_to_cells(poly, 9)
```

**⚠ v3→v4 gotchas:** renamed functions; `polygon_to_cells` takes `H3Shape` objects; GeoJSON-style (lng,lat) input is handled by `h3.geo_to_h3shape(geojson_dict)`. In UDFs always import inside or broadcast the module; install on cluster (library) not just notebook for jobs.

#### Pandas UDF pattern (when SQL function not enough)
```python
import pandas as pd, h3
from pyspark.sql.functions import pandas_udf

@pandas_udf("array<string>")
def disk_udf(cells: pd.Series, k: pd.Series) -> pd.Series:
    return pd.Series([h3.grid_disk(c, int(kk)) for c, kk in zip(cells, k)])
```
Prefer built-in Photon H3 SQL functions (10–100× faster than Python UDFs); use UDFs only for unsupported functions (`grid_path_cells`, edges, vertices, local IJ, area exact).

---

### 5.3 Databricks Labs Mosaic H3-related functions (legacy/maintenance mode 🔎)
Mosaic (`databricks-mosaic`) provided `grid_*` and `st_*` expressions with a choice of index system (H3 or BNG/custom):

| Mosaic function | Purpose |
|---|---|
| `grid_tessellate(geom, res, keep_core_geom)` | cells + core flag + chip geometry |
| `grid_tessellateexplode` | exploded rows |
| `grid_pointascellid(pt, res)` | point → cell |
| `grid_polyfill(geom, res)` | center-based fill |
| `grid_boundaryaswkb(cell)` | cell geometry |
| `grid_cellkring / grid_cellkloop` | neighbors |
| `grid_geometrykring/kloop` | rings around geometries |
| `st_*` (`st_area, st_buffer, st_intersects, st_contains, st_transform, st_simplify, st_unaryunion…`) | geometry functions (JTS-based) |
| `mos.enable_mosaic(spark, dbutils)`; `mos.enable_gdal(spark)` | init (GDAL for raster/OGR readers) |

> For new work prefer **built-in H3 SQL + native `ST_` functions** (and Sedona where more functionality is needed); keep Mosaic only for existing code or its raster/GDAL capabilities.

---
## 6. Other Spatial Tooling on Databricks

### 6.1 Native Databricks spatial SQL (🔎 Public Preview / rolling out, DBR 17.1+ / SQL warehouse)
- **Types:** `GEOMETRY(srid)` and `GEOGRAPHY(srid)` (Delta-supported; Photon-accelerated).
- **Representative `ST_` functions:** constructors `st_point`, `st_geomfromtext/wkt`, `st_geomfromwkb`, `st_geomfromgeojson`, `st_geogfromtext`; accessors `st_x`, `st_y`, `st_srid`, `st_geometrytype`, `st_isvalid`; outputs `st_astext`, `st_asbinary`, `st_asgeojson`; relations `st_contains`, `st_within`, `st_intersects`, `st_touches`, `st_covers`, `st_disjoint`; measures `st_area`, `st_length`, `st_distance`, `st_distancesphere`; processing `st_buffer`, `st_centroid`, `st_envelope`, `st_simplify`, `st_union`, `st_intersection`, `st_transform`, `st_setsrid`.
- **Verify** the exact list: `SHOW FUNCTIONS LIKE 'st_*'` in your workspace.
```sql
SELECT p.poi_id, z.zone_id
FROM   silver.pois p JOIN ref.zones z
  ON   ST_Contains(z.geom, ST_Point(p.lon, p.lat, 4326));
```
- Databricks can **accelerate spatial joins** (Photon spatial join) when predicates like `ST_Contains/Intersects/Within` are used between two tables 🔎 — still pre-filtering with H3 is a robust, version-independent pattern.

### 6.2 Apache Sedona (open source; `apache-sedona`, `sedona-spark-shaded`)
- Spark SQL `ST_*` (~300 functions), vector + raster, spatial partitioning (KDB-tree/quadtree), spatial join optimizer, readers: GeoParquet, GeoJSON, Shapefile, GeoPackage, WKT/WKB, raster (GeoTiff, NetCDF via raster).
- Setup on Databricks: install Maven coordinates `org.apache.sedona:sedona-spark-shaded-3.5_2.12:<ver>` + `org.datasyslabs:geotools-wrapper:<ver>`, PyPI `apache-sedona`; cluster Spark config `spark.serializer org.apache.spark.serializer.KryoSerializer`, `spark.kryo.registrator org.apache.sedona.core.serde.SedonaKryoRegistrator`, `spark.sql.extensions org.apache.sedona.sql.SedonaSqlExtensions`.
- Not Photon-accelerated (runs in JVM codegen) but very broad.
```python
from sedona.spark import SedonaContext
sedona = SedonaContext.create(spark)
sedona.sql("SELECT ST_Distance(ST_Point(8.68,50.11), ST_Point(8.70,50.12))").show()
```

### 6.3 Single-node Python libs (driver or pandas UDF)
| Lib | Use |
|---|---|
| **GeoPandas** + **Shapely 2** + **pyproj** + **pyogrio** | vector ops, CRS transforms, file IO |
| **rasterio / rioxarray / xarray / odc-stac / stackstac** | raster, STAC |
| **osmnx / networkx / igraph / pyrosm** | OSM + graph |
| **folium / pydeck / lonboard / keplergl / leafmap** | visualization in notebooks |
| **h3pandas** | H3 on GeoDataFrames |
| **duckdb + spatial ext.** | fast single-node spatial SQL, GeoParquet |
| **shapely.STRtree** | per-partition spatial index in `mapInPandas` |

### 6.4 Choosing the tool

| Need | Prefer |
|---|---|
| Point→cell, polyfill, aggregation, neighbors at scale | **Built-in H3 SQL** (Photon) |
| Exact predicates/measures at scale | **Native ST_** (if available) else **Sedona** |
| Raster at scale | Sedona raster / Mosaic+GDAL / rasterio in UDF |
| Routing/isochrones | External engine (OSRM/Valhalla/GraphHopper) in AKS/ACI + call via UDF/batch API; or Azure Maps Route API |
| Complex single-geometry processing on small data | GeoPandas/Shapely on driver |
| Visualization | Kepler/pydeck/lonboard in notebook; Power BI; Azure Maps; Databricks Apps |

---

## 7. Core Processing Patterns

### 7.1 Pattern catalog

| # | Pattern | Steps | Notes |
|---|---|---|---|
| P1 | **Point indexing** | lon/lat → `h3_longlatash3` → store BIGINT | Validate ranges first |
| P2 | **Point-in-polygon via H3 (approximate)** | polyfill polygons → explode → join on cell | Fast; error at borders |
| P3 | **Index-then-verify (exact)** | P2 as candidate filter → `ST_Contains` on candidates | Exact + fast |
| P4 | **Core/border tessellation** | tessellate polygons; core cells join w/o geometry test; border cells verify with chip | Best accuracy/perf ratio |
| P5 | **Density / heatmap** | `GROUP BY h3` (+ time bucket) → counts | Choose resolution for ≥30 obs/cell |
| P6 | **Smoothing / kernel** | `kring(k)` explode → weighted aggregate | Weights by `h3_kringdistances` |
| P7 | **Nearest-neighbor / radius search** | candidate = `kring(cell, k)` → exact distance → top-n | k from radius / edge length |
| P8 | **Hotspot detection** | cell counts → neighbor stats (Getis-Ord Gi*, Local Moran) | Use hex neighbors as weights |
| P9 | **Trajectory → cells sequence** | order pings by ts → cell per ping → run-length collapse → `h3_distance` | Flows, OD matrices |
| P10 | **Stay-point / dwell** | window: consecutive pings same cell/ring | Use res 10–11 |
| P11 | **OD flow aggregation** | origin cell, destination cell (+ time) → count | Directed edges for adjacency flows |
| P12 | **Multi-scale roll-up** | store finest res; `toparent` for coarser | Dashboard drill-down |
| P13 | **Geofencing (streaming)** | broadcast fence cells set; stream-static join on cell; verify edge cases | Structured Streaming |
| P14 | **Catchment / isochrone → cells** | isochrone polygon → polyfill → join demand per cell | Routing engine external |
| P15 | **Areal interpolation** | polygon→cells, distribute attributes by cell share → re-aggregate | Census → grid |
| P16 | **Geo-sharding / partition key** | `h3_toparent(cell, 3–5)` as partition/cluster key | Avoid > ~10k partitions |
| P17 | **Privacy aggregation (k-anonymity)** | aggregate to cells; suppress cells with n < k | Raise res only if k satisfied |
| P18 | **Feature engineering for ML** | per-cell features + neighbors' features (ring 1..k) | Embeddings, demand forecasting |

### 7.2 Choosing the join resolution (index-then-verify)
- Finer res → fewer false-positives, more cells per polygon (explode cost ↑).
- Coarser res → fewer rows, more refinement work.
- Heuristic: choose res where polygon ≈ **10–1000 cells**; test with sampling and `EXPLAIN`/Spark UI to compare shuffle volume.

### 7.3 Handling skew
- Hot cells (city centres) → **salt** the key (`concat(cell, pmod(hash(id), N))`), or use two-level approach (coarse for sparse, fine for dense = adaptive resolution), or `compact` polygons.
- Enable AQE: `spark.sql.adaptive.enabled=true`, `spark.sql.adaptive.skewJoin.enabled=true`.
- Broadcast small side: `/*+ BROADCAST(z) */` for polygon-cell table < ~100–500 MB.

### 7.4 Edge cases
- **Antimeridian (±180°)** polygons: split at the antimeridian before polyfill.
- **Poles:** lon/lat polygons around poles misbehave; use geography types or special handling.
- **Pentagons:** wrap `kring/hexring/distance` with `try_*` variants; accept NULL/exception handling.
- **Polygons smaller than a cell**: polyfill may return **0 cells** → use `coverash3` or tessellate or finer res.
- **Holes/MultiPolygons**: `h3_polyfillash3` handles holes; verify with test cases.
- **Invalid WKT/WKB:** use `h3_try_*` and route NULL results to a quarantine table.
- **Line features:** use `h3_coverash3` (works for lines) or densify + point index.

---

## 8. Sample Project Code

> All examples use Unity Catalog three-level names `geo_catalog.<schema>.<table>`. Replace with your own. Photon-enabled cluster / SQL warehouse recommended.

### 8.0 Project layout (Databricks Asset Bundle)

```text
geo-platform/
├── databricks.yml                 # bundle definition (jobs, pipelines, targets: dev/test/prod)
├── resources/
│   ├── jobs_geo_batch.yml
│   ├── pipeline_geo_dlt.yml
│   └── warehouse_serving.yml
├── src/
│   ├── geo_lib/                   # reusable python package
│   │   ├── __init__.py
│   │   ├── h3_utils.py
│   │   ├── validation.py
│   │   └── config.py
│   ├── notebooks/
│   │   ├── 01_ingest_reference.py
│   │   ├── 02_build_zone_cells.py
│   │   ├── 03_enrich_points.py
│   │   └── 04_publish_gold.py
│   └── pipelines/geo_dlt.py
├── sql/
│   ├── views/vw_cells_geojson.sql
│   └── functions/f_cells_in_radius.sql
├── tests/
│   ├── unit/test_h3_utils.py
│   └── integration/test_enrichment.py
├── .github/workflows/ci.yml       # or azure-pipelines.yml
└── README.md
```

### 8.1 Utility module (`geo_lib/h3_utils.py`)

```python
"""Reusable, testable H3 helpers built on Spark SQL expressions."""
from pyspark.sql import DataFrame, functions as F

VALID_LON = (-180.0, 180.0)
VALID_LAT = (-90.0, 90.0)

def with_valid_coords(df: DataFrame, lon="lon", lat="lat") -> tuple[DataFrame, DataFrame]:
    """Split into (valid, quarantined) based on coordinate sanity."""
    cond = (F.col(lon).between(*VALID_LON) & F.col(lat).between(*VALID_LAT)
            & F.col(lon).isNotNull() & F.col(lat).isNotNull()
            & ~((F.col(lon) == 0) & (F.col(lat) == 0)))      # "null island" guard
    return df.filter(cond), df.filter(~cond)

def add_h3(df: DataFrame, res: int | list[int], lon="lon", lat="lat", prefix="h3_r") -> DataFrame:
    """Add one or more H3 BIGINT columns."""
    resolutions = [res] if isinstance(res, int) else res
    for r in resolutions:
        df = df.withColumn(f"{prefix}{r}", F.expr(f"h3_longlatash3({lon}, {lat}, {r})"))
    return df

def parent_cols(df: DataFrame, base_col: str, targets: list[int], prefix="h3_r") -> DataFrame:
    for r in targets:
        df = df.withColumn(f"{prefix}{r}", F.expr(f"h3_toparent({base_col}, {r})"))
    return df

def polygons_to_cells(df: DataFrame, geom_col: str, res: int, id_col: str) -> DataFrame:
    """Explode polygons into rows of (id, cell). Uses safe variant, NULL -> dropped."""
    return (df.select(id_col, F.explode_outer(F.expr(f"h3_try_polyfillash3({geom_col}, {res})")).alias(f"h3_r{res}"))
              .filter(F.col(f"h3_r{res}").isNotNull()))
```

### 8.2 Reference data → polygon cell lookup (`02_build_zone_cells.py`)

```python
# Databricks notebook source
from geo_lib.h3_utils import polygons_to_cells
RES = 9

zones = spark.table("geo_catalog.ref.admin_zones")        # zone_id, name, geom_wkt (EPSG:4326)

# Strategy A: cheap polyfill (centre-in-polygon)
zone_cells = polygons_to_cells(zones, "geom_wkt", RES, "zone_id")

# Strategy B (robust for small zones): union with cover for tiny polygons
zone_cells_cover = (zones
    .select("zone_id", F.explode(F.expr(f"h3_try_coverash3(geom_wkt, {RES})")).alias(f"h3_r{RES}"))
    .where("zone_id NOT IN (SELECT DISTINCT zone_id FROM zone_cells_view)"))   # only fill gaps

(zone_cells.write.mode("overwrite")
   .option("overwriteSchema","true")
   .saveAsTable("geo_catalog.silver.zone_cells_r9"))

spark.sql("ALTER TABLE geo_catalog.silver.zone_cells_r9 CLUSTER BY (h3_r9)")   # Liquid Clustering
```

### 8.3 Enrich points with zones — approximate join

```sql
CREATE OR REPLACE TABLE geo_catalog.silver.pings_enriched
CLUSTER BY (h3_r9) AS
SELECT p.*, zc.zone_id
FROM   (SELECT *, h3_longlatash3(lon, lat, 9) AS h3_r9
        FROM   geo_catalog.bronze.gps_pings
        WHERE  lon BETWEEN -180 AND 180 AND lat BETWEEN -90 AND 90) p
LEFT   JOIN geo_catalog.silver.zone_cells_r9 zc USING (h3_r9);
```
Cells covered by >1 polygon create duplicates → decide on rule (`row_number()` by priority/area) or accept multi-membership.

### 8.4 Exact enrichment — index-then-verify with core/border split

```python
from pyspark.sql import functions as F

RES = 9
# 1) Tessellate zones → (zone_id, cell, is_core, chip_wkb)
tess = (spark.table("geo_catalog.ref.admin_zones")
        .select("zone_id", F.explode(F.expr(f"h3_tessellateaswkb(geom_wkb, {RES})")).alias("t"))
        .select("zone_id", F.col("t.cellid").alias("h3_r9"), F.col("t.core").alias("is_core"), F.col("t.chip").alias("chip_wkb")))
tess.write.mode("overwrite").saveAsTable("geo_catalog.silver.zone_tess_r9")

pts = (spark.table("geo_catalog.bronze.gps_pings")
       .withColumn("h3_r9", F.expr(f"h3_longlatash3(lon, lat, {RES})")))

joined = pts.join(F.broadcast(spark.table("geo_catalog.silver.zone_tess_r9")), "h3_r9")

result = joined.where("is_core OR ST_Contains(ST_GeomFromWKB(chip_wkb), ST_Point(lon, lat))") \
               .select("ping_id", "zone_id", "ts", "lon", "lat", "h3_r9")
```
(If `h3_tessellateaswkb` / native `ST_` are unavailable in your runtime, use polyfill + Sedona `ST_Contains` on the geometry for border cells only; border cells = `coverash3` minus `polyfill`-fully-inside set.)

### 8.5 Density aggregation + time bucketing (gold)

```sql
CREATE OR REPLACE TABLE geo_catalog.gold.density_r8_hourly
CLUSTER BY (h3_r8, hour_ts) AS
SELECT h3_toparent(h3_r9, 8)                 AS h3_r8,
       date_trunc('HOUR', ts)                AS hour_ts,
       COUNT(*)                              AS pings,
       COUNT(DISTINCT device_id)             AS devices,
       percentile_approx(speed_kmh, 0.5)     AS speed_p50
FROM   geo_catalog.silver.pings_enriched
GROUP  BY 1, 2
HAVING COUNT(DISTINCT device_id) >= 5;       -- k-anonymity guard
```

### 8.6 Radius / nearest-neighbour search with k-ring

```sql
-- Radius in meters → k via edge length (res 9 avg edge ≈ 174 m; cell-center spacing ≈ 1.73 × edge ≈ 301 m)
-- k = ceil(radius_m / (edge_m * sqrt(3))) + 1 (buffer)
CREATE OR REPLACE FUNCTION geo_catalog.geo.f_cells_in_radius(lon DOUBLE, lat DOUBLE, radius_m DOUBLE)
RETURNS ARRAY<BIGINT>
RETURN h3_kring(h3_longlatash3(lon, lat, 9), CAST(ceil(radius_m / 301.0) + 1 AS INT));

-- Candidates → exact great-circle distance
WITH q AS (SELECT 8.6821 AS lon, 50.1109 AS lat, 1000.0 AS r),
cand AS (
  SELECT p.*
  FROM   geo_catalog.silver.pois p, q
  WHERE  p.h3_r9 IN (SELECT explode(geo_catalog.geo.f_cells_in_radius(q.lon, q.lat, q.r)))
)
SELECT poi_id, name,
       2 * 6371008.8 * asin(sqrt(
          pow(sin(radians(lat - 50.1109)/2),2) +
          cos(radians(50.1109))*cos(radians(lat))*pow(sin(radians(lon - 8.6821)/2),2))) AS dist_m
FROM   cand
HAVING dist_m <= 1000
ORDER  BY dist_m LIMIT 20;
```
For bulk "each point vs. nearest N POIs": explode `kring` per point, join POIs on cell, `row_number() OVER (PARTITION BY point_id ORDER BY dist)`; increase k adaptively where no match is found.

### 8.7 Trajectories → cell sequences & OD flows

```python
from pyspark.sql import Window, functions as F
w = Window.partitionBy("device_id").orderBy("ts")

traj = (spark.table("geo_catalog.silver.pings_enriched")
        .withColumn("cell", F.expr("h3_longlatash3(lon, lat, 9)"))
        .withColumn("prev_cell", F.lag("cell").over(w))
        .withColumn("changed", (F.col("cell") != F.col("prev_cell")) | F.col("prev_cell").isNull())
        .withColumn("seg", F.sum(F.col("changed").cast("int")).over(w)))

stays = (traj.groupBy("device_id", "seg", "cell")
         .agg(F.min("ts").alias("t_start"), F.max("ts").alias("t_end"), F.count("*").alias("n"))
         .withColumn("dwell_s", F.col("t_end").cast("long") - F.col("t_start").cast("long")))

moves = (stays.withColumn("next_cell", F.lead("cell").over(Window.partitionBy("device_id").orderBy("t_start")))
         .where("next_cell IS NOT NULL")
         .groupBy(F.col("cell").alias("origin_r9"), F.col("next_cell").alias("dest_r9"))
         .count())
```

### 8.8 Hotspot / neighbor statistics (Getis–Ord style)

```sql
WITH cells AS (SELECT h3_r8, pings AS x FROM geo_catalog.gold.density_r8_daily WHERE day = '2026-10-01'),
stats AS (SELECT AVG(x) AS mu, STDDEV_POP(x) AS sd, COUNT(*) AS n FROM cells),
nbr AS (
  SELECT c.h3_r8, SUM(n.x) AS sum_nbr, COUNT(n.x) AS w
  FROM  (SELECT h3_r8, explode(h3_kring(h3_r8, 1)) AS nb FROM cells) c
  JOIN   cells n ON n.h3_r8 = c.nb
  GROUP  BY c.h3_r8
)
SELECT nbr.h3_r8,
       (sum_nbr - mu*w) / (sd * sqrt((n*w - w*w)/(n-1))) AS gi_star_z
FROM   nbr CROSS JOIN stats
ORDER  BY gi_star_z DESC;
```
z > 1.96 ≈ hot (95%), z > 2.58 ≈ hot (99%); negative → cold.

### 8.9 Delta Live Tables / Lakeflow Declarative Pipelines (streaming enrichment)

```python
import dlt
from pyspark.sql import functions as F

@dlt.table(comment="Raw GPS pings from Event Hubs (Kafka API)")
def bronze_pings():
    return (spark.readStream.format("kafka")
        .option("kafka.bootstrap.servers", "<ns>.servicebus.windows.net:9093")
        .option("subscribe", "gps")
        .option("kafka.security.protocol", "SASL_SSL")
        .option("kafka.sasl.mechanism", "PLAIN")
        .option("kafka.sasl.jaas.config",
                'kafkashaded.org.apache.kafka.common.security.plain.PlainLoginModule required '
                'username="$ConnectionString" password="{{secrets/geo/eh-conn}}";')
        .load()
        .select(F.from_json(F.col("value").cast("string"), "device_id string, ts timestamp, lon double, lat double, speed double").alias("j"))
        .select("j.*"))

@dlt.table(comment="Validated + indexed")
@dlt.expect_or_drop("valid_lon", "lon BETWEEN -180 AND 180")
@dlt.expect_or_drop("valid_lat", "lat BETWEEN -90 AND 90")
@dlt.expect("not_null_island", "NOT (lon = 0 AND lat = 0)")
def silver_pings():
    return (dlt.read_stream("bronze_pings")
            .withWatermark("ts", "10 minutes")
            .dropDuplicates(["device_id", "ts"])
            .withColumn("h3_r9", F.expr("h3_longlatash3(lon, lat, 9)")))

@dlt.table(cluster_by=["h3_r9"])
def silver_pings_zone():
    zones = spark.table("geo_catalog.silver.zone_cells_r9")          # static dimension, broadcast
    return dlt.read_stream("silver_pings").join(F.broadcast(zones), "h3_r9", "left")

@dlt.table
def gold_density_5min():
    return (dlt.read_stream("silver_pings_zone")
            .groupBy(F.window("ts", "5 minutes"), F.expr("h3_toparent(h3_r9, 8)").alias("h3_r8"))
            .agg(F.count("*").alias("pings"), F.approx_count_distinct("device_id").alias("devices")))
```

### 8.10 Geofence alerting (stateful streaming)

```python
fence_cells = spark.table("geo_catalog.silver.fence_cells_r10")     # fence_id, h3_r10 (static)

stream = (spark.readStream.table("geo_catalog.silver.pings")
          .withColumn("h3_r10", F.expr("h3_longlatash3(lon, lat, 10)"))
          .join(F.broadcast(fence_cells), "h3_r10"))                 # stream-static join

def foreach_batch(df, epoch):
    # exact check only for border cells if needed; then write events + notify
    (df.select("device_id","fence_id","ts").write.mode("append").saveAsTable("geo_catalog.gold.fence_events"))

(stream.writeStream.foreachBatch(foreach_batch)
 .option("checkpointLocation", "abfss://chk@<acct>.dfs.core.windows.net/fence")
 .trigger(processingTime="10 seconds").start())
```
Enter/exit detection: keep last-fence state per device with `applyInPandasWithState` / `transformWithState` 🔎.

### 8.11 Publish cells as GeoJSON (serving view)

```sql
CREATE OR REPLACE VIEW geo_catalog.serving.vw_density_geojson AS
SELECT h3_h3tostring(h3_r8)                    AS h3,
       hour_ts, pings, devices,
       h3_boundaryasgeojson(h3_r8)             AS geometry_geojson
FROM   geo_catalog.gold.density_r8_hourly;
```

### 8.12 Visualization in a notebook (pydeck H3 layer)

```python
import pydeck as pdk, pandas as pd
pdf = spark.sql("""SELECT h3_h3tostring(h3_r8) AS hex, pings
                   FROM geo_catalog.gold.density_r8_hourly
                   WHERE hour_ts = '2026-10-26 08:00:00'""").toPandas()
layer = pdk.Layer("H3HexagonLayer", pdf, get_hexagon="hex", get_fill_color="[255, 255 - pings/10, 0, 160]",
                  get_elevation="pings", extruded=True, elevation_scale=5, pickable=True)
pdk.Deck(layers=[layer], initial_view_state=pdk.ViewState(latitude=50.11, longitude=8.68, zoom=9, pitch=45))
```
Limit to ≲ 100k cells per render; aggregate/limit in Spark first.

### 8.13 Python library test (`tests/unit/test_h3_utils.py`)

```python
import pytest
from pyspark.sql import SparkSession
from geo_lib.h3_utils import with_valid_coords

@pytest.fixture(scope="session")
def spark(): return SparkSession.builder.master("local[1]").getOrCreate()

def test_quarantine(spark):
    df = spark.createDataFrame([(8.68, 50.11), (200, 10), (0.0, 0.0), (None, 5)], "lon double, lat double")
    ok, bad = with_valid_coords(df)
    assert ok.count() == 1 and bad.count() == 3
```
(H3 SQL functions require Databricks runtime; use `databricks-connect` or integration tests on a small cluster/serverless for those, and pure-Python `h3` for local unit tests.)

---

## 9. Delta Lake Layout & Performance Tuning

### 9.1 Data layout options for spatial data

| Technique | How | When |
|---|---|---|
| **Liquid Clustering** | `CLUSTER BY (h3_r8)` (or `ALTER TABLE … CLUSTER BY`) | **Default recommendation** for new Delta tables; incremental, handles changing keys; use with `OPTIMIZE` |
| **Z-ORDER** | `OPTIMIZE t ZORDER BY (h3_r8, ts)` | Existing tables without liquid; multi-column locality |
| **Partitioning** | `PARTITIONED BY (event_date)` | Only for low-cardinality, time-based, large tables (>1 TB); **avoid partitioning by H3 fine cells** |
| **Coarse H3 as clustering key** | cluster on `h3_toparent(cell, 4–6)` computed column | Billions of rows; balances file count vs pruning |
| **Predictive Optimization** | `ALTER SCHEMA … ENABLE PREDICTIVE OPTIMIZATION` | Auto OPTIMIZE/VACUUM on UC managed tables |
| **Data skipping stats** | Delta collects min/max on first 32 columns (`delta.dataSkippingNumIndexedCols`) | Put H3/time columns early; BIGINT stats are efficient, binary geometry is not useful |
| **Compression/file size** | Target 128 MB–1 GB files, `delta.targetFileSize` / auto compaction | |

**Note:** with H3 as `BIGINT`, hierarchy-aware range pruning works because child IDs of a parent fall in a contiguous numeric range → `h3_minchild/h3_maxchild`.

### 9.2 Query performance checklist
1. Filter on the clustering column(s) (cell, time) — verify **files pruned** in the query profile.
2. Use **Photon** (SQL warehouse or Photon-enabled cluster) – H3 functions are vectorized there.
3. Prefer **BIGINT** joins; avoid STRING cell joins.
4. **Broadcast** the polygon→cell lookup (small side) for point enrichment.
5. **Pre-compute** `h3_r*` columns in silver; do not recompute in each query.
6. Keep geometry **out of hot joins**; join on cell, fetch geometry later (late materialization).
7. Use **approx** functions (`approx_count_distinct`, `percentile_approx`) for dashboards.
8. Cache results with **materialized views** / gold tables for serving; dashboards must not scan bronze.
9. Prevent the **"too many cells per polygon" explosion**: cap number of cells (e.g. choose res adaptively, compact or use hierarchical join: coarse cell join then refine).
10. `ANALYZE TABLE … COMPUTE STATISTICS FOR COLUMNS h3_r8, ts` for CBO.
11. Use **AQE** + skew handling (section 7.3).
12. Right-size: shuffle partitions (`spark.sql.shuffle.partitions=auto` with AQE), photon-optimized VM families (e.g. `Standard_E8ds_v5`/`E16ds_v5` memory-optimized for joins; `L8s_v3` for local NVMe caching on repeated reads).

### 9.3 Adaptive (multi-resolution) joins — dense cities vs. sparse rural
```sql
-- Step 1: coarse count per r6
WITH dens AS (SELECT h3_toparent(h3_r10, 6) AS p6, COUNT(*) n FROM silver.pings GROUP BY 1),
-- Step 2: choose resolution per parent: dense → r10, sparse → r7
pick AS (SELECT p6, CASE WHEN n > 5e6 THEN 10 WHEN n > 5e5 THEN 9 ELSE 7 END AS res FROM dens)
SELECT s.*, CASE p.res WHEN 10 THEN s.h3_r10 WHEN 9 THEN h3_toparent(s.h3_r10, 9) ELSE h3_toparent(s.h3_r10, 7) END AS h3_adaptive
FROM   silver.pings s JOIN pick p ON p.p6 = h3_toparent(s.h3_r10, 6);
```

### 9.4 Compute sizing guidance (rules of thumb, Azure)

| Workload | Compute | Notes |
|---|---|---|
| Ad hoc SQL / BI / serving | **Databricks SQL Serverless warehouse** (Photon) | Auto-scale; best for dashboards |
| Batch ETL (TB-scale joins) | Job cluster, Photon, memory-optimized (E-series), autoscale 4–40, spot workers for non-critical | Use `SPOT_WITH_FALLBACK_AZURE` |
| Streaming | Job/DLT cluster fixed-size or autoscale (enhanced autoscaling in DLT) | Checkpoint on ADLS |
| Python/geo libs (GeoPandas, GDAL) | Standard (non-Photon) cluster with init script or Docker image | Pin versions |
| Raster/ML | GPU optional (NC/ND series) | |
| Serverless jobs/notebooks | Zero cluster management; check library support (pip OK, JVM libs like Sedona limited 🔎) | |

---

## 10. Operationalizing on Azure & Azure Databricks

### 10.1 Reference architecture

```text
┌─────────────┐  ┌────────────┐  ┌────────────┐  ┌─────────────┐
│ IoT/Vehicles│  │ Event Hubs │  │ ADF /      │  │ Partner APIs│
│ Mobile apps │─▶│ (Kafka API)│  │ Fabric DF  │◀─│ Open data   │
└─────────────┘  └─────┬──────┘  └─────┬──────┘  └─────────────┘
                       │ stream        │ batch files (GeoJSON, SHP, GPKG, CSV)
                       ▼               ▼
         ┌────────────────────────────────────────────────┐
         │ ADLS Gen2 (landing)  → Auto Loader / DLT       │
         └──────────────────────┬─────────────────────────┘
                                ▼
    ┌────────────── Azure Databricks (Unity Catalog) ──────────────┐
    │ Bronze (raw WKB/WKT) → Silver (valid, CRS-normalized, H3)    │
    │   → Gold (cell aggregates, features, zone assignments)       │
    │ Jobs / Lakeflow Pipelines / DABs / Workflows                 │
    └──────┬───────────────┬────────────────┬──────────────┬───────┘
           ▼               ▼                ▼              ▼
   SQL Warehouse    Model Serving     Databricks Apps   Delta Sharing /
   (BI, JDBC/ODBC,  (ML + geo-        (Streamlit/Dash/  Lakehouse Fed.
   SQL Statement    enrichment API)    FastAPI maps)    (external orgs)
   Execution API)
           │               │                │
           ▼               ▼                ▼
   Power BI / Excel   Azure API Mgmt    Azure Maps / Web apps / GIS (QGIS/ArcGIS)
                      (+ Front Door/WAF) 
         ─────────── Cross-cutting: Entra ID, Key Vault, Private Link, Purview, Monitor, Log Analytics, Azure DevOps/GitHub Actions ───────────
```

### 10.2 Azure service roles

| Azure service | Role in geo platform |
|---|---|
| **ADLS Gen2** | Lake storage (hierarchical namespace), GeoParquet/Delta/COG storage |
| **Azure Databricks** | Processing, SQL serving, ML, apps |
| **Unity Catalog** | Governance, lineage, ACLs, row/column security, Delta Sharing, Volumes for files |
| **Event Hubs** (Kafka endpoint) / **IoT Hub** | Streaming ingestion |
| **Azure Data Factory / Fabric Data Factory** | Orchestrate copy/ingest from external sources; trigger Databricks jobs |
| **Azure Maps** | Basemaps, geocoding, routing, isochrones, tiles (visual layer / enrichment API) |
| **Azure API Management** | Gateway for geo REST APIs: auth, throttling, caching, versioning, products per consumer group |
| **Azure Front Door / App Gateway + WAF** | Edge, TLS, geo-filtering, DDoS |
| **Azure Functions / Container Apps / AKS** | Light API wrappers, routing engines (OSRM/Valhalla), tile servers (pg_tileserv, Martin) |
| **Azure Database for PostgreSQL (PostGIS)** / **Azure Cosmos DB** | Low-latency serving/OLTP: geo-point lookup, vector tiles, transactional geo-data |
| **Azure Data Explorer (ADX)** | Time-series + geospatial (`geo_point_to_h3cell`, `geo_polygon_to_h3cells`) real-time analytics 🔎 |
| **Azure Synapse / Fabric** | Alternative engines; OneLake shortcuts to ADLS/Delta |
| **Power BI** | Reporting; Azure Maps visual, Icon Map, Deck.gl custom visuals; H3 hex via custom visuals |
| **Azure Key Vault** | Secrets (Databricks secret scope backed by Key Vault) |
| **Microsoft Entra ID** | SSO, service principals / managed identities, SCIM group sync |
| **Microsoft Purview** | Data catalog/classification across estate (location data = personal data!) |
| **Azure Monitor / Log Analytics / Application Insights** | Observability for jobs/APIs; diagnostic logs |
| **Azure DevOps / GitHub Actions** | CI/CD with Databricks Asset Bundles |
| **Azure Container Registry** | Custom container images (GDAL/GEOS/PROJ stack) |
| **Azure Policy / Defender for Cloud** | Compliance guardrails |

### 10.3 Environments, workspaces and Unity Catalog structure

```text
Metastore (per region)
 ├─ catalog: geo_dev    ├─ catalog: geo_test    └─ catalog: geo_prod
 │   schemas:  ref | bronze | silver | gold | serving | ml | tmp
 │   volumes:  landing (files), checkpoints, tiles
```
- **One workspace per environment** (dev/test/prod) or catalog-per-env bound to workspaces (`workspace-catalog binding`).
- Permissions via **Entra groups** (`geo-engineers`, `geo-analysts`, `geo-consumers-<bu>`): `GRANT USE CATALOG, USE SCHEMA, SELECT` on `serving`; engineers have write to bronze/silver/gold; consumers only `serving`.
- Use **external locations + storage credentials** (managed identity via access connector for Azure Databricks) — no account keys/SAS in code.
- Network: **VNet injection**, **Secure Cluster Connectivity (no public IP)**, **Private Link** (front-end & back-end), storage firewall + private endpoints, NCC for serverless → ADLS.
- Tag everything: `cost_center`, `owner`, `env`, `data_domain=geo` (cluster policies enforce tags).

### 10.4 CI/CD with Databricks Asset Bundles (DABs)

`databricks.yml`
```yaml
bundle:
  name: geo-platform

variables:
  catalog: { default: geo_dev }

include:
  - resources/*.yml

targets:
  dev:
    mode: development
    default: true
    workspace: { host: https://adb-<dev-id>.azuredatabricks.net }
    variables: { catalog: geo_dev }
  prod:
    mode: production
    workspace: { host: https://adb-<prod-id>.azuredatabricks.net }
    run_as: { service_principal_name: "<sp-application-id>" }
    variables: { catalog: geo_prod }
    permissions:
      - group_name: geo-engineers
        level: CAN_MANAGE
      - group_name: geo-analysts
        level: CAN_VIEW
```

`resources/jobs_geo_batch.yml`
```yaml
resources:
  jobs:
    geo_batch_enrichment:
      name: geo-batch-enrichment-${bundle.target}
      schedule: { quartz_cron_expression: "0 0 * * * ?", timezone_id: Europe/Berlin }
      max_concurrent_runs: 1
      email_notifications: { on_failure: ["geo-ops@contoso.com"] }
      job_clusters:
        - job_cluster_key: etl
          new_cluster:
            spark_version: 16.4.x-photon-scala2.12   # pick a current LTS 🔎
            node_type_id: Standard_E16ds_v5
            autoscale: { min_workers: 2, max_workers: 16 }
            azure_attributes: { availability: SPOT_WITH_FALLBACK_AZURE, first_on_demand: 1 }
            data_security_mode: USER_ISOLATION
            runtime_engine: PHOTON
            custom_tags: { cost_center: geo, env: ${bundle.target} }
      tasks:
        - task_key: build_zone_cells
          job_cluster_key: etl
          notebook_task: { notebook_path: ../src/notebooks/02_build_zone_cells.py, base_parameters: { catalog: ${var.catalog} } }
        - task_key: enrich_points
          depends_on: [{ task_key: build_zone_cells }]
          job_cluster_key: etl
          notebook_task: { notebook_path: ../src/notebooks/03_enrich_points.py }
        - task_key: publish_gold
          depends_on: [{ task_key: enrich_points }]
          job_cluster_key: etl
          notebook_task: { notebook_path: ../src/notebooks/04_publish_gold.py }
```

CLI flow:
```bash
databricks bundle validate -t dev
databricks bundle deploy   -t dev
databricks bundle run geo_batch_enrichment -t dev
databricks bundle deploy   -t prod     # from CI with service principal (OIDC / workload identity federation)
```

GitHub Actions / Azure DevOps stages: **lint (ruff/black) → unit tests (pytest, h3) → bundle validate → deploy dev → integration tests on small data → manual approval → deploy prod → smoke test**.

### 10.5 Orchestration options
| Option | Use |
|---|---|
| **Databricks Jobs/Workflows** (tasks, dependencies, repair runs, file-arrival & table-update triggers) | Default orchestrator |
| **Lakeflow Declarative Pipelines (DLT)** | Declarative streaming/batch with expectations (quality) |
| **ADF/Fabric pipelines** | Cross-system orchestration; call Databricks job via REST/linked service |
| **Azure Logic Apps / Event Grid** | Event-driven triggers (e.g., new file in landing) |
| **Airflow (Azure MWAA-like / self-hosted)** | If organization standard |

### 10.6 Ingestion patterns

**Files (Auto Loader):**
```python
(spark.readStream.format("cloudFiles")
   .option("cloudFiles.format", "json")                         # GeoJSON lines / CSV / parquet
   .option("cloudFiles.schemaLocation", "/Volumes/geo_prod/bronze/chk/poi_schema")
   .load("/Volumes/geo_prod/bronze/landing/poi/")
   .writeStream.option("checkpointLocation", "/Volumes/geo_prod/bronze/chk/poi")
   .trigger(availableNow=True).toTable("geo_prod.bronze.poi_raw"))
```
**Shapefile/GPKG/GeoJSON (non-Spark-native):** read with Sedona (`sedona.read.format("shapefile")`), or GeoPandas/pyogrio on driver for small reference data → `ST_AsBinary` → Delta. Large GeoJSON: convert to **GeoParquet/line-delimited GeoJSON** upstream.

**Streaming:** Event Hubs Kafka endpoint (section 8.9) with secrets in Key Vault-backed scope; use watermarks + dedup on `(device_id, ts)`; late data policy documented.

**APIs/partners:** ADF Copy → landing; or Databricks job with `requests` + retry/backoff; persist raw payload first (bronze), parse later.

### 10.7 Data quality for geospatial (expectations)
| Check | Expectation |
|---|---|
| Coordinate range | lon ∈ [-180,180], lat ∈ [-90,90] |
| Not null-island | not (lon=0 and lat=0) |
| Geometry validity | `ST_IsValid(geom)` / repair (`ST_MakeValid`, buffer(0)) |
| Non-empty geometry | `NOT ST_IsEmpty(geom)` |
| CRS consistency | all SRID = 4326 in silver |
| H3 validity | `h3_isvalid(h3_r9)` |
| Resolution consistency | `h3_resolution(h3_r9) = 9` |
| Spatial plausibility | within country bbox / not in ocean (if applicable) |
| Speed plausibility (trajectories) | implied speed < threshold between pings |
| Duplicates | unique (`device_id`,`ts`) |
| Freshness | max(ts) within SLA |
| Cardinality of join | zones per point ≤ expected |

Quarantine failed rows into `*_quarantine` tables with reason codes; publish DQ metrics to a monitoring table and alert (Databricks SQL alerts / Azure Monitor).

---

## 11. Serving Geospatial Data to Other Users & Groups

### 11.1 Delivery options matrix

| Consumer | Mechanism | Latency | Notes |
|---|---|---|---|
| **BI analysts** | **Databricks SQL warehouse** + Power BI (DirectQuery / Import / composite) | s–min | Serving schema with views; cells as hex strings + WKT/GeoJSON; for Power BI use Azure Maps visual or deck.gl H3 custom visual 🔎 |
| **Data scientists** | UC tables, notebooks, Feature Tables | — | Shared geo library via wheel on UC Volumes |
| **Apps / services (REST)** | **SQL Statement Execution API**, or Databricks **Apps** hosting FastAPI/Flask; or **Model Serving** custom pyfunc endpoint | 100 ms–s | Front with API Management |
| **Low-latency (ms) lookups** | Sync gold to **PostgreSQL/PostGIS**, **Cosmos DB**, **Redis**, or **Databricks Lakebase** 🔎 | ms | Reverse ETL for "point → zone", "cell → score" |
| **GIS users (QGIS/ArcGIS)** | JDBC/ODBC to SQL warehouse, GeoPackage/GeoParquet exports, or PostGIS replica | s–min | Provide WKB/WKT + SRID |
| **Web map clients** | Vector tiles (PMTiles in ADLS + CDN; or tile server on Container Apps) | ms | Pre-generate tiles (tippecanoe) from Delta for static layers |
| **External orgs/partners** | **Delta Sharing** (open protocol; Databricks-to-Databricks or open), or secure file export | — | No copy for D2D; governance via UC shares & recipients |
| **Other platforms** | Lakehouse Federation / OneLake shortcut / Fabric mirroring 🔎 | — | |
| **Event consumers** | Event Hubs output topic / webhook | s | Alerts, geofence events |
| **ML serving** | Model Serving endpoint with H3 feature lookup (Feature Serving / online tables 🔎) | 10–100 ms | |

### 11.2 Pattern: "Geo-enrichment as a service"

Goal: other teams call `POST /v1/locate {lat, lon}` → zone, h3 cells, attributes.

**Option A — Databricks Apps (FastAPI) + SQL warehouse**
```python
# app.py (Databricks App)
import os
from fastapi import FastAPI, HTTPException
from databricks import sql
from databricks.sdk.core import Config

app = FastAPI(title="Geo Enrichment API", version="1.0")
cfg = Config()                                    # picks up app service principal credentials

def q(statement, params):
    with sql.connect(server_hostname=cfg.host, http_path=os.environ["WAREHOUSE_HTTP_PATH"],
                     credentials_provider=lambda: cfg.authenticate) as c, c.cursor() as cur:
        cur.execute(statement, params)
        return cur.fetchall()

@app.get("/v1/locate")
def locate(lat: float, lon: float, res: int = 9):
    if not (-90 <= lat <= 90 and -180 <= lon <= 180):
        raise HTTPException(422, "invalid coordinates")
    rows = q("""SELECT zc.zone_id, z.name, h3_h3tostring(h3_longlatash3(:lon,:lat,:res)) AS h3
                FROM geo_prod.silver.zone_cells_r9 zc JOIN geo_prod.ref.admin_zones z USING (zone_id)
                WHERE zc.h3_r9 = h3_longlatash3(:lon,:lat,9)""", {"lon": lon, "lat": lat, "res": res})
    return [dict(r.asDict()) for r in rows]
```
Use app **resource bindings** (warehouse, secrets) and UC grants to the app service principal; front with API Management if multi-consumer.

**Option B — SQL Statement Execution API (no custom code)**
```bash
curl -X POST "$HOST/api/2.0/sql/statements" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{
  "warehouse_id":"<id>", "wait_timeout":"10s", "on_wait_timeout":"CONTINUE",
  "statement":"SELECT * FROM geo_prod.serving.vw_density_geojson WHERE h3 = :h3",
  "parameters":[{"name":"h3","value":"89283082803ffff","type":"STRING"}]}'
```
Use **Entra service principal** OAuth M2M tokens, not PATs, for production.

**Option C — Low latency: reverse ETL to PostGIS / Cosmos / Lakebase**
- Nightly/stream job: `gold.zone_cells_r9` → Postgres table `(h3_r9 bigint primary key, zone_id, …)` with B-tree index → sub-10 ms lookups; horizontal scale with read replicas.
- Keep Delta as system of record; sync via `foreachBatch` + JDBC upsert or CDC (`Change Data Feed` → merge).

**Option D — Model Serving pyfunc**: package H3 + polygon lookup + model in MLflow pyfunc (pure Python `h3`, `shapely` STRtree) → serverless endpoint with autoscaling/scale-to-zero; log with UC model registry (`models:/geo_prod.ml.zone_scorer@champion`).

### 11.3 Pattern: Publishing semantic layer to BI
- Create **serving views** with stable column names, documented comments, certified tags.
- For Power BI: use **Databricks SQL warehouse**, Entra SSO passthrough or service principal; publish certified dataset; for maps use hex string + Azure Maps/Deck.gl; keep cell counts per visual < ~30–100k; aggregate in SQL by zoom level (res 5 at country zoom → res 9 at street zoom via parameterized views / multiple tables `density_r5…r9`).
- **Multi-resolution pyramid tables**: `gold.density_r5, r6, r7, r8, r9` (materialized views) so the viz picks resolution per zoom.

### 11.4 Pattern: Delta Sharing to partners
```sql
CREATE SHARE geo_open_density COMMENT 'Aggregated H3 density (k>=5)';
ALTER SHARE geo_open_density ADD TABLE geo_prod.serving.density_r8_public;
CREATE RECIPIENT partner_x USING ID 'azure:westeurope:<metastore-uuid>';   -- D2D
GRANT SELECT ON SHARE geo_open_density TO RECIPIENT partner_x;
```
Share only **aggregated/anonymized** cells for personal-location data; use views to enforce k-anonymity.

### 11.5 Multi-tenant / group-based access

| Need | UC feature |
|---|---|
| Group sees only its region | **Row filters** on `h3_r4`/`region_id` using `is_account_group_member('geo-emea')` |
| Hide precise coordinates | **Column masks** (`lon, lat` → NULL / rounded; or only expose H3 r7) |
| Per-tenant isolation | separate schema/catalog per tenant, or **dynamic views** |
| Approvals / discoverability | UC tags (`pii=location`, `certified=true`), comments, **Discover/Marketplace** 🔎 |
| Audit | `system.access.audit`, `system.access.table_lineage`, query history system tables |

```sql
CREATE FUNCTION geo_prod.sec.region_filter(region STRING)
RETURN is_account_group_member('geo-global') OR is_account_group_member(concat('geo-', lower(region)));
ALTER TABLE geo_prod.gold.density_r8 SET ROW FILTER geo_prod.sec.region_filter ON (region);

CREATE FUNCTION geo_prod.sec.mask_coord(v DOUBLE)
RETURN CASE WHEN is_account_group_member('geo-precise') THEN v ELSE round(v, 2) END;
ALTER TABLE geo_prod.silver.pings ALTER COLUMN lat SET MASK geo_prod.sec.mask_coord;
```

### 11.6 API design guidance for geo services
- Versioned (`/v1`), idempotent GETs, pagination by cell range (`h3_minchild/maxchild`) or keyset on `(h3, ts)`.
- Input: lon/lat + `srid` (default 4326), **explicit axis order docs**; accept `h3` hex strings; output: GeoJSON (RFC 7946) + H3 hex string.
- Bound the request: max bbox area / max cells / max radius; async job pattern (`202 Accepted` + polling) for heavy queries.
- Cache on APIM / CDN keyed by `(h3, res, version)`; cache TTL aligned to gold refresh.
- SLAs: define freshness (e.g., gold refreshed hourly), availability, p95 latency per tier.
- Auth: OAuth2 via Entra; API Management products per consumer group with quotas; managed identity to backends.

---

## 12. Governance, Security, Cost, Monitoring

### 12.1 Privacy & compliance (location data is personal data — GDPR relevance)
- Treat device-level trajectories as **personal data**; minimize, pseudonymize device IDs (salted hash with Key Vault secret), set **retention** (`delta.deletedFileRetentionDuration`, `VACUUM`, TTL jobs), implement **right-to-erasure** via `DELETE` + `VACUUM` + Change Data Feed awareness.
- Publish **aggregates** at coarse res with k-anonymity (k≥5–20), add noise/suppression for small cells; avoid res > 9–10 for person-level public data; home/work inference risk from fine cells + time.
- Lawful basis/consent tracking; Purview classification (`Location`), UC tags `pii`.
- Data residency: choose region (e.g. Germany West Central / West Europe); UC metastore per region; Delta Sharing cross-region egress costs.
- Licensing: OSM (ODbL attribution/share-alike), national mapping agencies, Overture Maps (CDLA/ODbL variants) — record license per dataset in table properties.

### 12.2 Observability
| Layer | Tool |
|---|---|
| Job success/duration/cost | Jobs UI, **system tables** (`system.lakeflow.*`, `system.billing.usage`, `system.compute.*`, `system.query.history` 🔎 names) |
| Data quality | DLT expectations event log, Lakehouse Monitoring (profile/drift tables), custom DQ tables |
| Pipeline lag | Streaming query progress (`inputRowsPerSecond`, `processedRowsPerSecond`, `latestOffset`), alert on lag |
| API | APIM analytics + Application Insights (latency, 4xx/5xx, per-consumer) |
| Infra | Azure Monitor diagnostic settings → Log Analytics; alerts → Teams/PagerDuty/Email |
| Lineage | UC lineage (table & column), Purview |
| SQL alerts | Databricks SQL Alerts on DQ/freshness metrics (e.g. `max(ts) < now() - interval 2 hours`) |

### 12.3 Cost control
- **Serverless SQL** for bursty BI; **job clusters** (not all-purpose) for batch; spot + fallback; autotermination; cluster policies with max DBU/hour and mandatory tags.
- Reduce compute: indexed joins, broadcast, pre-aggregation, materialized views, incremental processing (Auto Loader / CDF / `MERGE` only changed cells), avoid recomputing polyfill (cache reference-cell tables).
- Reduce storage: only store needed resolutions; `VACUUM`; compaction; Zstd compression (`delta.parquet.compression.codec=zstd`).
- **Budgets & alerts** (Azure Cost Management, Databricks budgets); chargeback via tags and `system.billing.usage`.
- Monitor **cardinality explosion** (cells per polygon × polygons) as a cost driver.

### 12.4 Reliability & DR
- ADLS **GRS/ZRS**, Delta table **deep clone** to secondary region, UC metadata export (Terraform/DABs as code), re-runnable idempotent jobs (`MERGE`, `replaceWhere`, `INSERT OVERWRITE` by partition), job retries with backoff, **repair runs**.
- Time travel for rollback: `RESTORE TABLE … TO VERSION AS OF n`.
- Infrastructure as Code: **Terraform (databricks + azurerm providers)** for workspaces, networking, UC, cluster policies, groups.

### 12.4b Incremental processing for expensive geo-steps
```sql
-- Recompute only impacted cells when a polygon changes
MERGE INTO geo_prod.silver.zone_cells_r9 t
USING (SELECT zone_id, explode(h3_try_polyfillash3(geom_wkb, 9)) AS h3_r9
       FROM geo_prod.ref.admin_zones WHERE updated_at > :last_run) s
ON t.zone_id = s.zone_id AND t.h3_r9 = s.h3_r9
WHEN NOT MATCHED THEN INSERT *
WHEN NOT MATCHED BY SOURCE AND t.zone_id IN (SELECT zone_id FROM geo_prod.ref.admin_zones WHERE updated_at > :last_run)
  THEN DELETE;
```

---

## 13. Best Practices Checklist

### Modeling
- [ ] One canonical CRS (EPSG:4326 storage) + local metric CRS only for computation.
- [ ] Store raw geometry (WKB/native type) **and** H3 BIGINT columns for chosen resolutions.
- [ ] Document resolution choice (with rationale, expected cell counts) in table comments.
- [ ] Separate reference (slow-changing polygons) from facts (events); precompute polygon→cell tables.
- [ ] Version reference data (valid_from/valid_to) → point-in-time correct joins.

### Processing
- [ ] Validate/repair geometry; quarantine invalid rows with reason.
- [ ] Use built-in Photon H3 functions before Python UDFs.
- [ ] Index-then-verify for exactness; use core/border logic.
- [ ] Control cell explosion; test resolution on sample; use adaptive resolution for skew.
- [ ] Broadcast small lookup tables; enable AQE.
- [ ] Use `try_*` H3 functions in production pipelines.
- [ ] Idempotent writes; checkpoint streaming; watermark + dedup.
- [ ] Handle antimeridian, poles, pentagons explicitly.

### Storage & performance
- [ ] Liquid Clustering on `h3_*` (+ time); no partitioning on fine cells.
- [ ] Files 128 MB–1 GB; predictive optimization on.
- [ ] Late geometry materialization; WKB not WKT in Delta.
- [ ] Materialized aggregates for dashboards (multi-resolution pyramid).

### Engineering
- [ ] Code in Git, packaged as wheel/bundle; DABs for IaC of jobs/pipelines.
- [ ] Unit tests (pure Python h3/shapely), integration tests (small cluster), data tests (expectations).
- [ ] Cluster policies, pinned runtime/library versions, reproducible environments (Docker for GDAL stack).
- [ ] Service principals + OIDC federation for CI/CD; no PATs in pipelines.

### Operations & Governance
- [ ] UC governance: groups, grants, row filters, masks, tags, lineage.
- [ ] Private networking, Key Vault-backed secrets, managed identities.
- [ ] Monitoring: freshness, DQ, cost, API SLOs, alerts routed to on-call.
- [ ] Data contracts + SLA/SLO per gold/serving product; changelog and semantic versioning of schemas.
- [ ] Privacy: aggregation thresholds, retention, erasure process, DPIA documented.
- [ ] Documentation: data dictionary, CRS/resolution conventions, runbooks, how-to-consume guide.

---

## 14. Use Cases & Reference Architectures

| # | Use case | Industry | H3 res | Key patterns | Output / service |
|---|---|---|---|---|---|
| 1 | **Mobility demand heatmap & surge** | Ride-hailing, logistics | 8–9 | P5, P6, P12 | Streaming 5-min density → Event Hubs/API |
| 2 | **Retail catchment & site selection** | Retail, real estate | 8–10 | P14, P15, P7 | Isochrone → cells → demographics join → ranking in Power BI |
| 3 | **Fleet telematics & geofencing** | Transport | 10–11 | P13, P9, P10 | DLT streaming + alerts (Event Grid/Teams) |
| 4 | **Telecom network quality** | Telco | 8–10 | P5, P8, P12 | Cell-level KPI aggregation, hotspot/cold spot, coverage gaps |
| 5 | **Insurance risk (flood, wind, crime)** | Insurance | 8–10 | P2/P4, P15, P18 | Hazard polygons → cells; policy geocode → cell risk score |
| 6 | **Energy / utilities asset analytics** | Energy | 9–11 | P7, P4 | Asset-to-grid-cell proximity, outage clustering |
| 7 | **Public sector / census & mobility statistics** | Government | 6–9 | P15, P17 | Anonymized aggregates, Delta Sharing |
| 8 | **Supply-chain / last mile optimization** | Logistics | 8–10 | P11, P7, P18 | OD flows, depot catchments, ETA features |
| 9 | **Agriculture & environmental monitoring** | AgriTech | 7–10 | raster zonal stats + P2 | Raster → cell stats (NDVI per cell) |
| 10 | **Fraud / anomaly detection** | Banking | 7–9 | P18 | Impossible-travel (grid_distance vs time), cell risk features |
| 11 | **Foot-traffic & POI analytics** | Retail, advertising | 10–12 | P10, P17 | Dwell time, visit attribution — privacy critical |
| 12 | **Smart city & IoT sensors** | City | 9–11 | P5, P13 | Sensor cell aggregation, alerts |
| 13 | **Geo-sharding / data mesh keys** | Platform | 3–5 | P16 | Partition/cluster/shuffle key; multi-region routing |
| 14 | **Real-time location enrichment API** | Platform | 9–10 | Section 11.2 | Apps/API + PostGIS/Lakebase |

### 14.1 Example: Impossible-travel / fraud feature
```sql
SELECT t.*,
       h3_try_distance(prev_cell, h3_r8)                         AS cells_moved,
       h3_try_distance(prev_cell, h3_r8) * 0.92                  AS approx_km_moved,   -- r8 centre spacing ≈ 0.8 km (edge 0.46 × √3)
       unix_timestamp(ts) - unix_timestamp(prev_ts)               AS dt_s,
       (h3_try_distance(prev_cell, h3_r8) * 0.8) / nullif((unix_timestamp(ts)-unix_timestamp(prev_ts))/3600.0, 0) AS implied_kmh
FROM (SELECT *, lag(h3_r8) OVER (PARTITION BY card_id ORDER BY ts) AS prev_cell,
                lag(ts)    OVER (PARTITION BY card_id ORDER BY ts) AS prev_ts
      FROM geo_prod.silver.card_tx) t
WHERE h3_try_distance(prev_cell, h3_r8) IS NOT NULL;
```
(Grid distance is approximate for long distances; for accuracy use haversine on centres.)

### 14.2 Example: Demographics to H3 grid (areal interpolation, area-weighted)
```python
# census polygons (pop) → cells at r8, split population by share of cell count (simple) or by chip area (better)
cells = (spark.table("geo_prod.ref.census").select("tract_id", "pop", F.explode(F.expr("h3_try_polyfillash3(geom_wkb, 8)")).alias("h3_r8")))
w = Window.partitionBy("tract_id")
grid_pop = (cells.withColumn("n", F.count("*").over(w))
                 .withColumn("pop_cell", F.col("pop")/F.col("n"))
                 .groupBy("h3_r8").agg(F.sum("pop_cell").alias("pop")))
```
Better: use tessellation chips & `ST_Area(chip)/ST_Area(tract)` weights, or dasymetric weights (building footprints, land use).

### 14.3 Example: Raster zonal statistics into H3 (conceptual)
1. Store COGs on ADLS; read tiles with `rasterio` in `mapInPandas` per tile (or Sedona raster `RS_*`).
2. For each tile: compute pixel centroids → `h3.latlng_to_cell` (or vectorized with `h3-pandas`/numpy) → group by cell → mean/min/max.
3. Merge per-tile partials (sum, count) → final cell mean; write to `gold.raster_cells_r8`.

### 14.4 Example: Routing/isochrone integration pattern
- Deploy **Valhalla/OSRM/GraphHopper** on **AKS/Container Apps** with OSM extract (Azure Files/Blob volume); expose internal endpoint via Private Link.
- Databricks job: for each depot (hundreds/thousands) call isochrone API (batch via `mapInPandas`, rate-limited) → polygon → polyfill → store in `gold.catchment_cells(depot_id, minutes, h3_r9)`.
- Downstream joins (demand, competitors) are then pure H3 equi-joins.
- Alternative: **Azure Maps** Route Range (isochrone) API with quota/cost controls.

---

## 15. Pitfalls & Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Points land in wrong place (e.g. ocean / wrong continent) | lat/lon swapped | `h3_longlatash3(lon, lat, r)` — **lon first**; Python `h3.latlng_to_cell(lat, lng)` — lat first |
| `h3_longlatash3` error / NULL | out-of-range coords, NaN | Validate; use `try_` and quarantine |
| Polyfill returns empty array | polygon smaller than a cell, or invalid/CW-CCW or lat/lon swapped | Use `coverash3`, finer res, check geometry validity & orientation |
| Polyfill result misses borders | centre-based semantics | Use cover/tessellate; or buffer polygon; or verify with exact predicate |
| Duplicate rows after join | overlapping polygons/cells | Decide assignment rule; dedupe by priority/area |
| Join explodes / OOM / spill | too many cells per polygon, skew on hot cells | Coarser res, adaptive res, salting, broadcast, AQE |
| `kring` errors near pentagons | pentagon distortion | `try_*` functions; handle with fallback |
| `h3_distance` NULL/error | cells too far/different resolutions/pentagon | `h3_try_distance`; ensure same res; use haversine on centres |
| Different results in Python `h3` vs SQL | v3/v4 API differences, polyfill semantics, lat/lng order | Pin versions; cross-test known cells |
| BIGINT shows negative numbers | JVM signed long for high-bit IDs? (H3 IDs fit in 63 bits normally: top bit 0) | If you see negatives, upstream tool reinterpreted; use `h3_h3tostring` to validate |
| Hex string case/format mismatch | `8928308280fffff` lowercase 15 chars | Normalize with `lower()`; validate with `h3_isvalid` |
| Slow Python UDF | row-wise UDF | Use built-in SQL functions or vectorized pandas UDF |
| Slow queries despite H3 | cell not a clustering key; STRING joins; no broadcast; scanning bronze | Liquid clustering on BIGINT cells; precompute; check query profile |
| Antimeridian polygons wrong (huge polygons spanning the globe) | polygon crossing ±180° | Split at antimeridian / shift longitudes; use geography |
| Distances wrong by factor ~100,000 | degrees treated as meters | Use geography functions or project to metric CRS |
| Shapefile read fails | missing `.dbf/.shx/.prj`, encoding | Zip all sidecar files, specify encoding (`cp1252`/`utf-8`) |
| Sedona/Mosaic class not found | cluster library/Spark config missing; DBR mismatch | Install matching Maven artifact; set Kryo/SQL extension configs; pin DBR |
| Cannot install Sedona/GDAL on shared/serverless compute | UC shared access mode / serverless restrictions | Use dedicated (single-user) clusters, Docker image, or built-in ST/H3 |
| Dashboards slow or browser freezes | too many hexes rendered | Pyramid tables by zoom, limit by bbox, aggregate server-side |
| Privacy incident risk | fine-res person-level cells exposed | Aggregate, k-anonymity, masking, row filters |
| Stale enrichment | polygons updated but cell table not refreshed | Incremental MERGE on `updated_at`, add dependency in job DAG, DQ freshness check |
| Cross-region latency/egress cost | compute and storage in different regions | Co-locate; private endpoints |

---

## 16. Quick Reference Tables

### 16.1 One-page H3 SQL cheat sheet (Databricks)

```sql
-- INDEX
h3_longlatash3(lon, lat, res)                 -- → BIGINT cell
h3_longlatash3string(lon, lat, res)           -- → STRING cell
h3_pointash3(geom, res) / h3_pointash3string  -- from WKT/WKB/GeoJSON point
-- AREA → CELLS
h3_polyfillash3(geom, res)                    -- centre-in-polygon
h3_coverash3(geom, res)                       -- covering set (any geom)
h3_tessellateaswkb(geom, res)                 -- cells + core flag + chip WKB   (🔎 version)
h3_try_polyfillash3 / h3_try_coverash3        -- NULL on failure
-- CELL → GEOMETRY
h3_boundaryaswkt | aswkb | asgeojson (cell)
h3_centeraswkt   | aswkb | asgeojson (cell)
-- HIERARCHY
h3_resolution(c) · h3_toparent(c,res) · h3_tochildren(c,res)
h3_minchild(c,res) · h3_maxchild(c,res) · h3_ischildof(c,p)
h3_compact(arr) · h3_uncompact(arr,res)
-- NEIGHBORS / DISTANCE
h3_kring(c,k) · h3_kringdistances(c,k) · h3_hexring(c,k) · h3_distance(a,b) · h3_try_distance(a,b)
-- UTIL
h3_h3tostring(c) · h3_stringtoh3(s) · h3_isvalid(x) · h3_validate(x) · h3_try_validate(x)
```

### 16.2 Size relations
| Relation | Formula |
|---|---|
| Cells at res r | 2 + 120·7^r |
| Children per hex (one level down) | 7 (pentagon: 6) |
| k-ring size (hex) | 1 + 3k(k+1) |
| Hex ring size at distance k | 6k (k ≥ 1) |
| Cell centre spacing | ≈ √3 × edge length |
| Area per cell (hex) | (3√3/2) × edge² (average, ignoring distortion) |
| Radius → k | k ≈ ceil(radius / (√3 × edge)) (+1 safety) |
| Resolution change by Δ | area × 7^−Δ, edge × 7^−Δ/2 |

### 16.3 Resolution picker

| Analysis scale | Suggested res |
|---|---|
| Country / continent overview | 2–4 |
| Region / state | 4–6 |
| City overview | 6–8 |
| Neighbourhood analysis | 8–9 |
| Street / block | 9–10 |
| Building / venue | 11–12 |
| Sub-meter / sensor (rarely needed) | 13–15 |

### 16.4 Index-then-verify decision tree
```text
Need exact boundary result?
 ├─ No  → cell join (polyfill) ✔ fastest
 └─ Yes → Do polygons have many vertices / are they big?
          ├─ Yes → tessellate (core/border) → verify only border cells
          └─ No  → cover cells → join → ST_Contains on candidates
```

### 16.5 Spark config & table property snippets
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 256 * 1024 * 1024)   # tune carefully
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
```
```sql
ALTER TABLE geo_prod.silver.pings SET TBLPROPERTIES (
  'delta.enableChangeDataFeed' = 'true',
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.dataSkippingNumIndexedCols' = '8',
  'delta.parquet.compression.codec' = 'zstd',
  'geo.crs' = 'EPSG:4326', 'geo.h3.resolutions' = '6,8,10', 'owner' = 'geo-platform');
OPTIMIZE geo_prod.silver.pings;            -- with liquid clustering: clusters incrementally
VACUUM  geo_prod.silver.pings RETAIN 168 HOURS;
```

### 16.6 CRS cheat table
| EPSG | Name | Use |
|---|---|---|
| 4326 | WGS 84 (lon/lat) | Storage, GeoJSON, GPS, H3 input |
| 3857 | Web Mercator | Web map tiles (don't measure) |
| 25832 / 25833 | ETRS89 / UTM 32N / 33N | Germany metric analysis |
| 32632 / 32633 | WGS84 / UTM 32N / 33N | Metric analysis |
| 3035 | ETRS89-LAEA | Pan-EU equal area |
| 3857→4326 | pyproj `Transformer.from_crs(3857, 4326, always_xy=True)` | Always use `always_xy=True` |

### 16.7 Glossary of acronyms
AQE (Adaptive Query Execution) · COG (Cloud Optimized GeoTIFF) · CRS (Coordinate Reference System) · DAB (Databricks Asset Bundle) · DGGS (Discrete Global Grid System) · DLT (Delta Live Tables / Lakeflow Declarative Pipelines) · MV (Materialized View) · OD (Origin–Destination) · UC (Unity Catalog) · WKB/WKT (Well-Known Binary/Text) · NCC (Network Connectivity Configuration) · SP (Service Principal) · APIM (API Management).

### 16.8 Starter "day-1" runbook for a new geo data product
1. Define **question, accuracy needs, consumer groups, SLAs**.
2. Choose **CRS + H3 resolution(s)**; estimate cell counts & explosion.
3. Create **UC catalog/schemas, groups, grants**; external location via access connector.
4. Ingest raw → **bronze** (Auto Loader / Event Hubs); keep payloads.
5. Validate/standardize → **silver** (WKB, EPSG:4326, `h3_r*`, DQ expectations, quarantine).
6. Precompute **reference polygon→cell** tables (cache, version).
7. Enrich/aggregate → **gold** (cluster by cell/time; multi-res pyramid).
8. Expose via **serving schema** (views, masks, row filters), SQL warehouse, API/App, Delta Sharing.
9. Add **tests, DABs, CI/CD, monitoring, alerts, cost tags**.
10. Document (data dictionary, CRS/res conventions, runbook) & onboard consumers; review privacy (DPIA).
11. Iterate: profile, tune resolution, optimize layout, review cost monthly.

### 16.9 Further reading (verify current URLs)
- H3 docs: h3geo.org (core spec, API, resolution tables)
- Databricks docs: "H3 geospatial functions", "Spatial SQL / ST functions", "Liquid clustering", "Databricks Asset Bundles", "Unity Catalog row filters & column masks", "Databricks Apps", "SQL Statement Execution API", "Delta Sharing"
- Databricks Labs Mosaic repo; Apache Sedona docs; GeoParquet spec (geoparquet.org); OGC standards; Azure Architecture Center (geospatial / IoT / medallion guidance)

---
*End of reference card. Treat 🔎-marked items and function availability as "verify against your runtime", since Databricks geospatial capabilities evolve fast.*
