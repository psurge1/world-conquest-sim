# Globe rendering and geospatial stack

Research for [Select the globe rendering and geospatial stack](https://github.com/psurge1/world-conquest-sim/issues/2), 2026-09-25.

## Decision

Use **MapLibre GL JS 6.x** for the first playable Globe, behind a project-owned rendering adapter. Compile immutable Geographic Region geometry into **Mapbox Vector Tiles (MVT 2.1)** stored in a **PMTiles v3** archive. Use **TopoJSON only in the build pipeline** to preserve shared arcs and derive adjacency and boundary-edge assets; do not make it the runtime rendering format.

This choice is subject to a focused performance prototype. It best matches the actual product: a clean political Globe made of selectable vector regions whose colors, selection state, and ownership borders change while their geometry remains stable. It avoids making the project own a globe camera, spherical projection, tile pyramid, antimeridian handling, hit testing, and vector level-of-detail system.

Use **CesiumJS** as the fallback if the prototype shows that MapLibre cannot meet the required region count or update rate, or if the product later requires an ellipsoidal Globe at very close zoom, exact polar rendering, terrain, or substantial height-aware 3D content.

Do not begin with deck.gl `GlobeView`, a custom Three.js globe, Mapbox GL JS, or Web WorldWind.

## Why MapLibre is the best fit

MapLibre GL JS now has a first-class globe projection. Its own globe design guide says the same vector polygons and lines used in Mercator are projected onto a unit sphere in the vertex shader, with added subdivision so large polygons and lines curve correctly; the official example demonstrates a globe backed by vector tiles. This is the core rendering path this project needs, rather than an add-on mesh or a raster image pasted onto a sphere. [MapLibre globe design](https://github.com/maplibre/maplibre-gl-js/blob/main/developer-guides/globe.md), [vector-globe example](https://maplibre.org/maplibre-gl-js/docs/examples/display-a-globe-with-a-vector-map/)

The rest of its public API also matches the interaction model:

- A vector or GeoJSON feature with a stable ID can receive runtime state through `setFeatureState`; style expressions can read that state, and `fill-color`, `fill-outline-color`, and line paint properties support data-driven styling. Ownership color, selection, hover, and border visibility can therefore change without replacing polygon geometry. [Map API](https://maplibre.org/maplibre-gl-js/docs/API/classes/Map/), [feature-state expression](https://maplibre.org/maplibre-style-spec/expressions/#feature-state), [fill style properties](https://maplibre.org/maplibre-style-spec/layers/#fill)
- `queryRenderedFeatures` provides built-in hit testing for visible regions. The application can resolve the picked stable `region_id` to the current Country in the World rather than treating a map feature as simulation state. [Map query API](https://maplibre.org/maplibre-gl-js/docs/API/classes/Map/#queryrenderedfeatures)
- Vector tiles are fetched, decoded, laid out, and spatially indexed in Web Workers before render-ready buffers return to the main thread. This does not make rendering free, but it gives the project a credible non-blocking geometry path without building a worker-aware renderer. [MapLibre architecture](https://github.com/maplibre/maplibre-gl-js/blob/main/ARCHITECTURE.md#main-thread--worker-split)
- GeoJSON sources are internally tiled, and the API provides `updateData` for feature-level diffs, but the base geography should use prebuilt vector tiles and feature state. `setData` or `updateData` remains useful for small, genuinely dynamic overlays such as temporary conflict paths. [GeoJSONSource API](https://maplibre.org/maplibre-gl-js/docs/API/classes/GeoJSONSource/)
- Custom WebGL layers can share MapLibre's context and camera, including on the globe projection. That leaves an escape hatch for future visual effects without putting a second renderer into the first playable. [custom layer API](https://maplibre.org/maplibre-gl-js/docs/API/interfaces/CustomLayerInterface/), [custom globe layer example](https://maplibre.org/maplibre-gl-js/docs/examples/add-a-custom-layer-with-tiles-to-a-globe/)
- MapLibre GL JS is BSD-3-Clause and does not require a hosted map account or paid basemap. [MapLibre license](https://github.com/maplibre/maplibre-gl-js/blob/main/LICENSE.txt)

The Globe should use an intentionally sparse local style: background/ocean, region fills, ownership borders, selection/hover, and optional conflict or city overlays. A commercial or general-purpose basemap is unnecessary for the first playable.

## Known constraints in the recommendation

MapLibre's globe is a globe presentation of Web Mercator tiles rather than a general 3D ellipsoid scene. The prototype must test these documented boundaries:

- Globe projection smoothly transitions to Mercator around zoom 12 because 32-bit shader precision is insufficient for a full globe at very high zoom. That is acceptable for a world-scale simulation only if the first playable does not promise a visibly spherical Globe during street-level zoom. [MapLibre globe precision and transition](https://github.com/maplibre/maplibre-gl-js/blob/main/developer-guides/globe.md#floating-point-precision--transitioning-to-mercator)
- Web Mercator has no source data above roughly 85.05 degrees latitude. MapLibre fills the polar gap by stretching pole-adjacent tiles. This needs visual review around the Arctic and Antarctica and disqualifies the stack if exact polar geography becomes a requirement. [MapLibre polar behavior](https://github.com/maplibre/maplibre-gl-js/blob/main/developer-guides/globe.md#data-near-the-poles)
- Globe camera motion has special zoom and pole-crossing behavior. It should be evaluated as product interaction rather than assumed from the flat-map API. [MapLibre globe controls](https://github.com/maplibre/maplibre-gl-js/blob/main/developer-guides/globe.md#controls)
- MapLibre GL JS 6 requires WebGL2. A failed WebGL2 initialization must become a clear compatibility screen; WebGL1 is not a supported fallback. [v5-to-v6 migration guide](https://maplibre.org/maplibre-gl-js/docs/guides/v5-to-v6-migration-guide/#webgl2-is-now-required)
- The v6 package is ESM-only and uses a rendering worker. Follow the documented bundler-specific worker setup and pin the exact version proved by the prototype. [MapLibre installation](https://maplibre.org/maplibre-gl-js/docs/)

These are clearer and narrower risks than the amount of geospatial behavior a custom engine would need to create and maintain.

## Runtime design

### Keep the renderer downstream of the World

MapLibre must be an output adapter, never the owner of simulation state. A small project interface should isolate it, along these lines:

```ts
interface GlobeRenderer {
  initialize(container: HTMLElement, geography: GeographyManifest): Promise<void>;
  applyRegionVisualPatches(patches: readonly RegionVisualPatch[]): void;
  applyConflictOverlay(snapshot: ConflictOverlaySnapshot): void;
  setSelection(selection: GlobeSelection | null): void;
  onRegionSelected(listener: (regionId: RegionId) => void): () => void;
  dispose(): void;
}
```

The interface should expose project terms and serializable values, not MapLibre maps, sources, layers, or events. The simulation can then run in its own worker, tests can use a fake renderer, and a later move to Cesium does not cross the simulation boundary.

### Treat ownership as state, not geometry

The Geographic Region polygons and their shared edges are immutable during a World. A transfer changes which Country owns a region; it does not cut a polygon at runtime.

For every ownership patch, the adapter should:

1. Resolve the Country's display color outside the style expression.
2. Call `setFeatureState` for the changed `region_id` with the new fill color and any overlay flags.
3. Visit only that region's adjacent edge IDs. For each edge, compare the current owners of its left and right regions and update the edge's `is_country_border` feature state.
4. Coalesce all patches produced during a simulation frame and apply them once per animation frame. If a burst exceeds the prototype's frame budget, spread it across frames while retaining the latest value per region and edge.

This update cost scales with changed regions and their adjacent edges, rather than with all geometry. It also avoids calling `GeoJSONSource.setData`, which reparses and retiles the source.

Selection should query only the region fill layer, deduplicate by stable `region_id`, and ask the World model for the owning Country. A rendered tile is a view of the geography, not the canonical record of ownership.

### Render smooth ownership boundaries from shared edges

Do not draw every Geographic Region's polygon outline. That would reveal internal transfer units and can double-draw coincident lines. Generate one line feature per shared topological edge (or per joined edge chain), with stable IDs and `left_region_id` / `right_region_id`. A line layer displays the edge when the regions have different owners; coastlines live in a separate, always-visible layer.

Smoothness should come from:

- a topology-preserving source with shared arcs;
- zoom-specific simplification performed on shared arcs, before polygons and edge lines are emitted;
- enough tile extent and source detail for the target zooms;
- MapLibre's globe subdivision and antialiased line rendering;
- round line joins and caps where the visual design calls for them.

The asset build must cut antimeridian-crossing GeoJSON geometry, as RFC 7946 recommends, before topology or tiling. [RFC 7946, section 3.1.9](https://www.rfc-editor.org/rfc/rfc7946.html#section-3.1.9)

## Asset pipeline and formats

### 1. Canonical processed topology: TopoJSON

After ingesting the selected geographic dataset, normalize it to WGS84, repair invalid rings, assign immutable integer region IDs, and create a TopoJSON topology. TopoJSON stores shared sequences of positions as arcs, supports quantized and delta-encoded coordinates, and lets the build derive neighbors, merged shapes, and a boundary mesh without comparing duplicate polygon coordinates. [TopoJSON specification](https://github.com/topojson/topojson-specification), [TopoJSON client operations](https://github.com/topojson/topojson-client)

TopoJSON is an intermediate and review artifact, not the main browser asset. MapLibre's direct runtime paths are GeoJSON and vector tiles, so converting TopoJSON in the browser would add load and memory cost while throwing away its topology before rendering.

### 2. Runtime geometry: MVT 2.1 in PMTiles v3

Generate an MVT pyramid with at least these layers:

| Layer | Geometry | Required stable fields | Purpose |
| --- | --- | --- | --- |
| `regions` | Polygon / MultiPolygon | `region_id` | fill, selection, hover, ownership state |
| `region_edges` | LineString / MultiLineString | `edge_id`, `left_region_id`, `right_region_id` | dynamic Country ownership borders |
| `coastlines` | LineString / MultiLineString | `coastline_id` | stable land/ocean boundary |
| `places` | Point | `place_id`, `kind` | optional future capitals and simple city points |

MVT encodes tiled vector geometry and compact feature attributes as Protocol Buffers. Its specification deliberately leaves clipping and simplification to the producer, so the project must own and test those steps rather than assuming the format preserves topology automatically. [MVT 2.1 specification](https://mapbox.github.io/vector-tile-spec/)

Package the pyramid as PMTiles v3. PMTiles is a single-file tiled-data archive whose root directory fits in the first 16 KiB; the browser fetches only needed ranges. The official JavaScript adapter registers a `pmtiles://` protocol directly with MapLibre. The specification is CC0/public domain and its reference implementations are BSD-3-Clause. [PMTiles v3 specification](https://github.com/protomaps/PMTiles/blob/main/spec/v3/spec.md), [MapLibre integration](https://docs.protomaps.com/pmtiles/maplibre), [PMTiles license](https://github.com/protomaps/PMTiles/blob/main/LICENSE)

PMTiles keeps deployment static and serverless, but its host must support HTTP Range requests and correct CORS headers. If the chosen host cannot do that reliably, emit the same MVT pyramid as ordinary `{z}/{x}/{y}.pbf` files; this changes delivery, not the renderer or layer schema. [PMTiles hosting requirements](https://docs.protomaps.com/pmtiles/cloud-storage)

The renderer and format licenses do not grant rights to the underlying geographic data. Preserve source, version, license, required attribution, and transformation history in the asset manifest and the user-visible attribution control. PMTiles reserves an `attribution` metadata field for this purpose. [PMTiles v3 metadata](https://github.com/protomaps/PMTiles/blob/main/spec/v3/spec.md#5-json-metadata)

### 3. Non-geometric runtime metadata

Emit a small, versioned `geography-manifest` asset beside the tiles. It should contain:

- asset schema version and source-data attribution;
- stable region IDs and initial Country IDs;
- region area and other facts the simulation actually needs;
- region-to-edge and region-to-region adjacency;
- edge left/right region IDs;
- asset checksums and bounds.

The World should serialize IDs and simulation values, not polygon coordinate arrays. A saved World can then bind to a compatible geography asset by schema version and checksum.

### Formats not chosen

- **Raw GeoJSON** is useful for the prototype's smallest fixture and for dynamic overlays. It is human-readable and standardized, but repeats shared borders and loads a whole feature collection. MapLibre recommends vector tiling, chunking, and simpler styles for large GeoJSON datasets. [MapLibre large-data guide](https://maplibre.org/maplibre-gl-js/docs/guides/large-data/)
- **TopoJSON in production** retains topology but is not MapLibre's native tiled source. Keep it in the asset compiler.
- **3D Tiles / glTF** suit streamed 3D models, terrain-adjacent content, buildings, and point clouds. They add no benefit for flat political polygons on this Globe. Reserve glTF for future visual objects and reconsider 3D Tiles only if the product gains substantial 3D content.
- **Raster ownership textures** can make massive state updates cheap, but lose exact feature picking and vector boundary quality unless paired with a second geometry/index system. They are a later optimization path, not the first maintainable implementation.

## Alternatives evaluated

| Option | Strengths | Material constraints for this project | License | Conclusion |
| --- | --- | --- | --- | --- |
| **MapLibre GL JS 6.x** | Built-in globe projection for vector fills and lines; vector tiling, worker parsing, feature-state styling, hit testing, and custom layers | WebGL2 only; Web Mercator polar gap; transitions toward Mercator near zoom 12; dynamic border state still needs project-owned adjacency | BSD-3-Clause | **Choose for the prototype and first playable** |
| **CesiumJS** | True high-precision WGS84 globe; terrain/imagery; GeoJSON and TopoJSON loading; time-dynamic visualization; batched primitives, picking, and per-instance color updates | More engine than a stylized political Globe needs; high-volume region data needs a custom Primitive/3D Tiles pipeline; dynamic ownership borders remain project logic; integration and visual customization are heavier | Apache-2.0 | **Fallback** for polar/close-zoom/terrain or if MapLibre fails the benchmark |
| **Three.js** | Maximum shader and scene control; buffer attributes can be partially updated; built-in camera controls and raycasting | The project would own spherical projection, tile/LOD selection, topology, antimeridian and poles, triangulation, line rendering, hit-test acceleration, and resource lifecycle | MIT | Reject initially; consider only for a proven specialized renderer need |
| **deck.gl GlobeView** | Strong composable data layers, picking, and update model | `GlobeView` is explicitly experimental; high-zoom accuracy is limited and `TileLayer` / `MVTLayer` support is experimental | MIT | Reject as the base Globe; possibly add selected overlays later only after measuring need |
| **Mapbox GL JS** | Mature globe, vector-tile, style, and interaction capabilities similar to MapLibre | Current code is licensed for use with Mapbox products under Mapbox terms, requires an active account, and introduces billing/account coupling the first playable does not need | Mapbox TOS for current versions | Reject |
| **NASA Web WorldWind** | Purpose-built WGS84 web globe, terrain, geographic shapes, navigation, and picking | The current package remains 0.11.0 and its official build uses Grunt/RequireJS-era packaging; the asset and dynamic styling path is less direct than MapLibre's vector-tile/style system | Apache-2.0 | Reject for maintenance cost |

Supporting primary sources for the alternatives:

- CesiumJS describes an Apache-licensed WGS84 globe with vectors, 3D Tiles, imagery, and time-dynamic data. Its `Primitive` API batches geometry, supports asynchronous worker construction and per-instance picking, and lets code change per-instance color after creation. [CesiumJS overview](https://cesium.com/platform/cesiumjs), [Primitive API](https://cesium.com/learn/cesiumjs/ref-doc/Primitive.html), [per-instance updates](https://cesium.com/learn/cesiumjs-learn/cesiumjs-geometry-appearances/#updating-per-instance-attributes), [license](https://github.com/CesiumGS/cesium/blob/main/LICENSE.md)
- Three.js provides GPU buffer geometry, partial buffer update ranges, orbit controls, and raycasting, but those are general graphics primitives rather than a geospatial data system. [BufferGeometry](https://threejs.org/docs/pages/BufferGeometry.html), [BufferAttribute updates](https://threejs.org/docs/pages/BufferAttribute.html), [OrbitControls](https://threejs.org/docs/pages/OrbitControls.html), [Raycaster](https://threejs.org/docs/pages/Raycaster.html), [license](https://github.com/mrdoob/three.js/blob/dev/LICENSE)
- deck.gl documents `GlobeView` as experimental, with no high-precision rendering above zoom 12 and experimental MVT/tile support. [deck.gl GlobeView](https://deck.gl/docs/api-reference/core/globe-view), [license](https://github.com/visgl/deck.gl/blob/master/LICENSE)
- Mapbox GL JS supports a globe but its current repository license restricts the SDK to relevant Mapbox products through an active Mapbox account. [Mapbox globe guide](https://docs.mapbox.com/mapbox-gl-js/guides/globe/), [Mapbox GL JS license](https://github.com/mapbox/mapbox-gl-js/blob/main/LICENSE.txt)
- Web WorldWind's repository and package describe its globe capabilities, Apache license, 0.11.0 package, and build toolchain. [Web WorldWind repository](https://github.com/NASAWorldWind/WebWorldWind), [package.json](https://github.com/NASAWorldWind/WebWorldWind/blob/develop/package.json)

## Performance prototype contract

No first-party source publishes a benchmark for this project's combination of region topology, ownership changes, border changes, overlays, and laptop targets. The recommendation is therefore complete only with the already-planned rendering prototype.

### Build one vertical rendering path

The prototype should use production-shaped code, even though it is throwaway:

1. MapLibre GL JS 6.x with an exact pinned version and documented Vite worker configuration.
2. Globe projection, no terrain, no remote basemap, and the intended political style.
3. One PMTiles archive with `regions`, `region_edges`, and `coastlines` layers.
4. Stable numeric feature IDs in every vector tile and `promoteId` only if the encoder cannot emit MVT IDs directly.
5. A generated adjacency manifest.
6. Region fill and edge visibility driven by `feature-state`.
7. Click/hover selection through `queryRenderedFeatures`.
8. One small dynamic GeoJSON conflict overlay and an optional point layer for capitals.
9. A synthetic simulation Web Worker that sends visual patches while the Globe is moving.

Do not prototype a second renderer in parallel. Measure MapLibre first, then invoke the fallback only if a named failure threshold is crossed.

### Data tiers

Use the real candidate geography from the separate geographic-data decision. Also create reproducible stress fixtures so success is not tied to one friendly dataset:

| Tier | Geographic Regions | Source vertices before tiling | Shared edge features | Purpose |
| --- | ---: | ---: | ---: | --- |
| A | 10,000 | 500,000 | 30,000 | expected comfortable case |
| B | 50,000 | 2,500,000 | 150,000 | plausible high-resolution first playable |
| C | 100,000 | 5,000,000 | 300,000 | headroom / failure characterization |

These are benchmark inputs, not product commitments. Preserve the same World-facing IDs and patch protocol at every tier. Produce zoom-dependent simplification rather than sending maximum detail at every zoom.

### Workloads to measure

Run at 1920×1080 with device-pixel ratios 1 and 2 on at least one recent integrated-GPU laptop. Record browser, OS, CPU, GPU, RAM, exact library version, and asset checksums. Test current stable Chrome and Firefox; add Safari on macOS before calling browser support complete.

Measure:

- cold load, time to first interactive Globe, bytes transferred, decoded tile count, and peak JS/GPU memory;
- continuous orbit and zoom for 60 seconds at world, continent, and country scales;
- the documented globe-to-Mercator transition around zoom 12;
- the antimeridian, high northern latitudes, and Antarctica;
- hover at pointer-move frequency and repeated click selection along tile and region boundaries;
- ownership bursts of 1, 100, and 1,000 changed regions, including affected edge updates;
- 100 visual state patches per second for 60 seconds while orbiting, with updates coalesced once per animation frame;
- 25 and 100 simultaneous Conflict overlays with bounded point counts;
- a synthetic simulation worker consuming 8 ms every 50 ms, to confirm that rendering remains responsive while client-side computation runs;
- tab resize, WebGL context loss/recovery, and cleanup/recreation of the Globe.

### Initial pass/fail gates

Treat these as prototype gates to validate or revise, not final user-facing service levels:

- **Interaction:** p95 frame time at or below 33.3 ms during camera motion and steady updates, with a 16.7 ms median target at DPR 1.
- **Jank:** no main-thread task longer than 100 ms during steady operation; ownership bursts must not create repeated tasks over 50 ms.
- **Patches:** a 100-region ownership burst visible within 100 ms p95; a 1,000-region burst visible within 250 ms or progressively applied without blocking input.
- **Picking:** selected `region_id` returned within 50 ms p95 and correct on both sides of every sampled shared border. Deduplicate tile-split results by stable ID.
- **Memory:** Tier B remains below 512 MiB total page memory after the 60-second interaction run and returns near its pre-navigation plateau after tiles are evicted.
- **Visual correctness:** no visible cracks between same-owner regions; ownership borders remain coincident with fills at tested zooms; no wrong-side selection at the antimeridian.
- **Delivery:** PMTiles performs partial range loads rather than downloading the full archive, and production-like hosting returns correct Range, ETag, and CORS headers.

Record raw traces, fixture generators, screenshots, and the exact pass/fail table with the prototype. A single frames-per-second average is insufficient because it hides main-thread stalls during state bursts.

## Fallback triggers

Move the rendering prototype to CesiumJS if any of these are confirmed requirements or measured failures:

1. The Globe must remain visibly spherical beyond MapLibre's zoom-12 transition.
2. Geographic Regions or interactions must be correct above Web Mercator's latitude limit.
3. Tier B cannot hold the 30 FPS hard floor or patch gates after vector tiling, shared-edge assets, simplified styles, and frame-coalesced feature-state updates.
4. Terrain, height-aware occlusion, or large amounts of streamed 3D content become first-playable requirements.
5. MapLibre globe defects force reliance on private APIs or a long-lived project fork.

If Cesium also fails specifically on state-update volume, investigate a custom Three.js indexed-region texture renderer as a narrow experiment. Do not choose that maintenance burden based on hypothetical scale.

## Material uncertainties

- The geographic-data decision has not yet fixed the number of Geographic Regions, vertex density, topology quality, or data license. Those values can change the benchmark result more than the library choice.
- MapLibre documents the mechanisms used here, but it does not publish feature-state throughput numbers for tens of thousands of globe polygons. The prototype is the evidence for that limit.
- Dynamic Country borders require precomputed shared edges and adjacency. MapLibre cannot compare the ownership state of two other features inside one style expression; the adapter must update affected edge state.
- PMTiles depends on correct byte-range hosting. Its single-file operational simplicity should be compared with ordinary static MVT files on the intended host if caching behavior materially affects repeat loads.
- Smoothness is bounded by source quality and zoom-specific simplification. A renderer cannot recover detail removed or misaligned by the data pipeline.
- Future capital and city points fit MapLibre's circle/symbol layers. Detailed cities, terrain, and close-range 3D scenes would reopen the renderer decision because they were explicitly outside this destination.

## Final recommendation

Proceed to the globe performance prototype with **MapLibre GL JS + MVT-in-PMTiles**, backed by a **TopoJSON-based asset compiler and a separate adjacency manifest**. Keep all MapLibre types behind `GlobeRenderer`, update ownership through stable feature IDs and shared-edge feature state, and make the documented benchmark gates the acceptance test. This is the lowest-maintenance route that already owns the difficult geospatial rendering concerns while leaving the simulation and World serialization independent of the chosen renderer.
