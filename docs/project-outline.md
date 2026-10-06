# Signal K GRIB Resources — Project Outline

## Status

Draft proposal / project foundation document.

This is an initial proposal for community discussion, not an adopted Signal K specification. Resource names, schemas, URLs, and query parameters shown below are illustrative and remain open for review. The assumption that an initial implementation needs no server-core change also remains to be validated.

This document captures the current design milestone for introducing a standard way to discover, share, retrieve, subset, archive, and consume GRIB datasets in the Signal K ecosystem.

The goal at this stage is not to define a final protocol or implementation in full detail. It is to establish the use cases, architecture, boundaries, and implementation direction strongly enough to support discussion with the Signal K community and guide the first implementation.

---

## 1. Problem Statement

Several Signal K components already need access to GRIB weather data, but today they generally rely on private conventions, local file paths, plugin-specific APIs, or direct access to an external provider.

Examples include:

- a GRIB downloader fetching forecast datasets;
- a weather provider reading local GRIB files and exposing values through the Signal K Weather API;
- a routing plugin needing efficient access to many forecast points over time and space;
- a weather visualisation client displaying fields such as wind, pressure, precipitation, waves, or currents;
- a server ashore holding large forecast datasets while a vessel has only a low-bandwidth connection.

The missing layer is a standard way to treat GRIB datasets as **discoverable Signal K resources**.

GRIB already provides a World Meteorological Organization (WMO) standard format for exchanging gridded forecast data. This proposal builds on that format: the missing layer is a common way for Signal K components to discover available datasets, inspect their suitability, and retrieve their content.

The GRIB standard describes data and their encoding. It does not define how a Signal K downloader publishes its datasets to other plugins, how consumers discover them, or how one Signal K server retrieves datasets from another.

Such a layer should allow producers and consumers to remain decoupled. A downloader should not need to know which routing or weather plugin will consume its files, and a consumer should not need to know how or where the dataset was originally obtained.

---

## 2. Main Design Principle

The proposal is to build on the existing Signal K **Resources API** and **Resource Provider** model instead of creating a new GRIB-specific core API.

The Signal K server core remains responsible for generic resource discovery and CRUD operations.

GRIB-specific behaviour is implemented by plugins.

At this milestone, the working assumption is:

> **No Signal K server core change is required for the first implementation.**

The existing Resources API is used to expose GRIB metadata and discovery information, while GRIB binary content is served by provider-specific HTTP endpoints referenced from the resource metadata.

This follows the same general pattern already used by chart resources: the resource describes the data source, while the actual binary or tiled content is served outside the generic `getResource()` JSON response.

---

## 3. Why the Weather API Is Not Enough

The Signal K Weather API and GRIB resources solve different problems and should coexist.

### Weather API

The Weather API is well suited to queries such as:

> Give me the weather values at this position and this time.

It is ideal for providers such as Open-Meteo or OpenWeather and for clients that need a relatively small number of forecast values.

### GRIB resource access

A GRIB dataset is a multidimensional dataset covering space, time, vertical levels, and multiple meteorological parameters.

A routing engine may need thousands of interpolations over a route and forecast period. Downloading or opening a GRIB once and interpolating locally is much more efficient than issuing thousands of point queries through a weather provider.

A general-purpose GRIB reader can decode datasets from multiple forecast models without a separate integration for each model. Consumers still need to support the GRIB editions, encodings, and grid representations they encounter, and check that a dataset contains the fields, levels, geographic coverage, and forecast times required for their use case. These are dataset and reader requirements, rather than a requirement to integrate separately with every downloader.

A GRIB resource also has concepts that do not naturally belong in the Weather API:

- model and model run;
- geographic coverage;
- temporal coverage and forecast steps;
- grid resolution;
- available parameters;
- source and provenance;
- dataset freshness;
- original vs derived datasets;
- binary content retrieval;
- subsetting;
- retention and archival status.

The two APIs are therefore complementary:

- **Resources API:** dataset discovery and dataset metadata;
- **Weather API:** normalised weather-value queries.

A GRIB-backed Weather Provider becomes a bridge between the two layers.

---

## 4. User Stories / Use Cases

### 4.1 GRIB consumer

**As a routing, visualisation, weather, or analysis plugin or application, I want to discover available GRIB datasets matching my needs so that I can choose the most appropriate dataset without knowing which plugin downloaded or produced it.**

Possible selection criteria include:

- model;
- model run;
- geographic coverage;
- forecast period;
- resolution;
- available variables;
- freshness;
- producer;
- original or derived status.

Each consumer should remain free to choose the dataset that best fits its own use case. There is no need for one globally active GRIB for the entire Signal K server.

#### Weather routing example

A routing application evaluates many candidate routes across a geographic area and forecast period. It needs efficient access to forecast fields at many positions and times, and may use wind, waves, and currents according to the capabilities of its routing engine.

The application should be able to:

- discover datasets published by independent downloaders or other producers;
- inspect their parameters, vertical levels, geographic and temporal coverage, and resolution;
- select suitable datasets, such as a regional high-resolution forecast for a coastal passage or a global forecast for a longer passage;
- retrieve the GRIB content once, or request a suitable subset where supported;
- decode the data and interpolate locally during route computation.

For example, a regional AROME dataset could be selected where its coverage and available fields meet the routing application's needs. Wind, wave, and current fields may also come from separate datasets, if the consumer supports combining them.

A shared resource contract would allow a downloader to publish a dataset without a routing-specific integration, and a routing application to discover and retrieve it without depending on the downloader's private API or directory layout. The routing application remains responsible for checking dataset suitability and performing its own interpolation and route calculations.

This access layer does not guarantee that every consumer can use every valid GRIB file. It provides a common way to find and obtain standard datasets while leaving data interpretation and application requirements with the consumer.

### 4.2 GRIB producer / downloader

**As a GRIB downloader, I want to publish downloaded datasets as Signal K resources so that any compatible consumer can discover and use them.**

The downloader should not need direct knowledge of the consumers.

Its responsibilities are primarily:

- download the dataset;
- identify its metadata;
- make the binary data available;
- publish or update the corresponding resource.

### 4.3 GRIB aggregator / transformer

**As a GRIB processing plugin, I want to consume one or more existing GRIB resources and publish a new derived GRIB resource.**

Examples include:

- combining data from multiple models;
- combining atmospheric and ocean datasets;
- creating a reduced or optimised GRIB;
- building a linear or route-oriented GRIB;
- producing a blended forecast;
- converting or normalising GRIB encodings.

Derived datasets should preserve provenance information so that consumers can understand where the data came from.

### 4.4 Lifecycle and retention management

**As a server administrator or lifecycle-management plugin, I want to define retention, archival, compression, and deletion policies for GRIB datasets so that storage usage remains controlled while useful historical data can be preserved.**

Different installations may require different policies.

Examples:

- keep only the latest run of a model;
- keep the last N runs locally;
- archive older runs to slower or compressed storage;
- preserve selected datasets permanently;
- delete derived datasets earlier than source datasets;
- retain historical runs for forecast verification or research.

Lifecycle management should be independent of the downloader and consumers.

### 4.5 Historical GRIB access

**As an analysis or verification plugin, I want to access previous forecast runs so that I can compare forecasts with observations or compare successive model runs.**

This is one reason lifecycle management should support archival rather than only deletion.

### 4.6 Low-bandwidth vessel access

**As a Signal K server on a vessel with limited bandwidth, I want to discover a GRIB dataset held on another Signal K server and retrieve only the subset I need.**

For example, an ashore Signal K server may download the complete AROME or ARPEGE dataset, while the vessel requests only:

- the Mediterranean area around the planned route;
- the next 48 hours;
- wind, pressure, waves, and precipitation.

This can dramatically reduce transferred data.

### 4.7 Server-to-server federation

**As a Signal K server, I want to consume GRIB resources from another Signal K server using the same resource model that I expose locally.**

This enables recursive or federated workflows:

```text
Upstream weather source
        ↓
Signal K server ashore
        ↓
Signal K server on vessel
        ↓
Routing / Weather Provider / Visualisation
```

A server can therefore act as both a GRIB consumer and a GRIB producer.

### 4.8 External navigation application

**As a mobile or desktop navigation application already consuming vessel data from a Signal K server, I want to discover and retrieve forecast datasets from that server so that I can display weather forecasts alongside vessel observations and navigation information.**

The application may already use Signal K for position, measured wind, depth, or other vessel sensor data. The same server could also provide a catalogue of available GRIB datasets and access to their binary content.

A typical workflow would be to:

- connect to the vessel's Signal K server and receive vessel observations through the existing data APIs;
- discover forecast datasets through the Resources API;
- inspect model runs, available parameters, geographic and temporal coverage, and resolution;
- retrieve suitable GRIB content, optionally requesting a subset where supported;
- decode and display forecast fields alongside navigation information;
- optionally cache downloaded datasets for offline use, retaining their model run and forecast validity information.

The application does not need to be installed as a Signal K plugin or integrate separately with each downloader. Producers on the server can obtain and publish datasets that multiple independent applications retrieve and use. Each application remains responsible for GRIB decoding, dataset suitability, display, and any local interpolation.

This use case positions Signal K as a common access point for both vessel observations and forecast datasets in the mariner's digital hub. It does not require the two kinds of data to share the same API: observations and GRIB resource discovery retain their respective interfaces.

The resource contract should therefore be usable by external HTTP clients, without requiring access to the server filesystem or an internal plugin interface. Points to define include client authentication for both metadata and binary retrieval, resolution of content URLs from the client's server address, dataset identity and cache validation, and browser access from another origin where applicable. The exact mechanisms remain open for design.

---

## 5. Proposed Architecture

The architecture separates generic Signal K resource handling from GRIB-specific logic.

```text
                         ┌────────────────────────┐
                         │   Signal K Resources   │
                         │          API           │
                         └────────────┬───────────┘
                                      │
                    resource metadata │
                                      │
                         ┌────────────▼───────────┐
                         │  GRIB Resource Provider│
                         └────────────┬───────────┘
                                      │
                   binary/subset HTTP │
                                      │
       ┌──────────────────────────────┼──────────────────────────────┐
       │                              │                              │
┌──────▼───────┐              ┌───────▼────────┐             ┌──────▼───────┐
│ GRIB         │              │ GRIB lifecycle │             │ GRIB         │
│ downloader   │              │ / archive mgr  │             │ transformer  │
└──────────────┘              └────────────────┘             └──────────────┘
       │                                                             │
       └──────────────────── resources ───────────────────────────────┘
                                      │
                         ┌────────────▼───────────┐
                         │      Consumers         │
                         │ routing / weather / UI │
                         └────────────────────────┘
```

### Signal K server core

The core continues to provide:

- resource registration;
- generic resource listing;
- generic resource retrieval;
- provider selection;
- standard HTTP routing;
- resource change notifications.

The core does **not** need to understand GRIB encoding, meteorological models, spatial subsetting, or retention policies.

### GRIB Resource Provider

The GRIB Resource Provider is responsible for the GRIB-specific resource contract.

It should provide:

- discovery metadata through the Resources API;
- access to the original GRIB content;
- optional subsetting capabilities;
- an HTTP endpoint capable of returning binary GRIB content;
- content type and download metadata;
- capability information describing supported operations.

### Downloader plugin

The downloader obtains GRIBs from upstream weather services and registers them as resources.

### Lifecycle plugin

The lifecycle plugin applies retention, archival, and deletion rules independently of the downloader.

### Consumer plugins and applications

Routing, visualisation, weather providers, and analysis tools discover resources using the standard Resources API and choose the datasets they need. Consumers can be server plugins or external mobile, desktop, and web applications using the same HTTP resource contract.

---

## 6. Resource Metadata vs Binary Content

The Resources API is JSON-oriented. This is appropriate for resource discovery and metadata, but not for transporting large GRIB files.

The proposed model is therefore:

```text
GET /signalk/v2/api/resources/gribs
```

Returns the GRIB resource catalogue.

```text
GET /signalk/v2/api/resources/gribs/{id}
```

Returns one GRIB resource description.

The resource description contains a URL for the actual binary content.

For example:

```json
{
  "name": "AROME 0.01° 2026-10-06 06Z",
  "model": "AROME",
  "run": "2026-10-06T06:00:00Z",
  "resolution": 0.01,
  "bounds": [-12.0, 37.0, 16.0, 55.0],
  "forecast": {
    "from": "2026-10-06T06:00:00Z",
    "to": "2026-10-08T06:00:00Z"
  },
  "parameters": [
    "wind",
    "pressure",
    "temperature",
    "precipitation"
  ],
  "content": {
    "href": "/plugins/grib-resource-provider/gribs/arome-20261006-06/data",
    "type": "application/x-grib2"
  }
}
```

The exact schema remains to be defined.

---

## 7. Subsetting

Subsetting is an important capability, especially for vessel-to-shore communication.

A content endpoint could support requests such as:

```text
GET /plugins/grib-resource-provider/gribs/{id}/data
    ?bbox=5,42,8,44
    &parameters=wind,pressure
    &from=2026-10-06T12:00:00Z
    &to=2026-10-08T00:00:00Z
```

The provider would return a new GRIB containing only the requested subset.

Subsetting is a provider capability, not a mandatory feature of every GRIB resource.

The resource should therefore be able to advertise supported capabilities, for example:

```json
{
  "content": {
    "href": "/plugins/grib-resource-provider/gribs/arome-20261006-06/data",
    "type": "application/x-grib2",
    "capabilities": {
      "spatialSubset": true,
      "temporalSubset": true,
      "parameterSubset": true
    }
  }
}
```

A provider that cannot subset simply exposes the original GRIB.

---

## 8. Working Assumption: No Initial Core Change

The proposed approach relies on the Signal K Resource Provider model's support for custom resource types.

A plugin can therefore register a resource type such as:

```text
gribs
```

and expose resources through the normal Resources API.

The binary content does not need to pass through `getResource()`.

Instead, the resource metadata can expose an `href` handled by the GRIB Resource Provider plugin itself.

This is conceptually similar to chart resources, which describe tile or map sources while the actual chart imagery or vector content is retrieved separately.

This approach has several benefits:

- no server-core dependency for a first implementation;
- easier experimentation;
- independent plugin release cycle;
- no GRIB-specific logic in the Signal K core;
- the resource schema can evolve before proposing any generic server changes;
- binary transfer and streaming remain under direct control of the plugin.

A future generic enhancement to Signal K Resources may still be desirable if other resource types also need a standardised binary-content mechanism, but it is not a prerequisite for this project.

The no-core-change assumption should be checked against the target server version and reviewed with maintainers before implementation. The proposal does not establish a new core API or a final resource schema.

---

## 9. Relationship to Existing Components

### GRIB downloader

Current GRIB downloaders can evolve from writing files into a private directory structure to publishing discoverable GRIB resources.

### GRIB Weather Provider

A GRIB Weather Provider can discover and select appropriate GRIB resources, read or download them, and continue exposing normalised weather values through the existing Signal K Weather API.

This preserves compatibility with consumers already using the Weather API.

### Weather routing

A routing plugin can consume GRIB datasets directly, which is more efficient than querying individual forecast points through the Weather API.

### Weather visualisation

Visualisation tools can discover model runs and allow the user to choose the model, run, resolution, or field they want to display.

### External navigation applications

Mobile and desktop applications already using Signal K vessel observations can also retrieve GRIB resources to display forecast fields. Discovery and content retrieval should work through HTTP without requiring a server-side consumer plugin.

### Other Signal K servers

A remote Signal K server can be treated as another GRIB source, enabling low-bandwidth replication and federation.

---

## 10. Proposed Plugin Responsibilities

The project should avoid creating one monolithic plugin that performs every function.

Possible components include:

### `signalk-grib-resource-provider`

Core GRIB resource abstraction.

Responsibilities:

- resource metadata;
- binary content endpoint;
- dataset lookup;
- subsetting;
- capability advertisement;
- local storage abstraction.

### GRIB downloader plugins

Responsibilities:

- upstream service interaction;
- authentication where necessary;
- model-specific URLs or APIs;
- download scheduling;
- publication of resulting GRIB resources.

### GRIB lifecycle manager

Responsibilities:

- retention rules;
- archival;
- compression;
- cleanup;
- storage quotas.

### GRIB transformers / aggregators

Responsibilities:

- combining datasets;
- creating derived datasets;
- conversion;
- route-oriented extraction;
- model blending.

### Consumers

Examples:

- weather routing;
- GRIB-backed Weather Provider;
- Freeboard or other visualisation applications;
- external mobile, desktop, or web navigation applications;
- forecast-verification tools;
- remote Signal K replication.

---

## 11. Resource Discovery

The existing Resources API already supports query parameters when listing resources.

This can be used for GRIB discovery, for example:

```text
GET /signalk/v2/api/resources/gribs?model=AROME
```

or:

```text
GET /signalk/v2/api/resources/gribs
    ?bbox=[5,42,8,44]
    &from=2026-10-06T12:00:00Z
    &to=2026-10-08T00:00:00Z
```

The GRIB Resource Provider is responsible for interpreting and applying supported filters.

The exact list and semantics of discovery parameters still need to be standardised.

---

## 12. Provenance

Derived or mirrored datasets should retain provenance information.

A future schema should make it possible to identify:

- original upstream source;
- model;
- model run;
- server or plugin that created the current resource;
- parent resource(s);
- transformation history;
- creation timestamp;
- checksum or immutable content identifier.

This is important when GRIB resources are copied recursively between multiple Signal K servers.

---

## 13. Identity and Deduplication

Server-to-server exchange raises the question of dataset identity.

If the same AROME run is downloaded independently on two servers, it may be useful to determine that the content is identical.

Possible approaches include:

- provider-generated resource IDs;
- a stable dataset identifier derived from model/run/domain;
- cryptographic content hashes;
- both a local resource ID and a globally meaningful dataset identifier.

This is intentionally left open for further design.

---

## 14. Security Considerations

The first implementation should consider:

- whether GRIB discovery is publicly readable or follows normal Signal K authentication;
- access control for potentially expensive subsetting operations;
- protection against requesting unbounded subsets;
- maximum output size;
- request timeouts;
- rate limiting;
- remote-server authentication;
- avoiding arbitrary filesystem access through resource identifiers.

These are particularly important if a Signal K server exposes GRIB resources over the Internet.

---

## 15. Suggested Implementation Phases

### Phase 1 — Resource model and local provider

Define the GRIB resource metadata schema and implement a GRIB Resource Provider capable of:

- registering `gribs` as a custom resource type;
- listing local datasets;
- returning GRIB metadata;
- exposing original GRIB binary content through a plugin endpoint.

No subsetting is required for the first prototype.

Discovery and binary retrieval should also be demonstrated from an external HTTP client, without relying on a consumer plugin or direct filesystem access.

### Phase 2 — Existing downloader integration

Adapt a GRIB downloader to publish datasets through the new resource model instead of relying only on a private filesystem convention.

### Phase 3 — GRIB-backed Weather Provider

Adapt the GRIB Weather Provider to discover datasets through the Resources API.

This provides an immediate demonstration that producer and consumer are now decoupled.

### Phase 4 — Subsetting

Add spatial, temporal, and parameter subsetting to the binary content endpoint.

This enables the low-bandwidth vessel use case.

### Phase 5 — Server-to-server consumption

Implement a component capable of consuming remote GRIB resources and optionally republishing them locally.

### Phase 6 — Lifecycle management

Implement retention and archival policies as an independent plugin or service.

### Phase 7 — Aggregation and derived resources

Add support for plugins producing new GRIB resources from existing ones.

---

## 16. Open Questions

The following points still require design work and community discussion:

1. What should the canonical GRIB resource schema contain?
2. Should the resource type be named `gribs`, `weatherGribs`, `forecastDatasets`, or something more generic?
3. What MIME type(s) should be advertised for GRIB1 and GRIB2?
4. What query parameter names and formats should be standardised for subsetting?
5. How should a provider advertise its subsetting capabilities?
6. How should dataset identity work across multiple servers?
7. How should provenance and derived datasets be represented?
8. Should lifecycle metadata be part of the resource or remain provider-specific?
9. How should large subset-generation jobs be handled if they are too expensive for a synchronous HTTP request?
10. Should a future Signal K Resources extension define a generic binary-content relationship, based on lessons learned from GRIB and charts?
11. What authentication, content-URL resolution, and browser-origin access rules are needed for external mobile, desktop, and web clients?
12. How should clients identify changed content, validate cached datasets, and retain forecast validity information for offline use?

---

## 17. Current Working Conclusion

The current design milestone can be summarised as follows:

> GRIB datasets can be introduced into Signal K as a custom resource type using the existing Resources API and Resource Provider architecture.
>
> The Resources API provides dataset discovery and metadata, while the GRIB Resource Provider exposes the binary dataset through its own HTTP endpoint.
>
> Downloading, lifecycle management, transformation, routing, visualisation, and weather-value access remain separate plugins or consumers.
>
> Consumers include external navigation applications as well as server plugins, allowing Signal K to provide a common access point for vessel observations and forecast datasets.
>
> Spatial and temporal subsetting belongs in the GRIB Resource Provider, not in the Signal K server core.
>
> The proposed architecture is intended to support local datasets, derived datasets, historical datasets, low-bandwidth vessel access, and recursive server-to-server sharing. The working assumption is that the initial implementation can use existing server APIs without a core change; this remains to be validated.

This gives the project a practical path to an initial prototype while leaving room for later standardisation in the Signal K ecosystem.

---

## 18. Relevant Signal K References

- Signal K server Resources API implementation: `src/api/resources/index.ts`
- Signal K server Resource Provider interface: `packages/server-api/src/resourcesapi.ts`
- Built-in resources provider: `packages/resources-provider-plugin`
- Resource Provider plugin documentation: `docs/develop/plugins/resource_provider_plugins.md`
- Resources REST API documentation: `docs/develop/rest-api/resources_api.md`
- Chart resource schema: `packages/server-api/src/typebox/resources-schemas.ts`

Repository: <https://github.com/SignalK/signalk-server>

### GRIB standard and reader references

- [GRIB2: The WMO Standard for the Transmission of Gridded Data](https://www.weather.gov/media/mdl/GlahnLawrence2002GRIB2TheWMOStandard.pdf) — background on the standard format.
- [ECMWF GRIB2 format reference](https://codes.ecmwf.int/grib/format/grib2/) — GRIB structure, templates, and code tables.
- [ECMWF ecCodes geography keys](https://codes.ecmwf.int/grib/format/edition-independent/1/) — grid types and geographic metadata.
