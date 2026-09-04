# Draft reply to Chris on PR #304 (for the chair to edit and post)

---

Thanks Chris. Both points are now in the branch.

**1. Assertions vs `gpio`.** You are right that `gpio` is the 2.0 test tool; the first pass of
`docs/ogc/gaps.md` had mapped the abstract tests to GPQ and GDAL only. I went through
`geoparquet_io/core/validate.py` (main, `4ebaa9a`) check by check:

* Every abstract test in the document now ends with a NOTE naming the `gpio validate` (or
  `gpio check`) check that implements it, or saying that no tool does. `docs/ogc/gaps.md` has the
  full table, both directions: OGC test to `gpio` check, and `gpio` checks that are not OGC
  requirements (`v2_crs_in_parquet_type`, `v2_edges_consistency`, `epoch` datum checks, the version
  feature checks).
* With `gpio` counted, Core is 10 tests fully covered, 6 partially, 4 heuristic-only; the covering
  class is 5 partial and 1 uncovered. `gpio` closes the logical-type, CRS-consistency and covering
  structure checks that nothing else did.
* Things I found in `gpio` while doing this, in case they are useful (also in
  `docs/ogc/spec-issues.md`, section E): `orientation_matches_data` is a no-op; `bbox_valid` rejects
  the 2.0 8-element bbox; `bbox_contains_data` reads `bbox[:4]`, which is wrong for a 6-element
  bbox; `v2_crs_consistency` reads a bare `authority:code` as unset (your #814); the covering path
  checks do not verify the second element or that all four paths name one column.
* Two places where `gpio` is stricter than the spec and I did not follow `gpio`: it requires a
  non-default CRS to be inline PROJJSON in the Parquet type, while the spec recommends
  `authority:code` for writers that cannot produce PROJJSON (SI-25); and it fails a GEOGRAPHY column
  that omits `edges`, which the spec does not say but probably should (SI-12). Both are worth a
  decision one way or the other.

**2. A profile for "a good GeoParquet".** Drafted as Clause 8, *Requirements Class Cloud-Optimized
Distribution*, an optional class that Core does not depend on, so Portolan (or anyone) can require
"GeoParquet 2.0 conforming to the Cloud-Optimized Distribution class". It has five requirements, one
per `gpio check` factor, and four recommendations:

| Requirement | Source | Test |
| --- | --- | --- |
| geospatial statistics on every row group | guide, `check optimization` factor 2 | `native_geo_stats` |
| spatially ordered: fewer than 30 % of consecutive row-group bboxes overlap | guide, `check spatial` | bbox-stats method |
| 10 000–200 000 rows per row group (single group allowed below 64 MB) | guide, `check row-group` | `check row-group` |
| ZSTD on geometry columns | guide, `check compression` | `check compression` |
| any bbox column declared in `covering` | guide (PR 302 text), `check bbox` | `check bbox` |

Recommendations: ZSTD level 15 or more, 50 000–150 000 rows or 64–256 MB, spatial partitioning above
about 2 GB, STAC metadata. Everything is sourced line by line in `docs/ogc/CHANGES.md` (section
"Requirements Class Cloud-Optimized Distribution").

This is the one part of the document that is not a restatement of the community spec, so I have
marked it as a proposal throughout. Three things I would like your view on:

* The row-group numbers. The guide's tl;dr says 50k–150k, the body 50k–200k, `check row-group`
  64–256 MB, `check optimization` 10k–50k. I took the widest accepted range for the requirement and
  the tl;dr for the recommendation, but one agreed set would be better (SI-24).
* Whether the thresholds should live in the OGC document at all, or whether the class should say
  "spatially ordered" and leave the metric to the test suite.
* Whether Portolan needs anything the guide does not have.

Nothing in `format-specs/` changed; the guide is only cited.
