# Find Location — Feature Specification

## Problem

A `.torrent` file records what content is, not where a particular user keeps it. The association between a torrent and the directory holding its content lives only in the client that downloaded it.

When a user moves torrent management from one client to another, that association does not travel with the `.torrent` files. The receiving client has no basis for placing a torrent anywhere except the default location, so it treats content that is present on disk as absent and downloads it again.

The same risk remains after a torrent has entered the session. Starting a stopped torrent through the ordinary workflow can create its configured destination, allocate files there and request pieces before the application looks for content elsewhere. Opening that newly created destination can then resemble a successful location match even though discovery never ran.

Repairing this by hand means noticing each redownload, stopping the torrent, setting its location, and forcing a recheck. Across a migration of hundreds of torrents the effort is prohibitive, and every torrent the user fails to notice consumes bandwidth and disk space duplicating content that is already stored.

## Goals

* Place a newly added torrent at content that already exists on disk, rather than downloading that content again.
* Locate content for an existing stopped torrent before a Start request can create, allocate or download payload files at the configured destination.
* Resolve the location before the torrent requests pieces from peers.
* Recover partially downloaded content, including content the previous client left incomplete.
* Provide a manual trigger for torrents that are already in the session, singly and in bulk.
* Search directories the user points at, whether configured in advance or chosen for a single operation.
* Return an assigned torrent to service without further user action, seeding what is complete and downloading what is not.

## Non-goals

* Locating content whose on-disk name differs from the names the torrent describes. Matching is by path, not by hashing candidate directories.
* Deduplicating or hard-linking content shared between torrents.
* Cross-seed group management, meaning the linking of torrents so that acting on one acts on the others. Adopting content a torrent shares with one already in the session is what discovery does; managing the relationship between them is not.
* Changing how automatic torrent management derives base save paths.

## Terminology

* **Watched folder** — a directory monitored for new `.torrent` files. Each watched folder carries a save path that torrents found there are assigned.
* **Location**, or **save path** — the directory a torrent's content is stored under.
* **Candidate root** — a directory that discovery probes for a torrent's content, whatever its origin.
* **Root form**, **name form**, **source form** — the candidates a search root contributes, named in *Candidate roots* below. A probed root contributes all three; an enumerated root contributes the root form and the source form.
* **Discovery root** — a directory the user configures for searching, held with its own options.
* **Pointed root** — a directory the user chooses for one operation.
* **Probe** — a count of how many of the files a torrent describes are present beneath a candidate root.
* **Manual location mode** — a torrent mode in which automatic torrent management does not own the torrent's placement.
* **Pending Start** — a normal or forced Start request held by Find Location until discovery and any required assignment and recheck have completed.
* **Start transaction** — the single operation that owns a pending Start from interception through its match, miss, cancellation or failure outcome.

## Discovery

Discovery selects a save path for a torrent by probing a list of candidate roots and choosing the one where the torrent's content is found.

### Candidate roots

The list is ordered, and the first entries preserve the placement the user asked for:

1. The save path configured for the torrent.
2. The download path configured for the torrent.

Those two are destinations as well as search locations: they are where the torrent is written. Every root after them is a search location alone, and a tie decided in their favour leaves the torrent where it is already placed.

Where an operation carries a pointed root, it follows the torrent's own paths. A user who points at a directory has given the strongest available signal about where content lives, so it precedes every root the application infers.

The discovery roots follow, in the order the user configured them.

The watched folder save paths come last. Each save path, whether a discovery root or a watched folder's, contributes candidates in three forms, in this order:

* The **root form** — the save path itself.
* The **name form** — the save path joined with the torrent's name.
* The **source form** — the save path joined with the basename of the `.torrent` file.

The root form covers content stored as the torrent describes it, since a multi-file torrent carries its own root folder in the paths it declares. The name form covers content nested inside a directory of the torrent's name, which arises for single-file torrents and for content the previous client placed under a folder of its own. The source form accommodates a `.torrent` file whose name reflects the directory even where the torrent's internal name does not, which describes a library organised by hand; its position last keeps it from outranking a better match, and a root holding none of the content is abandoned after a few tests.

Where a watched folder has no save path configured it contributes the session default. Duplicate roots are collapsed, since watched folders commonly share a save path.

A root on storage that is not present when discovery runs contributes nothing and stops nothing. A discovery root on a removable disk or an unmounted share leaves every other root resolving as it would have.

### Enumerated roots

A discovery root carries a recursion flag, as a watched folder does.

A root marked recursive has its immediate subdirectories listed once, and a torrent is matched against that listing by name. A user pointing at a directory that holds their library gets every directory beneath it searched, at the cost of one listing rather than one probe per subdirectory.

The listing does the work of the name form, so an enumerated root contributes the root form and the source form alone. The directory named for the torrent is reached through the listing instead of being probed for.

Names are matched against the listing by the same case rule the platform applies to paths, so a listing resolves whatever probing that root directly would have resolved.

The listing is built for the operation that needs it and discarded when that operation ends.

A pointed root is enumerated on the same terms.

### Probing

A probe counts how many of the files the torrent declares are present relative to the candidate root. A file counts as present when it exists under its declared name, or under that name with qBittorrent's incomplete-file extension appended, which is how partially downloaded content is recognised.

A probe abandons a root once its count can no longer overtake the best root found so far.

A probe reads no file contents and computes no hashes.

### Selection

A root where no file is found is not a match. Among roots where at least one file is found, the root with the most files present wins, and the earliest root in the list wins a tie. Ordering therefore decides in favour of the user's configured placement whenever it holds the content.

A torrent that matches no root is placed at its configured destination.

Discovery records what it chose, and records a torrent that matched nothing, so a user asking why a torrent landed where it did has an account of it in the log.

### Completion

Discovery selects a location; it does not establish which pieces are valid. A torrent added at the selected location undergoes its add-time hash check before it can transfer payload data. A stopped torrent matched by a pending Start undergoes a recheck at the matched location before the Start request can proceed, whether or not assignment changed its location.

Any amount of recoverable content is a success. A torrent the user was midway through downloading when they changed clients is a valid case, so no completion threshold applies.

## Modes

### Automatic

Discovery runs when a torrent is added, and the selected location becomes the torrent's save path.

For a torrent added from a magnet link, the files it declares are unknown until metadata arrives from the swarm. Discovery for such a torrent runs at the point metadata is received, before the torrent requests content pieces.

### Start-triggered

While **Find location when starting stopped torrents** is enabled, a normal or forced Start request for a stopped torrent in manual location mode begins a Start transaction. A torrent whose placement is owned by automatic torrent management follows its existing Start behaviour instead.

The transaction holds the torrent stopped before the ordinary Start workflow can create its destination, allocate or write payload files, request content pieces, or otherwise begin downloading. It records whether the user requested normal or forced operation so the same request can be honored later.

Where the torrent lacks metadata, it may connect only to obtain the metadata needed to discover its files. Receipt of metadata does not release the hold: discovery runs then, before any payload piece is requested.

A match at the torrent's current save path or download path needs no location assignment but is rechecked through the transactional lifecycle below. A match at another location is assigned and then rechecked. A successful miss releases the hold and lets the ordinary Start workflow proceed at the configured destination.

### Manual

A **Find location** action sits alongside **Set location...** in the transfer list context menu. It runs discovery for the selected torrent and assigns the result.

Where discovery finds no match, the **Set location** file dialog opens, so the user targets the location by hand.

### Batch

Over a multiple selection, **Find location** runs discovery for every selected torrent. Torrents that match are assigned their locations.

Torrents that match nothing are collected and presented as a list, from which the user abandons the operation or steps through the entries setting each location. This reduces the number of dialogs a batch import requires.

### Pointed

The list of torrents that matched nothing offers a directory to search. The user chooses one, and discovery runs again for those torrents with that directory as a pointed root, enumerated as a recursive discovery root is and ordered ahead of every other search root, since a directory chosen for this operation is the most specific instruction available. Torrents that match are assigned and leave the list.

The action is repeatable while the list holds entries. A library split across several disks is recovered by pointing at each in turn, the list shrinking as each is searched, which serves the same purpose as choosing several directories at once without asking the user to assemble a set before anything happens.

This is the answer for a user who knows where their content is and has not configured the application to look there.

### Assignment

Assigning a location from the manual, batch or pointed modes, whether from a match or from the file dialog, rechecks the torrent while **Recheck automatically** is enabled and returns it to service as **Seed automatically** and **Leech automatically** allow. The recheck establishes which pieces the content holds.

A torrent started this way is auto-managed, so the session's queueing limits govern how many of a recovered library run at once, exactly as they govern any other torrent.

Rechecks arising from a batch are throttled by the session's limit on concurrently checking torrents, so a large selection queues rather than contending for the disk.

An assignment owned by a Start transaction follows the mandatory recheck and explicit start decision below instead. The optional automatic recheck, seed and leech preferences do not suppress a Start request the user already made.

## Transactional lifecycle

Each torrent has at most one Start transaction. Its lifecycle is:

1. **Hold.** Intercept the Start request before ordinary start behaviour. Keep the torrent stopped and record its requested normal or forced operating mode.
2. **Search.** Run discovery asynchronously. If metadata is unavailable, obtain metadata alone and search when it arrives while continuing to hold payload activity.
3. **Assign.** On a match at another location, assign that location without moving content from the former destination into it. If assignment requires storage movement, wait until the torrent reports the selected location. A match at the current save path or download path skips assignment and proceeds to recheck there.
4. **Recheck.** Recheck every matched location, after assignment and storage movement where they are needed. This recheck is mandatory for a pending Start and runs regardless of **Recheck automatically**.
5. **Decide.** Only successful completion of the required work permits the recorded Start request. Complete content starts seeding; partial content starts downloading only the pieces the check found missing; the requested normal or forced operating mode is preserved.

A successful search with no match ends the feature-owned transaction and returns control to ordinary Start behaviour at the configured destination. A miss is not an error and does not leave feature-owned state behind. Every match remains in the transaction until its recheck succeeds, including a match at the current save path or download path.

A Stop request during any phase cancels the pending Start and leaves the torrent stopped. Removing the torrent or shutting down also cancels the transaction, and no asynchronous result arriving afterward may assign, recheck or start it. Repeated Start requests coalesce into the active transaction rather than launching competing work; the eventual decision uses the user's latest explicit normal or forced mode.

A discovery-operation, assignment, storage-movement or recheck failure is distinct from a successful miss. It clears the transaction, does not honor the pending Start, and leaves the torrent stopped and able to retry. A reported or cached checking state alone is not evidence that a recheck began or succeeded. Only the successful check-completion outcome advances the transaction to its Start decision.

The execution log records whether the transaction continued after a match, continued after a miss, was cancelled, or failed, including the phase and reported reason for a failure. This is one final transaction outcome in addition to discovery's existing account of the location it selected.

## Timing

The selected save path is in place before the torrent requests content pieces from peers, so a torrent whose content is fully present transfers no content data.

A magnet link exchanges metadata with the swarm before its files are known. Discovery runs when that exchange completes, and content pieces are requested after it.

For Start-triggered discovery, holding begins before the ordinary Start workflow. Search completes before assignment, assignment and any storage movement complete before recheck, and a successful recheck completes before any matched location is allowed to seed or download. A miss releases the ordinary Start workflow only after search completes successfully. Cancellation or failure never releases it.

## Configuration

The feature's settings form a single group in the options dialog, titled **Find location**. The group carries a state of its own, so the feature is enabled or disabled as a whole, and the settings within it adjust how it behaves. The group and every setting are enabled by default, and each is held as a preference.

Every setting is reachable from the web interface as well as the desktop one, so a user running the headless daemon configures the feature on the same terms as a user running the application.

Disabling the group leaves the settings within it holding the values the user chose, so re-enabling the group restores that configuration.

**Find location automatically** governs discovery at torrent add. It selects a location for a torrent by finding where its content is stored.

**Find location when starting stopped torrents** governs Start-triggered discovery for stopped torrents in manual location mode. It delays Start while the transactional lifecycle searches and verifies a match, assigning it first where it differs from the current location. It is enabled by default and is independent of **Find location automatically**.

**Recheck automatically** governs the recheck that follows an assignment made without a pending Start. Disabled, such a torrent is assigned its location and left with its progress as it stood. It does not disable the mandatory recheck after a pending Start finds content.

**Seed automatically** starts a torrent whose automatic assignment recheck finds its content complete when no explicit Start is pending.

**Leech automatically** starts a torrent whose automatic assignment recheck finds its content incomplete when no explicit Start is pending.

Seeding and leeching are separated because the two carry different costs to the user. A user with limited upstream bandwidth, or on a connection where seeding is unwelcome, recovers a library without joining swarms as a seed; a user recovering a completed archive returns it to service without also resuming downloads they had abandoned.

**Discovery roots** is a list rather than a checkbox. Each entry is a directory with its own options, of which recursion is one, held the way watched folders are held. The list sits within the group and is empty by default.

## Scope

### First iteration — automatic discovery

* Candidate root construction and probing.
* Discovery at torrent add, and at metadata received.
* The **Find location** options dialog group and the **Find location automatically** preference it holds, with the corresponding WebAPI keys and their web interface controls.

The first iteration is behaviour rather than interface, and serves the GUI, the headless daemon and the WebUI alike.

### Second iteration — Start-triggered, manual and batch

* Start-triggered discovery for stopped torrents in manual location mode, with one transaction holding each Start through search, assignment, recheck and its final decision.
* The **Find location when starting stopped torrents** preference, with its place in the options dialog group and the corresponding WebAPI key and web interface control.
* The **Find location** context menu action, over single and multiple selections.
* The dialog listing torrents that matched nothing.
* Recheck and start on assignment, with the **Recheck automatically**, **Seed automatically** and **Leech automatically** preferences, their places in the options dialog group, and the corresponding WebAPI keys and web interface controls.

The manual and batch surfaces are confined to the GUI. Start-triggered discovery is session behaviour shared by the GUI, headless daemon and WebUI, and builds on the discovery delivered by the first iteration.

### Third iteration — discovery roots

* Discovery roots, their per-root options, and their place in the candidate root order.
* Enumeration of recursive roots.
* The pointed root offered from the list of torrents that matched nothing.
* The discovery root list in the options dialog group, its per-root dialog, and the corresponding WebAPI keys and web interface controls.

The third iteration removes the requirement that content sit under a path the application already knows.

### Out of scope

* A WebUI interface for the manual, batch and pointed modes.
* Fuzzy or heuristic name matching.
* Discovery options attached to individual watched folder entries. Watched folders contribute their save paths to the search and carry no settings of their own for this feature; directories the user wants searched on their own terms are configured as discovery roots.
* Repairing the application's pre-existing force-recheck failure. Find Location handles any recheck failure defensively by cancelling its pending Start, but the underlying recheck defect remains separate work.

Each iteration is a separate submission, keeping every change reviewable and limited to one feature.

## Related work

* Issue [#8261](https://github.com/qbittorrent/qBittorrent/issues/8261) proposes a button in the add torrent dialog that searches for matching folders, filenames and file sizes. It describes the same mechanism scoped to a single save path and driven from the add dialog, where this specification searches the watched folder set and drives from the transfer list.
* Pull request [#23578](https://github.com/qbittorrent/qBittorrent/pull/23578) reworks the interaction between automatic management and base save paths, which is the derivation this specification treats as a non-goal.
* Pull request [#20502](https://github.com/qbittorrent/qBittorrent/pull/20502) adds a rules engine setting categories and tags from torrent metadata. It is adjacent and independent.
* Issue [#16623](https://github.com/qbittorrent/qBittorrent/issues/16623) covers cross-seed linking and issue [#14536](https://github.com/qbittorrent/qBittorrent/issues/14536) covers periodic availability checking. Neither is addressed here.
