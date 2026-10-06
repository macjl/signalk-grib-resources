# Signal K GRIB Resources

An initial proposal for a shared way to discover and retrieve GRIB forecast datasets in the Signal K ecosystem.

**Status: draft for community discussion.** This repository contains a design proposal, not an implemented plugin or an adopted Signal K specification. The design is open for review.

## The problem

Downloaders, routing applications, weather providers, and visualisation clients need access to forecast datasets. A shared resource contract would let a producer publish a dataset and consumers discover and retrieve it without relying on private APIs or filesystem conventions.

GRIB already provides a standard format for exchanging gridded forecast data. This proposal addresses discovery and access within Signal K, building on that existing format.

Weather routing is a concrete example: a routing application should be able to find suitable datasets from independent downloaders, inspect their fields and coverage, and retrieve GRIB content for local interpolation. Each consumer remains responsible for deciding whether a dataset meets its needs.

## Proposed approach

- Use the existing **Resources API** for dataset discovery and metadata.
- Register a custom resource type, provisionally named `gribs`, through a **Resource Provider plugin**.
- Link resource metadata to plugin HTTP endpoints serving binary GRIB content.
- Keep downloading, lifecycle management, transformation, and consumption as separate responsibilities.
- Complement the **Weather API**, which provides normalised weather-value queries.

The working assumption is that a first implementation can use existing server APIs without a core change. This needs validation with the community. Resource names, schemas, URLs, and query parameters in the proposal are illustrative.

## Read the proposal

The [project outline](docs/project-outline.md) describes the use cases, architecture, implementation phases, and open questions.

Useful starting points:

- [Weather API and dataset access](docs/project-outline.md#3-why-the-weather-api-is-not-enough)
- [Consumer use cases, including weather routing](docs/project-outline.md#41-grib-consumer)
- [Proposed architecture](docs/project-outline.md#5-proposed-architecture)
- [Suggested implementation phases](docs/project-outline.md#15-suggested-implementation-phases)
- [Open questions](docs/project-outline.md#16-open-questions)

## Community feedback

[The shared plugin evaluation document](docs/consumer-fleet-evaluation.md), started by Bergie, examines the proposal from the perspective of weather producers, Weather API bridges, and consumers. Each section identifies the plugin author, repository, evaluator, and reviewed revision where recorded. It includes Bergie's original four evaluations and source-based evaluations for GRIB Downloader, GRIB Weather Provider, and Weather Map reviewed and approved by their maintainer.

The evaluations are a separate set of working notes intended to inform the discussion. Their recommendations are not adopted requirements of the proposal. Other plugin authors can contribute evaluations using the attributed template in that document.

## Initial scope

A first prototype would provide local dataset discovery, metadata, and original GRIB-file retrieval. Subsetting is not required for that prototype.

Later phases explore downloader and Weather Provider integration, spatial/temporal/parameter subsetting, transfer between shore and vessel servers over limited connections, retention and archival, and derived datasets.

## Join the discussion

Join the [Discord discussion in the Signal K Specifications channel](https://discord.com/channels/1170433917761892493/1557058626391384185) to exchange ideas with the community.

Use [GitHub Discussions](https://github.com/macjl/signalk-grib-resources/discussions) for general feedback on the proposal. In particular:

- Does this fit the intended use of Signal K Resource Providers?
- Is related work already available or underway?
- What metadata and access capabilities would routing, weather, and visualisation consumers need?
- Which use cases or constraints are missing?

Use [issues](https://github.com/macjl/signalk-grib-resources/issues) for specific design questions or corrections, and pull requests for proposed document improvements.

The [Signal K community page](https://signalk.org/community/) links to Discord and the wider community's GitHub Discussions. Exchanges there can inform this proposal; this repository provides a stable reference and history of its evolution.
