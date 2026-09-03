# Test-suite gaps: OGC abstract tests vs existing validators

**Status: PROPOSAL.** For every abstract test in the OGC document (`ogc/abstract_tests/`), this
table says which existing tool implements it and what is missing. It is the "gaps list" required
for the OGC submission: an OGC Standard needs an executable test suite eventually, and the SWG
should know how far the community tools are from one.

Tools audited (revisions in [`crosswalk.md`](crosswalk.md)):

* **GPQ** — `gpq validate` (planetlabs/gpq main, 2026-08-30). 1.0-era rule set.
* **GDAL** — `validate_geoparquet.py` (OSGeo/gdal master, 2026-08-14). `--check-data` needed for data rules.
* **Schema** — `format-specs/schema.json` 2.0.0 applied to the `geo` value (GDAL does this; GPQ does not).
* **Corpus** — `geoparquet/geoparquet-testing` `bad_data/` negative fixtures and `data/` positive fixtures.

Coverage: **full** = a tool checks the whole statement for 2.0 files; **partial** = a tool checks
part of it or checks it with a known defect; **none** = no tool checks it.

## Conformance Class Core (`/conf/core/...`)

| Test | GPQ | GDAL | Schema | Corpus fixture | Coverage | What is missing |
| --- | --- | --- | --- | --- | --- | --- |
| parquet-file | implicit (open fails) | implicit (`is not a Parquet file`) | — | — | full | — |
| geo-metadata | `RequiredGeoKey`, `RequiredMetadataType` | `'geo' … missing`, `not valid JSON`, schema validation | yes (step 3) | `missing-geo-metadata`, `geo-invalid-json`, `geo-invalid-utf8` | full | GPQ does not run the JSON Schema (step 3). |
| file-metadata | `RequiredVersion` (any string), `RequiredPrimaryColumn`, `RequiredColumns` (allows `{}`), `PrimaryColumnInLookup` | version/primary/columns presence, primary ∈ columns, schema | `required`, `const`, `minLength`, `minProperties` | `missing-version`, `version-unknown`, `missing-primary-column`, `missing-columns`, `primary-column-not-in-columns` | full (GDAL) | GPQ does not check the version value nor `minProperties`. |
| column-metadata | reverse direction only (`missing geometry column`) | reverse direction only (`not found in the Parquet fields`); `encoding`/`geometry_types` via schema | `required [encoding, geometry_types]` | — | **partial** | Step 2 (every `GEOMETRY`/`GEOGRAPHY` column is listed) is checked by nobody. Needs a fixture with an unlisted logical-typed column. |
| geometry-column-type | `GeometryDataType` (BYTE_ARRAY), `RequiredColumnEncoding` (`WKB`) | `encoding` via schema | `const "WKB"` | corpus self-test `test_geometry_files_have_geometry_logical_type` only | **partial** | Step 3 (logical type present) is not checked by GPQ or GDAL. Needs a fixture: 2.0 metadata on a plain `BYTE_ARRAY` column. |
| geometry-column-nesting | `GeometryUngrouped` | — | — | — | full (GPQ) | GDAL: none. No negative fixture. |
| geometry-column-repetition | `GeometryRepetition` | — | — | — | full (GPQ) | GDAL: none. No negative fixture. |
| wkb | `GeometryEncoding` (orb WKB) | `--check-data` `Invalid WKB` (OGR, accepts EWKB) | — | `wkb-truncated`, `wkb-wrong-type-byte`, `wkb-with-srid-prefix` | partial | GDAL accepts EWKB, so `wkb-with-srid-prefix` passes there. Neither checks the Parquet/ISO variant strictly. |
| axis-order | — | geographic range heuristic on `bbox` only | — | `crs-mismatch-schema-vs-geo-metadata` (indirectly) | **none** | Not machine-checkable except heuristically; the abstract test is written as a heuristic. |
| geometry-types | `RequiredGeometryTypes` (no ` M`/` ZM`), `GeometryTypes` (Z not verified) | schema; `--check-data` type in list (Point Z bug; no M/ZM); duplicates (only with `bbox`) | `pattern`, `uniqueItems` | `geometry-types-mismatch-*`, `geometry-types-missing-actual-type`, `zm-declared-*` | **partial** | Step 4 (agreement with Parquet `geospatial_types` statistics) is checked by nobody. Dimension check only in the corpus self-tests. Needs statistics-mismatch fixture. |
| crs-projjson | `OptionalCRS` (PROJJSON schema; first-column early return) | schema + PROJ parse | `$ref` PROJJSON v0.7 | `crs-invalid-projjson` | full (GDAL) | — |
| crs-default | — | bbox range heuristic | — | `data/crs/crs-default.parquet` (positive) | **none** (heuristic only) | Inherently heuristic; test written accordingly. |
| crs-consistency | — | 2.0.0 branch: null↔`srid:0`, form-level checks | — | none (corpus deferred the `srid:0`/`null` and `auth:code`/PROJJSON variants) | **partial** | GDAL checks the *form* combinations, not semantic CRS equality. Needs fixtures: PROJJSON in `geo` vs different `EPSG:` code on the logical type; `null` vs a defined Parquet CRS. |
| epoch | `OptionalEpoch` | schema | `type number` | `epoch-on-unsupported-crs` (not a spec violation, SI-19); `data/epoch/*` positive | full | — |
| orientation-value | `OptionalOrientation` | schema | `const` | — | full | — |
| orientation-rings | `GeometryOrientation` (Polygon only) | `--check-data` (Polygon, MultiPolygon, GeometryCollection) | — | `orientation-ccw-declared-rings-cw` | full (GDAL) | GPQ misses MultiPolygon. No MultiPolygon negative fixture. |
| edges-value | `OptionalEdges` (planar/spherical only) | schema | `enum` | `data/edges/*` positive | full (schema) | GPQ rejects the four geodesic values. |
| bbox-array | `OptionalBbox` (4/6 only) | schema; min/max ordering; geographic ranges; **crashes on 8 elements** | `oneOf` 4/6/8 | `data/bbox/*` positive | **partial** | No tool handles 8 elements correctly. Antimeridian ordering (step 2) unchecked except by GPQ's containment logic. Needs 8-element and antimeridian fixtures. |
| bbox-extent | `GeometryBounds` (X/Y only, antimeridian aware) | — | — | `bbox-does-not-contain-geometry` | partial | Z/M ranges unchecked; GDAL none. |
| bbox-crs | — | geographic range heuristic | — | — | **none** (heuristic only) | Inherently heuristic. |
| media-type | — | — | — | — | **none** | Not testable on a file; test is conditional on a media type being used. |

## Conformance Class Bounding Box Covering (`/conf/covering/...`)

| Test | GPQ | GDAL | Schema | Corpus fixture | Coverage | What is missing |
| --- | --- | --- | --- | --- | --- | --- |
| keys | — | schema (`required [bbox]`; unknown keys **pass**, SI-13) | partial | — | partial | Unknown covering keys are not rejected by anyone. |
| bbox-paths | — | schema (array shapes, `const` second element) | partial | — | **partial** | Same-column-name (step 3) and column existence (step 4) checked by nobody. |
| bbox-column-structure | — | — | — | — | **none** | Needs a schema-level check and fixtures (wrong names, wrong order, `zmin` without `zmax`). |
| bbox-column-type | — | — | — | — | **none** | Fixture: `INT32` children; mixed `FLOAT`/`DOUBLE`. |
| bbox-column-repetition | — | — | — | — | **none** | Fixture: `required` bbox on `optional` geometry; bbox present where geometry is null. |
| bbox-column-nesting | — | — | — | — | **none** | Fixture: bbox group nested in a struct. |
| bbox-column-values | — | — | — | — | **none** | Fixture: bbox values that do not match the geometry. |

The Bounding Box Covering class currently has **no data-level test coverage anywhere**. The 1.1
`covering` was validated by no community tool either; it is the largest gap if PR #302 merges.

## Summary

| | Core (21 tests) | Covering (7 tests) |
| --- | --- | --- |
| full | 10 | 0 |
| partial | 6 | 2 |
| none / heuristic only | 5 | 5 |

Recommendations and permissions have no abstract tests by design (`/rec/core/*`, `/per/core/*`);
metanorma reports them as "no corresponding Conformance test" warnings, which CI tolerates.

## Suggested next steps for the test suite (outside this repository)

1. `geoparquet-testing`: add negative fixtures for the "none" rows above (unlisted logical-typed
   column, missing logical type, `geometry_types` vs statistics mismatch, CRS semantic mismatch,
   8-element and antimeridian `bbox`, MultiPolygon orientation, and the seven covering cases),
   and relabel `primary-column-not-in-columns` (SI-3).
2. GDAL `validate_geoparquet.py`: fix the 8-element `bbox` assertion, the `Point Z` mapping and
   the M/ZM types; add checks for the logical type, root/repetition and `geospatial_types`
   agreement; that would make GDAL a near-complete Core validator.
3. GPQ: either update to 2.0 (logical types, M/ZM, geodesic edges, 8-element bbox, MultiPolygon
   orientation, per-column optional-field iteration) or document it as a 1.0 validator.
4. A future OGC executable test suite can be a thin wrapper that runs GDAL's validator plus a
   Parquet-schema inspection (pyarrow) for the rows marked "none".
