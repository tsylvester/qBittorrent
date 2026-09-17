# Find Location — Implementation Plan

Implement Find Location as three independently reviewable submissions, each landing as one commit with the subject and commit gate specified below.

[The workplan](find-location_workplan.md) is authoritative for the files, symbols, existing-code context, edits, tests, manual verification cases, integration scenarios and commit contents of every submission. Steps 0 through 25 in [the dependency map](find-location_dependency_map.md) are dependency-ordered implementation units inside the three commits, never separate commits.

The requirements are defined by [the product requirements](find-location_product_requirements.md), [the feature specification](find-location_feature_spec.md), [the technical requirements](find-location_technical_requirements.md), [the non-functional requirements](find-location_nfr.md), [the system architecture](find-location_system_architecture.md) and [the technical approach](find-location_technical_approach.md).

## Submission 1 — automatic discovery

Implement tickets T0 through T9 as one submission.

### Required implementation order

1. Complete step 0 first: register `testbittorrentfilesearcher.cpp`, `testbittorrentfilesearchermultiroot.cpp` and `testbittorrentcandidateroots.cpp` in `test/CMakeLists.txt`, build the `testbittorrentfilesearcher` target alone against the unmodified `filesearcher`, and run it green.
2. Steps 1, 2 and 3 share `filesearcher.h` and `filesearcher.cpp`. Add the `filesearcher.h` declarations next, so `testbittorrentfilesearchermultiroot` and `testbittorrentcandidateroots` compile and fail to link, which is their red state. Build only named test targets until `filesearcher.cpp` is complete, then run all three test executables green.
3. Complete step 4 independently of steps 0 through 3.
4. Complete step 5, `SessionImpl::findExistingContent()` together with `TorrentFilesWatcher::updateSessionWatchedFolderSavePaths()`, only after steps 2, 3 and 4.
5. Complete steps 6 and 7 only after step 5. Either may be implemented first.
6. Complete steps 8 and 9 only after step 4. Either may be implemented before or after steps 5 through 7, and both are included in this submission.

### Design

* `FileSearcher::search()`, `findInDir()` and `SessionImpl::findIncompleteFiles()` keep their signatures and behaviour. `countInDir()`, beside `findInDir()` in the anonymous namespace of `filesearcher.cpp`, counts a directory's matches without rewriting file names.
* `FileSearcher::searchRoots()` scores the torrent's save path, then its download path when one is set, then the search-only candidates, by the number of the torrent's files each holds. The directory holding the most files wins and the earlier directory wins a tie, so the torrent's own paths win a tie and lose to any candidate holding more. When no directory holds any file, the result is `search()`'s result.
* `candidateRoots()` is a pure function returning the search-only candidates, excluding the torrent's save path and download path.
* `TorrentFilesWatcher` pushes the watched folder save paths into the session through `Session::setWatchedFolderSavePaths()`. `SessionImpl::findExistingContent()` composes the search roots from them and records the outcome in the execution log: the torrent, location, originating folder and count when a search-only candidate wins; a miss when candidates were probed and no directory matched; nothing when the torrent's save path or download path wins.
* At add and at metadata receipt, a manual-mode torrent whose winning location is a search-only candidate takes that location as its save path with an empty download path, so no storage move follows its check. An automatic-mode torrent resolves through `findIncompleteFiles()`.
* The source form derives from a local `.torrent` file name alone; a magnet URI contributes none.
* The settings are the BitTorrent session settings `FindLocation/Enabled` and `FindLocation/OnAddEnabled`, each defaulting to enabled, exposed through `Session` and on `app/preferences` and `app/setPreferences` as `find_location_enabled` and `find_location_on_add_enabled`.

### Commit gate

Create one commit with the exact subject:

`Find existing torrent content before downloading`

Create it only after every Epic 1 workplan node is implemented; the three test executables pass against their fixtures; the manual verification cases for steps 6 through 9, as the Epic 1 workplan nodes enumerate them, pass; the full build and test suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows; the WebUI lint and format checks pass; and `WebAPI_Changelog.md` passes `rumdl`.

Include the `find_location_enabled` and `find_location_on_add_enabled` keys and one changelog entry linking the pull request by the number GitHub assigns when it is opened. `API_VERSION` and the changelog version headings are set by the maintainers.

The pull-request description states that `searchRoots()` probes the winning directory twice, once through `countInDir()` and once through the unchanged `findInDir()`, and offers a single pass that records the winner's names while counting as an alternative for the maintainers to choose.

## Submission 2 — Start-triggered, manual, batch and assignment

Implement tickets T10 through T15, T23 and T24 as one submission. Add-time and metadata-time discovery keep the behaviour Submission 1 delivers.

### Required implementation order

1. Complete step 10 independently.
2. Complete step 11 only after the session search method from step 5 and the settings from step 10 exist.
3. Complete step 12 only after step 5.
4. Complete step 13 only after steps 11 and 12.
5. Complete step 14 only after step 13.
6. Complete step 15 only after step 10. It may be implemented while steps 11 through 14 are in progress, and is included in this submission.
7. Complete step 23 only after steps 11 and 12.
8. Complete step 24 only after step 23.

### Transaction scope

The Start-triggered transaction lives in `SessionImpl`, held in `m_locationAssignments` and advanced through the existing lifecycle hooks the Epic 2 workplan identifies. `TorrentImpl` gains one `interceptFindLocationStart()` call in `start()` and the private `stop(bool)` Stop-origin overload that cancellation requires. `SessionImpl::searchExistingContent()` is the search shared by add-time discovery and the transaction, and `findExistingContent()` keeps its signature and add-time behaviour. Public `Torrent::start()` and `Torrent::stop()` signatures, the metadata path, the storage move queue, the location setters and `forceRecheck()` are unchanged, and the feature adds no lifecycle notification layer.

For each eligible Start request the transaction runs in this order: intercept and preserve the requested normal or forced Start mode; hold payload activity; wait for metadata when necessary; search; settle the selected location through the existing location operations; wait for any required move; issue the feature recheck; wait for successful check completion; then release the preserved Start request through the existing Start body. A search miss releases the preserved Start request without assignment or recheck.

Repeated Start requests for the same torrent coalesce, the last explicit Start mode is retained, and releasing the request passes through the interception without re-entering it. Explicit Stop, torrent removal and session shutdown cancel and erase feature-owned state. A search, metadata, movement or recheck failure erases the pending Start before ordinary stopped or error handling applies, and never releases Start. Late and unrelated lifecycle callbacks leave the transaction untouched.

With the Start-trigger setting disabled, the master gate disabled, an ineligible torrent, or no transaction owning the callback, behaviour is the existing behaviour. Discovery runs asynchronously and never blocks the GUI thread. Before a search miss or a successful feature recheck, no payload file is allocated, created, truncated or written and no payload block is requested.

The manual and batch modes reach the web interface and the headless daemon through `torrents/findLocation`, which holds one operation per web session in `TorrentsController`, assigns each match as its outcome arrives and answers HTTP 202 until no torrent is pending, and `torrents/assignLocation`, which assigns a named existing directory. Both reach discovery and assignment through `Session` alone. The web interface polls the first from its **Find location** context menu action and opens `setlocation.html` or `unmatchedtorrents.html` for torrents that matched nothing.

A manual **Find location** assignment follows **Set location...** semantics: an incomplete torrent with a download path keeps its storage in that download path. Add-time and metadata-time adoption from Submission 1 clears the download path instead.

The general force-recheck defect, [qBittorrent issue #14216](https://github.com/qbittorrent/qBittorrent/issues/14216), is separate work recorded in the workplan's To Do list. This submission adds only the feature-owned failure cleanup that keeps a failed Find Location transaction from releasing Start or retaining stale state, and `TorrentImpl::forceRecheck()` is unchanged.

### Commit gate

Create one commit with the exact subject:

`Find location before starting existing torrents`

Put `Closes #8261.` in the commit body. Create the commit only after every Epic 2 workplan node is implemented; all nineteen Epic 2 integration scenarios pass; the full build and test suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows; the WebUI lint and format checks pass; and `WebAPI_Changelog.md` passes `rumdl`.

Include `find_location_on_start_enabled`, `find_location_recheck_enabled`, `find_location_seed_enabled` and `find_location_leech_enabled` on both WebAPI preference endpoints, their desktop and WebUI controls, the `torrents/findLocation` and `torrents/assignLocation` actions with the web interface action and list, and one changelog entry naming the keys and actions and linking the pull request. `API_VERSION` and the changelog version headings are set by the maintainers.

## Submission 3 — discovery roots

Implement tickets T16 through T22 and T25 as one submission. Every mode the submission adds or extends is offered by the desktop interface, the web interface and the WebAPI.

### Required implementation order

1. Complete step 16 independently.
2. Complete step 17 only after step 16.
3. Complete step 18 only after steps 3, 16 and 17.
4. Complete step 22 only after steps 5, 16, 17, 18 and 23.
5. Complete steps 19 and 21 only after step 16. Either may be implemented while steps 17, 18 and 22 are in progress.
6. Complete step 20 only after steps 12, 14 and 22.
7. Complete step 25 only after steps 22 and 24.

### Commit gate

Create one commit with the exact subject:

`Search discovery roots and pointed folders`

Create it only after every Epic 3 workplan node is implemented; the manual cases for steps 19 through 22 and 25 and all ten Epic 3 integration scenarios pass; the Epic 2 integration scenarios still pass; the full build and test suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows; the WebUI lint and format checks pass; and `WebAPI_Changelog.md` passes `rumdl`.

Include `find_location_discovery_roots` on both WebAPI preference endpoints, its WebUI controls, the `root` parameter of `torrents/findLocation`, **Search folder...** in the web interface unmatched list, and one changelog entry naming the key and the parameter and linking the pull request. `API_VERSION` and the changelog version headings are set by the maintainers.

## Commit and pull-request requirements

The three commit subjects are exactly those specified by this plan: capitalized, imperative, no longer than 50 characters and without a trailing period. Each commit body explains the behaviour and the reason for the change, and records issue closure only where this plan requires it.

Each submission is independently reviewable and holds no unrelated change. Every dependency-map step, including its tests, user-interface changes and WebAPI changes, lands inside its submission's single commit. The submission is rebased onto the target branch before final verification, and its changelog entry sits under the version heading at the top of `WebAPI_Changelog.md` after that rebase.

Each pull-request description states the submission's scope and its dependency on preceding submissions, enumerates the automated and manual verification performed, describes the disabled-state regression result, and includes screenshots for user-interface changes. A human contributor reviews, takes responsibility for and submits each pull request in accordance with the repository contribution policy.
