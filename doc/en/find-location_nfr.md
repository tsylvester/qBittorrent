# Find Location — Non-Functional Requirements

The behaviour is set out in [the feature specification](find-location_feature_spec.md) and its implementation in [the technical approach](find-location_technical_approach.md). This records the qualities that behaviour holds to.

## Performance

* Discovery performs no hashing. A probe issues existence tests, and the hash check that establishes completed pieces is the one the torrent undergoes on being added.
* A probe abandons a candidate root once its count can no longer overtake the best root found so far.
* Cost per torrent scales as candidate roots multiplied by declared files, measured in existence tests. A probed save path contributes the root form, the name form and the source form; an enumerated one contributes the root form and the source form, with a lookup in place of the name form.
* Recheck throughput is governed by the session's `MaxActiveCheckingTorrents` limit, so a batch queues within the session.
* An enumerated root is listed once for the operation that needs it, never once per torrent. Each torrent costs a lookup against that listing.
* The scale the feature is sized for is a migration: thousands of torrents added in one pass, against a watched folder set in the low tens and an enumerated root that may hold thousands of subdirectories.

## Responsiveness

* Probing runs on the session I/O thread. No filesystem access occurs on the GUI thread.
* Discovery resolves asynchronously within the add path, so the session continues to accept torrents while a probe runs.
* A batch over a large selection leaves the transfer list interactive throughout, reporting each torrent's outcome as that torrent completes.

## Data safety

* Discovery reads. It creates, moves, renames and deletes nothing.
* Assignment does not overwrite content at the discovered location with data from the torrent's previous save path. Discovery exists to adopt content already on disk, and adopting it must never damage it.
* A torrent matching no root is placed at its configured destination.
* A wrong match is recoverable without loss. The recheck reports what is actually present, and **Set location** retargets the torrent.

## Compatibility

* With the group disabled, torrent add behaves as it does without the feature: the candidate list holds the save path and download path, and the search performs as its two-directory form does.
* Preferences, resume data and save paths written by earlier versions are read unchanged.
* The group and its settings default to enabled, so an upgrade changes placement for torrents added afterwards. Disabling the group is one action and restores prior behaviour. The discovery root list is empty by default, so an upgrade adds no search locations of its own.
* Torrents started after assignment are auto-managed, so `isQueueingSystemEnabled()` and `maxActiveTorrents()` govern them as they govern any other torrent. The feature does not start torrents in forced mode and does not bypass the queue.

## Portability

* Path comparison follows `Path::CASE_SENSITIVITY`, which is `Qt::CaseInsensitive` on Windows and `Qt::CaseSensitive` elsewhere. A probe matches by the platform's rule, so one watched folder set can resolve differently across platforms.
* An enumerated listing resolves the same names a probe of that root would resolve, on every platform. Its keys are folded to match `Path::CASE_SENSITIVITY`, since a hash lookup is otherwise case-sensitive everywhere.
* Save paths on network filesystems make existence tests substantially more costly than local ones. Early abandonment bounds that cost for a probed root, and a single listing bounds it for an enumerated one. `TorrentFilesWatcher` already treats network filesystems as a special case where recursion is concerned.
* The feature carries no platform-specific behaviour of its own beyond the case sensitivity above.

## Security

* Candidate roots are `Path` values. `Path::isValid()` rejects any component equal to `.` or `..` and validates each remaining component, so a name declared by a torrent cannot escape the root it is joined to.
* Discovery reads only beneath the torrent's own save and download paths, the discovery roots the user configured, a root the user pointed at, and the watched folder save paths.
* A discovery root or pointed root is trusted as a location because the user chose it. Names declared by a torrent and joined to it are not, and pass `Path::isValid()` on the same terms as every other root.
* Names declared by a `.torrent` file are untrusted input and are treated as data throughout.

## Observability

* A discovery that assigns a location records the torrent, the chosen root, the origin that root came from, and the count that chose it, through `Logger::addMessage`.
* A discovery that matches nothing records that outcome, so a user reporting an unexpected placement has a trail.
* Log volume stays proportional to torrents processed rather than to roots probed.

## Accessibility and localisation

* Every user-facing string is translatable through `tr()`.
* The options dialog group and its checkboxes are standard widgets, reachable by keyboard and exposed to screen readers.
* The web interface controls for the same settings are standard form elements on the preferences page, carrying labels that name the setting.
* The list of torrents that matched nothing is navigable by keyboard.

## Maintainability

* Candidate root construction and selection are pure functions, exercised without a populated disk.
* The change is additive: every signature callers depend on keeps its shape, and the one helper that changes is private to a single translation unit.
