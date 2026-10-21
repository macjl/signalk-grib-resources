# Signal K Forecast Dataset Resources — Project Outline

## Status

**Draft architecture revision for community review — 2026-10-21.** This is a design proposal, not an implemented plugin, an adopted Signal K specification, or a record of community consensus. The lack of further replies does not establish agreement.

This revision incorporates the [plugin fleet evaluations](consumer-fleet-evaluation.md), [Weather Router Plus feedback in discussion #3](https://github.com/macjl/signalk-grib-resources/discussions/3), and the [Discord architecture discussion](https://discord.com/channels/1170433917761892493/1557058626391384185). The evaluations remain attributed working notes about the revisions they assessed; they have not been rewritten as requirements or reassessed against this revision.

The proposed direction is one format-agnostic catalogue/storage plugin that also routes requests to independently installed, format-specific processing plugins. Producers publish persistent datasets; consumers can retrieve original content or request temporary subsets. GRIB is the first implementation profile. Zarr and harmonic data inform the boundaries without requiring their implementation in the first prototype.

`forecastDatasets` is the proposed public resource-type name used throughout this revision. The name and scope must be reviewed before the first prototype establishes a public contract. Repository and plugin package names are separate decisions. Schemas, endpoint names, and registration details below remain illustrative. The assumption that the prototype needs no server-core change also needs validation.

## 1. Problem Statement

Downloaders, routing applications, Weather Providers, and navigation clients need to share datasets. Today they often depend on private filesystem layouts, plugin-specific endpoints, or repeated upstream downloads.

Two needs drive this proposal:

- Producers should retain data as it arrives, using immutable content and atomic publication. A later indexing or transformation failure should not require another download.
- Consumers should be able to retrieve only the fields and forecast times they need, in a format they support. Simpler consumers should not have to assemble the producer's original files themselves.

Advanced consumers may instead prefer direct access to originals and detailed indexes, using HTTP Range to retrieve selected messages and performing their own decoding and interpolation.

GRIB provides an established exchange format, but does not define dataset discovery, publication, or retention within Signal K. Other relevant sources use Zarr or harmonic constants. The shared layer should make datasets discoverable while allowing independently developed plugins to provide the knowledge specific to each format.

## 2. Main Design Principle

Use the existing Signal K **Resources API** for discovery and metadata. A **catalogue/storage Resource Provider plugin** owns the persistent catalogue, immutable content access, and publication lifecycle. It also provides a common entry point for requesting operations from registered processors.

Format knowledge lives in **separate processing plugins**. GRIB and future Zarr processors have their own dependencies, configuration, and release cycles. Installing the catalogue does not require installing every processor.

The catalogue provides shared mechanics: identity, versions, completion state, notifications, access control, original-content serving, and temporary-result lifecycle. Processors supply verified format metadata and perform supported selections or transformations.

The catalogue is an explicit dependency for plugins using this contract. Its simplicity comes from a bounded responsibility, not from having no orchestration work. It routes one operation to one compatible processor; automatic pipelines and fallback chains are outside the initial scope.

The working assumption remains that a prototype can use existing Signal K interfaces without a server-core change. A processor-registration or dataset-job API is **proposed here**, not claimed to be an existing Signal K facility.

## 3. Why the Weather API Is Not Enough

The Weather API provides normalised weather-value queries, such as a forecast series for a position. A GRIB-backed Weather Provider can continue bridging dataset access to that interface.

A route calculation may require many spatial and temporal samples. Reading the relevant data once and interpolating locally avoids repeatedly requesting individual points. Dataset consumers also need model/run identity, available fields and levels, grid and encoding compatibility, coverage, provenance, and content access.

- **Resources API:** discover and inspect persistent datasets.
- **Catalogue content and processing endpoints:** retrieve originals, indexes, or requested temporary subsets.
- **Weather API:** query normalised values through a Weather Provider.

Reading a format remains a consumer capability: publishing a valid GRIB does not make it readable by every decoder. Optional processing can help supply compatible content, but conversion must be explicit and supported. Weather API schema reconciliation and extensions are a separate workstream; see §9.

## 4. User Stories / Use Cases

### 4.1 GRIB consumer

**As a routing, weather, visualisation, or analysis application, I want to select suitable datasets without depending on the downloader's private API or directory layout.**

An advanced consumer can inspect verified fields, levels, grids, steps, and encoding templates; retrieve originals or use an asset-bound message index and Range; then decode and interpolate locally. It can combine regional and global datasets according to its own policy. There is no globally active dataset for all consumers.

A simpler consumer can request selected parameters, levels, and forecast times from the catalogue and retrieve one GRIB result if an installed processor supports that operation. It does not need to select and combine source assets itself. Spatial subsetting is a separate capability and is not implied by parameter/time selection.

These access paths coexist. Original access must not trigger automatic conversion or re-packing.

### 4.2 GRIB producer / downloader

**As a downloader, file importer, or satellite/email receiver, I want to publish acquired content without carrying a decoder or depending on successful transformation.**

The producer supplies original bytes, acquisition context, and an explicit boundary for the content it has finished acquiring. It can publish one received file or a logical run containing several files. Download scheduling and upstream-specific requests remain its responsibility.

A minimal producer may know little beyond format, size, source, and acquisition time. Requested fields/domain must be distinguished from verified content. An indexer can enrich the catalogue later; missing metadata is not evidence that a field is absent.

### 4.3 GRIB aggregator / transformer

**As a processing plugin, I want to provide format-specific operations without implementing another catalogue or depending on other format processors.**

The plugin registers supported inputs, operations, and outputs. It can index content, select messages, create a spatial subset, or perform an explicitly requested conversion. A bounded job produces metadata or temporary content.

A plugin can also act as a producer and explicitly publish a derived dataset for persistent reuse, with source versions and transformation provenance. A temporary subset does not automatically become a persistent dataset.

### 4.4 Lifecycle and retention management

**As an administrator, I want storage usage controlled without datasets disappearing halfway through a read or processing job.**

The catalogue coordinates retention with active content access. Producers and policy tools request deletion or archival through that ownership contract, rather than removing catalogue-owned files independently. Policies may keep the latest N runs or preserve selected datasets.

Temporary jobs and results have a separate, bounded lifetime; they do not consume persistent catalogue entries by default.

### 4.5 Historical GRIB access

**As an analysis application, I want explicitly retained versions to compare runs or forecasts with observations.**

Persistent identity and provenance survive retrieval and mirroring. Historical retention is a policy, not an obligation to preserve every downloaded run indefinitely.

### 4.6 Low-bandwidth vessel access

**As a vessel with a limited connection, I want to retrieve only the required area, parameters, and forecast period from a shore server.**

The client discovers actual content and available operations. It can use index + Range or a processor-generated result. Size estimates, where supported, help the user decide before transfer. Unknown size must remain distinguishable from a reliable estimate.

### 4.7 Server-to-server federation

**As a Signal K server, I want to retrieve datasets from another server and optionally republish them locally with provenance.**

A local acquisition plugin acts as a consumer of the remote catalogue and a producer for its local catalogue. Federation, remote authentication, and automated replication follow the first local workflows.

### 4.8 External navigation application

**As a mobile, desktop, or web application, I want to discover and retrieve datasets through HTTP without direct access to the server filesystem.**

Original-content and processing interfaces should work for external applications and server plugins. URLs must resolve relative to the server used by the client, including proxied installations. Authentication and browser-origin behaviour need an explicit contract.

## 5. Proposed Architecture

```text
Producers / downloaders ── publish ──> Catalogue + immutable storage
                                            │
Consumers <── discovery / originals / index ─┤
Consumers ── processing request ────────────>│
                                            │ select registered processor
                                            ▼
                                  Independent processing plugins
                                  ├─ GRIB: index / subset / convert
                                  └─ Zarr / others: future profiles
                                            │
                                    short-lived worker
                                            │
Consumers <── temporary result ───── Catalogue

Persistent derived dataset: explicit publication through the producer path.
```

### Signal K server core

The server hosts plugins, provides the Resources API and HTTP routing, and transports resource notifications. It does not need to parse GRIB or Zarr or schedule dataset transformations.

### Catalogue/storage plugin

The catalogue owns consistent listing/filtering, identity/version handling, publication, content retrieval, notifications, access control, and deletion coordination. It maintains a registry of available processors and routes supported requests to them.

Binary ingestion, processor registration, and job handling use plugin-specific endpoints or interfaces defined by the contract. Generic Resource API JSON CRUD is not a binary-upload or job protocol.

### Independent processing plugins

Each processor registers a versioned capability description: accepted format/profile, operation, supported selectors/options, and output format/profile. Indexing is also an operation. A processor can reject unsupported grids or encodings without compromising originals.

The catalogue selects an available compatible processor. It reports unavailable or unsupported operations explicitly and updates advertised capabilities when a processor stops. The first implementation can explicitly configure one processor per operation/profile.

A preferred initial transport is a local HTTP contract between ordinary Signal K plugins, with registration during startup and withdrawal during shutdown. Restart/re-registration, availability, and authentication details must be specified. The catalogue does not dynamically load processor packages or add arbitrary properties to Signal K's shared `app` object.

Separately installed Signal K plugins may still run in the same Node.js process. Heavy parsing and transformation should execute in short-lived child processes controlled by the processing plugin. Independent installation alone does not isolate memory or crashes.

### Producers and consumers

Producers publish durable content and metadata. Consumers select originals or requested results. One plugin may fill several roles, but storage ownership and publication semantics remain the same.

## 6. Resource Metadata vs Binary Content

### Logical dataset and original assets

A dataset is a logical, versioned publication unit. It may contain one original file or several assets making up the producer's declared acquisition scope. Storage does not require combining, decoding, or re-encoding them.

The proposed resource paths are:

```text
GET /signalk/v2/api/resources/forecastDatasets
GET /signalk/v2/api/resources/forecastDatasets/{id}
```

Metadata stays compact. Large inventories and message indexes are linked documents, rather than embedded in every catalogue response. The exact summary/detail response contract remains to be specified.

An illustrative completed dataset might look as follows. Names, values, and URLs are examples; asset hashes and other detailed fields are omitted for brevity.

```json
{
  "name": "Regional forecast, declared 48-hour scope",
  "kind": "forecast",
  "format": "grib2",
  "version": "v1",
  "state": "complete",
  "model": "AROME",
  "run": "2026-10-21T06:00:00Z",
  "inventory": {
    "status": "verified",
    "href": "/plugins/forecast-catalogue/datasets/example/versions/v1/index"
  },
  "assets": [
    {
      "id": "part-1",
      "href": "/plugins/forecast-catalogue/datasets/example/versions/v1/assets/part-1",
      "byteRanges": true
    },
    {
      "id": "part-2",
      "href": "/plugins/forecast-catalogue/datasets/example/versions/v1/assets/part-2",
      "byteRanges": true
    }
  ]
}
```

Advanced clients can use the manifest directly. A consumer requesting an assembled subset uses the processing interface and receives a result link; the processor handles the source assets. Without a processor, original-content access remains available.

### Metadata requirements

Minimal publication should cover opaque resource identity, immutable content version, format/kind, publication state, assets with byte lengths and content hashes, and producer/acquisition context. It must permit sparse descriptive metadata from offline or file-drop producers.

Where available, verified inventories describe:

- model/run reference time, distinct from acquisition time;
- actual forecast steps/valid times, including irregular spacing and accumulation intervals;
- parameter identifiers, level value, and type of level;
- grid representation, resolution, exact extent, and longitude convention;
- GRIB edition and grid, product, and packing templates, per message where necessary;
- missing-value and instant/accumulated semantics.

Requested metadata and verified inventory remain distinguishable. Broad names such as `wind` alone are insufficient for precise selection. A summary must not imply that every field covers every advertised time and location; variable coverage belongs in the detailed inventory.

### Completion and atomic publication

Content is staged during acquisition. The producer declares the assets and scope it has finished acquiring; the catalogue publishes a complete version only once the declared content is durably available. A 48-hour subset can be complete without being the entire upstream run.

Acquisition completion and inventory verification are separate states. Indexing failure must not discard originals, force a re-download, or make requested fields appear verified. Consumers requiring verified coverage can exclude unindexed content. Partial acquisition may be shown explicitly, but must never be silently selected as complete.

Metadata enrichment does not change original bytes. Content changes produce a new immutable version. The metadata/index revision must also be identifiable so clients can notice enrichment without unnecessarily re-downloading content.

## 7. Subsetting and Processing Access

### Original bytes plus index and Range

The catalogue serves original assets with byte-range access for the initial local-file profile. Each range-capable asset has a stable URL, byte length, and validator for the exact served representation. Index offsets bind to that immutable asset, not merely a model/run label.

A GRIB indexer supplies per-message asset identity, offset, length, parameter, level, step, and encoding details. A compatible client can retrieve selected messages without a server-side subset job. Range alone does not interpret fields or crop grids.

The implementation must preserve the indexed representation, including avoiding transparent content encoding that changes byte offsets. Range semantics, validators, and content-URL authentication need interoperability testing.

### Catalogue-routed processing

A consumer requests an advertised operation on a specific dataset version, supplying selectors and the required output profile. The catalogue checks access, request validity, and processor availability, then delegates. The processor interprets format-specific selectors and verifies that the source satisfies them.

The contract distinguishes message selection from spatial cropping, regridding, and re-packing. Selecting fields/times need not imply conversion; output encoding/template changes must be explicit. Unsupported requests must fail rather than silently return incompatible or broader content.

Capabilities apply to compatible datasets/profiles and available processors, not every dataset indiscriminately. Upstream download-service subsetting does not establish local processing capability.

### Temporary job and result lifecycle

1. A request identifies immutable source versions. The catalogue prevents deletion while they are in active use by the job.
2. The processor executes a bounded job in a short-lived worker and a job-specific workspace. Long jobs expose status rather than indefinitely holding an HTTP request open.
3. On success, the complete result is handed over for delivery with its format, size, identity, and provenance. Worker exit must not remove a file still needed for delivery.
4. The client retrieves the result through the catalogue. The default result is temporary and does not appear in the persistent dataset listing.
5. Cleanup follows delivery completion or expiry, covering failure, cancellation, and restart recovery. Active transfers must not be deleted underneath the client. Worker exit reclaims its process memory; this is not a promise to flush the operating system's file cache.

Handover, job status, expiry/retry rules, and authorization need a small explicit contract. A job marked successful must have a complete result; acceptance of an asynchronous request does not mean completion. Processing failure preserves sources. Retrying processing must not trigger another upstream download.

### Persistent derived publication

A producer can explicitly publish a derived result through the persistent-publication path. It records source versions, processor/version, and transformation options. Persistence is separate from temporary subset delivery.

Reusable subset caches, automatic processing pipelines, and a shared decoded cache are deferred. A future decoded cache needs its own semantics and storage budget: interpolation, accumulation handling, coastal gap filling, and model-selection policies differ by consumer.

## 8. Working Assumption: No Initial Core Change

The Resource Provider interface accepts custom resource types, and plugins can expose HTTP endpoints. These facilities are the proposed basis for discovery, binary serving, and processor communication.

Registration, binary ingestion, temporary results, and dispatch are implemented by the catalogue and companion plugins. The Signal K core does not acquire a dataset scheduler or format registry through this proposal.

Before coding, validate the approach against a recorded target server version: listing/filtering, IDs, notifications, permissions, and plugin restarts. Generic resource CRUD must not be assumed to provide atomic multi-asset publication, binary upload, or processor lifecycle handling.

One catalogue owns the custom type's semantics initially. It does not rely on the server's aggregation of independently implemented catalogues to reconcile identities, filters, or errors. Multiple producers and processors connect to that catalogue.

## 9. Relationship to Existing Components

- **GRIB Downloader:** publishes declared complete runs without changing original encoding or requiring a decoder. Download/retention ownership needs an explicit handover during migration.
- **Offshore blogging / file-drop producers:** publish received bytes and sparse acquisition metadata. Indexing enriches them without adding a decoder to every receiver.
- **GRIB Weather Provider:** discovers and ingests content into its cache, continuing to serve normalised point forecasts. It need not request transformations it does not need.
- **Weather Router Plus:** uses completion/version metadata, originals, detailed inventories, and Range, preserving its decoding and source-selection policies.
- **Visualisation and external navigation clients:** use originals or request subsets through the same catalogue entry point.
- **Remote servers:** can later acquire and republish datasets with provenance.

Weather API work should be discussed with server maintainers separately. Discussion #3 reports divergent TypeScript and OpenAPI shapes at a specific server revision; this outline does not claim the report describes every subsequent version. Topics include canonical response fields, per-record provenance, parameter capabilities, tide/surge separation, level datum, and high/low-water events.

That feedback also corrects the fleet evaluation: swell fields exist in the API definitions it examined. Missing swell output from a provider does not mean the API cannot represent swell. The historical, attributed evaluation remains unchanged.

## 10. Proposed Plugin Responsibilities

| Component | Owns | Does not need to own |
| --- | --- | --- |
| Catalogue/storage Resource Provider | Catalogue/filter semantics, immutable content, publication, notifications, access control, processor registry/dispatch, temporary delivery and cleanup | GRIB/Zarr parsing libraries or meteorological transformations |
| Format-specific processor | Verified inventory/indexing, supported input/output profiles and operations, isolated workers and workspaces | Another catalogue or another format's dependencies |
| Producer | Acquisition, scheduling, acquisition metadata, declared completion, explicit persistent publication | Successful transformation as a prerequisite to retaining downloaded bytes |
| Consumer | Dataset suitability, original or processed access, application-specific decoding/interpolation and source policies | Producer directory conventions or contacting each processor directly |
| Optional lifecycle/policy tool | Retention/archival configuration through the catalogue | Independently deleting content owned or used by the catalogue |

These are roles, not a requirement for a package per row. Processors remain independently installable; original access continues without them.

## 11. Resource Discovery

The catalogue filters and orders resources consistently across producers. Criteria include format, model/run, state, verified parameters/levels, space/time coverage, freshness, and available operations.

Discovery selects an existing dataset; subsetting selects content from a chosen version. Filtering a catalogue by bounding box must not itself generate or download a dataset.

Before implementation, specify overlap versus complete coverage, unknown metadata, longitude/antimeridian rules, time intervals, field identifiers, freshness ordering, and pagination/summary behaviour. Strict required-field/coverage queries must not treat unknown inventory as a verified match.

Clients list the catalogue, inspect suitable entries and capabilities, then choose original access or an operation. No globally preferred model is imposed.

## 12. Provenance

Persistent and temporary results identify source versions and processor/options. Acquisition provenance distinguishes upstream source, arrival channel, request text, and acquisition time from model reference time and forecast validity.

Mirroring preserves upstream content identity while recording local publication context. Requester identity or credentials must not become public metadata by default; acquisition history may contain sensitive information.

Detailed provenance and cross-server links remain to be specified without embedding large histories in summaries.

## 13. Identity and Deduplication

Separate local resource ID, immutable content version, per-asset byte hashes, and metadata/index revision. Model/run/domain labels alone cannot establish identity: different parameter sets or horizons can refer to the same run.

A multi-asset version needs a deterministic manifest identity alongside individual asset hashes. Canonicalisation and identity rules must be specified before interoperability claims. Download settings or fetch fingerprints do not establish byte identity.

Indexes and jobs bind to exact source versions. Identical content may share physical storage without losing distinct acquisition provenance.

## 14. Security and Resource Limits

Metadata, originals, indexes, jobs, and results require consistent authorization. A processor receives only the source/result access needed for its job. Registration is restricted to configured or authenticated processors; it must not become an arbitrary executable or callback-URL mechanism.

Bound concurrency, deadlines, and input/output/workspace size. Cancellation and failure release temporary resources and source-use protection. Cleanup after restart is part of the temporary-lifecycle contract.

Publication defines safe content handover and ownership. IDs must not permit arbitrary filesystem access. Retention keeps catalogue entries, indexes, and bytes consistent while respecting active reads/jobs.

External clients need resolvable URLs, compatible metadata/binary authentication, and browser-origin rules where applicable. Size estimates for metered transfer are a capability, not automatic permission to transfer or a guaranteed processing cost.

## 15. Suggested Implementation Phases

### Phase 0 — Review the prototype contract

Review the public name/scope, role boundaries, minimal metadata, asset completion/version semantics, and original-access profile before implementation. Specify registration, requests/results, authentication, ownership, and restart behaviour. Record a target server version and validate the no-core-change assumption.

The first format profile is GRIB2. A generic type name does not require implementing Zarr, harmonic prediction, or cross-format conversion in the prototype.

### Phase 1 — Catalogue and original-content access

Implement local persistent publication, discovery/detail metadata, asset hashes/validators and Range, and publication/deletion notifications. Preserve sparse metadata and explicit completion. Integrate one producer and an independently installed GRIB indexer to demonstrate verified inventory and message-index enrichment.

Acceptance evidence includes version stability, correct index-bound Range reads, no partial run presented as complete, original access while indexing is unavailable, and retention coordination. Acquisition/indexing failures must not discard finished downloads. Demonstrate a multi-file run and a one-file received subset through an external HTTP client without filesystem access.

### Phase 2 — Delegated temporary subsetting

Implement one GRIB processor operation for parameter/level/time selection through the catalogue. Return one consumable GRIB in the advertised encoding/profile, using a short-lived worker and explicit result handover.

Exercise the external client's request/result workflow, unavailable processors, unsupported inputs, failure/cancellation, source retention, delivery before cleanup, expiry, and restart cleanup. Temporary jobs must not accumulate persistent entries. This completes the first demonstration of both consumer paths.

Spatial subsetting and re-packing are not required and must not occur implicitly.

### Phase 3 — Existing consumer integrations

Adapt the GRIB Weather Provider and exercise routing access with a compatible consumer. Preserve the filesystem bridge until resource-based workflows are demonstrated. Verify that producers need no consumer-specific integration and consumers retain their interpolation/model policies.

### Later phases

- Explicit persistent derived publication with provenance.
- Additional GRIB operations: spatial subsets, conversion, or blending.
- Independently installed Zarr/harmonic profiles justified by concrete consumers.
- Server-to-server acquisition, mirroring, and metered transfer.
- Advanced retention/archival tools and optional reusable-result caches.

A shared decoded cache is a separate investigation, not a prerequisite. Weather API changes follow their own review with maintainers.

## 16. Open Questions

### Resolve before the prototype

1. Is `forecastDatasets` the right public name/scope, including prediction inputs such as harmonic constants? Repository/package names need not determine the resource type.
2. Does one catalogue/storage provider with independent processors meet producer/consumer needs and fit the target Signal K interfaces?
3. What are the minimal publication, manifest, completion, content-version, and metadata-revision contracts? How is ownership transferred without coupling producers to storage paths?
4. Which GRIB2 metadata/index fields and unambiguous parameter/level/time selectors form the initial profile?
5. How do processors register, authenticate, report availability, and recover after restarts? How is one selected when several match?
6. What is the minimal job/result protocol, including cancellation, handover, delivery lifetime, and cleanup? Which operations can stream instead of materialising a file?
7. What exact URL, MIME, validator, Range, and authentication rules apply to originals and results?
8. Which discovery filters and unknown-metadata/coverage semantics are required?

### Resolve as later capabilities are introduced

9. How should persistent derived publication, mirroring, and cross-server identity/provenance work?
10. Which advanced retention policies and reusable-result caches help constrained installations?
11. How should spatial processing and reliable transfer-size estimates be advertised and limited?
12. What common metadata and distinct access profiles do real Zarr/harmonic consumers need?
13. Would lessons from datasets and charts justify a generic Signal K binary-content enhancement?

## 17. Current Working Conclusion

The revised direction is a shared catalogue/storage Resource Provider with a bounded dispatch role, plus independent format-specific processors. Producers retain immutable originals; indexing enriches metadata without changing those bytes. Consumers choose original/index/Range access or catalogue-routed processing.

Subsets are temporary by default. Persistent derived datasets use an explicit publication path. Processing workers and temporary files have a lifecycle separate from persistent sources and active downloads.

Review this outline and its prototype-blocking questions, then demonstrate both access paths with GRIB. This is a proposed direction, not a final protocol or community-approved architecture.

## 18. Relevant Signal K References

- [Resources API implementation](https://github.com/SignalK/signalk-server/blob/master/src/api/resources/index.ts).
- [Resource Provider interface](https://github.com/SignalK/signalk-server/blob/master/packages/server-api/src/resourcesapi.ts).
- [Resource Provider plugin documentation](https://github.com/SignalK/signalk-server/blob/master/docs/develop/plugins/resource_provider_plugins.md).
- [Plugin HTTP endpoints and lifecycle](https://github.com/SignalK/signalk-server/blob/master/docs/develop/plugins/README.md).
- [Resources REST API documentation](https://github.com/SignalK/signalk-server/blob/master/docs/develop/rest-api/resources_api.md).
- [Chart resource schemas](https://github.com/SignalK/signalk-server/blob/master/packages/server-api/src/typebox/resources-schemas.ts).

The Resource Provider interface was inspected at server revision [`3e12725`](https://github.com/SignalK/signalk-server/blob/3e12725925b2bc47c466329231a4bb73a7a19de9/packages/server-api/src/resourcesapi.ts) for this revision. This does not replace runtime testing against the prototype's target version.

### Community input

- [Weather Router Plus discussion #3](https://github.com/macjl/signalk-grib-resources/discussions/3).
- [Shared GRIB resources — Discord thread](https://discord.com/channels/1170433917761892493/1557058626391384185).
- [Attributed plugin fleet evaluations](consumer-fleet-evaluation.md).

### GRIB standard and reader references

- [GRIB2: The WMO Standard for the Transmission of Gridded Data](https://www.weather.gov/media/mdl/GlahnLawrence2002GRIB2TheWMOStandard.pdf).
- [ECMWF GRIB2 format reference](https://codes.ecmwf.int/grib/format/grib2/).
- [ECMWF ecCodes geography keys](https://codes.ecmwf.int/grib/format/edition-independent/1/).
