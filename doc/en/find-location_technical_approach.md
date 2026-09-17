# Find Location — Technical Approach

How the behaviour in [the feature specification](find-location_feature_spec.md) maps onto the codebase. The files, symbols, edits, tests and verification of every step are set out in [the workplan](find-location_workplan.md).

## Shape of the change

The change is additive around a small set of edits to existing code. `FileSearcher::searchRoots()` joins `FileSearcher` beside `search()`, `SessionImpl::findExistingContent()` joins `SessionImpl` beside `findIncompleteFiles()`, and the `BitTorrent::Session` interface gains `setWatchedFolderSavePaths()`, the Find Location setting accessors, `findTorrentLocation()`, `assignTorrentLocation()`, the `torrentLocationFound` signal and `findTorrentLocations()`. Existing callers keep their signatures, and `search()`, `findInDir()` and `findIncompleteFiles()` keep their behaviour.

The first iteration edits four existing places: the `resolveFileNames` lambda in `SessionImpl::addTorrent_impl` and its continuation; the call and continuation in `TorrentImpl::handleSaveResumeData`; and one push call each in `TorrentFilesWatcher::doSetWatchedFolder()` and `TorrentFilesWatcher::removeWatchedFolder()`.

The second iteration adds one conditional delegation to `TorrentImpl::start()`, after its existing error-clearing and operating-mode preamble but before its missing-files reload and resume paths, and a private `TorrentImpl::stop(bool)` overload: public `stop()` calls `stop(true)`, and the stop-after-check call in `TorrentImpl::handleTorrentChecked()` calls `stop(false)`. The existing Start and Stop bodies are not moved or rewritten.

When the Find Location group or **Find location when starting stopped torrents** is disabled, the delegation condition is false and the old Start body executes exactly as before. The new code in existing session callbacks is guarded by an entry in `m_locationAssignments`, so torrents outside the feature retain their current lifecycle.

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

`SessionImpl::addTorrent_impl` populates `lt::add_torrent_params`, then resolves the save path through a `resolveFileNames` lambda before the torrent reaches libtorrent. The lambda runs only when a local `needFindIncompleteFiles` flag is set, which happens when the torrent has metadata and carries no finished status, so a `.torrent` added in seed mode keeps its existing bypass.

The continuation assigns `result.savePath` to `p.save_path` and applies `result.fileNames` to `p.renamed_files`.

Because this runs before libtorrent receives the torrent, a save path chosen here is the path the torrent is added with. libtorrent's initial hash check runs against the discovered content and no content pieces are requested against a wrong location.

### Resolution at metadata received

A torrent added from a magnet link has no file list at add time, so `needFindIncompleteFiles` is not set and the add-time search does not run.

When metadata arrives, `TorrentImpl::handleSaveResumeData` runs under `MaintenanceJob::HandleMetadata`. It derives the file paths from the metadata, applies content layout adjustment through `Path::findRootFolder()`, `Path::stripRootFolder()` and `Path::addRootFolder()`, and resolves the location.

The continuation passes the result to `TorrentImpl::endReceivedMetadataHandling`, which assigns `p.save_path` from the discovered path, applies the renamed files, and calls `reload()`.

Both call sites reach `findExistingContent()` for a torrent in manual mode and `findIncompleteFiles()` for a torrent in automatic mode.

## First iteration

### Candidate roots

`TorrentFilesWatcher::updateSessionWatchedFolderSavePaths()` takes the watched folder paths, sorts them by `Path::data`, and pushes each folder's configured `addTorrentParams.savePath`, empty entries included, through `Session::setWatchedFolderSavePaths()` whenever a watched folder is set, replaced or removed. The sorted order makes ties between watched folders resolve the same way on every run. `SessionImpl` holds the list in `m_watchedFolderSavePaths`, and composition resolves a relative entry against `savePath()`; an empty entry passes through and `candidateRoots()` gives it the session default.

From each search root the construction contributes three candidates: the root form, the root itself; the name form, the root joined with the torrent's name; and the source form, the root joined with the `.torrent` file's name with its extension removed. The torrent's name comes from `TorrentInfo::name()`, except that a single-file torrent without a root folder uses the file's stem, the folder qBittorrent itself nests such a file in under the Subfolder layout. The source file name comes from `TorrentDescriptor::source()` when it names a local `.torrent` file; a magnet URI, including one whose last parameter ends in `.torrent`, and a descriptor loaded from memory contribute none.

The source form is ordered last of the three. It pays only where the directory on disk matches the `.torrent` filename and not the torrent's internal name, which describes a library organised by hand rather than one a client laid out. Its cost is small, since the counting pass abandons a root holding none of the content after a couple of `exists()` calls.

`candidateRoots()` returns candidates alone. A form equal to the torrent's save path or download path is omitted, because `searchRoots()` scores those paths itself. Duplicates are collapsed by `Path` equality at their earliest position, so they fold case where `Path::CASE_SENSITIVITY` is `Qt::CaseInsensitive`. A name form or source form whose name is empty, absolute, or whose first component is `.` or `..` is dropped by `isContainedName()`, so no candidate resolves outside its root.

Construction is a free function declared beside `FileSearcher` in `src/base/bittorrent/filesearcher.h`:

```c++
PathList candidateRoots(const Path &savePath, const Path &downloadPath, const PathList &searchRoots, const Path &defaultSavePath, const QString &torrentName, const QString &sourceFileName);
```

`searchRoots` is named for what it carries rather than where it came from, because the third iteration adds further origins to it.

Every input arrives as a parameter, so a test drives it with literals alone. `SessionImpl::findExistingContent()` composes `searchRoots` from `m_watchedFolderSavePaths` and passes `savePath()` as the default. With **Find location** or **Find location automatically** disabled, `findExistingContent()` returns `findIncompleteFiles()`'s result and composes nothing.

The third iteration places the discovery roots ahead of the watched folder save paths within `searchRoots`, and adds a trailing list of optional enumeration maps aligned with the search roots.

### Multi-root search

`countInDir()`, new in the anonymous namespace beside the unchanged `findInDir()`, returns how many of a torrent's file names are present under a directory, directly or with `QB_EXT` appended. It never modifies the names, and it returns as soon as the running count plus the names remaining cannot exceed a supplied best count.

`FileSearcher` gains:

```c++
void searchRoots(const PathList &originalFileNames, const Path &savePath, const Path &downloadPath, const PathList &candidates, bool forceAppendExt, QPromise<SearchRootsResult> &promise);
```

`search()` and the call to it inside `findIncompleteFiles()` are untouched.

The torrent's own paths are parameters rather than positions in a list, and the destination is derived from them. Candidates are only searched and never receive `forceAppendExt`, so no incomplete-file name is invented anywhere the torrent will not be written.

A root that cannot be read yields a count of zero and is passed over, since discovery must survive a discovery root on storage the user has disconnected.

### Selection

`searchRoots()` scores the save path, then the download path when one is set, then each candidate, in that order. An entry that is empty or does not exist costs one existence test and is skipped; otherwise `countInDir()` counts it against the best count so far. The entry with the highest count wins, the earlier entry wins a tie, and scoring stops once an entry holds every file. The torrent's own paths therefore win a tie and lose to any candidate holding more.

`findInDir()` rewrites the entries it is given, appending `QB_EXT` where the incomplete variant is the file present and, under `forceAppendExt`, where neither is. Scoring many directories with a function that mutates its input would score against names another directory had already altered. `countInDir()` yields a score per directory without touching the names, and `findInDir()` then runs once, on a copy of the names, for the winner: at the save path with `forceAppendExt` applied only when the download path is empty, at the download path with `forceAppendExt`, and at a candidate without it. Where no directory holds any file, `findInDir()` runs as `search()` runs it, so a miss produces `search()`'s result.

`SearchRootsResult` carries the winning `savePath` and its `fileNames`, the `matchCount` that chose it, `searchedCandidates`, true when candidates were reached, and `foundAtOwnPath`, true when the save path or download path won.

The `findExistingContent()` continuation records the outcome through `Logger::addMessage`. A candidate win records the torrent, the location, the watched folder save path whose exact candidate set contains that location, and the count. A search whose candidates were probed and matched nothing records the miss. A win at the torrent's own save path or download path records nothing, so an ordinary add logs as it does without the feature. One message per torrent keeps the log proportional to torrents processed.

### Adoption

In `addTorrent_impl`, a torrent in automatic mode resolves through `findIncompleteFiles()`, and a torrent in manual mode through `findExistingContent()`. When a manual-mode result is neither the torrent's save path nor its download path, the continuation assigns `loadTorrentParams.savePath` the discovered location and empties `loadTorrentParams.downloadPath`. `TorrentImpl` takes `m_savePath` from those parameters, and `adjustStorageLocation()` targets `savePath()` when the download path is empty, so the storage already where libtorrent placed it is not moved after the check, complete or not.

In `handleSaveResumeData`, the same condition clears `m_downloadPath` and calls `setSavePath()` before `endReceivedMetadataHandling()`. With the download path empty and `hasMetadata()` still false, `setSavePath()` reaches `moveStorage()` with `MoveStorageContext::ChangeSavePath`, whose no-metadata branch assigns `m_savePath` with no storage job. `setDownloadPath({})` is not used there, because with an empty path and no metadata it records the previous save path as the download path.

### Settings

Each setting is a `CachedSettingValue<bool>` with a getter and setter pair on `SessionImpl`, stored below a `FindLocation/` BitTorrent session key and exposed through the `Session` interface to desktop and WebUI callers. The setter returns early when the value is unchanged.

The group's checked state is a setting of its own, `FindLocation/Enabled`, gating the feature ahead of the individual settings. The settings within it keep their stored values while the group is unchecked, so re-checking it restores the configuration the user last chose.

**Find location automatically**, `FindLocation/OnAddEnabled`, gates discovery during add and metadata resolution. **Find location when starting stopped torrents** independently gates interception of an explicit Start for an eligible existing torrent. The three automatic-assignment settings arrive with the second iteration and gate recheck and post-check seeding or leeching when there is no explicit pending Start. They do not override explicit Start intent.

### WebAPI

Keys are added to `src/webui/api/appcontroller.cpp` serving `app/preferences` and `app/setPreferences`. Each iteration adds one entry in `WebAPI_Changelog.md`, under the version heading at the top of the file, naming the keys it introduces and linking its pull request. `API_VERSION` and the version headings are set by the maintainers.

Controls for the same keys go into `src/webui/www/private/views/preferences.html`. A preference reaching the WebAPI without a control there is settable by request and invisible in the interface, which is why the two are added together throughout this repository.

### Options dialog

A checkable `QGroupBox` titled **Find location** in `src/gui/optionsdialog.ui`, placed in the Downloads page's `QVBoxLayout` after `checkRecursiveDownload` and before `groupSavingManagement`, and wired in `src/gui/optionsdialog.cpp`, holds one checkbox per setting.

The first iteration introduces the group holding **Find location automatically**. **Find location when starting stopped torrents** and the three assignment checkboxes join it in the second, and the discovery root list in the third, at which point the group holds a list as well as checkboxes.

Enablement runs at two levels. Unchecking the group disables the feature and greys its contents. Within the group, **Seed automatically** and **Leech automatically** are enabled only while **Recheck automatically** is checked.

## Second iteration

The Start-triggered, manual and batch modes act on torrents held by the session, which do not pass through either add-time resolution point. An existing torrent's location changes through `Torrent::setSavePath()`, preceded by `setAutoTMMEnabled(false)`, which is the sequence `TransferListWidget::setSelectedTorrentsLocation()` performs.

This is why candidate root construction and selection are factored into shared functions in the first iteration.

Reaching them from the GUI and the WebAPI needs additions to the `BitTorrent::Session` interface. `findIncompleteFiles()` and `findExistingContent()` are declared on `SessionImpl`, while `src/gui` and `src/webui` hold `Session`, which the transfer list and `TorrentsController` obtain through `Session::instance()`.

`Session::findTorrentLocation()` is a `void` operation reporting through the `torrentLocationFound` signal, carrying one torrent's ID, location and whether content was found, so batch mode receives each torrent's result as that torrent completes. `Session::assignTorrentLocation()` assigns a chosen location. Both compose through `SessionImpl::searchExistingContent()`, which carries the enabled branch of `findExistingContent()` and is the one composition every call site reaches.

The GUI work sits in `src/gui/transferlistwidget.cpp`. A **Find location** action is constructed beside `actionSetTorrentPath` and added directly after it while the selection holds metadata and the group is enabled. Each invocation registers its torrents in one operation map before issuing any search; an invocation made while an operation is active merges only unseen torrents into it. Each match is assigned as its outcome arrives. Once every outcome has arrived, one torrent without a match opens the **Choose save path** dialog that **Set location...** opens, and several open `UnmatchedTorrentsDialog`.

The dialog listing unmatched torrents is a widget class and `.ui` file registered in `src/gui/CMakeLists.txt`.

### WebAPI and web interface discovery

`TorrentsController` carries the manual and batch modes for the web interface and the headless daemon. It is created once per web session and already holds state across requests, as `torrents/fetchMetadata` does in `m_torrentMetadataCache`, filled from `Session::metadataDownloaded` and reported through `APIStatus::Async`, which the web application answers with HTTP 202 until the result is ready.

`torrents/findLocation` follows that pattern. The controller holds one operation map from `TorrentID` to an entry carrying `Pending`, `Matched` with its location, or `Unmatched`, and connects `Session::torrentLocationFound` to a handler that acts only on its own pending entries: a match is recorded and assigned through `Session::assignTorrentLocation()` at once, and a miss is recorded. A request registers every named torrent holding metadata and absent from the map before issuing any search, then calls `Session::findTorrentLocation()` for each, as the transfer list does. It answers with the pending IDs, the matched IDs with their locations in the form `save_path` uses, and the unmatched IDs of torrents still in the session. While any entry is pending the answer carries `APIStatus::Async`; once none is, it is an ordinary result and the map is cleared, ending the operation. A client polls by repeating the request without `hashes`, which registers nothing, so a poll arriving after the operation has ended answers with empty lists rather than starting a new search; naming further torrents in a request merges them into the active operation. Matches are assigned whether or not the client polls, so a script can fire the request and leave. The transfer list and a web session each act only on their own operation, and an assignment of the location a torrent is already being assigned is coalesced by `assignTorrentLocation()`, so two surfaces locating one torrent at once produce one assignment.

`torrents/assignLocation` assigns a named location through `Session::assignTorrentLocation()`. Unlike `torrents/setLocation`, which creates a missing directory and calls `setAutoTMMEnabled(false)` and `setSavePath()` directly, it requires an existing directory and creates nothing, since the desktop dialog it mirrors can only choose one that exists, and it inherits the assignment semantics of the **Find location** action. Both actions are registered in `WebApplication::m_allowedMethod` as POST-only.

The web interface adds **Find location** after **Set location...** in `torrentsTableMenu`, shown by `TorrentsTableContextMenu::updateMenuItems()` while a selected row's `has_metadata` holds and the cached `find_location_enabled` preference is true. Its handler in `mocha-init.js` posts `torrents/findLocation` with the selected torrents holding metadata, then repeats the request without `hashes` each second while the answer is HTTP 202, as `addtorrent.js` polls `torrents/fetchMetadata`. One poll runs per page: a further invocation cancels the scheduled poll, merges its torrents with its own request, and continues from that answer. On HTTP 200, one unmatched torrent opens `setlocation.html` for it with an `assign` parameter that directs the window to `torrents/assignLocation`, and several open `unmatchedtorrents.html`. That window reads the names and save paths of the listed torrents through `torrents/info`, so a torrent removed in the meantime is not listed; it takes a location for the selected entry through the same path field and autofill `setlocation.html` uses, assigns it through `torrents/assignLocation`, drops the entry, and closes when the list is empty or the user closes it. The page is registered in `src/webui/www/webui.qrc`.

### Minimal pending-Start state

`SessionImpl` holds the feature state in `QHash<TorrentID, LocationAssignmentState> m_locationAssignments`, shared by manual assignment and pending Start. An entry holds only what the feature needs:

- `location`, the selected target path, once one exists;
- `phase`, one of `WaitingForMetadata`, `Searching`, `WaitingForMove` and `WaitingForCheck`;
- `startMode`, the optional pending Start's `TorrentOperatingMode`;
- `startPass`, a one-shot permission for an internal use of the existing Start path: `None`, `MetadataOnly`, `Recheck` or `Terminal`; and
- `token`, used by asynchronous search continuations.

Manual assignment and Start-triggered discovery share the waiting-for-move and waiting-for-check phases. A Start-triggered entry additionally uses waiting-for-metadata and searching. Under libtorrent 2, `SessionImpl::handleTorrentInfoHashChanged()` re-keys a waiting entry if magnet metadata changes its ID.

The token is local feature bookkeeping. A continuation captures the torrent ID and token, looks the entry up again on the main thread, and returns without action when the entry was cancelled, removed or replaced. No raw `TorrentImpl *` is retained across an asynchronous search.

### Minimal Start interception

The only Start-path addition is a conditional call after the existing safe preamble clears `hasError()` and records `m_operatingMode`, and before the `m_hasMissingFiles` reload branch or any resume:

```c++
if (m_session->interceptFindLocationStart(this, mode))
    return;
```

`interceptFindLocationStart()` decides in this order. An entry holding a pass other than `None` has that pass consumed, the entry is erased when the pass was `Terminal`, and the call returns `false`; this is the only bypass. An entry already carrying a `startMode` takes the new mode and the call returns `true`, coalescing a repeated Start in every phase. Otherwise the call returns `false` when the master gate or **Find location when starting stopped torrents** is disabled, the torrent is not stopped, it is in automatic mode, or `TorrentImpl::isChecking()` holds. That test reads the native state, which `forceRecheck()` sets to `checking_resume_data` before its internal `start()`, so a recheck is never intercepted. A stopped torrent never yet checked, whose files are already on disk, is in `checking_files` and falls through to libtorrent's own check of those files. An eligible torrent with a manual-assignment entry and no `startMode` has that mode attached and returns `true`. An eligible torrent without an entry gains one, is held by returning `true`, and either begins its search or, lacking metadata, waits for it. The remainder of `TorrentImpl::start()` is left where it is and executes unchanged. This position also provides the bounded workaround for the known force-recheck failure: a later user Start uses the existing preamble to clear the stale native error before Find Location searches and issues a new recheck, without briefly resuming payload transfer.

To release a successful transaction, `SessionImpl` arms `LocationStartPass::Terminal` and calls the existing public `torrent->start(savedMode)`. The conditional call consumes the pass, removes the completed entry, returns `false`, and the existing Start body continues.

`TorrentImpl::forceRecheck()` needs no rewrite. `forceLocationAssignmentRecheck()` sets the entry to waiting-for-check and arms `LocationStartPass::Recheck` before calling `forceRecheck()`, whose internal `start()` consumes that pass. The explicit pass is required because coalescing runs before the checking test, so without it the internal call would be taken for a repeated explicit Start. The existing `StopCondition::FilesChecked` and check-completion path remain responsible for making the torrent stopped again.

A metadata-less stopped torrent arms `LocationStartPass::MetadataOnly`, calls `torrent->start(mode)` to enter the application's existing metadata-only flow, and then sets `StopCondition::MetadataReceived`. Once the existing metadata pipeline calls `SessionImpl::handleTorrentMetadataReceived()`, the same entry begins its search. No new metadata notification or alternate magnet lifecycle is added.

### Existing lifecycle hooks

The feature advances by adding guarded map lookups to hooks that already exist:

1. `SessionImpl::handleTorrentMetadataReceived()` starts discovery only for an entry waiting for metadata. `handleTorrentInfoHashChanged()` preserves that entry across a magnet ID change.
2. The `searchExistingContent()` continuation validates its ID and token. A miss arms a terminal one-shot Start pass. An own-path match skips assignment but still starts the mandatory recheck. A different-path match uses `setAutoTMMEnabled(false)` and `setSavePath()` as manual location assignment does, then calls `setDownloadPath({})` when the torrent is incomplete, has a download path, and its actual storage location is not yet the match, so the matched location becomes the active storage location through the existing move.
3. `SessionImpl::handleTorrentStorageMovingStateChanged()` already receives completion from the existing move queue. A waiting entry advances only after no move is pending and `actualStorageLocation()` equals its target. A failed move reaches the same hook with the current path, so the mismatch clears the feature entry without changing move-queue behaviour.
4. `SessionImpl::handleTorrentChecked()` acts only for an entry placed in waiting-for-check immediately before the feature called `forceRecheck()`. With a pending Start, it releases Start. Without one, it starts the torrent in auto-managed mode under **Seed automatically** for complete content or **Leech automatically** for incomplete content, and otherwise leaves it stopped. Every other check keeps its existing behaviour.
5. The `onFailed` continuation of the search, `handleSaveResumeDataFailedAlert()`, `handleStorageMovedFailedAlert()` and `handleFileErrorAlert()` clear only a matching feature entry after their ordinary logging and torrent error handling. They discard pending Start and stop the torrent. Cached **Checking** never completes the transaction; only the existing successful checked callback does.
6. `removeTorrent()` and session destruction erase pending entries before late continuations can act.

Stop cancellation needs one narrowly bounded distinction because a searching torrent is deliberately still reported as stopped, while successful force recheck also performs an internal stop. Public `TorrentImpl::stop()` calls the private `stop(true)`, which calls `SessionImpl::cancelFindLocationStart()` and then runs the unchanged stop body, so a Stop request clears a pending entry even when the stop body has nothing to change. `TorrentImpl::handleTorrentChecked()` calls `stop(false)`, so the stop-after-check call does not cancel an entry immediately before `SessionImpl::handleTorrentChecked()` releases it. The public `Torrent::stop()` signature is unchanged.

### Force-recheck failure boundary

The feature works around the known defect only within its own operation. If its recheck produces an I/O error before `torrent_checked_alert`, the existing error-alert hook removes the waiting entry, discards pending Start, and calls the ordinary stop path to clear `StopCondition::FilesChecked`. A later explicit Start first executes the existing error-clearing preamble and can create a fresh Find Location transaction. The feature never accepts the cached **Checking** value as completion.

This does not change `forceRecheck()`, libtorrent error recovery, or the behaviour of a force recheck initiated outside Find Location. Repairing the underlying defect remains the separate post-feature task recorded in the workplan.

### Assignment

Assignment adds `forceRecheck()` to the `setAutoTMMEnabled(false)` and `setSavePath()` pair. A Start-triggered match always rechecks because successful verification is its release gate.

Manual assignment through `assignTorrentLocation()` follows **Set location...** semantics. An assignment to the location an entry already holds is coalesced and does nothing further; an assignment to a different location replaces the entry. The torrent is stopped before its entry is inserted. With **Recheck automatically** enabled, the assignment waits for a pending move, and rechecks only once the torrent's actual storage location equals the assigned location; an incomplete torrent with a download path keeps its storage there and receives no recheck.

Starting the torrent waits on the check, so the decision is taken where the check reports rather than at the point of assignment. The existing `SessionImpl::handleTorrentChecked()` callback performs the guarded feature lookup. For an explicit pending Start, the saved normal or forced mode is honored regardless of **Seed automatically** and **Leech automatically**. Without pending Start, completeness at that moment is tested against those two settings. Entries are keyed by `TorrentID` and correlated by phase and operation token so a check triggered by other means is unaffected.

Automatic assignment calls `Torrent::start()` in its default `TorrentOperatingMode::AutoManaged`; a terminal one-shot Start pass lets that call fall through to the existing body once. Forced mode bypasses the queueing system, so only an explicit forced Start may recover in forced mode. Auto-managed leaves `isQueueingSystemEnabled()` and `maxActiveTorrents()` governing it as they govern any other torrent.

Concurrency is bounded by the session. `MaxActiveCheckingTorrents`, defaulting to 1, is passed to libtorrent as `active_checking` in `SessionImpl`, so a batch of rechecks queues rather than contending for the disk. The feature adds no throttling of its own.

## Third iteration

### Discovery root storage

`src/base/discoveryroots.h` and its implementation add `DiscoveryRootOptions`, carrying `recursive`; `DiscoveryRoot`, carrying `path` and its options; and the `DiscoveryRoots` singleton holding the ordered list. The list persists to `discovery_roots.json` as a JSON array of objects carrying `path` and `recursive`. `TorrentFilesWatcher` keys its JSON object by folder path, which `QJsonObject` holds in sorted order; the discovery root list is ordered by the user, so it is held as an array.

`parseDiscoveryRoots()` converts the array into the list, dropping an entry that is not an object, whose path is empty or relative, or whose path repeats an earlier entry; `serializeDiscoveryRoots()` writes one object per root in order. Both read no singleton and write no log, so a `qbt_base` test covers them. `setRoots()` with a list equal to the stored one writes and emits nothing; otherwise it stores the list, writes the file and emits `rootsChanged()`. Loading follows `TorrentFilesWatcher::load()`: an absent file leaves the list empty silently, and a read error, parse error or document that is not an array logs a warning and leaves it empty.

`Application` owns the singleton's lifetime, calling `DiscoveryRoots::initInstance()` before `BitTorrent::Session::initInstance()` and `DiscoveryRoots::freeInstance()` after `BitTorrent::Session::freeInstance()`, because `SessionImpl` can compose a search for any torrent it holds from construction to destruction. Its header and implementation are registered in `src/base/CMakeLists.txt`.

### Enumeration

`enumerateSubdirectories()` walks every directory beneath a root once into `SubdirectoryMap`, a `QHash<QString, PathList>` from subdirectory name to every directory bearing that name, each list ordered by `Path::data()`. Recursion reaches the whole tree, as a watched folder's `recursive` flag does in `TorrentFilesWatcher::Worker::processFolder()`, so a volume structured as `volume/someDeterminant/someOtherDeterminant/contentFolder/content` is reached at the depth its content sits.

The walk is breadth-first over a pending list, each directory listed by a non-recursive `QDirIterator` over `QDir::Dirs | QDir::NoDotAndDotDot | QDir::NoSymLinks`, and an entry for which `QFileInfo::isJunction()` holds is skipped. The walk is explicit rather than `QDirIterator::Subdirectories` so the same tests govern descent as well as listing: a hidden directory, a symbolic link and a junction are neither listed nor entered, and a link pointing back up the tree cannot return the walk to a directory already listed. A directory that cannot be read lists nothing, and its siblings are walked as before.

`candidateRoots()` takes a trailing `QList<std::optional<SubdirectoryMap>>` aligned with the search roots. A search root with a map contributes the root form, then, for each directory the map holds under the torrent's name and then under the source name, that directory's parent followed by the directory itself. The parent is the root-form candidate for that depth, where a torrent whose declared paths begin with its own folder is found; the directory is the name-form candidate for that depth, where a single-file torrent or content nested in a folder of the torrent's name is found. A hit directly beneath the root has the root as its parent, which deduplication collapses into the root form. Where a name repeats in the tree, every directory bearing it contributes, and the count in `searchRoots()` chooses among them, the earlier path taking a tie. A tree holding five hundred content folders at any depth therefore costs one walk and a hash lookup per torrent rather than a probe of every directory.

Keys are folded to a single case where `Path::CASE_SENSITIVITY` is `Qt::CaseInsensitive`, and the lookup folds the same way. Without that, a lookup would resolve strictly where `exists()` on the same platform would not, and the form it replaces would lose coverage. `Path` cannot carry the map's key: `operator==(Path, Path)` compares through `Path::CASE_SENSITIVITY` while `qHash(const Path &, std::size_t)` hashes `key.data()` unfolded, so equality and hashing disagree on Windows.

Enumeration is filesystem I/O and runs on the session I/O thread with the probing. `SessionImpl::SearchOperation` holds one operation's roots with their origins, the composed `searchRoots`, the default save path and the maps. `composeSearchOperation()` builds it on the main thread, placing the pointed root first, then the discovery roots in configured order, then the watched folder save paths. `enumerateSearchOperation()` queues one functor onto the I/O thread that fills the maps for the pointed root and every recursive discovery root, before any search of that operation is queued; each search computes its candidates on the I/O thread once the maps exist. Every torrent of a `findTorrentLocations()` call shares one operation, and an automatic addition reuses an add operation still in flight whose composed roots and default save path are unchanged. The operation and its maps are released when the last continuation holding them ends. An enumerated listing retained across a session is a candidate for a future revision; it carries invalidation against a directory the user can change underneath the application.

### Pointed roots and batch operations

`Session::findTorrentLocations(ids, pointedRoot)` runs one operation over a list of torrents with an optional pointed root, reporting each torrent's result through `torrentLocationFound` as its own search completes. `findTorrentLocation()` keeps its declaration and runs as a one-torrent batch. The transfer list submits each invocation's newly registered torrents through one `findTorrentLocations()` call, and `torrents/findLocation` submits each request's newly registered torrents the same way.

`UnmatchedTorrentsDialog` gains **Search folder...**, taking one directory from the user and running one `findTorrentLocations()` operation over the listed torrents with it as the pointed root. Torrents that match are assigned and leave the list, and the action stays available while the list holds entries, so a library spread across several disks is covered by invoking it once per disk. The pointed root is written nowhere.

`torrents/findLocation` takes an optional `root`. The torrents a request registers are submitted through one `findTorrentLocations(submitted, root)` call, so a request carrying `root` is one pointed operation; a `root` that is not an existing directory answers HTTP 409 before anything is registered. `unmatchedtorrents.html` gains **Search folder...** beside **Set location...**. It posts every listed torrent with the directory in its path field as `root`, polls without `hashes` until HTTP 200, removes each listed torrent the answer reports in `matched`, whose assignment the controller has already made, and keeps the rest. The button is disabled while its poll runs and available again while entries remain, and the window closes once none does.

The continuation names the exact root whose candidate set contains the winning location, rebuilding each root's candidates in order and taking the first set that contains it, and labels it as a pointed root, a discovery root or a watched folder save path by its origin.

### Options dialog list

The discovery root list sits within the **Find location** group after the assignment checkboxes. `DiscoveryRootsModel`, a `QAbstractListModel` following `WatchedFoldersModel`, holds edits until `apply()` writes the list through `DiscoveryRoots::setRoots()` when the page is applied, and reloads on `rootsChanged()`. `DiscoveryRootOptionsDialog`, following `WatchedFolderOptionsDialog`, carries the **Recursive mode** checkbox for one root's options. The model's header and implementation, and the dialog's header, implementation and `.ui` file, are registered in `src/gui/CMakeLists.txt`.

### Web interface

`app/preferences` returns `find_location_discovery_roots` as an array of objects carrying `path` and `recursive`, writing each path with `Path::toString()` as `scan_dirs` writes each watched folder, while `discovery_roots.json` holds the `Path::data()` form. `app/setPreferences` receives the array through `parseDiscoveryRoots()`. The web interface builds each table row with DOM elements, writes the path through the input's `value` property, and reads the rows back in order.

## Threading and performance

Probing runs on the session I/O thread, inherited from where `FileSearcher` is constructed. Start interception only creates or coalesces transaction state and dispatches work; it does not wait for probing, movement or checking. Future continuations return to the `SessionImpl` context, and libtorrent lifecycle alerts already arrive through the session. No filesystem search or synchronous wait is added to the GUI thread, so Stop and other UI work remain responsive while discovery is active.

Cost scales as the directories scored — the torrent's own save path and download path plus the candidates — multiplied by the number of files in the torrent, in `exists()` calls, and multiplies across every torrent of a migration. The winning directory is probed once more to produce its file names. No hashing occurs during probing. Add-time discovery relies on the add lifecycle's check; a Start-triggered match and a rechecked assignment each run one explicit hash check at the winning location. Issue [#17111](https://github.com/qbittorrent/qBittorrent/issues/17111) records the cost of hashing at this point in the lifecycle.

The product is bounded on the probe. A directory holding none of a torrent's content is identified by its first few absent files, so the counting pass abandons a directory once it cannot overtake the leading score, and scoring stops once a directory holds every file.

An enumerated root inverts the cost. One walk, proportional to the number of directories in the tree, serves every torrent of an operation, and each torrent costs a hash lookup against the resulting map and two candidates per directory bearing its name, so a root holding many directories is cheaper per torrent than a root holding few.

## Testing

`candidateRoots()` takes every input as a parameter, so `testbittorrentcandidateroots` builds a `PathList` of search roots and asserts on the ordered result with no session, watcher or filesystem involved; the name lookup against an enumerated map is exercised the same way. `testdiscoveryroots` covers the discovery root JSON conversion without a filesystem.

Each test file links `Qt::Test` and `qbt_base`. `testbittorrentfilesearcher` pins `search()` before any change; `testbittorrentfilesearchermultiroot` covers probing and selection in `searchRoots()`; and `testbittorrentsubdirectories` covers enumeration, each against directory fixtures under `test/testdata/filesearcher`. `SessionImpl` is not made constructible through a new test-only abstraction for this feature; doing so would exceed the submission's scope.

The resolution call sites and every GUI and web interface surface are verified by the manual cases in the workplan. Nineteen Epic 2 integration scenarios cover the Epic 2 behaviour at the public boundary. Seventeen independently cover the TX requirements and the desktop surfaces: disabled and ineligible fall-through, normal and forced Start, metadata-only acquisition, own-path and moved-path matches, miss, repeated Start, Stop, removal, shutdown, late completion, unrelated check completion, and each failure path. They verify that no payload file is created, allocated, truncated or written before a miss or successful recheck, that no peer payload connection begins while held, that the force-recheck workaround clears only feature-owned state, and that the GUI remains responsive. Two cover the WebAPI actions and the web interface action and list. Ten Epic 3 integration scenarios cover batch enumeration, overlapping origins, persistence, absent and malformed files, unchanged writes, path forms, the web interface, automatic addition bursts, structured volumes with repeated names and links, and the pointed root through the WebAPI and the web interface.

## Risks

Pull request [#23578](https://github.com/qbittorrent/qBittorrent/pull/23578) reworks how `actualSavePath` is derived in `addTorrent_impl`, the same function that hosts `resolveFileNames`. The first iteration edits the `resolveFileNames` lambda and its continuation in that function, so the two changes overlap in its body and one rebases onto the other. The derivation of the torrent's own save path is untouched by discovery.

`searchRoots()` probes the winning directory twice, once through `countInDir()` and once through the unchanged `findInDir()`, so `findInDir()` stays untouched. The first iteration's pull request offers a single pass that records the winner's names while counting as an alternative for the maintainers to choose.

The review boundary is intentionally narrow. The feature changes no public Start or Stop signature, move queue, metadata handling or `forceRecheck()` implementation, and repairs no general force-recheck behaviour. It adds feature state in `SessionImpl`, one conditional Start delegation, the private `stop(bool)` overload required for cancellation, and guarded branches in existing callbacks. Those branches are unreachable when no Find Location entry exists.
