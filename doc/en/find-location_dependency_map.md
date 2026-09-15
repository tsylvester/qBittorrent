# Find Location — Dependency Map

The order the work is built in, from the components that depend on nothing to the surfaces that depend on everything. The behaviour is set out in [the feature specification](find-location_feature_spec.md), its implementation in [the technical approach](find-location_technical_approach.md), and the files, tests and verification of every step in [the workplan](find-location_workplan.md).

Steps 0, 3, 4, 10, 16 and 17 carry no prerequisite, so work may begin at any of them and a selectable step exists at every point in the walk. Steps 5, 11, 13, 15, 18, 19, 20, 21 and 22 are where independent paths rejoin.

## First iteration

### 0. Behaviour pin — chain A

* **Location** `test/testbittorrentfilesearcher.cpp`, with `testbittorrentfilesearcher.cpp`, `testbittorrentfilesearchermultiroot.cpp` and `testbittorrentcandidateroots.cpp` registered in `test/CMakeLists.txt`.
* **Produces** a test fixing the two-directory behaviour of `FileSearcher::search()` as it stands: the save path probed first, the download path second, `forceAppendExt` applied to the destination, and a miss returning the destination.
* **Consumes** nothing.
* **Notes** built as its own target and run green against the unmodified `filesearcher` before any change to it, so it stands as the regression guard for every step that follows.

### 1. Probe primitives — chain A

* **Location** `src/base/bittorrent/filesearcher.cpp`, anonymous namespace.
* **Produces** `countInDir()`, counting how many of a torrent's files are present under a directory, directly or with `QB_EXT` appended, without rewriting the names, and abandoning the directory once its count can no longer exceed a supplied best count. `findInDir()` is unchanged beside it.
* **Consumes** step 0, which pins the behaviour of the unchanged `search()`.
* **Visible to** the enclosing translation unit alone.

### 2. Multi-root search — chain A

* **Location** `src/base/bittorrent/filesearcher.h` and its implementation.
* **Produces** `FileSearcher::searchRoots()` and `SearchRootsResult`, beside the unchanged `search()`. It scores the torrent's save path, then its download path when one is set, then the supplied candidates, by the number of the torrent's files each holds; the most files wins, the earlier directory wins a tie, and scoring stops once a directory holds every file. A miss produces `search()`'s result for the same inputs. The result carries the winning location, its file names, the match count, whether candidates were probed, and whether an own path won.
* **Consumes** step 1.
* **Visible to** `SessionImpl`, through the session search method of step 5. `FileSearcher` records nothing in the log.

### 3. Candidate root construction — chain B

* **Location** `src/base/bittorrent/filesearcher.h` and its implementation.
* **Produces** `candidateRoots()`, mapping a list of search roots, a torrent name and a source file name to the ordered list of candidates, each search root contributing the root form, the name form and the source form. The source form is the `.torrent` file's name with its extension removed. Forms equal to the torrent's save path or download path are omitted, duplicates are collapsed by `Path` equality at their earliest position, the session default is substituted for an empty entry, and a name or source form that would resolve outside its root is dropped. The root list parameter is named for what it carries rather than its origin, so the third iteration extends it without renaming.
* **Consumes** nothing. Every input arrives as a parameter, and the lookups that compose them belong to the callers.
* **Notes** pure with respect to the filesystem and to application state, so its unit tests are written alongside it.

### 4. Discovery settings — chain C

* **Location** `src/base/bittorrent/session.h`, `sessionimpl.h` and `sessionimpl.cpp`.
* **Produces** the group's gate and **Find location automatically** as BitTorrent session settings under `FindLocation/Enabled` and `FindLocation/OnAddEnabled`, each a getter and setter pair on `Session` defaulting to enabled and writing only on a changed value.
* **Consumes** nothing.
* **Visible to** the options dialog, the WebAPI, and the session search method.

### 5. Session search method — convergence

* **Location** `src/base/bittorrent/sessionimpl.h` and `sessionimpl.cpp`, with `src/base/torrentfileswatcher.h` and `torrentfileswatcher.cpp`.
* **Produces** `SessionImpl::findExistingContent()`, a sibling to `findIncompleteFiles()` taking the torrent's save path, download path, file paths, name and source file name. With either setting off it returns `findIncompleteFiles()`'s result. With both on it composes the search roots, derives the name form's name, builds the candidates through `candidateRoots()`, dispatches `searchRoots()` onto the I/O thread as `findIncompleteFiles()` dispatches `search()`, and logs a hit when a candidate wins, naming the torrent, location, originating watched folder save path and count, or a miss when candidates were probed and nothing matched, logging nothing when an own path wins. `TorrentFilesWatcher` pushes the configured save path of every watched folder, ordered by watched folder path, through `Session::setWatchedFolderSavePaths()` whenever the watched folder set changes, and the session resolves a relative entry against its default save path.
* **Consumes** steps 2, 3 and 4.

### 6. Resolution at add

* **Location** `SessionImpl::addTorrent_impl`, the `resolveFileNames` lambda and its continuation.
* **Produces** a save path chosen by scoring before the torrent reaches libtorrent. A manual-mode torrent resolves through `findExistingContent()`, with a source file name only when the descriptor's source is a local `.torrent` file; when a candidate wins, the torrent takes that location as its save path with an empty download path, so no storage move follows its check. An automatic-mode torrent resolves through `findIncompleteFiles()` as before.
* **Consumes** step 5.
* **Notes** the first point at which the feature is observable. A torrent added from a `.torrent` file whose content sits in a watched folder save path is adopted rather than downloaded.

### 7. Resolution at metadata received

* **Location** `TorrentImpl::handleSaveResumeData`, under `MaintenanceJob::HandleMetadata`.
* **Produces** the same resolution for torrents added from magnet links, with no source file name. When a candidate wins for a manual-mode torrent, its download path is cleared and `setSavePath()` records the location before `endReceivedMetadataHandling()`.
* **Consumes** step 5.
* **Notes** independent of step 6 and buildable either side of it.

### 8. Options dialog group

* **Location** `src/gui/optionsdialog.ui` and `src/gui/optionsdialog.cpp`.
* **Produces** the checkable **Find location** group holding **Find location automatically**.
* **Consumes** step 4.
* **Notes** available from step 4 onward, and meaningful to a user from step 6 onward.

### 9. WebAPI keys

* **Location** `src/webui/api/appcontroller.cpp`, `WebAPI_Changelog.md`, `src/webui/www/private/views/preferences.html`.
* **Produces** the `find_location_enabled` and `find_location_on_add_enabled` keys on `app/preferences` and `app/setPreferences`, one changelog entry linking the pull request, and the controls presenting them in the web interface. `API_VERSION` is set by the maintainers.
* **Consumes** step 4.

The first iteration is complete and submittable at step 9.

## Second iteration

### 10. Existing-session settings

* **Location** `src/base/bittorrent/session.h`, `sessionimpl.h` and `sessionimpl.cpp`.
* **Produces** **Find location when starting stopped torrents**, **Recheck automatically**, **Seed automatically** and **Leech automatically** as cached BitTorrent session settings. The Start-trigger setting is independent of **Find location automatically**; the three assignment settings retain their existing automatic-assignment meanings.
* **Consumes** nothing.

### 11. Existing-session assignment and pending Start

* **Location** primarily `src/base/bittorrent/sessionimpl.*`, with one conditional delegation in `TorrentImpl::start()` and the private `stop(bool)` Stop-origin overload required for cancellation.
* **Produces** manual assignment through `assignTorrentLocation()`, following **Set location...** semantics and rechecking and starting as the assignment settings allow, and the Start transaction with its small feature-owned state: searching, optional metadata wait, assignment through the existing location operations, waiting for the existing move queue, feature-issued recheck, and either automatic assignment completion or release of a preserved normal/forced Start. Repeated Start requests coalesce, and a one-shot pass lets the existing Start body run without recursion. `searchExistingContent()` is factored from `findExistingContent()` as the composition every call site reaches.
* **Consumes** steps 5 and 10.
* **Notes** existing `SessionImpl` metadata, info-hash-change, movement, checked, removal and error callbacks advance or clear the feature entry. No new lifecycle notification layer, public Start/Stop signature, move queue, metadata path or `forceRecheck()` implementation is introduced. Disabled and ineligible Start requests execute the existing Start body unchanged. An I/O error clears pending Start and stops the torrent; the next explicit Start reaches the existing error-clearing preamble before a fresh search, while the general force-recheck defect remains separate follow-up work. Steps 6 and 7 are regression baselines for this iteration, not edit points.

### 12. Session discovery interface

* **Location** `src/base/bittorrent/session.h`, implemented in `SessionImpl`.
* **Produces** `Session::findTorrentLocation()`, a `void` operation running discovery for one torrent; `Session::assignTorrentLocation()`; and the `torrentLocationFound` signal carrying one torrent's result.
* **Consumes** step 5.
* **Notes** what makes discovery and assignment reachable from `src/gui`, which holds `Session` rather than `SessionImpl`.

### 13. Transfer list action

* **Location** `src/gui/transferlistwidget.h` and `transferlistwidget.cpp`.
* **Produces** the **Find location** context menu action directly after **Set location...**, shown while the selection holds metadata and the group is enabled, over single and multiple selections. Each match is assigned as its outcome arrives; once every outcome has arrived, one miss opens **Choose save path** and several open `UnmatchedTorrentsDialog`. Another invocation during an active operation merges only unseen torrents into it.
* **Consumes** steps 11, 12 and 14.

### 14. Unmatched torrent list

* **Location** `src/gui/unmatchedtorrentsdialog.h`, `unmatchedtorrentsdialog.cpp` and `unmatchedtorrentsdialog.ui`, registered in `src/gui/CMakeLists.txt`.
* **Produces** the dialog listing torrents that matched nothing, walking them one at a time into **Choose save path** and assigning each chosen location, or closing to abandon the rest.
* **Consumes** step 11.

### 15. Existing-session settings surfaces

* **Location** `src/gui/optionsdialog.ui`, `src/gui/optionsdialog.cpp`, `src/webui/api/appcontroller.cpp`, `WebAPI_Changelog.md`, `src/webui/www/private/views/preferences.html`.
* **Produces** the Start-trigger checkbox and the three assignment checkboxes inside the **Find location** group, their enablement, and the `find_location_on_start_enabled`, `find_location_recheck_enabled`, `find_location_seed_enabled` and `find_location_leech_enabled` keys with one changelog entry and their web interface controls. `API_VERSION` is set by the maintainers.
* **Consumes** steps 8, 9 and 10.

The second iteration is complete and submittable at step 15.

## Third iteration

### 16. Discovery root storage

* **Location** `src/base/discoveryroots.h` and `discoveryroots.cpp`, registered in `src/base/CMakeLists.txt`, with `test/testdiscoveryroots.cpp` and its lifetime owned by `Application` in `src/app/application.cpp`.
* **Produces** `DiscoveryRootOptions`, `DiscoveryRoot` and the `DiscoveryRoots` singleton holding the ordered list, empty by default and persisted to `discovery_roots.json` as a JSON array of objects carrying `path` and `recursive`, with `parseDiscoveryRoots()` and `serializeDiscoveryRoots()` converting it. `Application` calls `initInstance()` before `BitTorrent::Session::initInstance()` and `freeInstance()` after `BitTorrent::Session::freeInstance()`.
* **Consumes** nothing.

### 17. Enumeration

* **Location** `src/base/bittorrent/filesearcher.h` and its implementation, with `test/testbittorrentsubdirectories.cpp`, run on the session I/O thread.
* **Produces** `SubdirectoryMap` and `enumerateSubdirectories()`, listing a root's immediate subdirectories as a name-to-path map, keyed with the case folding `Path::CASE_SENSITIVITY` implies. A root that cannot be listed yields an empty map.
* **Consumes** nothing.
* **Notes** `Path` cannot key the map, since `qHash` hashes unfolded while `operator==` compares through `Path::CASE_SENSITIVITY`.

### 18. Candidate root extension

* **Location** `src/base/bittorrent/filesearcher.h` and its implementation.
* **Produces** `candidateRoots()` taking a trailing list of optional subdirectory maps. A search root with a map contributes the root form, the mapped directory in the name form's position, omitted when the lookup finds nothing, and the source form; a search root without one contributes the three forms as before. The order of the search roots is the order they arrive in, which step 22 sets.
* **Consumes** steps 3 and 17.

### 19. Discovery root list

* **Location** `src/gui/optionsdialog.ui` and `src/gui/optionsdialog.cpp`, with `src/gui/discoveryrootsmodel.h` and `discoveryrootsmodel.cpp`, and `src/gui/discoveryrootoptionsdialog.h`, `discoveryrootoptionsdialog.cpp` and `discoveryrootoptionsdialog.ui`, registered in `src/gui/CMakeLists.txt`.
* **Produces** `DiscoveryRootsModel`, following `WatchedFoldersModel`; `DiscoveryRootOptionsDialog`, following `WatchedFolderOptionsDialog` and carrying the recursion flag; and the list with **Add...**, **Options...** and **Remove** within the **Find location** group, after the assignment checkboxes.
* **Consumes** steps 15 and 16.

### 20. Pointed root

* **Location** the unmatched torrent list from step 14.
* **Produces** **Search folder...**, taking a directory from the user and running one `Session::findTorrentLocations()` operation over the listed torrents with that directory as the pointed root, enumerated and ahead of every other search root, assigning and removing each torrent that matches, repeatable while the list holds entries so a library across several disks is covered one disk at a time.
* **Consumes** steps 12, 14 and 22. It runs from the GUI, so it reaches discovery through the `Session` interface rather than through `SessionImpl`.

### 21. Discovery root WebAPI keys

* **Location** `src/webui/api/appcontroller.cpp`, `WebAPI_Changelog.md`, `src/webui/www/private/views/preferences.html`.
* **Produces** `find_location_discovery_roots` on `app/preferences` and `app/setPreferences` as an array of objects carrying `path` and `recursive`, returning each path in the native form the WebAPI uses for `save_path` and `scan_dirs`, with one changelog entry and its web interface control inside the **Find location** fieldset. `API_VERSION` is set by the maintainers.
* **Consumes** steps 15 and 16.

### 22. Session root composition

* **Location** `src/base/bittorrent/session.h`, `sessionimpl.h` and `sessionimpl.cpp`, extending the composition built at steps 5 and 11, with `src/gui/transferlistwidget.cpp`.
* **Produces** `Session::findTorrentLocations()`, a batch operation taking torrent IDs and an optional pointed root; each operation's search roots composed as the pointed root, then the discovery roots in configured order, then the watched folder save paths; the pointed root and every recursive discovery root listed once per operation and the maps shared by every torrent in it; an automatic addition reusing an add operation still in flight whose composed roots and default save path are unchanged; the log naming the exact root whose candidates contain the winning location, by its origin; and `findTorrentLocation()` running as a one-torrent batch. The transfer list submits each invocation's new torrents through one `findTorrentLocations()` call.
* **Consumes** steps 5, 12, 13, 16, 17 and 18.
* **Notes** composition stays in one place, so the add path, the metadata path and the GUI path gain discovery roots together.

The third iteration is complete and submittable at step 22.

## Test plan

Tests build under `-DTESTING=ON` and run through `cmake --build <build> --target check`. CI runs them on Ubuntu, macOS and Windows.

Each test file links `Qt::Test` and `qbt_base`. Coverage therefore reaches `src/base` and stops there: `src/gui` is not linked and no fixture constructs a running session, so the resolution call sites and every GUI step are verified by hand, and the second and third iterations by the integration scenarios the workplan enumerates.

### Automated

* **Step 0** `test/testbittorrentfilesearcher.cpp` pins the two-directory behaviour of `FileSearcher::search()` before it is touched: probe order, `forceAppendExt` reaching the destination, the incomplete variant being adopted, and a miss returning the destination.
* **Step 2** `test/testbittorrentfilesearchermultiroot.cpp` covers `searchRoots()`: own paths scored with candidates and winning a tie, a candidate holding more beating an own path, scoring stopping once an own path holds every file, the most files winning, the earliest directory taking a tie, matching through `QB_EXT` without carrying rewritten names, `forceAppendExt` applied at the destination alone, a miss falling back to the destination, and absent, unreadable and empty roots contributing nothing. Directory fixtures go under `test/testdata/filesearcher`, resolved from `__FILE__` as `testutilsio.cpp` resolves its own.
* **Step 3** `test/testbittorrentcandidateroots.cpp` covers the order of the returned list, the root form, name form and source form contributed per search root, exclusion of the torrent's own paths, substitution of `defaultSavePath` for an empty entry, duplicate collapse, the source extension stripped case-insensitively, names escaping their root being dropped, and duplicates folding case on Windows. It passes a `PathList` and asserts on the result.
* **Step 16** `test/testdiscoveryroots.cpp` covers `parseDiscoveryRoots()` and `serializeDiscoveryRoots()`: round-tripping order and options, the serialised shape, the recursion flag defaulting to false, and entries dropped for an empty or relative path, a non-object value or a repeated path.
* **Step 17** `test/testbittorrentsubdirectories.cpp` covers `enumerateSubdirectories()` against a fixture tree under `test/testdata/filesearcher`: immediate subdirectories listed, files and nested directories not listed, absent and empty roots yielding an empty map, and keys folding case on Windows.
* **Step 18** `test/testbittorrentcandidateroots.cpp` gains the enumerated cases: the name form served by lookup and using the mapped path, omitted on a miss, roots beyond the supplied maps and roots with `std::nullopt` probed as before, and lookup folding case on Windows.

The five test files are added to `testFiles` in `test/CMakeLists.txt`.

### Manual

Verified against a prepared directory tree and described in the pull request:

* **Step 5** setting, replacing and removing a watched folder changes the set the next torrent add searches without a restart, and two watched folders whose save paths tie resolve to the same folder before and after a restart.
* **Step 6** a torrent added in manual mode from a `.torrent` file, whose content sits under a watched folder save path, is adopted rather than downloaded, issues no content request, keeps that location after its check, reports an empty download path and moves nothing, including when a download path is configured. A torrent whose save path holds part of its content while a watched folder save path holds all of it resolves to the watched folder save path. With both settings on and no watched folder configured, add resolves and logs as it does with the feature off. An automatic-mode torrent resolves as it does without the feature, and a seed-mode `.torrent` retains its existing bypass. A local `.torrent` contributes its source form, while a magnet URI, including one whose last parameter ends in `.torrent`, and an in-memory upload do not. The log shows exactly one message for a hit and for a miss, and none for an add won by the torrent's own save path. On Windows, a configured root whose traversal is denied leaves the remaining roots resolving normally.
* **Step 7** the same adoption holds for a magnet link once its metadata arrives, including with a configured download path.
* **Step 8** unchecking the **Find location** group returns torrent add to its two-directory behaviour.
* **Step 9** the two keys appear on `app/preferences`, setting them through `app/setPreferences` is reflected in the options dialog, a request omitting a key leaves its setting unchanged, and their controls render on the web interface preferences page and drive the same settings.
* **Steps 10 through 15** are verified by the seventeen Epic 2 integration scenarios in the workplan.
* **Step 16** the discovery root list persists its paths, order and flags across restarts; an absent file yields an empty list without a warning, and malformed JSON or a top-level object yields an empty list with a warning.
* **Step 19** a discovery root configured with recursion resolves torrents whose content sits in its subdirectories.
* **Step 20** pointing the unmatched list at a directory holding the content resolves those torrents and shrinks the list, and pointing again at a second directory resolves the remainder.
* **Step 21** the discovery root list is readable through `app/preferences` and writable through `app/setPreferences`, a root added through the web interface appears in the options dialog list with its options intact, and on Windows each path is returned with the separators `save_path` uses while `discovery_roots.json` holds the `Path::data()` form.
* **Step 22** a discovery root outranks a watched folder save path holding the same content, a pointed root outranks a discovery root holding the same content, a torrent added with no watched folder configured resolves against a discovery root, and a discovery root on absent storage leaves other roots resolving normally. Each operation lists each recursive root and the pointed root once, and the log names the exact root that produced each found location.

The remaining Epic 3 integration scenarios in the workplan complete the third iteration's verification.
