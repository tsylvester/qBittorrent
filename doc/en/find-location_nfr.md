# Find Location — Non-Functional Requirements

The behaviour is set out in [the feature specification](find-location_feature_spec.md), its implementation in [the technical approach](find-location_technical_approach.md), and its files, tests and verification in [the workplan](find-location_workplan.md). This records the qualities that behaviour holds to.

## Performance

* Discovery performs no hashing. A probe issues existence tests; completed pieces are established by the check a torrent undergoes on being added or by the recheck required after Start-triggered discovery matches existing content.
* A directory that does not exist costs one existence test. A probe abandons a directory once its count can no longer overtake the best directory found so far, and scoring stops once a directory holds every file, so a torrent whose content is complete at its own save path or download path probes no searched location.
* Cost per torrent scales as the directories scored — the torrent's own save path and download path plus the candidate roots — multiplied by declared files, measured in existence tests. A probed save path contributes the root form, the name form and the source form; an enumerated one contributes the root form and, in place of the name form and the source form, the parent and the directory itself for each directory its listing holds under either name. The winning directory is probed once more to produce its file names.
* Recheck throughput is governed by the session's `MaxActiveCheckingTorrents` limit, so a batch queues within the session.
* An enumerated root is walked once for the operation that needs it, never once per torrent. The walk costs one directory listing per directory in the tree, at any depth. The listing is shared by every torrent in a selection, in a **Search folder...** action, and in a burst of automatic additions arriving while an unchanged add operation is in flight. Each torrent costs a lookup against that listing.
* The scale the feature is sized for is a migration: thousands of torrents added in one pass, against a watched folder set in the low tens and an enumerated root that may hold thousands of directories spread across several levels of determinant folders.

## Responsiveness

* Probing runs on the session I/O thread. No filesystem access occurs on the GUI thread.
* Discovery resolves asynchronously within the add path, so the session continues to accept torrents while a probe runs.
* Start-triggered discovery returns control to its caller after recording the pending Start and dispatching the search. Neither filesystem traversal nor waiting for metadata, assignment, storage movement or recheck completion blocks the GUI thread.
* Completion and failure notifications advance a Start transaction asynchronously. The transfer list, context menus and Stop action remain responsive while a transaction is held.
* A batch over a large selection leaves the transfer list interactive throughout, reporting each torrent's outcome as that torrent completes.
* `torrents/findLocation` answers without waiting for any probe. A client learns an operation's outcome by repeating the request, and the web interface does so once a second, so a web session's request thread is never held by a search.

## Data safety

* Discovery reads. It creates, moves, renames and deletes nothing.
* A torrent adopted at a searched location when it is added or its metadata arrives carries no separate incomplete-download location, so its content is not moved after its check.
* Assignment does not overwrite content at the discovered location with data from the torrent's previous save path. Discovery exists to adopt content already on disk, and adopting it must never damage it.
* From interception of a Start request until discovery has completed with a successful miss or the matched location has completed its required recheck successfully, the ordinary Start workflow performs no payload-file creation, allocation, truncation or write and requests no content piece from a peer.
* Metadata acquisition is the only network activity a held torrent may require before that release point. Receiving metadata does not permit payload allocation, writing or piece requests.
* A successful miss and successful completion of the feature-issued recheck are the only outcomes that release a held Start. Cancellation and failure leave it unexecuted.
* A search, assignment, storage-movement or recheck failure leaves the torrent stopped, clears feature-owned transaction state and permits a later retry. A cached or displayed checking state does not establish that a recheck began or succeeded.
* A torrent matching no root is placed at its configured destination.
* A wrong match is recoverable without loss. The recheck reports what is actually present, and **Set location** retargets the torrent.
* `torrents/assignLocation` assigns only a directory that already exists, and creates none.

## Compatibility

* With the group or **Find location automatically** disabled, add and metadata resolution follow the pre-feature two-directory behaviour. With the group or **Find location when starting stopped torrents** disabled, a stopped torrent's Start request is not intercepted.
* A torrent in automatic torrent management resolves at add and at metadata receipt as it does without the feature, whatever the settings.
* Preferences, resume data and save paths written by earlier versions are read unchanged.
* The group and its settings default to enabled, so an upgrade changes placement for torrents added afterwards and intercepts Start for eligible stopped torrents: the most complete copy is selected over a partial copy at a torrent's configured location, and a torrent adopted at a searched location carries no separate incomplete-download location. Disabling the group is one action and restores prior behaviour. The discovery root list is empty by default, so an upgrade adds no search locations of its own.
* Torrents started automatically after an assignment made without a pending Start are auto-managed, so `isQueueingSystemEnabled()` and `maxActiveTorrents()` govern them as they govern any other torrent. A Start transaction preserves the user's normal or forced operating mode; the feature changes neither mode's established queueing semantics and introduces no additional bypass.
* The WebAPI keys and actions are additive: a request omitting a key leaves its setting unchanged, `torrents/findLocation` and `torrents/assignLocation` join the torrent actions without changing any existing action, and `API_VERSION` is set by the maintainers.

## Portability

* Path comparison follows `Path::CASE_SENSITIVITY`, which is `Qt::CaseInsensitive` on Windows and `Qt::CaseSensitive` elsewhere. A probe matches by the platform's rule, so one watched folder set can resolve differently across platforms. Duplicate candidates are collapsed by `Path` equality, so they fold case on Windows.
* An enumerated listing resolves the same names a probe of the directories it holds would resolve, on every platform. Its keys are folded to match `Path::CASE_SENSITIVITY`, since a hash lookup is otherwise case-sensitive everywhere.
* Save paths on network filesystems make existence tests substantially more costly than local ones. Early abandonment bounds that cost for a probed root, and a single walk per operation bounds it for an enumerated one. `TorrentFilesWatcher` already treats network filesystems as a special case where recursion is concerned.
* The WebAPI returns discovery root paths in the platform's native form, as it returns `save_path` and the `scan_dirs` keys, while `discovery_roots.json` holds the `Path::data()` form.

## Security

* Candidate construction drops a name form or source form whose normalised name is empty, absolute, or begins with `.` or `..`, so a name declared by a torrent cannot escape the root it is joined to.
* The source form is derived only from a local `.torrent` file name, never from a magnet URI.
* Discovery reads only beneath the torrent's own save and download paths, the discovery roots the user configured, a root the user pointed at, the watched folder save paths, and the session default save path standing in for a watched folder with no save path configured.
* The walk beneath an enumerated root neither lists nor enters a hidden directory, a symbolic link or a junction, so it stays within the tree the user chose and a link cycle cannot hold it.
* A discovery root or pointed root is trusted as a location because the user chose it. Names declared by a torrent and joined to it are not, and pass the same containment check as every other candidate.
* Names declared by a `.torrent` file are untrusted input and are treated as data throughout.
* Discovery roots received through `app/setPreferences` are parsed with empty, relative and repeated paths dropped.
* A pointed root received through `torrents/findLocation` is used only when it is an existing directory, applies to the operation that request starts, and is written nowhere.
* The web interface writes each discovery root path into its input through the `value` property, never into markup.
* The web interface list of torrents that matched nothing writes each torrent name through `textContent` and each location through the `value` property, never into markup.
* `torrents/findLocation` and `torrents/assignLocation` are accepted by POST alone, as every other state-changing torrent action is, and are subject to the same authentication as every other WebAPI action.

## Observability

* A search won by a searched location records the torrent, the location, the exact root whose candidates contain it and that root's origin, and the count that chose it, through `Logger::addMessage`.
* A search that probed searched locations and matched nothing records that outcome, so a user reporting an unexpected placement has a trail.
* A search won by the torrent's own save path or download path records nothing, so an ordinary add logs as it does without the feature.
* A Start transaction records one final outcome—continued after a match, continued after a miss, cancelled, or failed—and a failure identifies its phase and reported reason.
* Log volume stays proportional to torrents processed rather than to roots probed.

## Accessibility and localisation

* Every user-facing string is translatable through `tr()`.
* The options dialog group and its checkboxes are standard widgets, reachable by keyboard and exposed to screen readers.
* The web interface controls for the same settings are standard form elements on the preferences page, carrying labels that name the setting.
* Every control of the web interface discovery root list is a native element with an accessible name, reached by Tab in row order; **Add...** moves focus to the new row's path, and **Remove** moves focus to **Add...**.
* Web interface labels repeat the desktop source strings and translation contexts, and are translated through the WebUI catalogs separately from the desktop catalogs.
* The list of torrents that matched nothing is navigable by keyboard, in the desktop interface and in the web interface, where the list, the location field, **Search folder...**, **Set location** and **Close** are native elements reached by Tab.

## Maintainability

* Candidate root construction, the enumerated name lookup and the discovery root JSON conversion are pure functions, exercised without a populated disk; probing, selection and enumeration are exercised against fixtures.
* The change is additive: every signature callers depend on keeps its shape, `findInDir()` is unchanged, and the helpers added beside it — `countInDir`, `isContainedName` and `foldedName` — are private to `filesearcher.cpp`.
* Existing session callbacks gain branches that act only when a Find Location entry exists, and no test-only session abstraction is introduced.
