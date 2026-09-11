# Find Location — Implementation Plan

The order the work is committed in, and the points at which it can stop. Requirements are set out in [the technical requirements](find-location_technical_requirements.md), whose ticket identifiers this plan carries, with build order and verification in [the dependency map](find-location_dependency_map.md), behaviour in [the feature specification](find-location_feature_spec.md), product requirements in [the product requirements](find-location_product_requirements.md), implementation in [the technical approach](find-location_technical_approach.md), qualities in [the non-functional requirements](find-location_nfr.md), and systems in [the system architecture](find-location_system_architecture.md).

Twenty-two commits across three submissions, with eleven milestones. Every commit builds and leaves the suite green, satisfying BT-7 at each step rather than at the end. Every milestone is a point at which the work can be set down.

## Submission 1 — automatic discovery

| Commit | Subject | Tickets |
| --- | --- | --- |
| 1 | `Add tests for existing file search behaviour` | T0 |
| 2 | `Return match count from file search helper` | T1 |
| 3 | `Add multi-root variant of file search` | T2 |
| 4 | `Add candidate root construction` | T3 |
| 5 | `Add find location preferences` | T4 |
| 6 | `Compose search roots in the session` | T5 |
| 7 | `Find existing content when adding a torrent` | T6, T7 |
| 8 | `Add Find location options group` | T8 |
| 9 | `WebAPI: expose find location preferences` | T9 |

Commit 1 is test-only, written against code the submission goes on to change. Commit 2 is internal to `filesearcher.cpp` and alters no behaviour. Commit 3 carries scoring, early abandonment and logging together, since they are one method. Commit 7 joins the two resolution call sites, so that a torrent from a file and a torrent from a magnet link gain the behaviour at the same moment.

Commits 4 and 5 depend on nothing in this submission and may be taken at any point. Commits 8 and 9 depend only on commit 5.

* **M1**, after commit 1 — the baseline is pinned and the suite is green. Commit 2 changes the code this covers, so nothing proceeds until it passes.
* **M2**, after commit 3 — the search machinery is complete and commit 1's test passes unmodified. This is the gate the submission rests on: it demonstrates that the refactor preserved existing behaviour.
* **M3**, after commit 6 — the plumbing is complete and behaviour is unchanged. The last point before anything is user-visible.
* **M4**, after commit 7 — the feature is observable. The dependency map's manual cases for steps 6 and 7 run here.
* **M5**, after commit 9 — rebase onto `master`, run the manual cases for steps 8 and 9, submit.

## Submission 2 — manual, batch and assignment

| Commit | Subject | Tickets |
| --- | --- | --- |
| 10 | `Add assignment preferences` | T10 |
| 11 | `Start torrents after an assignment recheck` | T11 |
| 12 | `Expose content discovery on the session interface` | T12 |
| 13 | `Add Find location to the transfer list menu` | T13 |
| 14 | `Add dialog for torrents that matched nothing` | T14 |
| 15 | `WebAPI: expose assignment preferences` | T15 |

Commit 13 carries `Closes #8261.` in its body, since the transfer list action is what that issue describes.

* **M6**, after commit 12 — discovery is reachable from `src/gui`. No GUI work can begin before it.
* **M7**, after commit 14 — the manual and batch modes work end to end. The manual cases for steps 13 and 14 run here, including the per-torrent reporting that IF-13 requires.
* **M8**, after commit 15 — rebase onto `master`, run the manual cases for steps 11 and 15, submit.

## Submission 3 — discovery roots

| Commit | Subject | Tickets |
| --- | --- | --- |
| 16 | `Add discovery root storage` | T16 |
| 17 | `Enumerate recursive discovery roots` | T17 |
| 18 | `Accept an enumerated map in candidate roots` | T18 |
| 19 | `Compose discovery roots in the session` | T22 |
| 20 | `Add discovery root list to options` | T19 |
| 21 | `Search a directory the user points at` | T20 |
| 22 | `WebAPI: expose the discovery root list` | T21 |

* **M9**, after commit 19 — discovery roots work on the automatic path. The manual cases for step 22 run here, covering the ordering of a discovery root against a watched folder save path, of a pointed root against a discovery root, and the case of storage that is absent.
* **M10**, after commit 21 — the pointed root works. The manual cases for steps 19 and 20 run here.
* **M11**, after commit 22 — rebase onto `master`, run the manual case for step 21, submit.

## Commit conventions

Subjects are limited to fifty characters, capitalised, in the imperative mood, and carry no trailing period. The body explains what and why. A commit addressing a reported issue names it there.

Conflicts are resolved by rebasing onto the target branch. Each submission carries one `API_VERSION` bump and one `WebAPI_Changelog.md` entry, in its final commit, so the version moves once per submission rather than once per key.

## Pull request descriptions

Each submission's description states its scope, the submissions it depends on, and the manual verification performed.

The first submission's description states that commit 1 pins existing behaviour deliberately, so that the change to `findInDir()` can be shown to preserve it. A test-only commit against untouched code reads as out of place without that, and with it, it is the strongest evidence the submission carries.

The second and third submissions' descriptions carry screenshots of the transfer list action, the dialog listing torrents that matched nothing, and the options dialog group in each of its states.
