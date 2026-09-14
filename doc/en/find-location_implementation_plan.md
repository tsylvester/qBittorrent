# Find Location — Implementation Plan

Implement Find Location as three independently reviewable submissions. Produce one commit per submission, using the exact commit subject and commit gate specified below.

Treat steps 0 through 22 in [the dependency map](find-location_dependency_map.md) as dependency-ordered implementation units inside those three commits. Do not turn those steps into separate commits. Follow [the workplan](find-location_workplan.md) for the exact files, symbols, existing-code context, required edits, tests, integration scenarios and commit contents. Where an older planning document conflicts with the workplan, the workplan controls the implementation.

The requirements are defined by [the product requirements](find-location_product_requirements.md), [the feature specification](find-location_feature_spec.md), [the technical requirements](find-location_technical_requirements.md), [the non-functional requirements](find-location_nfr.md), [the system architecture](find-location_system_architecture.md) and [the technical approach](find-location_technical_approach.md).

## Submission 1 — automatic discovery

Implement tickets T0 through T9 as one submission.

### Required implementation order

1. Complete dependency-map step 0 before step 1, and step 1 before step 2.
2. Complete steps 3 and 4 independently of steps 0 through 2.
3. Complete step 5 only after steps 2, 3 and 4.
4. Complete steps 6 and 7 only after step 5. Either may be implemented first.
5. Complete steps 8 and 9 only after step 4. Either may be implemented before or after steps 5 through 7, but include both in this submission.
6. Preserve `FileSearcher::findInDir()` and its existing behaviour. Add the non-mutating counting primitive and the multi-root search exactly as directed by the Epic 1 workplan; do not implement the dependency map's stale statement that `findInDir()` returns a count.
7. Keep search-root composition and outcome logging in the session implementation as directed by the workplan. Do not move outcome logging into `FileSearcher`.
8. Store the Find Location settings in the BitTorrent session settings identified by the workplan. Do not relocate them to `Preferences`.

### Commit gate

Create one commit with the exact subject:

`Find existing torrent content before downloading`

Create it only after every Epic 1 workplan node is implemented; the automated tests and fixtures required by the workplan pass; the dependency map's manual cases for steps 6 through 9 pass; the full build and test suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows; the WebUI lint and format checks pass; and `WebAPI_Changelog.md` passes `rumdl`.

Include the two WebAPI preference keys and one changelog entry required by the workplan. Leave `API_VERSION` and the changelog version headings unchanged.

## Submission 2 — Start-triggered, manual, batch and assignment

Implement tickets T10 through T15 as one submission. Extend the existing-session work without changing the add-time and metadata-time behaviour implemented by Submission 1.

### Required implementation order

1. Complete step 10 independently.
2. Complete step 11 only after the session search method from step 5 and the settings from step 10 exist.
3. Complete step 12 only after step 5.
4. Complete step 13 only after steps 11 and 12.
5. Complete step 14 only after step 13.
6. Complete step 15 only after step 10. It may be implemented while steps 11 through 14 are in progress, but include it in this submission.

### Required transaction scope

Implement the Start-triggered transaction in `SessionImpl` through the existing lifecycle hooks identified by the architecture, technical approach and Epic 2 workplan. Make only the specified conditional delegation from `TorrentImpl::start()` and the minimum Stop-origin distinction required for cancellation. Do not introduce a replacement Start/Stop API, lifecycle notification layer, metadata path, storage-move queue or force-recheck implementation.

Implement the transaction in this order for each eligible Start request: intercept and preserve the requested normal or forced Start mode; hold payload activity; wait for metadata when necessary; search; settle the selected location through the existing location operations; wait for any required move; issue the feature recheck; wait for successful check completion; then release the preserved Start request through the existing Start body. On a search miss, release the preserved Start request without assignment or recheck.

Coalesce repeated Start requests for the same torrent, retain the last explicit Start mode, and prevent recursive interception when releasing that request. Cancel and erase feature-owned state on explicit Stop, torrent removal and session shutdown. On search, metadata, movement or recheck failure, erase the pending Start before applying ordinary stopped/error handling and never release Start. Ignore late and unrelated lifecycle callbacks.

When the Start-trigger setting is disabled, the master gate is disabled, the torrent is ineligible, or no transaction owns the callback, preserve the existing behaviour unchanged. Perform discovery asynchronously and never block the GUI thread. Before a search miss or a successful feature recheck, do not allocate, create, truncate or write payload files and do not request payload blocks.

Treat the general force-recheck defect as separate work. Add only the feature-owned failure cleanup required to prevent a failed Find Location transaction from releasing Start or retaining stale transaction state. Do not repair or redesign `TorrentImpl::forceRecheck()` in this submission.

### Commit gate

Create one commit with the exact subject:

`Find location before starting existing torrents`

Put `Closes #8261.` in the commit body. Create the commit only after every Epic 2 workplan node is implemented; all seventeen Epic 2 integration scenarios pass; the full build and test suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows; the WebUI lint and format checks pass; and `WebAPI_Changelog.md` passes `rumdl`.

Include `find_location_on_start_enabled`, `find_location_recheck_enabled`, `find_location_seed_enabled` and `find_location_leech_enabled` on both WebAPI preference endpoints, their desktop and WebUI controls, and one changelog entry as directed by the workplan. Leave `API_VERSION` and the changelog version headings unchanged.

## Submission 3 — discovery roots

Implement tickets T16 through T22 as one submission.

### Required implementation order

1. Complete step 16 independently.
2. Complete step 17 only after step 16.
3. Complete step 18 only after steps 3, 16 and 17.
4. Complete step 22 only after steps 5, 16, 17 and 18.
5. Complete steps 19 and 21 only after step 16. Either may be implemented while steps 17, 18 and 22 are in progress.
6. Complete step 20 only after steps 12, 14 and 22.

### Commit gate

Create one commit with the exact subject:

`Search discovery roots and pointed folders`

Create it only after every Epic 3 workplan node is implemented; the manual cases for steps 19 through 22 and all eight Epic 3 integration scenarios pass; the Epic 2 integration scenarios still pass; the full build and test suite pass under `-DTESTING=ON`; the WebUI lint and format checks pass; and `WebAPI_Changelog.md` passes `rumdl`.

Include `find_location_discovery_roots` on both WebAPI preference endpoints, its WebUI controls and one changelog entry as directed by the workplan. Leave `API_VERSION` and the changelog version headings unchanged.

## Commit and pull-request requirements

Use exactly the three commit subjects specified by this plan. Keep each subject capitalized, imperative, no longer than 50 characters and without a trailing period. In each commit body, explain the behaviour and the reason for the change, and record issue closure only where the workplan requires it.

Make each submission independently reviewable and keep unrelated changes out of it. Rebase the submission onto the target branch before final verification. Do not split dependency-map steps into preparatory commits, test-only commits, UI commits or WebAPI commits.

In each pull-request description, state the submission's scope and dependency on preceding submissions, enumerate the automated and manual verification performed, and describe the disabled-state regression result. Include screenshots for user-interface changes. A human contributor must review, take responsibility for and submit each pull request in accordance with the repository contribution policy.
