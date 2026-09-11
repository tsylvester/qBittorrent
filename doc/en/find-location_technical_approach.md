# Find Location — Technical Approach

How the behaviour in [the feature specification](find-location_feature_spec.md) maps onto the codebase.

## Shape of the change

The change is additive. A multi-root search method joins `FileSearcher` beside `search()`, a session method joins `SessionImpl` beside `findIncompleteFiles()`, and a virtual joins the `BitTorrent::Session` interface. Every signature callers depend on keeps its shape.

`findInDir()`, private to a single translation unit, changes its return type.

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

### Preference

Each preference is a getter and setter pair declared in `src/base/preferences.h` and defined in `src/base/preferences.cpp`, reading through `value()` with a default of `true` and writing through `setValue()` behind an early return when the value is unchanged. `closeSearchTabWithMiddleClick()` is a close model. Keys under the `Downloads/` section suit behaviour concerning content placement.

The group's checked state is a preference of its own, gating the feature ahead of the individual settings. The settings within it keep their stored values while the group is unchecked, so re-checking it restores the configuration the user last chose.

The discovery preference gates the extra candidate roots. Where it is disabled the list holds the save path and download path alone, and the search performs as its two-directory form does. The three assignment preferences arrive with the second iteration and gate the recheck and the two start decisions.

### WebAPI

Keys are added to the map in `src/webui/api/appcontroller.cpp` serving `app/preferences` and `app/setPreferences`, with a patch-level bump of `API_VERSION` in `src/webui/webapplication.h` and an entry in `WebAPI_Changelog.md` naming them. Each iteration carries its own bump and entry for the keys it introduces.

Controls for the same keys go into `src/webui/www/private/views/preferences.html`. A preference reaching the WebAPI without a control there is settable by request and invisible in the interface, which is why the two are added together throughout this repository.

### Options dialog

A checkable `QGroupBox` titled **Find location** in `src/gui/optionsdialog.ui`, placed among the download options and wired in `src/gui/optionsdialog.cpp`, holds one checkbox per setting. The `.ui` file is subject to the grid item ordering pre-commit hook, so the group and its contents are inserted at the correct grid position rather than appended.

The first iteration introduces the group holding **Find location automatically**. The three assignment checkboxes join it in the second, and the discovery root list in the third, at which point the group holds a list as well as checkboxes.

Enablement runs at two levels. Unchecking the group disables the feature and greys its contents. Within the group, **Seed automatically** and **Leech automatically** are enabled only while **Recheck automatically** is checked.

## Second iteration

The manual and batch modes act on torrents held by the session, which do not pass through either resolution point. An existing torrent's location changes through `Torrent::setSavePath()`, preceded by `setAutoTMMEnabled(false)`, which is the sequence `TransferListWidget::setSelectedTorrentsLocation()` performs.

This is why candidate root construction and selection are factored into shared functions in the first iteration.

Reaching them from the GUI needs an addition to the `BitTorrent::Session` interface. `findIncompleteFiles()` is declared on `SessionImpl`, while `src/gui` holds `Session`, which the transfer list obtains through `Session::instance()`. Neither that method nor anything beneath it is reachable from there.

A virtual on `Session` supplies it, backed by a sibling to `findIncompleteFiles()` on `SessionImpl`. It is a `void` operation reporting through a completion signal, in the manner of the signals `Session` declares, so batch mode receives each torrent's result as that torrent completes.

The GUI work sits in `src/gui/transferlistwidget.cpp`. An action is constructed beside `actionSetTorrentPath`, added to the context menu next to it, and connected to a slot mirroring `setSelectedTorrentsLocation()`. A selection producing no match opens the same `QFileDialog` that action opens.

The dialog listing unmatched torrents is a widget class and `.ui` file registered in `src/gui/CMakeLists.txt`.

### Assignment

Assignment adds `forceRecheck()` to the `setAutoTMMEnabled(false)` and `setSavePath()` pair, under the **Recheck automatically** preference.

Starting the torrent waits on the check, so the decision is taken where the check reports rather than at the point of assignment. `TorrentImpl::handleTorrentChecked` is that point. Whether the torrent starts is a question of completeness at that moment, answered against the **Seed automatically** and **Leech automatically** preferences, with the torrents awaiting a decision tracked by info hash so that a check the user triggered by other means is unaffected.

Starting is `Torrent::start()` taking its default `TorrentOperatingMode::AutoManaged`. Forced mode bypasses the queueing system, so a recovered library would start in full; auto-managed leaves `isQueueingSystemEnabled()` and `maxActiveTorrents()` governing it as they govern any other torrent.

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

Probing runs on the session I/O thread, inherited from where `FileSearcher` is constructed. Nothing is added to the GUI thread.

Cost scales as the number of candidate roots multiplied by the number of files in the torrent, in `exists()` calls, and multiplies across every torrent of a migration. No hashing occurs during probing; the hash check runs once, on the winning location, as part of the add that would occur regardless. Issue [#17111](https://github.com/qbittorrent/qBittorrent/issues/17111) records the cost of hashing at this point in the lifecycle.

The product is bounded on the probe. A root holding none of a torrent's content is identified by its first few absent files, so the counting pass abandons a root once it cannot overtake the leading score. Every root is examined.

An enumerated root inverts the cost. One listing serves every torrent of an operation, and each torrent costs a hash lookup against the resulting map, so a root holding many subdirectories is cheaper per torrent than a root holding few.

## Testing

`candidateRoots()` takes every input as a parameter, so a test builds a `PathList` of search roots and asserts on the ordered result with no session, watcher or filesystem involved. Selection is a pure function of probe counts, and the name lookup against an enumerated map is a pure function of that map; both are exercised the same way.

Each test file links `Qt::Test` and `qbt_base`, which bounds automated coverage to `src/base`. Probing and enumeration are exercised against directory fixtures under `test/testdata/`; the resolution call sites, the GUI and a running session are verified by hand.

The test plan is set out step by step in [the dependency map](find-location_dependency_map.md).

## Risks

Pull request [#23578](https://github.com/qbittorrent/qBittorrent/pull/23578) reworks how `actualSavePath` is derived in `addTorrent_impl`, the same function that hosts `resolveFileNames`. The additive shape keeps the two apart: discovery adds methods beside the ones that function calls, so the two changes share no declaration. The derivation of the torrent's own save path is untouched by discovery.
