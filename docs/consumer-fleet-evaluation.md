# GRIB Resources — plugin fleet evaluation

An evaluation of the GRIB Resources proposal from the perspective of weather-producing and weather-consuming Signal K plugins, one attributed section per plugin.

**Status: working notes.** This document is not part of the proposal itself; it is consumer- and producer-side feedback intended to inform the community discussion. The original four evaluations by Henri Bergius, were reviewed against proposal commit `f4bd072` (2026-10-06) and contributed in [PR #1](https://github.com/macjl/signalk-grib-resources/pull/1). The three additional source-based evaluations were prepared against proposal commit `8c922f7` (2026-10-06) and reviewed and approved by their maintainer, Jean-Laurent Girod.

Plugin authorship and evaluation authorship are recorded separately. Author fields were checked against the linked repositories' package metadata; the plugin revisions assessed in the original contribution are recorded from the local working checkouts used for review. The new evaluations cite specific source revisions and are based on code and documentation inspection, not runtime testing. Existing evaluations retain their original analysis; adding attribution does not independently verify their implementation claims or trial results.

## Purpose

The proposal asks consumers what metadata and access capabilities they actually need (see the README's discussion questions). Rather than answering from a single plugin, this report answers from a fleet of independent weather-handling plugins with deliberately different access patterns — point queries, bulk interpolation, visualization, downloads, and out-of-band acquisition — so common requirements and outliers become visible.

Each plugin evaluation is self-contained and follows the same template, so sections can be added (or pasted in from separate analysis sessions) without restructuring the document.

This shared document is organized by plugin rather than by developer, so producer, bridge, catalogue, and point-query requirements can be compared across contributors.

## How to add a plugin section

Add a `### <plugin name>` subsection under *Plugin evaluations*, following the template verbatim so sections stay comparable. Keep each section able to stand alone: a reader should not need the other sections to understand one plugin. Then add one row to the fleet matrix in *Fleet-wide observations* and move any new, generally applicable points into *Feedback items*.

Identify the plugin author or maintainer separately from the evaluator, link its repository, and record the plugin revision assessed. Cite source files for implementation claims and distinguish current behaviour from proposed adoption. If a revision or attribution is unknown, say so rather than inferring it. An evaluation is a contributor's assessment, not automatically a maintainer endorsement.

The template:

```markdown
### <plugin name>

**Plugin author / maintainer:** name and handle, with an attribution source where available.
**Repository:** link to the plugin repository.
**Evaluation by:** contributor name; note assistance and pending maintainer review where applicable.
**Plugin version / commit reviewed:** version and immutable source revision, or explicitly not recorded.

**Role:** one sentence on what the plugin does.
**Weather access today:** how it obtains forecast data (APIs, providers, files), and where that sits on the hot path.
**Verdict:** one line — direct consumer / indirect consumer / producer / not affected.

#### Data needs

Parameters, resolution, coverage, freshness, and access pattern (point vs bulk, cadence, offline requirements).

#### Direct GRIB resource consumption

Would the plugin consume `gribs` resources itself? Why or why not?

#### Indirect consumption via the Weather API bridge

How the proposal's producer → resource → Weather Provider → Weather API chain affects the plugin, if at all.

#### Metadata and capabilities wanted

Resource metadata, discovery filters, or content capabilities (e.g. subsetting) the plugin would use, and whether it can use them today.

#### Risks and caveats

Draft-status exposure, dependency weight, offline-first constraints, trust concerns.

#### Recommendation and feedback

What the plugin should do now, what to watch, and specific feedback this plugin's use case contributes to the proposal discussion.
```

## The proposal in brief

GRIB datasets become a custom `gribs` resource type in the existing Resources API: JSON metadata and discovery through `GET /signalk/v2/api/resources/gribs`, binary content (with optional subsetting) through plugin HTTP endpoints referenced from the metadata. Downloading, lifecycle, transformation, and consumption stay separate responsibilities. The Resources API layer is deliberately complementary to the Weather API — datasets vs normalized point queries — with a GRIB-backed Weather Provider as the bridge between the two. The proposal is a draft: schemas and URLs are illustrative, and the no-core-change assumption is unvalidated. See [project outline](project-outline.md) for the full design.

## Plugin evaluations

### signalk-dead-reckoning

**Plugin author / maintainer:** Henri Bergius (@bergie), as declared in [package metadata](https://github.com/meri-imperiumi/signalk-dead-reckoning/blob/main/package.json).
**Repository:** [meri-imperiumi/signalk-dead-reckoning](https://github.com/meri-imperiumi/signalk-dead-reckoning).
**Evaluation by:** Henri Bergius (@bergie), original contribution; LLM-assisted evaluation with GLM-5.3-flash.
**Plugin version / commit reviewed:** 0.13.1, [`1311a12`](https://github.com/meri-imperiumi/signalk-dead-reckoning/commit/1311a1267e8d391b01df5863def2bda939e8aa6a).

**Role:** offline-first dead reckoning and sensor fusion engine that maintains a continuously computed "shadow boat" position from water-track sensors, learning vessel-specific leeway and current corrections against GPS ground truth.
**Weather access today:** Signal K Weather API point queries (`app.weatherApi.getForecasts()`, in-process) for the surface current at the vessel position, polled on a slow interval and cached; never on the 1 Hz hot path.
**Verdict:** indirect consumer only — the Resources API layer should stay invisible to this plugin.

#### Data needs

A single point series: surface current set/drift at the vessel position, roughly six time steps ahead, refreshed on the order of 30 minutes. Parameters beyond current (wind, pressure) matter only indirectly through other integrations (polar performance, calibration context). Offline behaviour is the defining constraint: the plugin degrades through its current hierarchy of truth — manual override, derived residual from the boat's own GPS-vs-water-track observation, Weather API forecast, offline pilot charts, zero vector — so forecast data is always best-effort and its absence must never break the DR solution. Modelled currents are deliberately second-class: they are integrated into the shadow boat but never used to train the calibration matrix, since training on modelled data bakes the model's error into the learned bins as fake corrections (confirmed by sea trials: boat-own observation ~11 nm DR error over 622 nm vs 47 nm for the modelled-current tier).

#### Direct GRIB resource consumption

No. The access pattern — one position, one parameter, a handful of time steps, slow cadence — is exactly the Weather API's sweet spot, and the proposal's own efficiency argument for direct GRIB access (thousands of interpolations for routing engines) does not apply. Direct consumption would drag an ecCodes-class decoder into a plugin whose entire runtime dependency list is `astronomy-engine`, and would couple the DR hot path to a draft spec. The Weather API integration already exists, is in-process, and its parser already tolerates both real-world response shapes.

#### Indirect consumption via the Weather API bridge

This is where the proposal genuinely helps. Today tier 3 of the current hierarchy only exists if a specific provider plugin happens to serve currents. If downloaders publish `gribs` resources and a GRIB-backed Weather Provider bridges them to the Weather API, the tier-3 supply chain widens — any downloader, any provider, no private filesystem conventions — with zero code changes on the DR side. This matches what the DR SPEC's execution architecture already assumes upstream of the current hierarchy (a "GRIB Provider Service"), validating that design rather than changing it.

#### Metadata and capabilities wanted

One thing, currently missing end to end: **provenance pass-through**. The proposal's dataset metadata (model, model run, freshness, resolution) is information a safety-adjacent consumer could actually use — bounding the forecast-cache TTL by dataset freshness instead of a fixed multiple of the poll interval, or surfacing "currents from AROME 06Z run, 14 h stale" in status and notifications. But the Weather API carries no provenance today, and the proposal leaves the Weather-Provider-to-Weather-API bridge undefined. Until provenance crosses that bridge, none of this metadata is reachable from this plugin, no matter how rich the resource schema is. Subsetting is irrelevant to this plugin's own access pattern.

#### Risks and caveats

The proposal is a draft with illustrative schemas and an unvalidated no-core-change assumption; anything built directly against `gribs` today would be built on sand. This plugin has no such exposure as long as it stays behind the Weather API boundary. The plugin's defensive posture (tolerant parser, TTL-bounded cache, graceful fall-through to observed tiers and zero vector) already absorbs ecosystem churn.

#### Recommendation and feedback

Keep the plugin a Weather API point-query consumer; no direct `gribs` integration. Track the proposal as ecosystem plumbing that makes tier-3 currents reliably available, and prefer it over private downloader-provider couplings as it matures. Revisit only if the Weather API proves unreliable as the tier-3 source in practice, provenance becomes standardized, or archived historical datasets become a viable enrichment source for the offline backfill (which wants reanalysis-class products, not archived forecast runs — the proposal's historical use case targets the latter).

Feedback to the discussion:

- **Provenance through the Weather API bridge** is the single most valuable capability this consumer lacks: define how dataset identity, model run, and freshness (§12) surface in normalized point-query responses, not only in resource metadata.
- The proposal's routing-centric framing underrepresents **safety-adjacent small consumers** like navigation fallback engines; their needs are modest (point queries plus trust metadata) but their tolerance for stale or misattributed data is far lower than a map layer's.

### signalk-energy-predictor

**Plugin author / maintainer:** Henri Bergius (@bergie), as declared in [package metadata](https://github.com/meri-imperiumi/signalk-energy-predictor/blob/main/package.json).
**Repository:** [meri-imperiumi/signalk-energy-predictor](https://github.com/meri-imperiumi/signalk-energy-predictor).
**Evaluation by:** Henri Bergius (@bergie), original contribution; LLM-assisted evaluation with GLM-5.3-flash.
**Plugin version / commit reviewed:** 0.11.3, [`1c057ce`](https://github.com/meri-imperiumi/signalk-energy-predictor/commit/1c057ce11b19296fa8d3f4250a3dbcb53b09fce0).

**Role:** predicts the vessel's energy generation (solar, wind generator, hydrogenerator) and battery trajectory for the next 24–48 h by combining weather forecasts with models learned from the vessel's actual generation history, and publishes actionable advisories (stow a deployable, run the genset or engine, surplus/deficit outlook).
**Weather access today:** a 4-tier ingestion FSM: tier 1 fetches Open-Meteo hourly point forecasts (shortwave radiation passed through as GHI plus wind speed, direction and gusts; 48 h default, up to 168 h configured), tier 2 queries the server's Weather API in-process via `app.weatherApi.getForecasts()` (Kasten-Czeplak synthesis from `outside.cloudCover`, wind fields when the provider carries them), tier 3 reads observed cloud cover from signalk-logbook through the `logentries` resource API, tier 4 is clear-sky geometry. Refetch runs on a debounced schedule plus uplink-transition events (Starlink online, metered↔unmetered), and points land in an on-disk per-day weather cache tagged by tier with TTL-bounded reuse; tier 1 is suppressed on metered uplinks in favour of the in-process tier 2. Not a 1 Hz hot path, but safety-adjacent: outputs gate deployable stowage and genset/engine run recommendations.
**Verdict:** indirect consumer through the Weather API bridge — the Resources API layer stays invisible to it, but the proposal standardizes the private downloader→provider chain that currently feeds its tier 2.

#### Data needs

A point series at the vessel position, hourly, 24–48 h ahead (up to a week when configured), refreshed on a debounced schedule plus uplink events. Parameters: cloud cover (drives the solar synthesis), wind speed and direction (wind-generator power curves, anchorage wind-protection bins, and — via a polar lookup — forecast boat speed, which is what hydrogenerator prediction actually consumes, making it effectively a one-boat weather-routing computation), and gusts (the deployable stow gate: "take down FLINsail, gusts over 20kn overnight"). Offline behaviour is the defining constraint, with a longer memory than the dead-reckoning case: the FSM degrades through logbook persistence to a clear-sky baseline and must always publish a best-effort forecast. Accuracy appetite is modest at anchor — the learned per-vessel efficiency absorbs systematic derating — but staleness converts directly into over-confident advisories.

#### Direct GRIB resource consumption

No. The access pattern is a single point series; direct consumption would drag an ecCodes-class decoder into a plugin that deliberately runs on a thin dependency budget, and the plugin must still produce a forecast when no GRIB exists at all (tier 4). Per-model parameter availability is exactly what a normalised provider should absorb: cloud cover is widely carried in marine GRIBs, gust availability varies by model and download, and the plugin treats gust absence as "advisory unavailable" rather than "dataset unusable". Building directly against a draft schema with an unvalidated no-core-change assumption would also couple the learning pipeline's upstream to shifting sand for no gain over the existing in-process Weather API call.

#### Indirect consumption via the Weather API bridge

This is where the proposal touches the plugin today, because its tier-2 supply is already GRIB-backed through private conventions: signalk-grib-downloader orchestrates GFS/AROME/ARPEGE/ICON-EU downloads and feeds signalk-grib-weather-provider, which answers the Weather API the plugin calls. Phases 2–3 replace that private producer→provider coupling with `gribs` resources and a bridge, so any downloader — including ones not yet written — widens tier-2 supply with zero code change on this side; the same insulation argument as the dead-reckoning tier-3 case, one tier up. The metered-uplink design is where the proposal earns its keep: tier 2 is preferred on metered links on the assumption that in-process reads are free, and without a shared dataset pool that cheapness is accidental — the alternative is every weather-hungry plugin doing its own forecast downloads over the same link. A central `gribs` pool makes the cheap path architectural instead: one download per model run, fetched once and served in-process to every consumer, schedulable for unmetered windows by a lifecycle plugin, with subsetting and server-to-server federation (Phases 4–5) shrinking even that single transfer to a route-sized subset when the source is a shore server.

#### Metadata and capabilities wanted

Three things, all currently inferred heuristically because the Weather API carries no dataset truth. Provenance: tier-2 datasets are accepted by checking their newest point is roughly current (2 h grace) and cached forecasts are bounded by TTLs; a model-run timestamp and freshness crossing the bridge would replace those guesses with facts, gate advisories (don't recommend an engine run off a day-old model run), and feed webapp transparency ("solar forecast from ICON-EU 00Z, 20 h stale"). Parameter-level capability: tier-2 acceptance is dataset-level (any cloud cover → accepted), so a dataset with cloud but no gusts silently disables every gust-dependent advisory with no warning path; propagating the resource schema's `parameters` list through the bridge would let the plugin degrade per parameter and say so — for which the schema should keep gusts a first-class parameter (not derivable from speed), distinguish total from low cloud (solar wants total), and carry forecast step spacing (3-hourly beyond 48 h on global models; the plugin interpolates, but knowingly is better). Historical runs (§4.5, Phase 6) are a niche want: backtesting uses reanalysis-class actuals as ground truth, but archived forecast runs — a different product — would let backtests decompose prediction error into learned-model vs forecast error and replay advisories ("would the stow advice have been right?") in hindsight.

#### Risks and caveats

Draft-status exposure is fully avoided as long as the plugin stays behind the Weather API boundary; direct consumption bears all of it. The plugin already carries a masquerade guard — a tier-2 response without cloud cover is rejected so a wind-and-pressure-only provider cannot shadow the fallbacks below — and a bridge that lets partial datasets register as complete would recreate that failure one layer down. The trust calculus is sharper than a visualisation consumer's: advisories trigger physical action (stowage, genset and engine runs), so stale or misattributed data costs diesel or a shredded sail, not just a wrong map colour. And the plugin's weather cache tags points by tier, not model run, so a fresher run silently overwrites a stale same-tier point today; provenance would make cache coherency visible too.

#### Recommendation and feedback

Keep the plugin a Weather API point-query consumer with no direct `gribs` integration; it already made the Phase-3-shaped bet (tier 2 in-process) and the proposal validates it. Watch the Phase-3 bridge contract as the highest-leverage deliverable for this plugin, and Phases 4–5 for the metered-vessel path. Revisit direct consumption only if the roadmap ever gains bulk spatio-temporal access, e.g. weather along a planned track.

Feedback to the discussion:

- **Parameter-level capability advertisement through the bridge** is the gap a dataset-level view misses: a cloud-without-gusts dataset silently disables gust-dependent advisories; propagate the resource `parameters` list (gusts first-class, total vs low cloud, step spacing) into what providers expose downstream.
- **Metered-uplink economics is the download-once argument:** point-query consumers on metered links delegate their bandwidth problem to the provider layer, and a shared `gribs` pool is what solves it — one fetch per model run served in-process to every plugin, instead of per-plugin forecast downloads; subsetting and federation (Phases 4–5) complete it by replacing the vessel-side fetch with a route-sized transfer from shore.
- **Provenance crossing the bridge** (shared with dead-reckoning) is what would replace the plugin's freshness heuristics with dataset truth and let it gate automation advisories on it.

### signalk-passage-briefing

**Plugin author / maintainer:** Henri Bergius (@bergie), as declared in [package metadata](https://github.com/meri-imperiumi/signalk-passage-briefing/blob/main/package.json).
**Repository:** [meri-imperiumi/signalk-passage-briefing](https://github.com/meri-imperiumi/signalk-passage-briefing).
**Evaluation by:** Henri Bergius (@bergie), original contribution; LLM-assisted evaluation with GLM-5.3-flash.
**Plugin version / commit reviewed:** 0.8.0, [`bd0471b`](https://github.com/meri-imperiumi/signalk-passage-briefing/commit/bd0471b7b50f04844568652aa905d7b405d715be).

**Role:** offshore passage daily briefing webapp — simulates the passage hour by hour from the boat's actual position (polar boat speed plus current) along whatever route is currently active, regardless of its source (signalk-weather-router-plus, an Orca or LuckGrib plan, a hand-drawn track), recommends sail changes learned from the electronic logbook, rates every hour on the Sereno comfort scale, and assembles bulletins, official alerts, hazards and sky events into one unified timeline the crew walks through each day underway.
**Weather access today:** track-weather fetch — waypoints sampled every 30 nm along the active route over a 7-day horizon; `weather_source: auto` prefers the server's Weather API in-process (`app.weatherApi.getForecasts()` per waypoint, served by signalk-weather-router-plus from its decoded ECMWF run) and falls back to Open-Meteo for that window otherwise (three batched multi-location requests: forecast, GFS-wave marine, SMOC currents). The payload is cached to disk (`weather/latest-<route>.json` plus dated snapshots) and must survive the roughly 23 offline hours between uplink windows; fetches are gated on `network.internet.state` (cron at 02/08/14/20 UTC in harbour, online/metered transitions offshore). Runtime dependency budget: three packages, no GRIB decoder.
**Verdict:** hybrid consumer — values stay behind the Weather API bridge (no decoder), but the plugin is a natural direct consumer of the resource *catalogue*: its selection criterion is "freshest, parameter-complete data for how the remaining trip will feel", not agreement with the router's plan, and making that choice needs exactly the metadata the `gribs` type advertises.

#### Data needs

The widest parameter set in the fleet, because the briefing models motion rather than conditions: surface wind, gusts and MSL pressure drive the hourly simulation and the Sereno comfort scale; combined sea **and** the swell/wind-sea partitions are both needed, since the encounter-period motion math at the scale's core wants the split; surface current set/drift feeds the ETA percentiles; CAPE and the pressure-layer fields behind the K-index drive the convective warnings. The access pattern sits between the single-point consumers and the routing engine the proposal centres on: a 600 nm route samples ~20 waypoints over the horizon — up to 168 hourly steps each, a few thousand point-time values per fetch — but the fetch happens once per window (cron in harbour, online/metered transitions offshore) and the cached payload then serves the whole day, so the cost is amortized and the defining constraint is offline-first: the fetch must complete inside the uplink window, and the payload must carry the remaining 23 hours. And the need is tactical, not static: underway the briefing re-simulates from the boat's actual position and trims the plan to route progress, so the coverage it wants shrinks day by day to the remaining corridor, and "fresh" is defined at briefing time — yesterday's run that planned the route is exactly what today's briefing should replace. The grading is the plugin's own product: the point of the exercise is "how will this trip feel" — comfort bands, sail work, departure timing — so the best dataset is the one that serves the motion model, not the one ranked for routing quality.

#### Direct GRIB resource consumption

Metadata yes; binary values not yet. The plugin's contract with the crew is the best and freshest forecast available, translated into how the trip will feel, for whatever route is currently active — the route may come from signalk-weather-router-plus, an Orca or LuckGrib plan, or a hand-drawn track, and there may be no weather-routing plugin on board at all, so the briefing's weather supply must stand entirely on its own; that it shares a source with the router today is a bandwidth side effect of `weather_source: auto`, not a constraint, and as a trip unfolds they already diverge by design (plan versus progress). Reading the `gribs` catalogue directly — plain JSON over the Resources API, no decoder — is therefore attractive: run timestamps and freshness to tell whether the answering provider is stale relative to what is on board, coverage to find datasets for the remaining corridor, parameter lists to know whether a dataset serves the motion model's swell partitions and upper air at all, and a resource ID to anchor provenance. Decoding GRIBs itself stays declined for now: the payload contract (UnifiedWeatherPayload, SPEC §3.1) is a point series, the webapp's worker is zero-dependency, the server side carries three runtime packages, and interpolation would be a new subsystem — but the refusal is conditional, not principled: if the bridge ecosystem cannot deliver the freshest parameter-complete data to the Weather API, the best-available goal eventually demands direct consumption, with a spool-ingested HF subset — possibly the freshest data on board mid-ocean — as the case that would force it.

#### Indirect consumption via the Weather API bridge

The bridge is today's delivery path — the router's decoded ECMWF run reaches the briefing through `app.weatherApi`, and proposal Phases 2–3 standardize the private downloader→provider chain behind it — but for a best-freshest consumer it has a second role the proposal does not yet name: letting the briefing detect when the answering provider is *stale* relative to what is on board. Today "a provider answered" is the whole signal; with run identity crossing the bridge, `auto` could become "freshest adequate source wins" instead of "first provider wins". The distinctive consumer-side gap is capability: the Weather API carries combined sea only, so whenever the router answers, the briefing's swell/wind-sea partitions and upper-air fields degrade to null even though its own Open-Meteo fallback carries them — a documented, silently accepted regression, because `weather_source` is all-or-nothing per window and the plugin cannot compose (wind and pressure from the router, partitions and CAPE from another source). The plugin already merges three independent Open-Meteo endpoints with per-endpoint graceful degradation, so the proposal's composition model (§4.1: "wind, wave, and current fields may also come from separate datasets") maps one-to-one onto its payload shape — it just needs capability signals to do the same behind the Weather API.

#### Metadata and capabilities wanted

- **Per-parameter capability through the bridge, for composition:** dataset-level "a provider answered" is not enough; the plugin needs to know "answered with wind and pressure but no swell partitions, no CAPE" so it can fill those blocks from another source instead of degrading them to null. Today that knowledge exists only as conventions about which providers carry what.
- **Provenance — and the specific-record anchor it enables:** the model label is currently scraped heuristically from the provider's free-text step description ("ECMWF IFS 0.25° open data, cycle …, +3 h"), and the briefing's provenance affordance deep-links every sourced item to the *specific* record — a GDACS event page, the Marine Regions gazetteer entry keyed by MRGID, a satellite tracker keyed by the NORAD catalog number parsed from the TLE — but weather is the one source whose anchor stops short: the record stays label-only with no URL, and the viewer deep link carries position and time, not the dataset. A particular GRIB's identity — resource ID, model, run, per §12/§13 — crossing the bridge would give weather the same anchor every other source already has: a resolvable dataset reference stamped on the briefing the crew reads, deep-linkable into a viewer at the exact run, stable across the payload's cached snapshots. Structured freshness would also gate publication ("comfort ratings from a 20 h-old run") the way the energy predictor gates its advisories.
- **Run identity and change validation (§13, open question 12):** offshore the plugin gets roughly one uplink window per day, and refetching ~20 waypoints × 7 days of the *same* model run spends scarce bandwidth for no new information. A cheap "has a new run published?" check — resource metadata carrying run timestamp and content identity — would let the window be skipped or pointed at a different dataset instead.
- **Best-freshest selection from the catalogue:** atmosphere, waves and currents as separately selectable datasets mirrors how the plugin already fetches; discovery filters by bbox (the *remaining* corridor, which shrinks as the trip progresses), parameter completeness and run-freshness ordering would let it rank what is on board and spend a scarce fetch window only when something better exists, instead of guessing per provider which source supports which waters.
- **Time-parameterized spatial subsetting — the moving window:** a static bbox must cover the whole route, but the boat only ever needs a bubble around where it will be *at each forecast step*. Saildocs-style course/speed windows have existed for decades for exactly this reason, and the plugin holds better inputs than course and speed: activeRoute plus the vessel polar give the expected position per forecast time. A subset request shaped as track-of-(position, time) pairs plus a bubble radius would cut a long-passage extraction to a fraction of the static-corridor area, and the daily variant — "just tomorrow's bubble" — matches how metered uplink windows actually happen. This is also what would make the plugin's conditional direct-consumption case (spool-ingested HF subsets) small enough to order mid-ocean at all.

#### Risks and caveats

Draft-status exposure stays behind the Weather API boundary, and the plugin's degradation conventions (nulls, absent blocks rather than invented values) absorb capability churn without lying to the crew. The trust stakes are underway-decision stakes: comfort ratings and sail recommendations drive deck work and departure timing mid-passage — costlier than a wrong map colour, though below the physical-action tier where genset runs and deployable stowage hang off the forecast. Offline is the hard constraint: provider reads work today because they are in-process and need no connectivity, and any bridge evolution that assumes a reachable upstream at query time would break the briefing exactly when it matters. And once sources can mix between windows, provenance stops being a nicety: a briefing that swaps datasets overnight must say so per block, or the crew reads a comfort rating that silently changed basis.

#### Recommendation and feedback

Values stay indirect — no decoder — but the plugin should grow into a metadata-level `gribs` consumer as the ecosystem matures: read the catalogue at each fetch window, rank what is on board for the remaining corridor by freshness and parameter completeness, spend the window only when something better exists, and anchor the briefing's provenance to the particular dataset that produced it. Agreement with the router is a benefit of shared sources, not a goal; the crew-facing contract is "how will the trip feel, from the best data on board". The route side is already source-agnostic, which mirrors the proposal's external-application use case (§4.8) from the other side: Orca and LuckGrib are route *producers* as much as GRIB consumers, and the briefing is the junction that briefs their routes — route provenance and forecast provenance are independent chains meeting in the derived product the crew reads. Watch Phases 2–3 for the bridge contract that makes this possible (run identity, parameter capability, provenance), and Phases 4–5 for the metered-window and remaining-corridor wins. The spool option stands: HF/Iridium-delivered subset GRIBs dropped into the "Local GRIB/Text Ingestion Directory" (today text-only) could be published as `gribs` resources by a producer plugin and served back through an offline Weather Provider bridge — the freshest data on board mid-ocean arriving by radio, visible to the catalogue, no decoder in this plugin.

Feedback to the discussion:

- **Advertise parameter-level capability through the bridge so point consumers can compose sources:** when the answering provider carries fewer parameters than the consumer's fallback, all-or-nothing source selection silently loses data — this plugin's swell/wind-sea partitions and convective indices degrade to null every time the router serves, by design today; per-parameter signals would turn that degradation into composition.
- **Treat run identity and change validation as a metered-bandwidth lever, not only a dedup concern:** scarce-window consumers refetch the same model run today because nothing cheaply answers "has a new run published?" before the window's bandwidth is spent; a run timestamp plus content identity, in resource metadata and across the bridge, would gate the fetch first.
- **Keep out-of-band ingestion in the producer story:** the downloader use case (§4.2) assumes a reachable upstream service, but offshore vessels obtain GRIBs by radio and email into file drops; a `gribs` producer that also accepts spool/file-drop ingestion would let the resource model cover the offline half of the supply chain.
- **Expose freshness and completeness as first-class discovery signals, not just content access:** consumers rank datasets differently by design — a routing engine for route quality, a briefing for motion-model parameter completeness plus run freshness on the remaining corridor — so per-consumer selection (the proposal's no-global-active-GRIB stance) only works if the catalogue can answer "freshest dataset covering this bbox with parameters X, Y, Z"; discovery filters and ordering deserve as much design attention as the binary endpoint.

### signalk-offshore-blogging

**Plugin author / maintainer:** Henri Bergius (@bergie), as declared in [package metadata](https://github.com/meri-imperiumi/offshore-blogging-system/blob/main/package.json).
**Repository:** [meri-imperiumi/offshore-blogging-system](https://github.com/meri-imperiumi/offshore-blogging-system) (package `@meri-imperiumi/signalk-offshore-blogging`).
**Evaluation by:** Henri Bergius (@bergie), original contribution; LLM-assisted evaluation with GLM-5.3-flash.
**Plugin version / commit reviewed:** 0.3.3, [`45825b4`](https://github.com/meri-imperiumi/offshore-blogging-system/commit/45825b47edfbf9158a4ffb23b9da3bdcb04fa6de).

**Role:** low-bandwidth lifeline plugin plus companion cloud server: encodes blog posts for InReach/Winlink transmission, turns crew weather requests into Saildocs queries, and receives the resulting GRIB files back over the same channels, persisting them on the boat's Signal K server for shared use.
**Weather access today:** it *acquires* rather than consumes — the vessel's primary weather access is high-bandwidth Starlink (signalk-grib-downloader feeding signalk-grib-weather-provider), and this plugin is the backup acquisition path for when that is unavailable or uneconomic; with Starlink pricing pushing cruisers off the primary path (the plugin SPEC's stated motivation), the backup is progressively becoming a co-primary. Outbound: a crew request or preset becomes a Saildocs email query (`model:lat1,lat2,lon1,lon2|grid|hours|params`); the cloud server stores the query text for reply correlation, and when the reply GRIB arrives it chunks it (compact header, type `G`, ~96 base64 chars per message) for InReach transmission behind a size-consent gate (above 15 chunks ≈ 1 KB the crew must approve the estimated cost with a YES). Inbound: the crew pastes received chunks into the plugin webapp; the server reassembles, validates the `GRIB` magic and edition byte, and persists to `…/gribs/inreach/<id>.grb`, where signalk-grib-weather-provider discovers it by directory convention; the Winlink variant uploads the whole email attachment through a single POST endpoint instead. No decoding anywhere: the plugin's entire GRIB awareness is magic bytes and the edition byte.
**Verdict:** producer — the out-of-band, file-drop producer the proposal's downloader story (§4.2) does not yet cover; never a weather consumer itself, and the fleet's first producer-side evaluation.

#### Data needs

Inverted for a producer: what matters is what flows *through* the plugin and what it can honestly know about those flows. The payload profile is the satellite echo of exactly the subsetting §7 describes — tiny, bespoke subsets (one model, a few parameters, a short horizon, ~1 KB per consent gate) chosen at request time by cloud-side query grammar, possibly hours old by the time they arrive. And the request side can get smarter with data the boat already holds: the same `navigation.course.activeRoute` plus Resources API route-geometry pattern passage-briefing uses (course-provider href, `pointIndex` progress, remaining-legs trim) would let the plugin compute the request area from the active route — bounding the *remaining* passage instead of a hand-entered box — so the default Saildocs query shrinks to the corridor ahead; a route-derived static corridor today, and the plugin becomes a natural client for track-shaped subset windows if providers ever advertise them. Metadata it can supply without decoding: acquisition channel (InReach assembly vs Winlink upload), the exact request text that produced the file, requester identity hash, timestamps, size, chunk count, and a content-derived id (uploads dedup on it; InReach-assembled files carry a cloud-generated random transmission id and do not). Metadata it cannot supply today: model, run, bounds, resolution, parameter list, forecast window — most of what the proposal's example schema shows — though that is a choice, not a hard limit: the organization maintains a WASM build of ecCodes (`@meri-imperiumi/eccodes-wasm`, ~5–10 MB, no native build step) that could be adopted as an opt-in dependency when the metadata value justifies it. Freshness inverts too: the path is exercised rarely, but when it is, the radio-ingested subset may simultaneously be the stalest *and* the freshest dataset on board.

#### Direct GRIB resource consumption

It would not consume `gribs` — it would *publish* them. Today publication runs through three private conventions: the directory layout that signalk-grib-weather-provider scans (exactly the private-filesystem coupling the proposal exists to eliminate), and the plugin's own HTTP endpoints (`/api/gribs` listing latest-first, `/api/gribs/:id/download` with edition-based MIME) serving the crew and external tools; the SPEC's target visualisation consumer, LuckGrib, gets files by manual download-and-import — §4.8's external-application pattern minus standard discovery. Adoption would be a Phase-2-shaped change on this side: register each persisted file as a `gribs` resource instead of relying on a sibling plugin's directory scanner, with the private endpoints re-plumbed onto the Resources API and a content `href`. One asymmetry matters for the contract: as the *backup* producer, its datasets must be distinguishable from primary-path ones — a full fresh run via Starlink and a bespoke radio subset cut from an older run are not interchangeable for any consumer doing best-freshest selection, so acquisition channel and arrival context are selection signals, not provenance decoration.

#### Indirect consumption via the Weather API bridge

Downstream, not upstream: everything the plugin acquires reaches the rest of the boat only through signalk-grib-weather-provider and the Weather API (dead-reckoning tier 3, energy-predictor tier 2, briefing fallbacks). The bridge is therefore what determines whether a hard-won radio subset is actually usable by point-query consumers — and the provenance gap other fleet sections flag has a producer-side corollary: this plugin holds requester, request text, and channel in its manifest, and has no path to push any of it toward the consumers on the other side of the bridge.

#### Metadata and capabilities wanted

- **A metadata-optional schema is the make-or-break item:** §4.2 assigns "identify its metadata" to producers, but an out-of-band producer may be structurally unable to decode what it holds — this one validates magic bytes and nothing else today, and while an org-maintained WASM ecCodes build makes decoding feasible when wanted, minimal installs and other producers will legitimately skip it. If the schema requires model, run, bounds, and parameters, the fallback supply chain is locked out of the ecosystem it exists to serve. Required fields should stay minimal; a lightweight GRIB2 section-1 scan (centre, reference time — a few dozen lines, no dependency) is a reasonable recommended middle ground; consumers should tolerate metadata-poor resources.
- **The acquisition request belongs in provenance (§12):** the query text that produced the file (`gfs:58n,60n,018e,022e|2,2|0,12|wind`) is the closest coverage-and-parameter description this producer will ever hold — for radio-ingested datasets the request *is* the transformation history. Channel (satellite vs upload vs direct download) and requester identity (already stamped as `requestedBy`) belong there too; in federated flows, who asked for a dataset is provenance the proposal does not currently name.
- **Content-derived identity works in practice (§13):** upload-path dedup on a content hash has run on this boat from day one, supporting the "local id plus globally meaningful identifier" pairing — the content hash is the floor every producer can compute, the model/run-domain identifier the better key when a decoder exists. The InReach path shows the gap from the other side: re-requesting the same subset after losing chunks creates a duplicate today because the id is random; deriving it from content at the cloud end would fix that, and is the kind of practice a shared identity discussion should encourage.
- **Cost estimation before transfer (§14):** the consent gate (estimate message count, require YES above threshold) is how satellite transfers stay affordable; "protection against unbounded subsets" should treat estimated-size reporting as a first-class provider capability, not just rate limiting.
- **MIME data point (open question 3):** the plugin already detects the edition from the GRIB header and serves `application/x-grib` vs `application/x-grib2`; harmless precedent for the discussion.

#### Risks and caveats

Draft exposure is real for a producer — registering resources means writing to a moving schema — but the plugin's role argues for early adoption once stable, not avoidance: backup systems are exercised rarely and must work when needed, and a transition can dual-run (private store plus resource publication) cheaply. Dependency posture: decoding is allowed when useful (the org's WASM ecCodes build removed the classic objections) but must never become a condition for producing valid resources — metadata-optional producers remain first-class citizens. The path also has a human in the loop — chunks arrive by hand, assembly can fail late (a 409 with a missing-chunk list), and incomplete state lives only in the browser session; if a resource can only exist once its content is complete, the model has no place for the "transmission 60% received, resumable across sessions" state that would materially improve the crew workflow — open question 8's lifecycle metadata could distinguish complete from partial datasets.

#### Recommendation and feedback

Adopt the resource contract as a producer once the schema stabilises: publish radio-ingested and uploaded GRIBs as `gribs` resources, retire the private list/download endpoints in favour of the Resources API plus content `href`, and drop the directory-layout coupling when signalk-grib-weather-provider moves to resource discovery (proposal Phases 2–3). Compute request areas from the active route the way passage-briefing already trims to it — the inputs are on board, and a corridor default turns every satellite weather request smaller and cheaper regardless of the proposal. Until then the filesystem bridge stays as-is — it works. Derive InReach transmission ids from content at the cloud end so re-requests dedup regardless of the proposal. Feed the discussion the producer-side gaps the consumers cannot see: metadata-optional schema, acquisition request/channel/requester in provenance, a lifecycle place for partial transmissions, and cost estimation as a capability.

Feedback to the discussion:

- **The schema must admit metadata-poor producers:** out-of-band ingestion often means the producer cannot decode the GRIB it holds (this one checks magic bytes by design); if model/run/bounds/parameters are required, the offline supply chain is locked out — keep required metadata minimal, consider a lightweight GRIB2 section-1 scan as recommended practice, and expect consumers to tolerate metadata-poor resources.
- **Producer-side confirmation for out-of-band ingestion (§4.2):** a real file-drop producer exists — satellite chunk reassembly plus whole-file upload, publishing today via directory convention — so the downloader story can be written from practice, not just inference.
- **Record the acquisition request, channel, and requester in provenance (§12):** for a radio-ingested dataset the request text is the only coverage-and-parameter description the producer holds, and in federated flows who asked for the dataset is provenance too.
- **Treat content hashes as the identity floor (§13):** content-derived ids already dedup re-uploads in production; model/run-derived identity is better but needs a decoder — the schema should carry the hash as the always-computable minimum alongside the better key.
- **Make cost estimation a first-class subset capability (§7/§14):** satellite flows gate transfers on an estimated-size YES; estimated-size reporting belongs in the content capability advertisement next to rate limiting.

### signalk-grib-downloader

**Plugin author / maintainer:** Jean-Laurent Girod (@macjl), as declared in [package metadata](https://github.com/macjl/signalk-grib-downloader/blob/61eca905212e9ae51ee4013e4e0062d4b3575cf2/package.json).
**Repository:** [macjl/signalk-grib-downloader](https://github.com/macjl/signalk-grib-downloader).
**Evaluation by:** Codex, source-based evaluation reviewed and approved by Jean-Laurent Girod.
**Plugin version / commit reviewed:** 0.2.4, [`61eca90`](https://github.com/macjl/signalk-grib-downloader/commit/61eca905212e9ae51ee4013e4e0062d4b3575cf2).

**Role:** obtains forecast runs from upstream services and manages automatic or manual downloads, source configuration, and local retention.
**Weather access today:** in-process HTTP acquisition for GFS, AROME, ARPEGE, and ICON-EU, writing files under a shared `<root>/<model>-<resolution>` directory convention. The Weather Provider discovers those directories independently; the downloader does not currently register GRIB resources or serve a binary-content catalogue.
**Verdict:** producer — a candidate for publishing completed datasets through the shared resource contract.

#### Data needs

The producer knows the requested model, resolution, forecast horizon, and run stamp, plus upstream-specific parameters, levels, packages, or groups. GFS supports upstream area subsetting; the other download paths obtain distributed files. These request settings describe intended content and should not be mistaken for a verified inventory of fields and coverage in the resulting GRIBs.

Automatic scheduling pauses when the published internet state is offline or captive and stretches scheduling waits on metered links. Manual downloads remain available. The management webapp already estimates download volume. A run may consist of several files, rather than one binary object.

#### Direct GRIB resource consumption

The downloader would publish resources, rather than use them for weather calculations. Its existing file writes use temporary files and rename; a completion marker records the fetch fingerprint once the run has finished. Older runs are then archived or removed according to configuration. See [download and completion handling](https://github.com/macjl/signalk-grib-downloader/blob/61eca905212e9ae51ee4013e4e0062d4b3575cf2/src/downloader.ts).

Resource publication should define whether each file is a resource or a completed multi-file run is one logical dataset. Publishing a run only after completion would provide a stronger dataset boundary than directory scanning alone.

#### Indirect consumption via the Weather API bridge

Today the bridge is the shared filesystem layout. Resource publication would allow the Weather Provider and other consumers to discover the results without depending on those directory names. The acquisition code and scheduling policy could remain in the downloader, while binary serving and metadata enrichment are handled by the Resource Provider. The publication interface between the two plugins remains to be designed.

#### Metadata and capabilities wanted

- Model, run, requested domain, horizon, and acquisition context, with a distinction between requested and verified content.
- An explicit representation of multi-file datasets and their completion state.
- Content identity as well as acquisition settings: the existing [fetch fingerprint](https://github.com/macjl/signalk-grib-downloader/blob/61eca905212e9ae51ee4013e4e0062d4b3575cf2/src/scheduler.ts) detects settings changes, but is not a hash of downloaded bytes.
- Publication and deletion notifications that keep catalogues synchronized with retention operations.
- A way to enrich metadata without requiring every downloader to carry a decoder.

#### Risks and caveats

The downloader already performs retention, whereas the proposal envisages an independent lifecycle manager. Adoption needs a clear ownership rule so the catalogue and stored content remain consistent when runs are archived or deleted. Requested capabilities also differ from serving capabilities: being able to ask NOMADS for a GFS subset does not make the local producer a generic subset server.

#### Recommendation and feedback

Adopt producer-side publication incrementally after the resource schema and publication interface stabilize, retaining the filesystem bridge during migration. Prioritize dataset granularity, completion, identity, and retention synchronization before advanced subsetting. This evaluation is based on the [README](https://github.com/macjl/signalk-grib-downloader/blob/61eca905212e9ae51ee4013e4e0062d4b3575cf2/README.md) and source inspection; it does not validate upstream downloads or volume estimates at runtime.

### signalk-grib-weather-provider

**Plugin author / maintainer:** Jean-Laurent Girod (@macjl), as declared in [package metadata](https://github.com/macjl/signalk-grib-weather-provider/blob/5acb4997f91ed27c763a24c83f2cce0a4aa3fcf1/package.json).
**Repository:** [macjl/signalk-grib-weather-provider](https://github.com/macjl/signalk-grib-weather-provider).
**Evaluation by:** Codex, source-based evaluation reviewed and approved by Jean-Laurent Girod.
**Plugin version / commit reviewed:** 0.5.0, [`5acb499`](https://github.com/macjl/signalk-grib-weather-provider/commit/5acb4997f91ed27c763a24c83f2cce0a4aa3fcf1).

**Role:** decodes locally available GRIB files, caches forecast fields, and exposes interpolated point forecasts through the Weather API.
**Weather access today:** periodically discovers source directories and ingests files with WebAssembly ecCodes into per-validity-time `.gribcache` slices. Each source is registered as an independently selectable Weather Provider. Point queries interpolate from the surrounding grid values; daily forecasts, observations, and warnings return empty arrays.
**Verdict:** direct GRIB consumer and Weather API bridge — its present ingestion pipeline could consume resource content instead of relying only on private directory discovery.

#### Data needs

The supported ingestion profile described in the [README](https://github.com/macjl/signalk-grib-weather-provider/blob/5acb4997f91ed27c763a24c83f2cce0a4aa3fcf1/README.md) is regular latitude/longitude GRIB2 data with a common grid within each file. Extraction targets 10 m wind, gusts, 2 m temperature, surface/sea-surface temperature, pressure, humidity, precipitation, and total cloud cover. Conversion is bounded in concurrency and cached locally, so subsequent point queries do not require upstream connectivity.

The current [extraction table](https://github.com/macjl/signalk-grib-weather-provider/blob/5acb4997f91ed27c763a24c83f2cce0a4aa3fcf1/src/ingest-wasm.ts) does not extract ocean currents, wave or swell fields, or upper-air convective parameters. Discovering resources carrying those fields would not by itself add support for them.

#### Direct GRIB resource consumption

Yes. Replace or supplement directory discovery with catalogue discovery, suitability checks, and retrieval of resource content into a local ingestion cache. Keep decoding and interpolation in this plugin. A standard GRIB file can still be outside this consumer's supported profile; the catalogue should help reject or explain unsupported inputs before expensive processing where possible.

The [store](https://github.com/macjl/signalk-grib-weather-provider/blob/5acb4997f91ed27c763a24c83f2cce0a4aa3fcf1/src/grib-store.ts) currently chooses the most recent reference time for overlapping validity times. A resource-aware implementation should retain the actual dataset reference used by each returned slice, rather than attaching one assumed run identity to a potentially mixed forecast series.

#### Indirect consumption via the Weather API bridge

This plugin is the bridge on which the indirect consumers depend. Its [Weather API mapper](https://github.com/macjl/signalk-grib-weather-provider/blob/5acb4997f91ed27c763a24c83f2cce0a4aa3fcf1/src/weather-mapper.ts) currently returns dates, types, and supported weather values without structured resource identity or model-run provenance. Resource metadata alone therefore does not reach downstream clients.

Dataset availability, decoder support, and parameters actually exposed through the Weather API are separate capabilities. Provider selection also needs a mapping between registered Weather Providers and resource IDs; selecting a directory-backed source today is not equivalent to selecting one immutable dataset or run.

#### Metadata and capabilities wanted

- Edition, grid representation, levels, verified parameters, run reference time, forecast steps, and coverage for ingestion and selection.
- Stable content identity to reuse decoded caches and invalidate them when bytes change.
- Per-slice or per-parameter provenance when a forecast response uses multiple runs or datasets.
- A clear source-to-resource mapping for clients selecting a dataset through the catalogue but querying values through the Weather API.
- Offline retrieval policy and explicit lifecycle behaviour for already cached content after resource removal.

#### Risks and caveats

The reviewed version depends on WebAssembly ecCodes and documents Node.js 24 or newer. Resource adoption does not remove decoder, grid, or field limitations. The current mapper can expose generic surface temperature as `water.temperature`; verified parameter semantics matter when consumers interpret that as actual sea-surface temperature. Metadata enrichment and expanded weather extraction should be considered separately, and changes to Weather API provenance need their own contract review.

#### Recommendation and feedback

Prioritize resource-based discovery and local caching while preserving existing point-query clients. Define the bridge's identity, capability, and source-selection contract early. Treat support for currents, waves, swell, or additional grids as explicit implementation work, not an automatic consequence of resource registration. The assessment is source-based; no conversion or forecast-query runtime validation was performed.

### signalk-weather-map

**Plugin author / maintainer:** Jean-Laurent Girod (@macjl), as declared in [package metadata](https://github.com/macjl/signalk-weather-map/blob/d8e40b324446f0a18eb891b1b27b5d422eaa97f2/package.json).
**Repository:** [macjl/signalk-weather-map](https://github.com/macjl/signalk-weather-map).
**Evaluation by:** Codex, source-based evaluation reviewed and approved by Jean-Laurent Girod.
**Plugin version / commit reviewed:** 0.4.0, [`d8e40b3`](https://github.com/macjl/signalk-weather-map/commit/d8e40b324446f0a18eb891b1b27b5d422eaa97f2).

**Role:** browser weather visualisation displaying forecast fields, a time slider, and vessel position on a MapLibre globe.
**Weather access today:** discovers registered providers through the Weather API, samples point forecasts on a viewport-dependent display grid, and renders weather layers in the browser. It uses memory and localStorage caches with a 30-minute TTL and batches point requests with up to 15 concurrent workers. It does not download or decode GRIB binaries.
**Verdict:** indirect consumer today; potential catalogue consumer for dataset metadata and selection. Direct binary consumption would be a separate architectural change.

#### Data needs

The displayed layers are wind, gusts, temperature, water temperature, cloud cover, precipitation, and pressure. The client needs geographically distributed point series covering the current viewport and available forecast times; grid density adapts to zoom. Vessel position and heading come from the normal vessel-data API. The current cache helps reuse loaded forecasts, but its TTL and cached viewport coverage do not constitute a complete offline forecast package.

#### Direct GRIB resource consumption

There is no direct consumption in the reviewed version. The [browser client](https://github.com/macjl/signalk-weather-map/blob/d8e40b324446f0a18eb891b1b27b5d422eaa97f2/public/index.html) discovers `/signalk/v2/api/weather/_providers` and queries `/signalk/v2/api/weather/forecasts/point` with a provider ID for each sampled position. It builds the display grid itself, rather than using a provider-specific grid endpoint.

The resource catalogue could add model/run choices, verified coverage, and dataset freshness without requiring browser-side GRIB decoding. Connecting such a choice to Weather API queries requires the bridge's provider-to-resource mapping. If direct binary rendering is later desired, it needs a decoder, field extraction, interpolation, and a distinct cache design; it should remain optional so non-GRIB Weather Providers continue to work.

#### Indirect consumption via the Weather API bridge

The map already benefits from locally decoded GRIBs through the Weather Provider. Changing upstream discovery to resources need not change its existing value-query path. Its provider-information panel currently infers available layers from loaded point responses; it lacks an authoritative dataset-level inventory and provenance reference. Bridge capability metadata would help distinguish unavailable fields, missing coverage, and fields not rendered by the map.

#### Metadata and capabilities wanted

- Dataset and model-run identity for source labels and cache invalidation: the current cache key uses provider and position, not run identity.
- Geographic and temporal coverage, including explicit behaviour around the antimeridian, for efficient viewport requests and source choice.
- Per-parameter availability and semantics, separate from which layers the UI can render.
- A mapping from a chosen catalogue dataset to an appropriate Weather API source.
- If binary retrieval is added later, browser-usable authenticated content URLs and advertised subset capabilities.

#### Risks and caveats

The reviewed application is hosted by Signal K and uses `window.location.origin` as its server address. It demonstrates a browser consumer, but does not yet implement arbitrary remote-server configuration for a standalone navigation application. Fetching a dataset once could reduce repeated point-query overhead for large views; that is a design opportunity to measure, not a demonstrated performance result. Loading map assets and retaining an offline-ready dataset are also separate concerns.

#### Recommendation and feedback

Keep the existing Weather API path while adding trustworthy catalogue metadata and source mapping as the contract matures. Treat browser-side binary rendering as an optional later experiment. Specify model-run cache validation and provider capabilities before expanding the source picker. Sources: the [README](https://github.com/macjl/signalk-weather-map/blob/d8e40b324446f0a18eb891b1b27b5d422eaa97f2/README.md) and browser code at the recorded revision; no visual or runtime tests were performed for this evaluation.

## Fleet-wide observations

Accumulates as plugin sections are added. The matrix gives one row per evaluated plugin; keep cells short and link to the section for detail.

| Plugin | Plugin author | Role | Access pattern | Direct `gribs` consumer? | Key want |
|---|---|---|---|---|---|
| [signalk-dead-reckoning](#signalk-dead-reckoning) | Henri Bergius (@bergie) | DR / sensor fusion | Weather API point queries, slow poll, offline-degrading | No | Provenance through the Weather API bridge |
| [signalk-energy-predictor](#signalk-energy-predictor) | Henri Bergius (@bergie) | Energy generation prediction & run advisories | 4-tier FSM; in-process Weather API point queries; metered-aware, offline-degrading | No | Provenance and per-parameter capability through the Weather API bridge |
| [signalk-passage-briefing](#signalk-passage-briefing) | Henri Bergius (@bergie) | Passage briefing: comfort rating, sail work, unified timeline; tactical re-brief underway, route-source-agnostic | Track series (30 nm waypoints, 7 d) via Weather API with Open-Meteo fallback; cached across the offline day | Metadata yes, values no — catalogue drives best/freshest selection | Per-parameter capability for composition; freshness and identity for best-dataset picks per window |
| [signalk-offshore-blogging](#signalk-offshore-blogging) | Henri Bergius (@bergie) | Backup GRIB acquisition (InReach/Winlink ingestion) behind a Starlink-primary vessel | Producer: Saildocs queries out, chunked satellite GRIBs in (~1 KB per consent gate); persists for shared download, no decoding | Producer (out-of-band file-drop); never a consumer | Metadata-optional schema; acquisition request/channel/requester in provenance; content-hash identity floor |
| [signalk-grib-downloader](#signalk-grib-downloader) | Jean-Laurent Girod (@macjl) | Scheduled/manual acquisition and local retention | Upstream HTTP to completed local runs; shared directory layout today | Producer; future resource publisher | Multi-file dataset boundaries, completion, and retention synchronization |
| [signalk-grib-weather-provider](#signalk-grib-weather-provider) | Jean-Laurent Girod (@macjl) | GRIB decoding and Weather API bridge | Local file ingestion to cached, interpolated point forecasts | Direct binary consumer; future resource discovery | Verified capabilities, source/resource mapping, and per-slice provenance |
| [signalk-weather-map](#signalk-weather-map) | Jean-Laurent Girod (@macjl) | Browser forecast visualisation | Viewport-grid sampling via Weather API; browser cache | Indirect today; catalogue use possible | Run-aware cache invalidation and authoritative layer/coverage metadata |

Shared patterns observed so far:

- Plugins behind the Weather API boundary are insulated from the proposal's draft-status risk; direct consumers bear all of it.
- The Weather-Provider-to-Weather-API bridge (proposal Phase 3) is the load-bearing wall for every indirect consumer; its contract deserves earlier definition than the resource schema itself.
- Point-query consumers on metered uplinks delegate their bandwidth problem to the provider layer, and a shared dataset pool is what solves it — one download per model run instead of per-plugin fetching, with subsetting and federation (Phases 4–5) to shrink even that transfer for the vessel.
- The bridge's parameter coverage, not just its existence, gates what point consumers can do: an answering provider that carries fewer parameters than the consumer's fallback forces all-or-nothing source choice and silently degrades data (passage-briefing's swell partitions and convective indices) — capability signals turn degradation into composition.
- Model-run identity is a bandwidth lever for metered vessels, not just a dedup concern: scarce fetch windows should be spent only when a new run has actually published.
- Subset size is a function of route geometry, not just area: a boat occupies one small bubble at a time, so time-parameterized (moving) extraction windows beat whole-route bounding boxes on every long passage.
- The producer's view inverts the consumers' wish list: consumers want richer metadata, but out-of-band producers may be structurally unable to compute it (no decoder on a radio lifeline), so the schema must degrade gracefully or the offline half of the supply chain is locked out — producer-side confirmation of the ingestion gap first raised under passage-briefing.
- The main HTTP downloader supplies a complementary producer case: a completed model run may span several GRIB files, and request settings are not a verified inventory of the downloaded content.
- Resource discovery, decoder support, and Weather API field support are independent layers. The reviewed GRIB Weather Provider extracts a limited field set; publishing a current or wave dataset does not automatically make those fields available to indirect consumers.
- Catalogue-based selection needs an explicit resource-to-Weather-Provider mapping. A source/provider ID is not necessarily an immutable run ID, and the bridge can use different runs at different validity times.
- A visualisation client may generate many point queries while remaining entirely behind the Weather API. Browser-side GRIB decoding is an optional redesign, not a prerequisite for benefiting from shared resources.

## Feedback items for the proposal discussion

Accumulating list of concrete points to raise, tagged with the contributing plugin:

- Define how dataset provenance (model run, freshness, identity per §12/§13) passes through a GRIB-backed Weather Provider into Weather API responses — resource metadata alone is unreachable for point-query consumers, and consumers evaluated so far substitute heuristics (cache TTLs, newest-point acceptance, free-text model-label scraping) where dataset truth would do. *(signalk-dead-reckoning, signalk-energy-predictor, signalk-passage-briefing)*
- Make identity a first-class part of bridge provenance, not just freshness: consumers publish provenance records downstream that deep-link to specific records — this plugin links a GDACS event page, a gazetteer MRGID, a satellite catalog number, but its weather record stays label-only because the Weather API carries no dataset identity; a stable, resolvable dataset identifier per §12/§13 (stable across subsets and federation, per the provenance chain) is what lets a point-query consumer anchor a provenance link the way every other source type already can. *(signalk-passage-briefing)*
- Consider small, safety-critical point-query consumers explicitly in the discussion framing, alongside routing and visualization — the energy predictor's advisories trigger physical action (stowage, genset/engine runs), so misattributed or stale data has an automation cost, not just a display cost. *(signalk-dead-reckoning, signalk-energy-predictor)*
- Advertise parameter-level availability through the Weather API bridge, not just dataset-level: a dataset carrying cloud cover but no gusts is accepted by a point-query consumer and silently disables every gust-dependent advisory; propagate the resource schema's `parameters` list (gusts as a first-class parameter, total vs low cloud, forecast step spacing) into provider-side capability signals — and enable composition, not just rejection: when the answering provider carries fewer parameters than the consumer's fallback, all-or-nothing source selection loses data (passage-briefing's swell/wind-sea partitions and convective indices degrade whenever the combined-sea-only provider serves). *(signalk-energy-predictor, signalk-passage-briefing)*
- Generalize subsetting from static bbox to time-parameterized windows: a moving window centered on the boat's expected position at each forecast step (Saildocs' course/speed GRIBs are the folk precedent; a consumer holding route plus polar can compute a better track than stated course and speed) shrinks long-passage subsets by an order of magnitude versus a whole-route bbox, and its daily variant — fetch only the next day's bubble — matches how metered uplink windows actually happen. A request shape of track-of-(position, time) pairs plus a width subsumes the static bbox (a degenerate single-point track), course/speed windows, and route/polar-aware planning; GRIB carries a grid definition per message natively, so providers implement this as per-timestep subsetting; advertise it as a content capability (§7) and record the window definition in derived-dataset provenance (§12) so results stay reproducible and dedupable (§13). The pattern has a second witness in the fleet: the backup-acquisition plugin sits on the same inputs — it can read `navigation.course.activeRoute` and the route geometry from the Resources API exactly as the briefing does — and could compute route-derived default areas for its satellite weather requests today (a tighter remaining-corridor bbox), making it a ready-made client for track-shaped windows if providers advertise them. *(signalk-passage-briefing, signalk-offshore-blogging)*
- Highlight the download-once economics of the shared pool in the discussion framing: one fetch per model run served in-process to every weather-consuming plugin replaces per-plugin forecast downloads on metered vessels, and subsetting/server-to-server federation (Phases 4–5) shrink the remaining transfer to a route-sized subset from a shore server. *(signalk-energy-predictor)*
- Treat model-run identity and change validation (§13, open question 12) as a metered-bandwidth lever, not only a dedup concern: scarce-window consumers refetch the same model run today because nothing cheaply answers "has a new run published?" before the window's bandwidth is spent; a run timestamp plus content identity, in resource metadata and across the bridge, would gate the fetch first. *(signalk-passage-briefing)*
- Include out-of-band ingestion in the producer story: the downloader use case (§4.2) assumes a reachable upstream service, but offshore vessels obtain GRIBs by radio and email into file drops; a `gribs` producer that also accepts spool/file-drop ingestion would let the resource model cover the offline half of the supply chain. Producer-side confirmation now exists: one fleet plugin reassembles satellite-chunked GRIBs and accepts whole-file uploads, publishes via directory convention to a weather provider, and deliberately contains no GRIB decoder. *(signalk-passage-briefing, signalk-offshore-blogging)*
- Keep the resource schema metadata-optional for producers: §4.2 assigns "identify its metadata" to the producer, but an out-of-band producer may be structurally unable to decode what it holds; required metadata should stay minimal, a lightweight GRIB2 section-1 scan (centre, reference time) can be recommended practice, and consumers should expect metadata-poor resources. *(signalk-offshore-blogging)*
- Record the acquisition request, channel, and requester in provenance (§12): for a radio-ingested dataset the request text (e.g. a Saildocs query) is the only coverage-and-parameter description the producer holds — the request is the transformation history — and in federated flows the requester identity is provenance the proposal does not yet name. *(signalk-offshore-blogging)*
- Treat content hashes as the identity floor (§13): content-derived ids already dedup re-uploads in production on a radio-ingestion producer, while model/run-derived identity requires a decoder — the schema should pair the always-computable content hash with the better model/run key where one exists. *(signalk-offshore-blogging)*
- Make estimated-size reporting a first-class content capability (§7/§14): metered flows gate transfers on consent driven by a size estimate (this fleet's cloud server requires an explicit YES above ~1 KB), so a subset request should be answerable with an estimate before bytes move — a sibling of the spatialSubset/temporalSubset/parameterSubset advertisement. *(signalk-offshore-blogging)*
- Define dataset granularity and publication completion: a logical run can consist of several files. Decide how the catalogue represents those files and when the dataset is ready for use, rather than exposing a changing directory as if it were immutable content. *(signalk-grib-downloader)*
- Distinguish requested metadata from verified content, and dataset fields from decoder/bridge capabilities. A producer can report what it asked for; an ingestion service can enrich what it actually found; a Weather Provider must advertise what it can expose. *(signalk-grib-downloader, signalk-grib-weather-provider)*
- Define the mapping from catalogue resources to Weather API providers and dataset selection. Retain provenance at the slice or parameter level when a response combines runs, so a chosen source name does not imply one immutable dataset. *(signalk-grib-weather-provider, signalk-weather-map)*
- Specify lifecycle ownership and synchronization during migration: existing downloaders already delete or archive files, while the resource proposal assigns lifecycle management a separate role. Catalogue records, original content, and decoded caches need consistent rules. *(signalk-grib-downloader, signalk-grib-weather-provider)*
- Give browser clients a model-run/content version for cache validation and trustworthy coverage/capability metadata. The current Weather Map keys cached point forecasts by provider and position and infers layers from responses; resource identity and bridge metadata would allow more precise invalidation and source display. *(signalk-weather-map)*
