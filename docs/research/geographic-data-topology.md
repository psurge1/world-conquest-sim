# Geographic data and territorial topology

Research date: 2026-09-25

## Question

Which primary geographic data sources, licenses, topology formats, and preprocessing strategy should supply modern Country boundaries and stable, high-resolution Geographic Regions whose ownership can transfer while combined Country borders render smoothly?

## Decision

Build the first modern-world geography pack from a **pinned geoBoundaries Comprehensive Global Administrative Zones (CGAZ) ADM2 release**, then convert it offline into a versioned, validated shared-arc topology owned by this project.

The pack should have three independent layers:

1. **Immutable geography:** Geographic Region polygons, shared border arcs, adjacency, centroids, area, and source provenance.
2. **Scenario assignment:** the initial `ownerCountryId` for every Geographic Region and the declared political worldview used to create that assignment.
3. **Mutable World state:** a compact array from Geographic Region index to current `ownerCountryId`.

Country borders should never be stored as mutable polygons and should never be recomputed by unioning polygons on every simulation update. A border segment is visible when the Geographic Regions on its two sides have different current owners; coastline segments have only one incident region. Both sides reference the same geometry at each level of detail, so a transfer changes state and border visibility without cutting or repairing polygons at runtime.

Use **TopoJSON as the canonical processed topology**, because it stores shared boundaries once as arcs, supports quantized coordinates, and permits topology-preserving simplification. Its reference client can also derive polygon neighbors and filter an arc mesh by the two features that share each arc ([TopoJSON format specification](https://github.com/topojson/topojson-specification), [TopoJSON project documentation](https://github.com/topojson/topojson), [topojson-client neighbor and mesh APIs](https://github.com/topojson/topojson-client)). The later rendering prototype may additionally package the same IDs and geometry as vector tiles in PMTiles for progressive delivery. Tiled data is a rendering artifact, not the source of adjacency or canonical topology.

This is a source and topology decision, not a claim that unprocessed ADM2 data will meet the browser performance target. The rendering prototype must measure the processed levels of detail and may establish deterministic aggregation rules if the full ADM2 partition is too dense.

## Why geoBoundaries CGAZ

geoBoundaries publishes global administrative boundary data and a CGAZ product for ADM0, ADM1, and ADM2. Unlike its high-precision single-country files, where each country represents its own claimed extent and neighboring files can overlap, CGAZ is a single global composite with gaps filled and international edges clipped to one worldview. That property makes CGAZ the best starting point for a non-overlapping simulation partition ([geoBoundaries repository and product definitions](https://github.com/wmgeolab/geoBoundaries)).

The project publishes versioned releases for replication. The build must pin a formal release and its checksum rather than consume the rolling `current` API or `main` branch during every install. The API documents source dates, licenses, vertex counts, and full and simplified download links; it also states that a `boundaryID` changes when its underlying data changes, which is why upstream identifiers cannot be the permanent identity used by saves ([geoBoundaries API](https://www.geoboundaries.org/api.html)).

The current CGAZ artifacts also show why preprocessing is mandatory. At review time, the rolling ADM2 file is **525 MB as GeoJSON** and **149 MB as a zip archive**; the ADM1 GeoJSON is 344 MB ([ADM2 GeoJSON](https://github.com/wmgeolab/geoBoundaries/blob/main/releaseData/CGAZ/geoBoundariesCGAZ_ADM2.geojson), [ADM2 zip](https://github.com/wmgeolab/geoBoundaries/blob/main/releaseData/CGAZ/geoBoundariesCGAZ_ADM2.zip), [ADM1 GeoJSON](https://github.com/wmgeolab/geoBoundaries/blob/main/releaseData/CGAZ/geoBoundariesCGAZ_ADM1.geojson)). These are build inputs, not browser assets.

### Source comparison

| Source | Strengths | Material limits | Decision |
| --- | --- | --- | --- |
| **geoBoundaries CGAZ ADM2** | Global ADM2 composite; gaps filled; versioned releases; attribution-friendly project license; strong starting granularity for territorial transfer | Large raw payload; administrative depth varies by country; CGAZ is clipped to a U.S. Department of State worldview; upstream IDs and geometry change between data releases | **Canonical source** |
| **geoBoundaries high-precision single-country files** | Highest detail offered by the project; per-file metadata | Neighboring countries may overlap because each file reflects that country's own view; reconciling them recreates the hardest part of CGAZ | Do not combine as the global partition |
| **Natural Earth 1:10m** | Public domain; polished cartography; explicit disputed-area and point-of-view products; compact; populated-place points | ADM1 has over 4,500 units, while global ADM2 is not supplied; therefore too coarse for the intended transfer resolution | Use for future city/capital points and dispute-reference overlays, not Region geometry |
| **GADM 4.1** | Detailed worldwide administrative hierarchy | Official license permits academic and other non-commercial use; redistribution or commercial use requires prior permission | Reject for a distributable game |
| **Overture Divisions / OpenStreetMap** | Rich division areas and explicit shared boundaries; Overture boundary records identify left/right divisions and support disputed perspectives | Divisions are ODbL; attribution and share-alike obligations add avoidable release and data-export complexity; dataset is much broader than this game needs | Keep as a future alternative, not the default |

Natural Earth publishes 258 country features, over 4,500 ADM1 units, disputed-area themes, and populated places; its ADM2 product is limited to the United States. Its Admin 0 zip is 4.7 MB and Admin 1 zip is 14.22 MB, illustrating the lighter but coarser option ([Natural Earth 1:10m cultural vectors](https://www.naturalearthdata.com/downloads/10m-cultural-vectors/)). All Natural Earth vector and raster data is public domain and may be modified and used commercially ([Natural Earth terms](https://www.naturalearthdata.com/about/terms-of-use/)).

GADM cannot be the default because its official license says redistribution or commercial use is not allowed without prior permission ([GADM license](https://gadm.org/license.html)).

Overture is technically attractive: a boundary feature identifies the divisions on its left and right and carries `is_disputed` and political perspective fields ([Overture DivisionBoundary schema](https://docs.overturemaps.org/schema/reference/divisions/division_boundary/)). However, Overture's Divisions theme is ODbL because it includes OpenStreetMap data ([Overture attribution and licensing](https://docs.overturemaps.org/attribution/)). OpenStreetMap requires attribution and says an altered or built-upon database may be distributed only under the same license ([OpenStreetMap copyright and license](https://www.openstreetmap.org/copyright/en)). This does not prevent commercial use, but it introduces compliance and data-export questions that geoBoundaries avoids for this first game.

## License and attribution plan

geoBoundaries' repository license places project-generated code and derivative works under CC BY 4.0, allowing commercial adaptation with attribution and change notices. It also warns that individual boundary files are governed by the license recorded in each file's metadata ([geoBoundaries license](https://github.com/wmgeolab/geoBoundaries/blob/main/LICENSE)). Therefore the importer must preserve the selected release's license metadata and fail the build if any selected source is outside the approved license set.

Ship the following in an in-game About/Data Sources view and alongside downloadable geography data:

- geoBoundaries name and project link;
- the exact release/tag and import checksum;
- `CC BY 4.0` with a license link;
- a statement that the geometry was normalized, topologized, simplified, and repackaged;
- the chosen political worldview and a boundary-accuracy disclaimer.

If Natural Earth points or disputed-area metadata are included later, attribution is optional under its public-domain terms, but “Made with Natural Earth” is still useful provenance.

## Geographic Region identity and World state

The source dataset and a running World have different lifecycles. Preserve that separation explicitly.

### Geography-pack manifest

Each generated pack should have a manifest similar to:

```json
{
  "geographyPackId": "modern-cgaz-adm2-v1",
  "schemaVersion": 1,
  "source": {
    "name": "geoBoundaries CGAZ",
    "release": "pinned-release",
    "sha256": "...",
    "license": "CC-BY-4.0",
    "worldview": "CGAZ / U.S. Department of State clipping"
  },
  "regionCount": 0,
  "countryRosterVersion": 1,
  "lods": []
}
```

The project should assign dense numeric Region indexes for runtime arrays and stable string Region IDs scoped to the geography-pack version. Keep a crosswalk from each Region ID to the upstream feature ID and, if a multipart source unit is split, to its component ordinal. A serialized World must record `geographyPackId`; a newer pack requires an explicit migration rather than silently applying new polygons to an old save.

Country IDs should also be internal stable IDs. ISO codes, Natural Earth codes, and geoBoundaries names are aliases in a curated country-roster table because disputed and dependent entities do not map cleanly to one external code system.

### Runtime data shape

Use structure-of-arrays data for the hot path:

- `regionOwner[regionIndex] -> countryIndex` for mutable ownership;
- `neighborOffsets` and `neighborRegionIndexes` as a compact adjacency graph;
- `edgeLeftRegion` and `edgeRightRegion` for shared arcs (`right = none` for coastline);
- immutable Region metadata such as area, centroid, parent source unit, and initial owner;
- geometry buffers keyed by the same dense Region and edge indexes.

At around 100,000 Regions, ownership itself is only roughly 0.2 MB with 16-bit Country indexes or 0.4 MB with 32-bit indexes. Adjacency and geometry are larger, but geometry will dominate. This supports the client-first model as long as rendering data is simplified and progressively delivered.

## Preprocessing pipeline

The geography build should be deterministic and run outside the browser.

1. **Acquire and pin.** Download one formal CGAZ release, record its URL, release identifier, SHA-256 checksum, metadata, and licenses. Never pull `current` during a production build.
2. **Read the global ADM2 coverage.** Prefer one Geographic Region per connected ADM2 feature component. Where ADM2 coverage is absent or invalid, fall back in a recorded order to ADM1 and then ADM0 so every playable land area has a Region.
3. **Normalize coordinates.** Convert to WGS 84 longitude/latitude, normalize ring winding and holes, repair invalid polygons, and split geometry at the antimeridian. GeoJSON requires WGS 84 longitude/latitude and recommends cutting antimeridian-crossing geometry for interoperability ([RFC 7946](https://www.rfc-editor.org/rfc/rfc7946)).
4. **Create one coverage.** Snap near-equal border coordinates, remove or resolve overlaps and gaps, eliminate zero-area slivers, and make each shared boundary a single canonical arc. Preserve an audit log for every repair.
5. **Assign project IDs.** Generate version-scoped Region IDs, dense indexes, source crosswalks, and initial Country owners. Do not hash raw floating-point geometry as the sole identity.
6. **Build topology before simplifying.** Convert the cleaned coverage to shared arcs, derive left/right Region indexes and symmetric adjacency, and only then simplify the arcs. Simplifying polygons independently creates cracks or overlaps. TopoJSON explicitly stores common arcs once, supports integer quantization and delta encoding, and supports topology-preserving simplification ([TopoJSON specification](https://github.com/topojson/topojson-specification), [TopoJSON documentation](https://github.com/topojson/topojson)).
7. **Generate levels of detail.** Produce at least globe, regional, and close-view geometries from the same arcs and Region IDs. A lower level may omit small visual components, but it must not change simulation identity or adjacency.
8. **Generate renderer-neutral products.** Emit canonical TopoJSON, Region metadata, compact adjacency/edge arrays, and triangulation inputs. If the rendering prototype benefits from tiles, also emit vector tiles with globally stable feature IDs and package them into PMTiles. PMTiles is a single-file tiled archive suitable for static hosting; clients fetch needed byte ranges without a custom tile server ([PMTiles project and v3 specification](https://github.com/protomaps/PMTiles)).
9. **Attach optional point data.** Join capital or city points to internal Country and Region IDs only after the polygon topology is final.
10. **Write provenance.** Generate attribution text, source/version details, transformation history, checksums, and repair statistics with the pack.

Do not use vector tiles as the canonical topology. Tile clipping can duplicate or cut features, and the Mapbox Vector Tile specification stores each tile in its own integer screen-coordinate extent ([Mapbox Vector Tile 2.1 specification](https://github.com/mapbox/vector-tile-spec/blob/master/2.1/README.md)). Tiles are useful for delivery; canonical adjacency and ownership must come from the untiled coverage.

## Smooth dynamic borders

The visible Country outline should be a filtered shared-edge mesh:

```text
draw edge when:
  edge.rightRegion is absent
  OR owner(edge.leftRegion) != owner(edge.rightRegion)
```

The fill color for a Region is looked up from its owner's current presentation state. A transfer updates `regionOwner`; the renderer updates the affected Region fill and only edges incident to that Region. There is no polygon mutation and no whole-Country union.

The word “smooth” should mean:

- neighboring Regions use exactly the same border coordinates;
- simplification is applied once per shared arc;
- the selected level of detail has enough vertices for the current zoom;
- a line on the sphere is subdivided enough that projection onto the Globe does not reveal long straight chords;
- switching levels of detail does not expose cracks.

It should not mean procedurally smoothing a political boundary after transfer. Smoothing that moves an edge independently can create overlaps, gaps, or a border that no longer matches selectable Region fills.

## Disputed boundaries and political viewpoint

No modern border dataset is politically neutral. CGAZ uses a global composite clipped to U.S. Department of State international boundaries, while its high-precision country files can overlap because each country represents itself ([geoBoundaries product definitions](https://github.com/wmgeolab/geoBoundaries)). Natural Earth defaults to de facto control, publishes disputed polygons and claim lines, and supports multiple point-of-view variants ([Natural Earth disputed-boundaries policy](https://www.naturalearthdata.com/about/disputed-boundaries-policy/)).

For the first geography pack:

- declare CGAZ's worldview in the manifest and product UI;
- call the data approximate simulation geography rather than an authoritative legal statement;
- keep `initialOwner`, current `owner`, and any future `claimedBy` or `disputedBy` metadata as separate concepts;
- do not silently combine polygon edges from CGAZ and Natural Earth, because slightly different borders would create seams;
- treat an alternative worldview as a separately built geography pack or a reviewed reassignment that follows the same Region edges.

Natural Earth's disputed-area features may later tag approximate Region sets for explanatory overlays, but any automated spatial join must be reviewed where its geometry does not align with the CGAZ partition.

## Capital and city points

Cities should remain an optional point layer and must not define polygon topology. Natural Earth's populated-places dataset contains all Admin 0 capitals, many Admin 1 capitals, major cities, towns, scale ranks, and names; its full download is 2.68 MB and a reduced-column version is 636.84 KB ([Natural Earth populated places](https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-populated-places/)). It is a suitable low-friction source for future capital and sparse city points.

On import, map each point to:

- an internal point ID;
- `regionId` by point-in-polygon;
- `countryId` through the curated roster;
- point kind such as country capital, administrative capital, or city;
- display rank and names.

Points should not be embedded in Region polygons, so the game can add, remove, or replace city data without changing Region IDs or invalidating a serialized World.

## Payload and delivery implications

Raw CGAZ is far beyond a reasonable browser startup payload. The build should measure all emitted artifacts both uncompressed and with the actual HTTP compression used in deployment. TopoJSON commonly reduces duplication substantially by sharing arcs and supports quantized delta encoding, but the project's “80% or more” observation is not a guarantee for this dataset ([TopoJSON documentation](https://github.com/topojson/topojson)). RFC 7946 also warns that excessive coordinate precision can nearly double detailed GeoJSON payloads; six decimal places are already about ten-centimeter precision, much finer than this game's cartographic need ([RFC 7946 coordinate precision](https://www.rfc-editor.org/rfc/rfc7946)).

Use these delivery rules:

- load a coarse whole-Globe level first;
- decode and triangulate off the interaction-critical path, preferably in a worker if the renderer does not already do so;
- stream close geometry by view if the full processed pack misses the prototype budget;
- keep ownership and adjacency loaded independently from detailed geometry so the simulation never waits on camera detail;
- cache immutable pack assets by content hash;
- preserve Region IDs across every level and tile.

If the full ADM2 pack remains too expensive or too uneven for play, apply a deterministic aggregation and subdivision policy during geography generation rather than dropping arbitrary features in a tile encoder. The policy should preserve Country coastlines, avoid disconnected Regions where practical, define minimum and maximum area targets, and record source membership so a later higher-resolution pack can migrate ownership intentionally.

## Required validation

The generated geography pack is acceptable only if the build verifies:

- every Region polygon is valid, non-empty, and uses the expected ring winding;
- land coverage has no unintended overlaps or gaps above a declared tolerance;
- every non-coast edge has exactly two incident Regions and every coast edge has one;
- adjacency is symmetric and agrees with the shared-edge table;
- all initial owners and all Regions resolve to curated IDs;
- every level of detail preserves Region count, Region IDs, adjacency, and shared edges;
- antimeridian, polar, island, enclave, exclave, hole, and multipart cases render and pick correctly;
- no Region disappears because of simplification;
- country unions at the initial assignment match the selected source within declared tolerances;
- attribution and provenance artifacts are present.

## Inputs for the globe-rendering prototype

The later prototype should receive a small, reproducible geography fixture and the full processed pack, each generated by the same pipeline.

Required artifacts:

- manifest with pack/version/worldview/license/checksum;
- Region metadata and dense index mapping;
- initial and mutable owner arrays;
- shared edges with left/right Region indexes;
- adjacency arrays;
- three geometry levels of detail;
- polygon triangulation or enough canonical geometry to benchmark triangulation;
- optional MVT/PMTiles output using the same Region IDs;
- a sample capital/city point layer;
- source-to-project ID crosswalk and attribution text.

The fixture should cover dense borders, islands and holes, an enclave/exclave, and an antimeridian case. The prototype should measure:

- compressed startup and total bytes;
- decode, topology setup, and triangulation time;
- peak JavaScript and GPU memory;
- frame time while rotating and zooming the Globe;
- picking latency;
- cost of changing 1, 100, and 1,000 Region owners;
- border correctness after simultaneous transfers;
- cracks or mismatches across levels of detail, tiles, and the antimeridian.

The prototype must compare at least a whole-world TopoJSON path with a progressively delivered tile path if the whole-world artifact misses the agreed budget. It should not revisit which dataset defines Region identity.

## Material uncertainties

1. **ADM2 density is uneven.** Administrative levels do not represent equal physical or gameplay scale across countries. The rendering and conflict prototypes must decide whether direct ADM2 granularity feels consistent enough or whether a deterministic aggregation layer is necessary.
2. **Multipart units complicate transfer.** A source ADM2 unit may include disconnected islands. Splitting connected components improves physical adjacency but increases Region count and can make tiny islands individually transferable. The generator needs a documented component and minimum-area policy.
3. **The political worldview is a product choice.** CGAZ's single partition is technically useful, but its U.S. Department of State clipping may be inappropriate for some audiences. Supporting alternative worldviews means separate reviewed packs, not a runtime polygon toggle.
4. **The browser budget is unmeasured.** Source sizes prove that raw delivery is unsuitable, but only the rendering prototype can choose simplification thresholds, whole-pack versus tiled delivery, and the final Region count.
5. **Upstream updates require migrations.** Corrected boundaries can alter source IDs and topology. Saves must remain tied to their geography-pack version until an explicit crosswalk migration exists.
6. **License metadata must be audited at import.** The project-level license is CC BY 4.0, while geoBoundaries explicitly records file-specific licenses. The selected release's actual metadata is the release gate.

## Recommendation in one line

Pin geoBoundaries CGAZ ADM2, transform it offline into a versioned shared-arc geography pack with separate owner state and precomputed adjacency, add Natural Earth points later, and let the rendering prototype choose whole-pack or PMTiles delivery without changing Geographic Region identity.
