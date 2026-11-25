# PR #304 description

This is the text of the description of opengeospatial/geoparquet#304, kept here so it is versioned
with the document. Last updated 2026-11-25.

Title: **OGC Standard document for GeoParquet 2.0 (metanorma), aligned with the 2.0.0 release**

---

## Why

The OGC GeoParquet SWG needs an OGC-formatted version of the specification (metanorma, OGC
Standard template) to take to the OGC Architecture Board and to a membership vote. That work
stalled in December 2024 with #224 (@m-mohr, based on @ghobona's migration) and #247 (@cholmes),
both written for GeoParquet 1.0. The SWG (with OGC staff) agreed to target **2.0** directly: 1.x
got the community to implement, 2.x is what implementers should adopt, and it is far less likely
to need another major revision.

GeoParquet 2.0.0 was tagged on 2026-11-05, and this document now follows the tagged text. It
changes nothing under `format-specs/`.

## Status (2026-11-25)

Ready for a final review by the maintainers. After that it goes to an SWG vote to send it to the
OAB, then to public comment and the membership vote. I am working out the calendar with OGC staff.

Changes since the September review round:

* **Synced with the v2.0.0 tag.** The normative text of the tag differs from rc.1 only by the
  `covering` text of #302, which Clause 7 already restated, so no provision had to change. Every
  line citation now points at the tagged file.
* **Parquet references pinned to 2.14.0** (September 2026) instead of 2.12.0. 2.12.0 does not
  contain the inline PROJJSON and `<authority>:<code>` forms of the `crs` parameter that the
  specification recommends; 2.13.0 and 2.14.0 do (SI-9, CHANGES G7).
* **Cloud-Optimized Distribution keeps only its two requirements.** The six best-practice
  recommendations (row group size, compression, bbox covering, declared bbox column,
  partitioning, STAC) are removed, as Matthias proposed and Joris asked for in review. The
  community guide is cited as informative instead.
* **New Native Geospatial Types class (Clause 9)**, as agreed in #308. It is for Parquet files
  whose geometry columns use the native `GEOMETRY`/`GEOGRAPHY` types and that carry no `geo`
  metadata. It does not make such files GeoParquet files, and Core is unchanged.
* The spatial-order test describes the pruning measure in terms that fit the method
  geoparquet-io 1.8.0 and Portolan now use.

## What is in the PR

Everything lives in a new `ogc/` directory plus `docs/ogc/`:

* `ogc/` is the metanorma document for GeoParquet **2.0.0**, with four requirements classes, all
  targeting the Parquet file:

  | Class | Applies to | Requirements | Recommendations | Permissions |
  | --- | --- | --- | --- | --- |
  | Core | every GeoParquet file | 20 | 6 | 3 |
  | Bounding Box Covering | files that declare a `covering` (#302) | 6 | | |
  | Cloud-Optimized Distribution | optional profile for direct access from object storage | 2 | | |
  | Native Geospatial Types | native Parquet geometry, no `geo` metadata | 5 | 2 | |

  * There is one abstract test per requirement (33). Every requirement file starts with a
    `// Source:` comment that points at the line of `geoparquet.md` 2.0.0 it restates.
  * The normative references pin Apache Parquet 2.14.0 (`LogicalTypes.md`, `Geospatial.md`)
    instead of restating the encoding.
  * The community spec's expectations on *readers* are informative (Clause 10): interpret
    PROJJSON and authority:code `crs`, don't reject unknown fields, forward compatibility. The
    SWG chose a single standardization target, the file, to keep approval simple.
* `docs/ogc/CHANGES.md`: **start here if you review.** It lists every place where the OGC wording
  differs from `geoparquet.md`, with the reason. The intent is that the OGC text is purely
  derivative: where the markdown implies something without stating it, the OGC text has a NOTE,
  not a requirement. The one exception is the Native class, which restates five Core
  requirements for files without `geo` metadata (see question 1).
* `docs/ogc/spec-issues.md`: things in the community spec that the drafting surfaced. SI-9 is
  resolved. SI-33 is new: `geoparquet.md` says readers *should* read native-only files, while
  the compatibility guide says they are *required* to. Nothing was changed in `format-specs/`.
* `docs/ogc/gaps.md`: for each abstract test, which existing tool implements it. The
  `geoparquet-validator` (validator.geoparquet.org) implements all 33 tests under the same
  identifiers once geoparquet/geoparquet-validator#8 is merged.
* `docs/ogc/AUDIT.md` and `docs/ogc/crosswalk.md`: the audit of #224/#247 against 1.1 and 2.0
  that this work started from (rc.1 line numbers).
* `.github/workflows/ogc-document.yml` builds the document with the `metanorma/metanorma` Docker
  image on pushes and PRs that touch `ogc/**`. It fails on metanorma errors or on a requirement
  without a test, and uploads the HTML. It replaces the old `docs.ogc.org/DRAFTS/24-013` cron
  build, which is gone (#277).

The #224 and #247 branches are merged into this branch so the original authorship is preserved
in history; the files were then moved from `format-specs/` to `ogc/` and rewritten for 2.0.

## Questions for reviewers

1. **Shape of the Native class.** It is a separate class that restates five Core requirements
   (column type, nesting, repetition, WKB, axis order) for files without `geo` metadata, so the
   reviewed Core text stays untouched. The alternative is Core inheriting Native: each statement
   would appear once, but five Core requirements would move to new identifiers and
   `/req/core/geometry-column-type` would be split. Which do you prefer?
2. **Native files and the Distribution profile.** The profile inherits Core, so a native-only
   file cannot claim it, although its two requirements (statistics, spatial order) do not need
   the `geo` key. Should the profile also be claimable by Native files? This matters for table
   formats such as Iceberg.
3. **Editors and contributors.** `document.adoc` lists Chris Holmes, Tim Schaub, Joris Van den
   Bossche, Kyle Barron and Javier de la Torre as editors (carried over from #224). Who else
   should be there, and who should be in the Contributors table? @cholmes, your view first.
4. **Document number 24-013** was assigned for the 1.0 draft. I have asked OGC staff to confirm
   that it carries over.

## How to build locally

```sh
cd ogc
docker run --rm -v "$PWD":/metanorma metanorma/metanorma \
  metanorma compile --agree-to-terms -t ogc -x xml,html document.adoc
open document.html
```

Supersedes #224 and #247. Refs #277, #302, #308.
