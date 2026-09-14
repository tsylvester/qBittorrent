# Find Location — Technical Approach

How the behaviour in [the feature specification](find-location_feature_spec.md) maps onto the codebase.

## Shape of the change

The change is additive except for two narrow implementation edits. A multi-root search method joins `FileSearcher` beside `search()`, a session method joins `SessionImpl` beside `findIncompleteFiles()`, and a virtual joins the `BitTorrent::Session` interface. Existing callers keep their signatures.

`findInDir()`, private to a single translation unit, changes its return type. `TorrentImpl::start()` gains one conditional delegation after its existing error-clearing and operating-mode preamble, but before its missing-files reload and resume paths. The existing Start body is not moved or rewritten. A small Stop-request discriminator is the only other torrent-internal touch required for cancellation; the existing stop-after-check behavior remains unchanged.

When the Find Location group or **Find location when starting stopped torrents** is disabled, the delegation condition is false and the old Start body executes exactly as before. The new code in existing session callbacks is likewise guarded by an entry in the feature's location-assignment map, so torrents outside the feature retain their current lifecycle.

## Components

### FileSearcher

`FileSearcher`, in `src/base/bittorrent/filesearcher.h` and its implementation, probes a directory for a torrent's files:

```c++
void search(const PathList &originalFileNames, const Path &savePath, const Path &downloadPath, bool forceAppendExt, QPromise<FileSearchResult> &promise);
```

Its helper `findInDir()` sits in an anonymous namespace in `filesearcher.cpp`, private to that translation unit. It walks the expected file names against one directory and reports whether any were found. Where a name is absent it also tries that name with `QB_EXT` — the `.!qB` incomplete-file suffix defined in `src/base/bittorrent/common.h` — and rewrites the entry in the returned list when the incomplete variant is the one on disk. This is what makes partially downloaded content recoverable.

The two directories are not treated alike. `forceAppendExt` is passed to whichever of them is the torrent's destination, the download path where one is set and the save path otherwise, because that is where incomplete files are written. A search that finds nothing returns that same destination, so a miss places the torrent where it was configured to go rather than where it was last looked for.

The component lives on the session I/O thread, constructed in `SessionImpl` and moved to `m_ioThread`. It is never called from the GUI thread. `SessionImpl::findIncompleteFiles()` marshals calls onto that thread through `QMetaObject::invokeMethod` and returns a `QFuture<FileSearchResult>`.

`FileSearchResult` carries the chosen `savePath` and the adjusted `fileNames`.

### Resolution at add

`SessionImpl::addTorrent_impl` populates `lt::add_torrent_params`, then resolves the save path through a `resolveFileNames` lambda before the torrent reaches libtorrent. The lambda calls `findIncompleteFiles()` with the torrent's save path and download path, guarded by a local `needFindIncompleteFiles` flag that is set when the torrent has metadata and carries no finished status.

The continuation assigns `result.savePath` to `p.save_path` and applies `result.fileNames` to `p.renamed_files`.

Because this runs before libtorrent receives the torrent, a save path chosen here is the path the torrent is added with. libtorrent's initial hash check runs against the discovered content and no content pieces are requested against a wrong location.

### Resolution at metadata received

A torrent added from a magnet link has no file list at add time, so `needFindIncompleteFiles` is not set and the add-time search does not run.

When metadata arrives, `TorrentImpl::handleSaveResumeData` runs under `MaintenanceJob::HandleMetadata`. It derives the file paths from the metadata, applies content layout adjustment through `Path::findRootFolder()`, `Path::stripRootFolder()` and `Path::addRootFolder()`, and calls `m_session->findIncompleteFiles(savePath(), downloadPath(), filePaths)`.

The continuation passes the result to `TorrentImpl::endReceivedMetadataHandling`, which assigns `p.save_path` from the discovered path, applies the renamed files, and calls `reload()`.

Magnet links therefore reach discovery through this second call site, and both call sites require the candidate list.

## First iteration

### Candidate roots

`TorrentFilesWatcher::instance()->folders()` returns `QHash<Path, WatchedFolderOptions>`. Each `WatchedFolderOptions` holds an `addTorrentParams` whose `savePath` member is the save path for torrents found in that folder, and which is empty where the folder inherits the session default from `Session::savePath()`. A caller extracts those save paths and passes them down.

From each of those save paths the construction contributes three candidates: the root form, the path itself; the name form, the path joined with the torrent's name; and the source form, the path joined with the basename of the `.torrent` file. The torrent's name comes from `TorrentInfo::name()` where metadata is available, and the source file name is known to `AddTorrentManager` through the source string it tracks per info hash.

The source form is ordered last of the three. It pays only where the directory on disk matches the `.torrent` filename and not the torrent's internal name, which describes a library organised by hand rather than one a client laid out. Its cost is small, since the counting pass abandons a root holding none of the content after a couple of `exists()` calls, and its position denies it any chance of outranking a better match.

The torrent's own save path and download path lead the list. Those two are destinations as well as search locations; the roots derived from watched folders are search locations only. A search that matches nothing returns the destination, never a watched folder root that happened to be probed last.

Roots are deduplicated, since watched folders commonly share a save path.

Construction is a free function declared beside `FileSearcher` in `src/base/bittorrent/filesearcher.h`:

```c++
PathList candidateRoots(const Path &savePath, const Path &downloadPath, const PathList &searchRoots, const Path &defaultSavePath, const QString &torrentName, const QString &sourceFileName);
```

`searchRoots` is named for what it carries rather than where it came from, because the third iteration adds a second origin to it.

Every input arrives as a parameter. The lookups producing them — `TorrentFilesWatcher::instance()->folders()`, `Session::savePath()`, and the settings gating them — belong to the `SessionImpl` sibling below, which composes `searchRoots` and passes it down. `SessionImpl::addTorrent_impl`, `TorrentImpl::handleSaveResumeData` and the on-demand caller the second iteration introduces all reach discovery through that sibling, so composition has one implementation rather than one per call site. An empty entry in `searchRoots` takes `defaultSavePath` in its place, and the returned list is ordered and deduplicated.

Where discovery is disabled the sibling composes `searchRoots` empty, and `candidateRoots()` returns the torrent's save path and download path alone. Reading a preference inside the function would put a singleton behind it and cost the property that makes it testable.

The third iteration places the discovery roots ahead of the watched folder save paths within `searchRoots`, and adds a parameter carrying the enumerated name-to-path map.

### Multi-root search

`findInDir()` returns the number of files found in place of a boolean, and gains a counting variant beside it that tests names without rewriting them. Both are private to `filesearcher.cpp`.

`FileSearcher` gains a method taking an ordered list of roots together with the destination a miss falls back to. `search()` retains its signature and its behaviour, delegating to that method with a list built from its save path and download path, so the call site at `sessionimpl.cpp:3180` needs no change.

The destination is a parameter of the new method rather than a position in the list, which is what carries `forceAppendExt` to it. Roots that are only searched never receive the flag, so no incomplete-file name is invented anywhere the torrent will not be written.

A root that cannot be read yields a count of zero and is passed over, since discovery must survive a discovery root on storage the user has disconnected.

Matching is what `findInDir()` performs: a file is present when it exists under its declared name, or under that name with `QB_EXT` appended.

### Selection

Scoring across many roots uses the count `findInDir()` returns, with the caller taking the highest count and the earliest root on a tie.

`findInDir()` rewrites the entries it is given, appending `QB_EXT` where the incomplete variant is the file present and, under `forceAppendExt`, where neither is. Scoring many roots with a function that mutates its input means scoring against names another root has already altered.

The counting variant resolves it. It yields a score per root without touching the names, and `findInDir()` then runs once, on the winner, to produce the names the result carries. The counting pass is also free to stop early, since a score it cannot bring above the leader decides nothing.

The adjusted `fileNames` list belongs to whichever root wins, so the search retains the names produced by the winning probe rather than the last one attempted.

The search records its outcome through `Logger::addMessage`, naming the torrent, the winning root, the origin that root came from, and the count that chose it. A search matching nothing records that. One message per torrent keeps the log proportional to torrents processed.

### Settings

Each setting is a cached value and getter/setter pair on `SessionImpl`, stored below a `FindLocation/` BitTorrent session key. The setter returns early when the value is unchanged. This follows the ownership of the existing Find Location implementation: the session consumes the settings and exposes them through the `Session` interface to desktop and WebUI callers.

The group's checked state is a setting of its own, gating the feature ahead of the individual settings. The settings within it keep their stored values while the group is unchecked, so re-checking it restores the configuration the user last chose.

**Find location automatically** gates discovery during add and metadata resolution. **Find location when starting stopped torrents** independently gates interception of an explicit Start for an eligible existing torrent. The three automatic-assignment settings arrive with the second iteration and gate recheck and post-check seeding or leeching when there is no explicit pending Start. They do not override explicit Start intent.

### WebAPI

Keys are added to the map in `src/webui/api/appcontroller.cpp` serving `app/preferences` and `app/setPreferences`, with a patch-level bump of `API_VERSION` in `src/webui/webapplication.h` and an entry in `WebAPI_Changelog.md` naming them. Each iteration carries its own bump and entry for the keys it introduces.

Controls for the same keys go into `src/webui/www/private/views/preferences.html`. A preference reaching the WebAPI without a control there is settable by request and invisible in the interface, which is why the two are added together throughout this repository.

### Options dialog

A checkable `QGroupBox` titled **Find location** in `src/gui/optionsdialog.ui`, placed among the download options and wired in `src/gui/optionsdialog.cpp`, holds one checkbox per setting. The `.ui` file is subject to the grid item ordering pre-commit hook, so the group and its contents are inserted at the correct grid position rather than appended.

The first iteration introduces the group holding **Find location automatically**. **Find location when starting stopped torrents** and the three assignment checkboxes join it in the second, and the discovery root list in the third, at which point the group holds a list as well as checkboxes.

Enablement runs at two levels. Unchecking the group disables the feature and greys its contents. Within the group, **Seed automatically** and **Leech automatically** are enabled only while **Recheck automatically** is checked.

## Second iteration

The Start-triggered, manual and batch modes act on torrents held by the session, which do not pass through either add-time resolution point. An existing torrent's location changes through `Torrent::setSavePath()`, preceded by `setAutoTMMEnabled(false)`, which is the sequence `TransferListWidget::setSelectedTorrentsLocation()` performs.

This is why candidate root construction and selection are factored into shared functions in the first iteration.

Reaching them from the GUI needs an addition to the `BitTorrent::Session` interface. `findIncompleteFiles()` is declared on `SessionImpl`, while `src/gui` holds `Session`, which the transfer list obtains through `Session::instance()`. Neither that method nor anything beneath it is reachable from there.

A virtual on `Session` supplies it, backed by a sibling to `findIncompleteFiles()` on `SessionImpl`. It is a `void` operation reporting through a completion signal, in the manner of the signals `Session` declares, so batch mode receives each torrent's result as that torrent completes.

The GUI work sits in `src/gui/transferlistwidget.cpp`. An action is constructed beside `actionSetTorrentPath`, added to the context menu next to it, and connected to a slot mirroring `setSelectedTorrentsLocation()`. A selection producing no match opens the same `QFileDialog` that action opens.

The dialog listing unmatched torrents is a widget class and `.ui` file registered in `src/gui/CMakeLists.txt`.

### Minimal pending-Start state

`SessionImpl` extends the location-assignment map already planned for manual assignment. It does not introduce a general lifecycle coordinator. An entry is keyed by `TorrentID` and contains only what the feature needs:

- the current Find Location phase;
- the selected target path, once one exists;
- optional pending Start intent and its `TorrentOperatingMode`;
- a one-shot permission for an internal use of the existing Start path, marked terminal or metadata-only; and
- an operation token used by asynchronous search continuations.

Manual assignment and Start-triggered discovery share the existing waiting-for-move and waiting-for-check phases. A Start-triggered entry additionally uses waiting-for-metadata and searching. Under libtorrent 2, the existing `SessionImpl::handleTorrentInfoHashChanged()` re-keys a waiting entry if magnet metadata changes its ID.

The operation token is local feature bookkeeping, not a new application identity or event system. A continuation captures the torrent ID and token, looks the entry up again on the main thread, and returns without action when the entry was cancelled, removed or replaced. No raw `TorrentImpl *` is retained across an asynchronous search.

### Minimal Start interception

The only Start-path addition is a conditional call after the existing safe preamble clears `hasError()` and records `m_operatingMode`, and before the `m_hasMissingFiles` reload branch or any resume:

```c++
if (m_session->interceptFindLocationStart(this, mode))
    return;
```

The method returns `false` immediately when the master feature gate or **Find location when starting stopped torrents** is disabled, when the torrent is checking, or when it is not an eligible stopped manual-mode torrent. The checking exclusion lets every existing force-recheck path retain its internal use of `start()`. The remainder of `TorrentImpl::start()` is left where it is and executes unchanged. This position also provides the bounded workaround for the known force-recheck failure: a later user Start uses the existing preamble to clear the stale native error before Find Location searches and issues a new recheck, without briefly resuming payload transfer.

An active entry coalesces another Start and updates the saved normal or forced mode without issuing duplicate work. To release a successful transaction, `SessionImpl` arms a terminal one-shot Start pass and calls the existing public `torrent->start(savedMode)`. The same conditional call consumes the pass, removes the completed entry, returns `false`, and the existing Start body continues. For metadata acquisition the pass is non-terminal: consuming it leaves the entry waiting for metadata. This avoids a new internal Start API, friendship, or movement of the current implementation.

`TorrentImpl::forceRecheck()` also needs no rewrite. It sets the cached state to checking before its existing internal call to `start()`, so a checking torrent is excluded from Start-trigger eligibility. For the feature-issued recheck, the waiting-for-check entry is installed before `forceRecheck()` is called. The existing `StopCondition::FilesChecked` and check-completion path remain responsible for making the torrent stopped again.

A metadata-less stopped torrent uses the same one-shot mechanism to enter the application's existing metadata-only flow, followed by `StopCondition::MetadataReceived`. Once the existing metadata pipeline calls `SessionImpl::handleTorrentMetadataReceived()`, the same entry begins its search. No new metadata notification or alternate magnet lifecycle is added.

### Existing lifecycle hooks

The feature advances by adding guarded map lookups to hooks that already exist:

1. `SessionImpl::handleTorrentMetadataReceived()` starts discovery only for an entry waiting for metadata. `handleTorrentInfoHashChanged()` preserves that entry across a magnet ID change.
2. The existing `searchExistingContent()` continuation validates its ID and token. A miss arms a terminal one-shot Start pass. An own-path match skips assignment but still starts the mandatory recheck. A different-path match uses `setAutoTMMEnabled(false)` and `setSavePath()` exactly as manual location assignment already does.
3. `SessionImpl::handleTorrentStorageMovingStateChanged()` already receives completion from the existing move queue. A waiting entry advances only after no move is pending and `actualStorageLocation()` equals its target. A failed move reaches the same hook with the current path, so the mismatch clears the feature entry without changing move-queue behavior.
4. `SessionImpl::handleTorrentChecked()` releases Start only for an entry placed in waiting-for-check immediately before the feature called `forceRecheck()`. Every other check keeps its existing behavior.
5. Existing search failure handling, `handleSaveResumeDataFailedAlert()`, `handleStorageMovedFailedAlert()` and `handleFileErrorAlert()` clear only a matching feature entry after their ordinary logging and torrent error handling. They discard pending Start and stop the torrent. Cached **Checking** never completes the transaction; only the existing successful checked callback does.
6. `removeTorrent()` and session destruction erase pending entries before late continuations can act.

Stop cancellation needs one narrowly bounded distinction because a searching torrent is deliberately still reported as stopped, while successful force recheck also performs an internal stop. The public Stop request must clear a pending entry even when the normal stop body has nothing to change, but the existing stop-after-check call must not cancel immediately before `handleTorrentChecked()` releases it. Implement this as a private request-origin discriminator local to `TorrentImpl`; do not change the public `Torrent::stop()` signature or general stop semantics. The old stop body remains unchanged and is used in both cases.

### Force-recheck failure boundary

The feature works around the known defect only within its own operation. If its recheck produces an I/O error before `torrent_checked_alert`, the existing error-alert hook removes the waiting entry, discards pending Start, and calls the ordinary stop path to clear `StopCondition::FilesChecked`. A later explicit Start first executes the existing error-clearing preamble and can create a fresh Find Location transaction. The feature never accepts the cached **Checking** value as completion.

This does not change `forceRecheck()`, libtorrent error recovery, or the behavior of a force recheck initiated outside Find Location. Repairing the underlying defect remains the separate post-feature task recorded in the workplan.

### Assignment

Assignment adds `forceRecheck()` to the `setAutoTMMEnabled(false)` and `setSavePath()` pair. A Start-triggered match always rechecks because successful verification is its release gate. For manual assignment without an explicit pending Start, **Recheck automatically** determines whether the assignment enters the waiting-for-check phase.

Starting the torrent waits on the check, so the decision is taken where the check reports rather than at the point of assignment. The existing `SessionImpl::handleTorrentChecked()` callback performs the guarded feature lookup. For an explicit pending Start, the saved normal or forced mode is honored regardless of **Seed automatically** and **Leech automatically**. Without pending Start, completeness at that moment is tested against those two settings. Entries are keyed by `TorrentID` and correlated by phase and operation token so a check triggered by other means is unaffected.

Automatic assignment calls `Torrent::start()` in its default `TorrentOperatingMode::AutoManaged`; a terminal one-shot Start pass lets that call fall through to the existing body once. Forced mode bypasses the queueing system, so only an explicit forced Start may recover in forced mode. Auto-managed leaves `isQueueingSystemEnabled()` and `maxActiveTorrents()` governing it as they govern any other torrent.

Concurrency is bounded by the session. `MaxActiveCheckingTorrents`, defaulting to 1, is passed to libtorrent as `active_checking` in `SessionImpl`, so a batch of rechecks queues rather than contending for the disk. The feature adds no throttling of its own.

## Third iteration

### Discovery root storage

Discovery roots carry per-root options, so they persist as `TorrentFilesWatcher` persists watched folders: a JSON object per root, with the recursion flag a key within it, following `OPTION_RECURSIVE` in `torrentfileswatcher.cpp`. A flat path list with a separate flag would diverge from the pattern the application already has for a rooted option.

Storage is a singleton on the same terms, so `Application` owns its lifetime, calling `initInstance()` where it calls `TorrentFilesWatcher::initInstance()` and `freeInstance()` where it frees that one. Its header and implementation are registered in `src/base/CMakeLists.txt`.

### Enumeration

A root marked recursive is listed once, and the listing becomes a `QHash<QString, Path>` from subdirectory name to path. Candidate root construction takes that map and looks the torrent's name up in it, so a directory holding five hundred subdirectories costs one listing and a hash lookup per torrent rather than fifteen hundred candidate forms.

The lookup replaces the name form, so an enumerated root contributes the root form and the source form, with the name-matched directory supplied by the map.

Keys are folded to a single case where `Path::CASE_SENSITIVITY` is `Qt::CaseInsensitive`, and the lookup folds the same way. Without that, a lookup would resolve strictly where `exists()` on the same platform would not, and the form it replaces would lose coverage. `Path` cannot carry the map's key: `operator==(Path, Path)` compares through `Path::CASE_SENSITIVITY` while `qHash(const Path &, std::size_t)` hashes `key.data()` unfolded, so equality and hashing disagree on Windows.

The map is built for the operation that needs it and discarded when that operation ends. Persisting it across a session would spare the listing on every add, and is a candidate for a future revision; it carries invalidation against a directory the user can change underneath the application, which this iteration does not take on.

Enumeration is filesystem I/O and runs on the session I/O thread with the probing.

### Pointed roots

The dialog listing torrents that matched nothing gains an action taking a directory from the user and running discovery again for those torrents, with that directory as an enumerated root ahead of every other. Torrents that match are assigned and leave the list.

The action stays available while the list holds entries, so a library spread across several disks is covered by invoking it once per disk. A single directory per invocation keeps `QFileDialog` in its directory mode, which selects one at a time, and keeps a set-building widget out of the dialog.

### Options dialog list

The discovery root list is a list backed by a model within the **Find location** group, with a per-root dialog carrying the recursion flag. `watchedfolderoptionsdialog.h`, its `.cpp` and its `.ui` are the model to follow, down to the `QCheckBox` labelled for recursive mode. The new widget class and `.ui` file are registered in `src/gui/CMakeLists.txt`.

## Threading and performance

Probing runs on the session I/O thread, inherited from where `FileSearcher` is constructed. Start interception only creates or coalesces transaction state and dispatches work; it does not wait for probing, movement or checking. Future continuations return to the `SessionImpl` context, and libtorrent lifecycle alerts already arrive through the session. No filesystem search or synchronous wait is added to the GUI thread, so Stop and other UI work remain responsive while discovery is active.

Cost scales as the number of candidate roots multiplied by the number of files in the torrent, in `exists()` calls, and multiplies across every torrent of a migration. No hashing occurs during probing. Add-time discovery relies on the add lifecycle's check; a Start-triggered match and a rechecked assignment each run one explicit hash check at the winning location. Issue [#17111](https://github.com/qbittorrent/qBittorrent/issues/17111) records the cost of hashing at this point in the lifecycle.

The product is bounded on the probe. A root holding none of a torrent's content is identified by its first few absent files, so the counting pass abandons a root once it cannot overtake the leading score. Every root is examined.

An enumerated root inverts the cost. One listing serves every torrent of an operation, and each torrent costs a hash lookup against the resulting map, so a root holding many subdirectories is cheaper per torrent than a root holding few.

## Testing

`candidateRoots()` takes every input as a parameter, so a test builds a `PathList` of search roots and asserts on the ordered result with no session, watcher or filesystem involved. Selection is a pure function of probe counts, and the name lookup against an enumerated map is a pure function of that map; both are exercised the same way.

Each test file links `Qt::Test` and `qbt_base`. Probing and enumeration are exercised against directory fixtures under `test/testdata/`. `SessionImpl` is not made constructible through a new test-only abstraction for this feature; doing so would exceed the submission's scope.

Epic 2 integration scenarios independently cover the TX requirements at the public behavior boundary: disabled and ineligible fall-through, normal and forced Start, metadata-only acquisition, own-path and moved-path matches, miss, repeated Start, Stop, removal, shutdown, late completion, unrelated check completion, and each failure path. They verify that no payload file is created, allocated, truncated or written before a miss or successful recheck, that no peer payload connection begins while held, that the force-recheck workaround clears only feature-owned state, and that the GUI remains responsive.

The test plan is set out step by step in [the dependency map](find-location_dependency_map.md).

## Risks

Pull request [#23578](https://github.com/qbittorrent/qBittorrent/pull/23578) reworks how `actualSavePath` is derived in `addTorrent_impl`, the same function that hosts `resolveFileNames`. The additive shape keeps the two apart: discovery adds methods beside the ones that function calls, so the two changes share no declaration. The derivation of the torrent's own save path is untouched by discovery.

The review boundary is intentionally narrow. This iteration does not introduce a generic torrent-operation state machine, change public Start or Stop signatures, replace the move queue, alter metadata handling, modify `forceRecheck()`, or repair its general failure behavior. It adds feature state in `SessionImpl`, one conditional Start delegation, the minimum Stop-origin distinction required for cancellation, and guarded branches in existing callbacks. Those branches are unreachable when no Find Location entry exists.
