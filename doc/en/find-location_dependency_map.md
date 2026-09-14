# Find Location — Dependency Map

The order the work is built in, from the components that depend on nothing to the surfaces that depend on everything. The behaviour is set out in [the feature specification](find-location_feature_spec.md), its implementation in [the technical approach](find-location_technical_approach.md).

Steps 0, 3, 4, 10 and 16 carry no prerequisite, so work may begin at any of them and a selectable step exists at every point in the walk. Steps 5, 11, 13, 18, 22 and 20 are where independent paths rejoin.

## First iteration

### 0. Behaviour pin — chain A

* **Location** `test/testbittorrentfilesearcher.cpp`, registered in `test/CMakeLists.txt`.
* **Produces** a test fixing the two-directory behaviour of `FileSearcher::search()` as it stands: the save path probed first, the download path second, `forceAppendExt` applied to the destination, and a miss returning the destination.
* **Consumes** nothing.
* **Notes** written against the implementation before any change to it, so it stands as the regression guard for every step that follows.

### 1. Probe primitives — chain A

* **Location** `src/base/bittorrent/filesearcher.cpp`, anonymous namespace.
* **Produces** `findInDir()` returning the number of files found, and a counting variant beside it that tests names without rewriting them.
* **Consumes** step 0, which pins the behaviour this step changes.
* **Visible to** the enclosing translation unit alone.

### 2. Multi-root search — chain A

* **Location** `src/base/bittorrent/filesearcher.h` and its implementation.
* **Produces** a `FileSearcher` method taking an ordered list of roots together with the destination a miss falls back to, and a `search()` that delegates to it from its existing signature. The method records its outcome through `Logger::addMessage`, one message per torrent.
* **Consumes** step 1.
* **Visible to** `SessionImpl`. The call site at `sessionimpl.cpp:3180` is unchanged, so the search is exercised through the existing path before any new caller exists.

### 3. Candidate root construction — chain B

* **Location** `src/base/bittorrent/`, callable from the session and from torrent handling.
* **Produces** `candidateRoots()`, mapping a list of search roots, a torrent name and a source file name to the ordered root list, each search root contributing the root form, the name form and the source form, with the torrent's save and download paths leading it, duplicates collapsed, and the session default substituted for an empty entry. The root list parameter is named for what it carries rather than its origin, so the third iteration extends it without renaming.
* **Consumes** nothing. Every input arrives as a parameter, and the lookups that compose them belong to the callers.
* **Notes** pure with respect to the filesystem and to application state, so its unit tests are written alongside it.

### 4. Discovery preferences — chain C

* **Location** `src/base/preferences.h` and `src/base/preferences.cpp`.
* **Produces** the group's gate preference and **Find location automatically**, each a getter and setter pair defaulting to enabled.
* **Consumes** nothing.
* **Visible to** the options dialog, the WebAPI, and the callers that compose the search root list.

### 5. Session search method — convergence

* **Location** `src/base/bittorrent/sessionimpl.h` and `sessionimpl.cpp`.
* **Produces** a sibling to `findIncompleteFiles()` taking the root list, marshalling onto the I/O thread as that method does, and the composition of that list from the watched folder set, the session default and the settings that gate them.
* **Consumes** steps 2, 3 and 4.

### 6. Resolution at add

* **Location** `SessionImpl::addTorrent_impl`, the `resolveFileNames` lambda.
* **Produces** a save path chosen from the full root list before the torrent reaches libtorrent.
* **Consumes** step 5.
* **Notes** the first point at which the feature is observable. A torrent added from a `.torrent` file whose content sits in a watched folder save path is adopted rather than downloaded.

### 7. Resolution at metadata received

* **Location** `TorrentImpl::handleSaveResumeData`, under `MaintenanceJob::HandleMetadata`.
* **Produces** the same resolution for torrents added from magnet links.
* **Consumes** step 5.
* **Notes** independent of step 6 and buildable either side of it.

### 8. Options dialog group

* **Location** `src/gui/optionsdialog.ui` and `src/gui/optionsdialog.cpp`.
* **Produces** the checkable **Find location** group holding **Find location automatically**.
* **Consumes** step 4.
* **Notes** available from step 4 onward, and meaningful to a user from step 6 onward.

### 9. WebAPI keys

* **Location** `src/webui/api/appcontroller.cpp`, `src/webui/webapplication.h`, `WebAPI_Changelog.md`, `src/webui/www/private/views/preferences.html`.
* **Produces** the two keys on `app/preferences` and `app/setPreferences`, with a patch-level `API_VERSION` bump, a changelog entry, and the controls presenting them in the web interface.
* **Consumes** step 4.

The first iteration is complete and submittable at step 9.

## Second iteration

### 10. Existing-session settings

* **Location** `src/base/bittorrent/session.h`, `sessionimpl.h` and `sessionimpl.cpp`.
* **Produces** **Find location when starting stopped torrents**, **Recheck automatically**, **Seed automatically** and **Leech automatically** as cached BitTorrent session settings. The Start-trigger setting is independent of **Find location automatically**; the three assignment settings retain their existing automatic-assignment meanings.
* **Consumes** nothing.

### 11. Existing-session assignment and pending Start

* **Location** primarily `src/base/bittorrent/sessionimpl.*`, with one conditional delegation in `TorrentImpl::start()` and the minimum private Stop-origin distinction required for cancellation.
* **Produces** the existing-torrent assignment flow and its small feature-owned state: searching, optional metadata wait, assignment through the existing location operations, waiting for the existing move queue, feature-issued recheck, and either automatic assignment completion or release of a preserved normal/forced Start. Repeated Start requests coalesce, and a one-shot pass lets the existing Start body run without recursion.
* **Consumes** steps 5 and 10.
* **Notes** existing `SessionImpl` metadata, info-hash-change, movement, checked, removal and error callbacks advance or clear the feature entry. No new lifecycle notification layer, public Start/Stop signature, move queue, metadata path or `forceRecheck()` implementation is introduced. Disabled and ineligible Start requests execute the existing Start body unchanged. An I/O error clears pending Start and stops the torrent; the next explicit Start reaches the existing error-clearing preamble before a fresh search, while the general force-recheck defect remains separate follow-up work. Steps 6 and 7 are regression baselines for this iteration, not edit points; if their tests expose an add-time contract violation, that finding is replanned explicitly rather than folded into step 11.

### 12. Session discovery interface

* **Location** `src/base/bittorrent/session.h`, implemented in `SessionImpl`.
* **Produces** a `void` virtual running discovery for one torrent and a completion signal carrying its result.
* **Consumes** step 5.
* **Notes** what makes discovery reachable from `src/gui`, which holds `Session` rather than `SessionImpl`.

### 13. Transfer list action

* **Location** `src/gui/transferlistwidget.cpp`.
* **Produces** the **Find location** context menu action over single and multiple selections, assignment through the step 11 session flow, and the fallback to the **Set location** dialog on no match.
* **Consumes** steps 11 and 12.

### 14. Unmatched torrent list

* **Location** `src/gui/`, registered in `src/gui/CMakeLists.txt`.
* **Produces** the widget class and `.ui` file listing torrents that matched nothing, with the walk through them.
* **Consumes** step 13.

### 15. Existing-session settings surfaces

* **Location** `src/gui/optionsdialog.ui`, `src/gui/optionsdialog.cpp`, `src/webui/api/appcontroller.cpp`, `src/webui/webapplication.h`, `WebAPI_Changelog.md`, `src/webui/www/private/views/preferences.html`.
* **Produces** the Start-trigger checkbox and the three assignment checkboxes inside the **Find location** group, their enablement, and four WebAPI keys with a changelog entry and their web interface controls. `API_VERSION` remains for the maintainers to set.
* **Consumes** step 10.

The second iteration is complete and submittable at step 15.

## Third iteration

### 16. Discovery root storage

* **Location** `src/base/`, registered in `src/base/CMakeLists.txt`, with its lifetime owned by `Application` in `src/app/application.cpp`.
* **Produces** the per-root options object and its JSON form, with the recursion flag a key within it, and the `initInstance()` and `freeInstance()` pair `Application` calls alongside `TorrentFilesWatcher`.
* **Consumes** nothing.

### 17. Enumeration

* **Location** `src/base/bittorrent/`, run on the session I/O thread.
* **Produces** a listing of a root's immediate subdirectories as a name-to-path map, keyed with the case folding `Path::CASE_SENSITIVITY` implies, built for one operation and discarded with it.
* **Consumes** step 16.
* **Notes** `Path` cannot key the map, since `qHash` hashes unfolded while `operator==` compares through `Path::CASE_SENSITIVITY`.

### 18. Candidate root extension

* **Location** `src/base/bittorrent/filesearcher.h` and its implementation.
* **Produces** `candidateRoots()` taking the enumerated map, and an enumerated root contributing the root form and the source form, its name form served by a lookup. The order of the search roots is the order they arrive in, which step 22 sets.
* **Consumes** steps 3, 16 and 17.

### 19. Discovery root list

* **Location** `src/gui/optionsdialog.ui`, `src/gui/optionsdialog.cpp`, and a new widget class and `.ui` file registered in `src/gui/CMakeLists.txt`.
* **Produces** the list within the **Find location** group and the per-root dialog carrying the recursion flag, following `watchedfolderoptionsdialog`.
* **Consumes** step 16.

### 20. Pointed root

* **Location** the unmatched torrent list from step 14.
* **Produces** the action taking a directory from the user and running discovery again for the listed torrents with that directory as an enumerated root ahead of every other, repeatable while the list holds entries so a library across several disks is covered one disk at a time.
* **Consumes** steps 12, 14 and 22. It runs from the GUI, so it reaches discovery through the `Session` virtual rather than through `SessionImpl`.

### 21. Discovery root WebAPI keys

* **Location** `src/webui/api/appcontroller.cpp`, `src/webui/webapplication.h`, `WebAPI_Changelog.md`, `src/webui/www/private/views/preferences.html`.
* **Produces** the discovery root list on `app/preferences` and `app/setPreferences`, with an `API_VERSION` bump, a changelog entry, and its web interface control.
* **Consumes** step 16.

### 22. Session root composition

* **Location** `src/base/bittorrent/sessionimpl.h` and `sessionimpl.cpp`, the sibling built at step 5.
* **Produces** the sibling reading the discovery root list, placing those roots ahead of the watched folder save paths within the composed list, supplying the enumeration map for the roots marked recursive, and accepting a pointed root for one operation, placed ahead of every other search root.
* **Consumes** steps 5, 16, 17 and 18.
* **Notes** composition stays in one place, so the add path, the metadata path and the GUI path gain discovery roots together.

The third iteration is complete and submittable at step 22.

## Test plan

Tests build under `-DTESTING=ON` and run through `cmake --build <build> --target check`. CI runs them on Ubuntu, macOS and Windows.

Each test file links `Qt::Test` and `qbt_base`. Coverage therefore reaches `src/base` and stops there: `src/gui` is not linked and no fixture constructs a running session, so the resolution call sites and every GUI step are verified by hand.

### Automated

* **Step 0** `test/testbittorrentfilesearcher.cpp` pins the two-directory behaviour of `FileSearcher::search()` before it is touched. Probe order, `forceAppendExt` reaching the destination, and a miss returning the destination.
* **Step 2** the same file gains the multi-root cases: scoring by count, the earliest root taking a tie, abandonment once a root cannot overtake the leader, and matching through `QB_EXT`. Directory fixtures go under `test/testdata/`, resolved from `__FILE__` as `testutilsio.cpp` resolves its own.
* **Step 3** `test/testbittorrentcandidateroots.cpp` covers the order of the returned list, the root form, name form and source form contributed per search root, substitution of `defaultSavePath` for an empty entry, and duplicate collapse. It passes a `PathList` and asserts on the result.
* **Step 17** the same file covers the name lookup against an enumerated map, which is pure once the map is built, including a name differing from its directory only by case. The listing that builds it is exercised against a fixture tree under `test/testdata/`.
* **Step 18** the same file covers discovery roots taking their place after the torrent's own paths and ahead of the watched folder save paths, and an enumerated root omitting the name form that a probed root contributes.

Both files are added to `testFiles` in `test/CMakeLists.txt`.

### Manual

Verified against a prepared directory tree and described in the pull request:

* **Step 6** a torrent added from a `.torrent` file, whose content sits under a watched folder save path, is adopted rather than downloaded, and issues no content request.
* **Step 7** the same holds for a magnet link once its metadata arrives.
* **Step 8** unchecking the **Find location** group returns torrent add to its two-directory behaviour.
* **Step 9** the two keys appear on `app/preferences`, setting them through `app/setPreferences` is reflected in the options dialog, their controls render on the web interface preferences page and drive the same preferences, and `app/webapiVersion` reports the bumped value.
* **Step 11** with the Start-trigger setting disabled, a stopped torrent follows the existing Start path unchanged. With it enabled, a stopped manual-mode torrent is held before missing-files reload or payload resume; a miss releases the requested normal or forced Start, while an own-path or moved-path match releases it only after the feature-issued recheck completes. Repeat Start to verify coalescing; cancel with Stop; remove and shut down during asynchronous phases; deliver late and unrelated completions; and exercise metadata arrival, movement failure, search failure and recheck I/O error. Confirm that failure never releases Start, a later Start clears the stale error before a fresh transaction, and automatic-mode and checking torrents retain their existing behavior. Separately verify the existing automatic assignment decisions for complete and partial content and the session queueing limits.
* **Step 13** a torrent already in the session is located, rechecked and returned to service, and a torrent matching nothing opens the **Set location** dialog.
* **Step 14** a selection mixing matches and misses assigns the matches and lists the misses. Over a selection large enough to take visible time, outcomes appear as each torrent completes rather than arriving together at the end.
* **Step 15** the Start-trigger key and three assignment keys behave as step 9's do. Desktop and web controls preserve their values while the group is disabled, and the seed/leech controls honour the recheck dependency. The Start-trigger control remains independent of **Find location automatically**.
* **Step 19** a discovery root configured with recursion resolves torrents whose content sits in its subdirectories.
* **Step 21** the discovery root list is readable through `app/preferences` and writable through `app/setPreferences`, a root added through the web interface appears in the options dialog list with its options intact, and `app/webapiVersion` reports the bumped value.
* **Step 22** a discovery root outranks a watched folder save path holding the same content, a pointed root outranks a discovery root holding the same content, and a torrent added with no watched folder configured at all resolves against a discovery root. A discovery root on storage that is absent leaves other roots resolving normally.
* **Step 20** pointing the unmatched list at a directory holding the content resolves those torrents and shrinks the list, and pointing again at a second directory resolves the remainder.
