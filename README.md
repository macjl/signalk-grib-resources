# Signal K GRIB Resources

An initial proposal for shared forecast dataset discovery and access in the Signal K ecosystem, with GRIB as the first implementation profile.

**Status: draft for community discussion.** This repository contains a design proposal, not an implemented plugin or an adopted Signal K specification. The design is open for review.

## The problem

Downloaders, routing applications, weather providers, and visualisation clients need access to forecast datasets. A shared resource contract would let a producer publish a dataset and consumers discover and retrieve it without relying on private APIs or filesystem conventions.

Producers should retain downloaded content in its original, immutable form without requiring a successful transformation. Consumers should be able to retrieve originals or request just the data they need through a common entry point. GRIB is the first format profile; other formats should be handled by independently installed processors.

Weather routing is a concrete example: a routing application should be able to find suitable datasets from independent downloaders, inspect their fields and coverage, and retrieve GRIB content for local interpolation. Each consumer remains responsible for deciding whether a dataset meets its needs.

## Proposed approach

- Use the existing **Resources API** for dataset discovery and metadata.
- Use one **catalogue/storage Resource Provider plugin**, with `forecastDatasets` as the proposed public resource type to review before prototyping.
- Preserve immutable original assets and expose content links, optional detailed indexes, and HTTP Range access.
- Let **independent format-specific processing plugins** register indexing, subsetting, or transformation capabilities. The catalogue advertises and routes supported requests without carrying GRIB/Zarr processing dependencies.
- Return subsets as **temporary results** by default; publish persistent derived datasets only through an explicit producer action.
- Keep acquisition, format-specific processing, catalogue/storage lifecycle, and consumption as separate responsibilities.
- Complement the **Weather API**, which provides normalised weather-value queries.

The working assumption is that a first implementation can use existing server APIs without a core change. This needs validation with the community. Processor registration and request/result handling are proposed plugin contracts, not existing Signal K APIs. Resource names, schemas, URLs, and query parameters remain illustrative.

The [2026-10-21 outline revision](docs/project-outline.md) consolidates the fleet evaluations and [discussion #3](https://github.com/macjl/signalk-grib-resources/discussions/3) alongside the Discord architecture exchange. It proposes a direction for review; silence in the discussion is not treated as community approval.

## Read the proposal

The [project outline](docs/project-outline.md) describes the use cases, architecture, implementation phases, and open questions.

Useful starting points:

- [Weather API and dataset access](docs/project-outline.md#3-why-the-weather-api-is-not-enough)
- [Consumer use cases, including weather routing](docs/project-outline.md#41-grib-consumer)
- [Proposed architecture](docs/project-outline.md#5-proposed-architecture)
- [Original access and delegated temporary processing](docs/project-outline.md#7-subsetting-and-processing-access)
- [Suggested implementation phases](docs/project-outline.md#15-suggested-implementation-phases)
- [Open questions](docs/project-outline.md#16-open-questions)

## Community feedback

[The shared plugin evaluation document](docs/consumer-fleet-evaluation.md), started by Bergie, examines the proposal from the perspective of weather producers, Weather API bridges, and consumers. Each section identifies the plugin author, repository, evaluator, and reviewed revision where recorded. It includes Bergie's original four evaluations and source-based evaluations for GRIB Downloader, GRIB Weather Provider, and Weather Map reviewed and approved by their maintainer.

The evaluations are a separate set of working notes intended to inform the discussion. Their recommendations are not adopted requirements of the proposal. Other plugin authors can contribute evaluations using the attributed template in that document.

## Initial scope

A contract-review phase should settle the public type name, publication/ownership and completion semantics, processor registration, and temporary-result lifecycle before implementation.

The first implementation stages demonstrate local publication, original access with Range and a separate GRIB indexer, then one delegated parameter/level/time-selection operation delivering a temporary GRIB to an external client. A processing failure must not require downloading originals again.

Spatial subsetting, additional formats, persistent derived workflows, federation, advanced archival, and shared decoded caches are later work. Weather API reconciliation/extensions belong in a separate discussion with server maintainers. The attributed fleet evaluations remain historical input rather than being rewritten for the new architecture.

## Join the discussion

Join the [Discord discussion in the Signal K Specifications channel](https://discord.com/channels/1170433917761892493/1557058626391384185) to exchange ideas with the community.

Use [GitHub Discussions](https://github.com/macjl/signalk-grib-resources/discussions) for general feedback on the proposal. In particular:

- Does this fit the intended use of Signal K Resource Providers?
- Is related work already available or underway?
- What metadata and access capabilities would routing, weather, and visualisation consumers need?
- Does the shared catalogue plus independent processor model meet both original-access and simpler subset-consumer needs?
- Which use cases or constraints are missing?

Use [issues](https://github.com/macjl/signalk-grib-resources/issues) for specific design questions or corrections, and pull requests for proposed document improvements.

The [Signal K community page](https://signalk.org/community/) links to Discord and the wider community's GitHub Discussions. Exchanges there can inform this proposal; this repository provides a stable reference and history of its evolution.
