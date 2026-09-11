# Find Location — Product Requirements

## Summary

qBittorrent adopts content that is already on disk when a torrent is added, instead of downloading it again. A user moving a library from another client, or rebuilding one after losing configuration, recovers it without repairing each torrent by hand.

Behaviour is set out in [the feature specification](find-location_feature_spec.md), implementation in [the technical approach](find-location_technical_approach.md), qualities in [the non-functional requirements](find-location_nfr.md), and build order in [the dependency map](find-location_dependency_map.md).

## Problem

A `.torrent` file records what content is, not where a user keeps it. That association lives only in the client that downloaded the content, and it does not travel with the `.torrent` files.

A client receiving those files has no basis for placing a torrent anywhere but the default location, so content present on disk is treated as absent and downloaded again. The user pays for it in bandwidth, in disk space, and in the time it takes to notice and repair each torrent individually.

## Users

**The migrator** moves a library from another BitTorrent client. They hold the `.torrent` files and the content, and have lost the mapping between them. Library sizes run to thousands of torrents. They know where their content is, and need only a way to say so. This is the primary user.

**The rebuilder** has lost qBittorrent's own configuration or resume data while the content survived. Their need is identical to the migrator's, arriving by a different route.

**The reinstaller** sets up qBittorrent on a new machine against storage that already holds the content, whether a moved disk or a network share.

**The routine user** adds torrents one at a time and occasionally adds one whose content they already hold. They benefit from the same mechanism without ever thinking about it.

## User stories

* As a migrator, I add my `.torrent` files and my library is recognised, so I neither redownload it nor repair it torrent by torrent.
* As a migrator, torrents I had partly downloaded resume from what is on disk rather than starting over.
* As a rebuilder, my completed torrents return to seeding without my intervention.
* As a routine user, a torrent I add whose content I already hold is recognised, and one whose content I do not hold behaves as it always has.
* As a user who wants placement under my own control, I turn the feature off in one action and nothing about torrent add changes.
* As a user repairing a handful of torrents, I select them, ask for their locations to be found, and deal by hand only with the ones that were not.
* As a migrator whose library sits in one place, I point at that directory once and every torrent beneath it is linked up, without configuring a watched folder I have no other use for.
* As a migrator whose library is split across several disks, I configure each as a place to look, or point at them one after another and watch the unresolved list shrink.
* As a user looking at torrents that matched nothing, I point at the directory I know holds them and have them resolved without leaving the list.

## Goals

* Eliminate redownload of content that is already on disk in a location the user has configured.
* Recover partial content as readily as complete content.
* Require no user action for the common case, and no more than one action for the exceptions.
* Leave placement behaviour unchanged for users who disable the feature.

## Success measures

* A migration of a library whose content sits under a path the application searches, whether configured as a save path, configured as a discovery root, or pointed at, completes with no content bytes transferred for torrents that are complete.
* Torrents partly downloaded before the migration resume at their existing progress rather than at zero.
* The number of torrents a user must repair by hand after a migration falls to those whose content the user cannot point the application at.
* Issue [#8261](https://github.com/qbittorrent/qBittorrent/issues/8261) is closable.

## Requirements

### Must

* Discovery runs when a torrent is added and selects a save path from the candidate roots.
* Discovery runs for magnet links when metadata arrives, before content pieces are requested.
* A torrent whose content is fully present transfers no content data.
* Partial content is adopted at whatever proportion is present.
* Discovery reads only; it creates, moves, renames and deletes nothing.
* A torrent matching nothing is placed at its configured destination and behaves as it does without the feature.
* The feature is disabled in one action, returning torrent add to its prior behaviour.
* Every setting is reachable from each interface the application presents, so a user running the headless daemon configures the feature as fully as a user running the desktop application.
* Discovery records what it chose, and records a torrent that matched nothing, so an unexpected placement can be accounted for after the fact.

### Should

* A **Find location** action runs discovery on demand for torrents already in the session, singly and over a selection.
* Assignment rechecks the torrent and returns it to service, seeding what is complete and downloading what is not.
* Torrents matching nothing in a batch are listed, and the user walks them or abandons the operation.
* Seeding and downloading are separately controllable, so a user on constrained upstream recovers a library without joining swarms as a seed.
* Torrents started after assignment are auto-managed, so the session's queueing limits govern how many of a recovered library run at once.
* Directories the user configures are searched alongside the watched folder save paths, ahead of them, and without requiring a watched folder to exist.
* A directory searched recursively is listed once and matched by name, so pointing at a library costs one listing rather than one probe per subdirectory.
* The list of torrents that matched nothing accepts a directory to search, resolving them without leaving the list, and accepts another while entries remain, so a library across several disks is covered a disk at a time.

### Could

* A WebUI interface for the manual, batch and pointed modes.
* An enumerated listing retained across a session rather than rebuilt per operation.

### Will not

* Locating content whose on-disk name differs from the names the torrent declares.
* Deduplicating or hard-linking content shared between torrents.
* Cross-seed group management, meaning the linking of torrents so that acting on one acts on the others.
* Changes to how automatic torrent management derives base save paths.

## Assumptions

* The user knows where their content is, and can either point the application at it or has it under a path the application already knows.
* The content on disk carries the names the torrent declares, allowing for qBittorrent's own incomplete-file suffix.
* libtorrent hash-checks a torrent added against existing data, which is what establishes completed pieces.
* A user's library is reachable by directory rather than scattered arbitrarily, so a small number of pointed roots covers it.

## Dependencies

* Watched folder configuration and the discovery root list, which together supply the search space.
* The session's checking limit, which bounds recheck throughput for a batch.
* The session's queueing limits, which bound how many adopted torrents start.
* No external service, library or protocol change.

## Risks

* A user whose content sits outside every path qBittorrent knows sees no benefit until they configure a discovery root or point at a directory. The first two iterations carry that limitation; the third removes it.
* Enabled by default, the feature changes placement behaviour on upgrade. The single disabling action is the mitigation, and the discovery root list starts empty so an upgrade adds no search locations of its own.
* A batch recheck across a large library is disk-intensive for its duration, bounded by the session's checking limit rather than by the feature.
* A discovery root pointed at a very large directory costs a listing proportional to its subdirectory count. The listing is taken once per operation, so the cost does not scale with the number of torrents.

## Release

The work ships as three submissions. The first delivers discovery at add and at metadata received, with its setting and WebAPI key, and serves the GUI, the headless daemon and the WebUI alike. The second delivers the manual and batch modes, assignment, and their settings. The third delivers discovery roots, enumeration and the pointed root, removing the requirement that content sit under a path the application already knows.

Each submission stands alone and depends only on those before it. The third carries the largest GUI surface, a list with a per-root dialog, which is why it is separated rather than folded into the first.
