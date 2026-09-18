# Find Location — Workplan

Nodes for the feature set out in [the feature specification](find-location_feature_spec.md), against the requirements in [the technical requirements](find-location_technical_requirements.md) and the build order in [the dependency map](find-location_dependency_map.md).

Each node addresses one source file and the support that file requires. Ticket identifiers name the requirements a node delivers.

## Epic 1 — automatic discovery

*   `[X]` [BE] src/base/bittorrent/`filesearcher` — pin the two-directory behaviour of `FileSearcher::search()`, add a non-mutating counting helper, add `FileSearcher::searchRoots()` scoring a torrent's own save path, its download path and search-only candidates together by match count, and add `candidateRoots()`. T0, T1, T2, T3

    *   `[X]` `objective`
        *   `[X]` Problem: `FileSearcher::search()` looks for a torrent's files in exactly two directories, the save path then the download path, and takes the first holding any file. Content held anywhere else is treated as absent and downloaded again.
        *   `[X]` Functional: given a torrent's save path, its download path and an ordered list of search-only candidates, score the save path, then the download path when one is set, then the candidates, in that order; select the directory holding the most of the torrent's files, the earlier directory winning a tie, so the torrent's own paths win a tie and lose to any candidate holding more; and when no directory holds any file, produce exactly `search()`'s result.
        *   `[X]` Functional: build the ordered list of search-only candidates from a list of search roots, each search root contributing the root form, the name form and the source form, omitting any form equal to the torrent's save path or download path.
        *   `[X]` Functional: `FileSearcher::search()` keeps its signature and its first-found behaviour exactly, proven by a test written and run against it before any code in this node changes.
        *   `[X]` Non-functional: a directory that does not exist costs one existence test; a directory is abandoned once its count can no longer exceed the best count; scoring stops once a directory holds every file, so a save path or download path holding every file probes no candidate.
        *   `[X]` Non-functional: probing tests existence only, reading no file contents and computing no hashes. A directory that cannot be read, whether an own path or a candidate, contributes a count of zero and does not fail the search.

    *   `[X]` `role`
        *   `[X]` Domain logic in `src/base/bittorrent`. `searchRoots()` runs on the session I/O thread, where `SessionImpl` moves `FileSearcher` into `m_ioThread`; `candidateRoots()` runs on the calling thread.
        *   `[X]` Out of scope: reading settings, the watched folder set, the session default save path or any singleton; logging; deriving the name form's name; any edit to `SessionImpl`, `TorrentImpl` or another caller. Each belongs to the `sessionimpl` or `torrentimpl` node.

    *   `[X]` `module`
        *   `[X]` Inside: probing a directory for a torrent's relative file names, scoring the torrent's own two paths together with search-only candidates, choosing the directory holding the most files, and deriving candidates from search roots and names. Outside: where search roots come from, whether discovery is enabled, what the torrent does with the result, and recording the outcome.

    *   `[X]` `deps`
        *   `[X]` `base/path.h` — `Path`, `PathList`, `Path::exists()`, `Path::isEmpty()`, `Path::isRelative()`, `Path::rootItem()`, `Path::removedExtension(QStringView)`, `operator/`, `operator+`, `operator==`. A lower layer of `src/base`, depended on inward.
        *   `[X]` `base/bittorrent/common.h` — `QB_EXT`. The same module.
        *   `[X]` `base/global.h` — `TORRENT_FILE_EXTENSION` and the `_s` literal. A lower layer, depended on inward.
        *   `[X]` `<QPromise>` — result delivery, used exactly as `search()` uses it.

    *   `[X]` `context_slice`
        *   `[X]` `SessionImpl::findIncompleteFiles()` calls `search()` through `QMetaObject::invokeMethod` on `m_fileSearcher`. `FileSearchResult` is consumed by the continuations in `SessionImpl::addTorrent_impl()` and `TorrentImpl::handleSaveResumeData()`. `searchRoots()` and `candidateRoots()` gain their consumer in the `sessionimpl` node.

    *   `[X]` test/testdata/`filesearcher`
        *   `[X]` test/testdata/filesearcher/complete/`alpha.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/complete/`beta.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/partial/`alpha.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/partialtwin/`alpha.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/incomplete/`alpha.txt.!qB` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/nested/folder/`alpha.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` The directories `test/testdata/filesearcher/absent` and `test/testdata/filesearcher/absentdownload` are named by the tests and never created.

    *   `[X]` test/`testbittorrentfilesearcher.cpp`
        *   `[X]` New file pinning `search()` as it stands. It references no symbol this node adds, so it builds and links against the untouched `filesearcher`.
        *   `[X]` Form copied from `test/testutilsio.cpp`: the licence header naming the contributor as `Full Name <email>`, as the existing headers name theirs, `class TestBittorrentFileSearcher final : public QObject` with `Q_OBJECT` and `Q_DISABLE_COPY_MOVE`, `private slots`, `QTEST_APPLESS_MAIN(TestBittorrentFileSearcher)` and `#include "testbittorrentfilesearcher.moc"`. Includes `<QFuture>`, `<QObject>`, `<QPromise>`, `<QTest>`, `"base/bittorrent/common.h"`, `"base/bittorrent/filesearcher.h"`, `"base/global.h"` and `"base/path.h"`.
        *   `[X]` Fixture root `Path(QString::fromUtf8(__FILE__)).parentPath() / Path(u"testdata/filesearcher"_s)`. The file names are `alpha.txt` then `beta.txt`. Each slot constructs a `FileSearcher` and drives it as `findIncompleteFiles()` does: `QPromise<FileSearchResult> promise`, `promise.start()`, `search(...)`, `promise.finish()`, then `promise.future().result()`.
        *   `[X]` `testSavePathHoldingAnyFileIsUsed` — save `complete`, download empty, `forceAppendExt` true → `savePath` is `complete`; `fileNames` are `alpha.txt`, `beta.txt`.
        *   `[X]` `testForceAppendsAtSavePathWithoutDownloadPath` — save `partial`, download empty, `forceAppendExt` true → `savePath` is `partial`; `fileNames` are `alpha.txt`, `beta.txt` + `QB_EXT`.
        *   `[X]` `testNoForceLeavesAbsentNamesUnchanged` — save `partial`, download empty, `forceAppendExt` false → `fileNames` are `alpha.txt`, `beta.txt`.
        *   `[X]` `testIncompleteVariantIsAdopted` — save `incomplete`, download empty, `forceAppendExt` false → `savePath` is `incomplete`; `fileNames` are `alpha.txt` + `QB_EXT`, `beta.txt`.
        *   `[X]` `testFirstDirectoryHoldingAnyFileWins` — save `partial`, download `complete`, `forceAppendExt` false → `savePath` is `partial`, though `complete` holds more of the files.
        *   `[X]` `testDownloadPathSuppressesForceAtSavePath` — save `partial`, download `complete`, `forceAppendExt` true → `fileNames` are `alpha.txt`, `beta.txt`.
        *   `[X]` `testDownloadPathUsedWhenSavePathHoldsNothing` — save `absent`, download `complete`, `forceAppendExt` false → `savePath` is `complete`.
        *   `[X]` `testMissAppendsAtDownloadPath` — save `absent`, download `absentdownload`, `forceAppendExt` true → `savePath` is `absentdownload`; `fileNames` are `alpha.txt` + `QB_EXT`, `beta.txt` + `QB_EXT`.
        *   `[X]` `testMissWithoutDownloadPathStaysAtSavePath` — save `absent`, download empty, `forceAppendExt` true → `savePath` is `absent`; `fileNames` are `alpha.txt` + `QB_EXT`, `beta.txt` + `QB_EXT`.

    *   `[X]` test/`testbittorrentfilesearchermultiroot.cpp`
        *   `[X]` New file proving `searchRoots()`. Once the header element declares `searchRoots()` and before the implementation element defines it, it fails to link with an unresolved symbol `FileSearcher::searchRoots`, and that link failure is its red state.
        *   `[X]` Form, fixture root and file names as `testbittorrentfilesearcher.cpp`, with `class TestBittorrentFileSearcherMultiRoot`, `QPromise<SearchRootsResult>` in place of `QPromise<FileSearchResult>`, and `searchRoots(...)` in place of `search(...)`. Add `<QDir>`, `<QFile>`, `<QFileInfo>`, `<QScopeGuard>` and `<QTemporaryDir>` for the process-state and permission tests below, and `<QtSystemDetection>` for their platform guards. Save is `absent`, download is `absentdownload`, candidates are empty and `forceAppendExt` is false unless a slot states otherwise.
        *   `[X]` `testSearchRootWithMoreBeatsSavePath` — save `partial`, download empty, candidates `complete` → `savePath` is `complete`; `fileNames` are `alpha.txt`, `beta.txt`; `matchCount` is 2; `searchedCandidates` is true; `foundAtOwnPath` is false.
        *   `[X]` `testSearchRootWithMoreBeatsDownloadPath` — download `partial`, candidates `complete` → `savePath` is `complete`; `matchCount` is 2; `searchedCandidates` is true; `foundAtOwnPath` is false.
        *   `[X]` `testCompleteSearchRootBeatsIncompleteSavePath` — save `incomplete`, download empty, candidates `complete`, `forceAppendExt` true → `savePath` is `complete`; `fileNames` are `alpha.txt`, `beta.txt`; `matchCount` is 2; `foundAtOwnPath` is false. This covers a torrent started against the wrong location while a complete copy of its content exists under a search root.
        *   `[X]` `testSavePathWinsTieWithSearchRoot` — save `partial`, download empty, candidates `partialtwin` → `savePath` is `partial`; `matchCount` is 1; `searchedCandidates` is true; `foundAtOwnPath` is true.
        *   `[X]` `testForceAppliedAtSavePathWithoutDownloadPath` — save `partial`, download empty, candidates `partialtwin`, `forceAppendExt` true → `savePath` is `partial`; `fileNames` are `alpha.txt`, `beta.txt` + `QB_EXT`; `matchCount` is 1; `foundAtOwnPath` is true.
        *   `[X]` `testDownloadPathWithMoreBeatsSavePath` — save `partial`, download `complete` → `savePath` is `complete`; `fileNames` are `alpha.txt`, `beta.txt`; `matchCount` is 2; `searchedCandidates` is false; `foundAtOwnPath` is true.
        *   `[X]` `testOwnPathFullMatchSkipsSearchRoots` — save `complete`, candidates `partial` → `savePath` is `complete`; `matchCount` is 2; `searchedCandidates` is false; `foundAtOwnPath` is true.
        *   `[X]` `testOwnPathsMatchSearchWhenNoSearchRoots` — save `partial`, download `absentdownload`, `forceAppendExt` true → `savePath` is `partial`; `fileNames` are `alpha.txt`, `beta.txt`; `matchCount` is 1; `foundAtOwnPath` is true. Then save `absent`, download `absentdownload`, `forceAppendExt` true → `savePath` is `absentdownload`; `fileNames` are `alpha.txt` + `QB_EXT`, `beta.txt` + `QB_EXT`; `matchCount` is 0; `foundAtOwnPath` is false. Both results have `searchedCandidates` false, and each equals `search()`'s result for the same inputs.
        *   `[X]` `testMostFilesWins` — candidates `partial`, `complete` → `savePath` is `complete`; `matchCount` is 2; `searchedCandidates` is true.
        *   `[X]` `testEarlierRootWinsTie` — candidates `partial`, `partialtwin` → `savePath` is `partial`; `matchCount` is 1.
        *   `[X]` `testIncompleteVariantCounts` — candidates `incomplete` → `savePath` is `incomplete`; `matchCount` is 1; `fileNames` are `alpha.txt` + `QB_EXT`, `beta.txt`.
        *   `[X]` `testScoringDoesNotCarryRewrittenNames` — candidates `incomplete`, `complete` → `savePath` is `complete`; `matchCount` is 2; `fileNames` are `alpha.txt`, `beta.txt`.
        *   `[X]` `testMissFallsBackToDestination` — candidates `absent`, `forceAppendExt` true → `savePath` is `absentdownload`; `matchCount` is 0; `searchedCandidates` is true; `foundAtOwnPath` is false; `fileNames` are `alpha.txt` + `QB_EXT`, `beta.txt` + `QB_EXT`.
        *   `[X]` `testForceNotAppliedAtSearchOnlyWinner` — candidates `partial`, `forceAppendExt` true → `savePath` is `partial`; `fileNames` are `alpha.txt`, `beta.txt`.
        *   `[X]` `testUnavailableRootContributesNothing` — candidates `absent`, `partial` → `savePath` is `partial`; `matchCount` is 1. This covers disconnected or missing storage, whose path cannot be traversed because it is absent.
        *   `[X]` `testUnreadableRootContributesNothing`, its body guarded by `#ifdef Q_OS_UNIX` with an `#else` branch calling `QSKIP("Requires Unix permission semantics.")` — create a temporary directory holding a child directory, write `alpha.txt` and `beta.txt` into the child, remove the child's permissions with `QFile::setPermissions(child, {})`, and register a `qScopeGuard` that restores owner read, write and execute permissions before `QTemporaryDir` cleans it up. `QVERIFY` each setup operation. When `QFileInfo::exists()` still reports the child's `alpha.txt`, use `QSKIP`, because the running account can traverse the child and the environment cannot establish the precondition. Otherwise, with candidates the child then `partial`, `savePath` is `partial` and `matchCount` is 1. Windows permission semantics do not provide a stable equivalent for an unprivileged unit test, so the manual verification below covers a permission-denied root there.
        *   `[X]` `testEmptyRootIsSkipped` — `QVERIFY` that the fixture root is absolute. Save the current working directory, enter a nested scope, immediately register a `qScopeGuard` in that scope that restores the saved directory, and use `QVERIFY` when changing the current directory to `complete` with `QDir::setCurrent`. Run the search in the nested scope; after the scope and guard have completed, use `QCOMPARE` to prove that `QDir::currentPath()` is the saved directory. With candidates `Path()`, `partial`, `savePath` is `partial` and `matchCount` is 1. An earlier assertion may return without reaching the restoration comparison, but the guard still prevents process-global state from leaking into another test.
        *   `[X]` `testNamesUnderPresentFolderCountedEach` — file names `folder/alpha.txt`, `folder/beta.txt`, candidates `complete`, `nested` → `savePath` is `nested`; `matchCount` is 1.
        *   `[X]` `testAbsentFolderCountsNone` — file names `folder/alpha.txt`, `folder/beta.txt`, candidates `complete` → `savePath` is `absentdownload`; `matchCount` is 0; `searchedCandidates` is true.

    *   `[X]` test/`testbittorrentcandidateroots.cpp`
        *   `[X]` New file proving `candidateRoots()`, which touches no filesystem, so no fixture is used. Once the header element declares `candidateRoots()` and before the implementation element defines it, it fails to link with an unresolved symbol `candidateRoots`, and that link failure is its red state.
        *   `[X]` Form as `testbittorrentfilesearcher.cpp`, with `class TestBittorrentCandidateRoots`, includes `<QObject>`, `<QTest>`, `"base/bittorrent/filesearcher.h"`, `"base/global.h"` and `"base/path.h"`, and each slot after the first comparing the returned `PathList` to an expected `PathList` with `QCOMPARE`. Inputs are save `/save`, download `/download`, default `/default`, torrent name `Album`, source file name `Album Rip.torrent` and search roots `/watch`, unless a slot states otherwise.
        *   `[X]` `testPathListComparisonDiscriminates`, the first slot — `QCOMPARE((PathList {Path(u"a/b"_s), Path(u"c"_s)}), (PathList {Path(u"a/b"_s), Path(u"c"_s)}));`, `QCOMPARE_NE((PathList {Path(u"a/b"_s), Path(u"c"_s)}), (PathList {Path(u"c"_s), Path(u"a/b"_s)}));` and `QCOMPARE_NE((PathList {Path(u"a/b"_s), Path(u"c"_s)}), (PathList {Path(u"a/b"_s), Path(u"d"_s)}));`, proving that equal lists compare equal and that a reordered list and a list with one differing element compare unequal.
        *   `[X]` `testOrderFollowsSearchRoots` — search roots `/watch`, `/other` → `/watch`, `/watch/Album`, `/watch/Album Rip`, `/other`, `/other/Album`, `/other/Album Rip`.
        *   `[X]` `testOwnPathsAreExcluded` — search roots `/save`, `/download`, `/watch` → `/save/Album`, `/save/Album Rip`, `/download/Album`, `/download/Album Rip`, `/watch`, `/watch/Album`, `/watch/Album Rip`.
        *   `[X]` `testEmptyDownloadPathExcludesNothing` — download `Path()`, search roots `/download` → `/download`, `/download/Album`, `/download/Album Rip`.
        *   `[X]` `testEmptySearchRootTakesDefault` — search roots `Path()` → `/default`, `/default/Album`, `/default/Album Rip`.
        *   `[X]` `testEmptySearchRootWithoutDefaultContributesNothing` — default `Path()`, search roots `Path()`, `/watch` → `/watch`, `/watch/Album`, `/watch/Album Rip`.
        *   `[X]` `testDuplicateKeepsEarliestPosition` — search roots `/watch`, `/other`, `/watch` → `/watch`, `/watch/Album`, `/watch/Album Rip`, `/other`, `/other/Album`, `/other/Album Rip`.
        *   `[X]` `testAbsentNameOmitsNameForm` — torrent name empty → `/watch`, `/watch/Album Rip`.
        *   `[X]` `testAbsentSourceOmitsSourceForm` — source file name empty → `/watch`, `/watch/Album`.
        *   `[X]` `testSourceExtensionStrippedCaseInsensitively` — source file name `Album Rip.TORRENT` → `/watch`, `/watch/Album`, `/watch/Album Rip`.
        *   `[X]` `testSourceFormMatchingNameFormCollapses` — source file name `Album.torrent` → `/watch`, `/watch/Album`.
        *   `[X]` `testEscapingNameIsDropped` — torrent name `../escape` → `/watch`, `/watch/Album Rip`.
        *   `[X]` `testEscapingSourceIsDropped` — source file name `...torrent`, whose stem is `..` → `/watch`, `/watch/Album`.
        *   `[X]` `testDuplicatesFoldCaseOnWindows`, its body guarded by `#ifdef Q_OS_WIN` with an `#else` branch calling `QSKIP("Requires case-insensitive path comparison.")` — save `C:/Save`, download `Path()`, search roots `c:/save`, `D:/Watch`, `d:/watch` → `c:/save/Album`, `c:/save/Album Rip`, `D:/Watch`, `D:/Watch/Album`, `D:/Watch/Album Rip`.

    *   `[X]` test/`CMakeLists.txt`
        *   `[X]` Add `testbittorrentcandidateroots.cpp`, `testbittorrentfilesearcher.cpp` and `testbittorrentfilesearchermultiroot.cpp` to `testFiles`, after `testalgorithm.cpp` and before `testbittorrentpeeraddress.cpp`. The existing `foreach` builds each as its own executable linked to `Qt::Test` and `qbt_base` and adds it to `check`.
        *   `[X]` With this element in place and before the header element is written, `cmake --build <build> --target testbittorrentfilesearcher` builds the pin alone against the untouched `filesearcher`, and the executable it produces runs green. Build only named targets until the implementation element is complete; `check` and the default target fail to build while the two new tests are in their red state.

    *   `[X]` src/base/bittorrent/`filesearcher.h`
        *   `[X]` Add `struct SearchRootsResult` after `FileSearchResult`, declaring in order `Path savePath;`, `PathList fileNames;`, `qsizetype matchCount = 0;`, `bool searchedCandidates = false;` and `bool foundAtOwnPath = false;`.
        *   `[X]` Add to `FileSearcher`, after `search()`: `void searchRoots(const PathList &originalFileNames, const Path &savePath, const Path &downloadPath, const PathList &candidates, bool forceAppendExt, QPromise<SearchRootsResult> &promise);`
        *   `[X]` Declare after the class: `PathList candidateRoots(const Path &savePath, const Path &downloadPath, const PathList &searchRoots, const Path &defaultSavePath, const QString &torrentName, const QString &sourceFileName);`
        *   `[X]` Add `#include <QString>` to the Qt header group, after `#include <QObject>`.
        *   `[X]` With the header in place, `testbittorrentfilesearchermultiroot` and `testbittorrentcandidateroots` compile and fail to link, which is their red state.

    *   `[X]` `interaction.spec`
        *   `[X]` `search()` keeps its body. Its branches: the save path holds any file → the result is the save path, with `forceAppendExt` applied there only when the download path is empty; the save path holds none and the download path is set → the result is the download path, with `forceAppendExt` applied there; the save path holds none and the download path is empty → the result is the save path, with `forceAppendExt` applied there.
        *   `[X]` `countInDir(dirPath, fileNames, mustExceed)`, new in the anonymous namespace, returns the number of `fileNames` present under `dirPath`. It walks `fileNames` in order, with `remaining` starting at `fileNames.size()`. A file counts as present when `dirPath / name` or `dirPath / (name + QB_EXT)` exists, and `remaining` falls by one after each file. After each file it returns as soon as the running count plus `remaining` is no greater than `mustExceed`. It never modifies `fileNames`.
        *   `[X]` `searchRoots()`, scoring: `bestCount` starts at 0 and there is no winner. Visit the save path entry, then the download path entry when `downloadPath` is non-empty, then each entry of `candidates`, in that order; stop once `bestCount` equals `originalFileNames.size()`; skip an entry that is empty or for which `exists()` is false; otherwise `countInDir(entry, originalFileNames, bestCount)`, and the entry becomes the winner only on a count strictly greater than `bestCount`, which takes that count. The winner is recorded as the save path entry, the download path entry or a candidate. `searchedCandidates` is true when `candidates` is non-empty and `bestCount` is below `originalFileNames.size()` once the own path entries are visited.
        *   `[X]` `searchRoots()` with the save path entry the winner: `findInDir(savePath, names, (forceAppendExt && downloadPath.isEmpty()))` over a copy of `originalFileNames`, then `promise.addResult(SearchRootsResult {.savePath = savePath, .fileNames = names, .matchCount = bestCount, .searchedCandidates = searchedCandidates, .foundAtOwnPath = true})`. The `findInDir()` arguments are those `search()` passes for the save path.
        *   `[X]` `searchRoots()` with the download path entry the winner: `findInDir(downloadPath, names, forceAppendExt)` over a copy of `originalFileNames`, then `promise.addResult(SearchRootsResult {.savePath = downloadPath, .fileNames = names, .matchCount = bestCount, .searchedCandidates = searchedCandidates, .foundAtOwnPath = true})`. The `findInDir()` arguments are those `search()` passes for the download path.
        *   `[X]` `searchRoots()` with a candidate the winner: `findInDir(winner, names, false)` over a copy of `originalFileNames`, then `promise.addResult(SearchRootsResult {.savePath = winner, .fileNames = names, .matchCount = bestCount, .searchedCandidates = true})`.
        *   `[X]` `searchRoots()` with no winner: `findInDir(savePath, names, (forceAppendExt && downloadPath.isEmpty()))` over a copy of `originalFileNames`, then, when `downloadPath` is non-empty, `findInDir(downloadPath, names, forceAppendExt)`, then `promise.addResult(SearchRootsResult {.savePath = (downloadPath.isEmpty() ? savePath : downloadPath), .fileNames = names, .searchedCandidates = searchedCandidates})`, which is `search()`'s result for the same save path, download path and `forceAppendExt`.
        *   `[X]` `candidateRoots()`: for each search root in order, an empty entry takes `defaultSavePath`, and a root that is still empty contributes nothing. A contributing root yields the root form `root`; the name form `root / Path(torrentName)` when `Path(torrentName)` is contained; and the source form `root / Path(sourceFileName).removedExtension(TORRENT_FILE_EXTENSION)` when that stem is contained. A form is appended unless it equals `savePath` or `downloadPath` by `operator==` or is already in the list, so each keeps its earliest position.
        *   `[X]` `isContainedName(name)`, new in the anonymous namespace: true when `name` is non-empty, `name.isRelative()`, and `name.rootItem().data()` is neither `u"."_s` nor `u".."_s`.
        *   `[X]` Exclusion and deduplication use `operator==(const Path &, const Path &)`, through `QList::contains` for deduplication, and so fold case where `Path::CASE_SENSITIVITY` is `Qt::CaseInsensitive`.

    *   `[X]` src/base/bittorrent/`filesearcher.cpp`
        *   `[X]` Add `#include "base/global.h"` after `#include "base/bittorrent/common.h"`.
        *   `[X]` Add `countInDir` and `isContainedName` to the anonymous namespace beside `findInDir`, per the interaction spec.
        *   `[X]` Implement `FileSearcher::searchRoots()` after `FileSearcher::search()`, and `candidateRoots()` after that, per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `filesearcher` depends inward on `base/path.h`, `base/global.h` and `base/bittorrent/common.h`, and on nothing that depends on it. Its consumers, `sessionimpl` and `torrentimpl`, sit beside it in `src/base/bittorrent` and depend on it one way. The tests depend on `qbt_base` only.

    *   `[X]` `requirements`
        *   `[X]` BT-4, CN-1: `testbittorrentfilesearcher` is built as its own target and passes against the untouched `filesearcher` before the header element, and passes again against this node's `filesearcher.cpp`; `search()` keeps its declaration and body.
        *   `[X]` The torrent's own paths compete by count with the search-only candidates, win a tie, and produce the file names `search()` produces for the path that wins: `testSearchRootWithMoreBeatsSavePath`, `testSearchRootWithMoreBeatsDownloadPath`, `testCompleteSearchRootBeatsIncompleteSavePath`, `testSavePathWinsTieWithSearchRoot`, `testForceAppliedAtSavePathWithoutDownloadPath`, `testDownloadPathWithMoreBeatsSavePath`, `testOwnPathsMatchSearchWhenNoSearchRoots`, `testMissFallsBackToDestination`.
        *   `[X]` PS-1, PS-2: `testIncompleteVariantCounts`, `testMostFilesWins`.
        *   `[X]` PS-3: `countInDir` and `searchRoots()` call no function other than `Path::exists()` against the filesystem, directly or through `findInDir`.
        *   `[X]` PS-4: `testMostFilesWins`, `testEarlierRootWinsTie` and `testScoringDoesNotCarryRewrittenNames` prove that selection remains correct when abandonment is allowed. `testOwnPathFullMatchSkipsSearchRoots` proves that scoring stops before any candidate once the save path holds every file. They cannot observe how many `Path::exists()` calls occurred. The implementation review verifies the abandonment condition, the skip of an own path or candidate that does not exist, and the stop once `bestCount` equals the file count directly. No production API or test-only seam is added solely to expose an internal existence-call count.
        *   `[X]` PS-5: `testMostFilesWins`, `testEarlierRootWinsTie`, `testSavePathWinsTieWithSearchRoot`, `testDownloadPathWithMoreBeatsSavePath`, `testSearchRootWithMoreBeatsSavePath`.
        *   `[X]` PS-6, PS-7: `testMissFallsBackToDestination`, `testAbsentFolderCountsNone`.
        *   `[X]` PS-8, PS-9: `testScoringDoesNotCarryRewrittenNames`.
        *   `[X]` PS-11: `testForceNotAppliedAtSearchOnlyWinner`, `testForceAppliedAtSavePathWithoutDownloadPath`.
        *   `[X]` PS-12: `testUnavailableRootContributesNothing`, the Unix-only `testUnreadableRootContributesNothing`, and `testEmptyRootIsSkipped`. On Windows, the permission-denied case is verified manually because changing `QFile` permissions does not reliably deny traversal there.
        *   `[X]` `testNamesUnderPresentFolderCountedEach` and `testAbsentFolderCountsNone` prove the counts for names beneath a top-level folder.
        *   `[X]` CR-1: `candidateRoots()` reads no singleton and no setting; `testbittorrentcandidateroots.cpp` calls it with literals alone.
        *   `[X]` CR-2, CR-4: `testOrderFollowsSearchRoots`, `testOwnPathsAreExcluded`, `testEmptyDownloadPathExcludesNothing`.
        *   `[X]` CR-5: `testEmptySearchRootTakesDefault`, `testEmptySearchRootWithoutDefaultContributesNothing`.
        *   `[X]` CR-6: `testDuplicateKeepsEarliestPosition`, `testSourceFormMatchingNameFormCollapses`, `testDuplicatesFoldCaseOnWindows`.
        *   `[X]` A name form or source form cannot resolve outside its root: `testEscapingNameIsDropped`, `testEscapingSourceIsDropped`.
        *   `[X]` CN-6: `countInDir` and `isContainedName` are private to `filesearcher.cpp`, and `findInDir` is unmodified.
        *   `[X]` BT-3, BT-5, BT-6: the three test executables are registered in `testFiles`; `testbittorrentcandidateroots.cpp` uses no fixture, and its `testPathListComparisonDiscriminates` proves the `PathList` comparison its other slots rely on; the other two resolve fixtures under `test/testdata/filesearcher`.

*   `[X]` [BE] src/base/bittorrent/`sessionimpl` — add the session settings `isFindLocationEnabled()` and `isFindLocationOnAddEnabled()` with their setters, each defaulting to enabled; declare `Session::setWatchedFolderSavePaths()` and hold the paths it receives; add `SessionImpl::findExistingContent()` composing the search-only candidates, searching through `FileSearcher::searchRoots()` and logging the outcome of a search of those candidates with its originating folder; and in `addTorrent_impl()` resolve a manual-mode torrent through it and adopt the resolved location as that torrent's save path, clearing its download path. T4, T5, T6

    *   `[X]` `objective`
        *   `[X]` Problem: `addTorrent_impl()` resolves a torrent's save path through `findIncompleteFiles()`, which searches the save path and the download path alone.
        *   `[X]` Problem: the continuation assigns the resolved location to `p.save_path` only. `TorrentImpl` takes `m_savePath` from `LoadTorrentParams`, and `handleTorrentChecked()` and `handleTorrentFinished()` call `adjustStorageLocation()`, which moves storage to `savePath()` whenever it differs from libtorrent's save path. A location found anywhere other than the torrent's own save path would be moved back to that save path after the first check.
        *   `[X]` Functional: a gate for the feature as a whole and a setting for discovery at torrent add, each a session setting read and written through `Session`, each defaulting to enabled, each held under its own key, so writing the gate never writes the add setting and disabling the gate leaves the add setting's stored value intact.
        *   `[X]` Functional: for a torrent in manual mode, while both settings hold, score the torrent's save path, its download path and the candidates composed from the watched folder save paths together, and resolve the torrent's save path from the directory holding the most of its files, the torrent's own paths winning a tie.
        *   `[X]` Functional: a torrent in automatic mode, and every torrent while either setting is off, resolves exactly as `findIncompleteFiles()` resolves it. Automatic Torrent Management owns an automatic-mode torrent's location, and discovery leaves it alone.
        *   `[X]` Functional: a manual-mode torrent resolved to a search-only candidate takes that location as its save path and has no download path, so `adjustStorageLocation()` targets `savePath()`, finds its storage already there, and moves nothing whether the torrent is complete or not.
        *   `[X]` Functional: when a search-only candidate wins, record the torrent, the location, the folder that location came from and how many files were found; when search-only candidates are probed and no directory holds any file, record that nothing was found. When the torrent's save path or download path wins, record nothing, so torrent add logs what it logs without the feature.
        *   `[X]` Non-functional: composition has one implementation, reached by both call sites; probing stays on the session I/O thread; the caller is never blocked; a setter writes only when the new value differs from the stored one.

    *   `[X]` `role`
        *   `[X]` Application service in `src/base/bittorrent`, owning the session's settings, its add path and the I/O thread that `FileSearcher` runs on.
        *   `[X]` Out of scope: reading `TorrentFilesWatcher`; the metadata-received call site, which is the `torrentimpl` node; the derivation of `actualSavePath` and `actualDownloadPath`; every member of `LoadTorrentParams` other than `savePath` and `downloadPath`; presentation of the settings, which is the `optionsdialog`, `appcontroller` and `preferences.html` nodes.

    *   `[X]` `module`
        *   `[X]` Inside: two persisted booleans and their accessors, holding the watched folder save paths, deciding whether discovery runs, composing the search, adopting the resolved location as the save path with an empty download path, and naming the outcome. Outside: where the watched folder save paths originate, how candidates are built and probed, and what the torrent does once its save path is set.

    *   `[X]` `deps`
        *   `[X]` `filesearcher.h` — `FileSearcher::searchRoots()`, `SearchRootsResult`, `candidateRoots()` and `FileSearchResult`. Beside it in `src/base/bittorrent`, depended on one way, and already included by `sessionimpl.cpp`.
        *   `[X]` `base/settingvalue.h` — `CachedSettingValue`, already included by `sessionimpl.h`, with `BITTORRENT_SESSION_KEY` defined in `sessionimpl.cpp`.
        *   `[X]` `base/global.h` — `TORRENT_FILE_EXTENSION`, already included by `sessionimpl.cpp`.
        *   `[X]` `base/logger.h` — `LogMsg()` and `Log::INFO`. A lower layer, already included by `sessionimpl.cpp`.

    *   `[X]` `context_slice`
        *   `[X]` `setWatchedFolderSavePaths()` is called by the `torrentfileswatcher` node on the main thread, which `SessionImpl` shares, so the held list needs no lock. `findExistingContent()` is called by `addTorrent_impl()` in this node and by `TorrentImpl::handleSaveResumeData()` in the `torrentimpl` node, through its member `SessionImpl *const m_session`.
        *   `[X]` `isUnwantedFolderEnabled()` and `isI2PPeXEnabled()` are the precedent for a session toggle: a `Session` virtual pair, a `CachedSettingValue<bool>` member initialised in the constructor under `BITTORRENT_SESSION_KEY`, and a setter that assigns only on a change. `OptionsDialog` and `AppController` read and write them through `BitTorrent::Session::instance()`.
        *   `[X]` In `addTorrent_impl()`, `loadTorrentParams` is moved into the continuation, which is `mutable`, and from there into the add-alert handler that constructs the `TorrentImpl`. In manual mode `actualSavePath` is `loadTorrentParams.savePath` and `actualDownloadPath` is `loadTorrentParams.downloadPath`.

    *   `[X]` src/base/bittorrent/`session.h`
        *   `[X]` After `virtual void setDownloadPathEnabled(bool enabled) = 0;`, add `virtual void setWatchedFolderSavePaths(const PathList &paths) = 0;`.
        *   `[X]` After `virtual void setUnwantedFolderEnabled(bool enabled) = 0;`, add in order `virtual bool isFindLocationEnabled() const = 0;`, `virtual void setFindLocationEnabled(bool enabled) = 0;`, `virtual bool isFindLocationOnAddEnabled() const = 0;` and `virtual void setFindLocationOnAddEnabled(bool enabled) = 0;`.
        *   `[X]` `SessionImpl` is the only class deriving from `Session`.

    *   `[X]` src/base/bittorrent/`sessionimpl.h`
        *   `[X]` After `void setDownloadPathEnabled(bool enabled) override;`, add `void setWatchedFolderSavePaths(const PathList &paths) override;`.
        *   `[X]` After `void setUnwantedFolderEnabled(bool enabled) override;`, add in order `bool isFindLocationEnabled() const override;`, `void setFindLocationEnabled(bool enabled) override;`, `bool isFindLocationOnAddEnabled() const override;` and `void setFindLocationOnAddEnabled(bool enabled) override;`.
        *   `[X]` After the `findIncompleteFiles()` declaration, in the same `public` section, add `QFuture<FileSearchResult> findExistingContent(const Path &torrentSavePath, const Path &torrentDownloadPath, const PathList &filePaths, const QString &torrentName, const QString &sourceFileName);`.
        *   `[X]` After `CachedSettingValue<bool> m_isUnwantedFolderEnabled;`, add `CachedSettingValue<bool> m_isFindLocationEnabled;` then `CachedSettingValue<bool> m_isFindLocationOnAddEnabled;`.
        *   `[X]` After `FileSearcher *m_fileSearcher = nullptr;`, add `PathList m_watchedFolderSavePaths;`.

    *   `[X]` `interaction.spec`
        *   `[X]` Settings: `m_isFindLocationEnabled` is initialised with `BITTORRENT_SESSION_KEY(u"FindLocation/Enabled"_s)` and `true`, and `m_isFindLocationOnAddEnabled` with `BITTORRENT_SESSION_KEY(u"FindLocation/OnAddEnabled"_s)` and `true`. Each getter returns its member. Each setter assigns its member only when `enabled` differs from it, in the form of `setI2PPeXEnabled()`, and has no other effect.
        *   `[X]` `setWatchedFolderSavePaths(paths)` assigns `paths` to `m_watchedFolderSavePaths`, with no other effect.
        *   `[X]` `findExistingContent()` with `isFindLocationEnabled()` false or `isFindLocationOnAddEnabled()` false returns `findIncompleteFiles(torrentSavePath, torrentDownloadPath, filePaths)`.
        *   `[X]` `findExistingContent()` with both true takes the name form's name `nameFormName` as `filePaths.at(0).removedExtension().toString()` when `filePaths.size() == 1` and `Path::findRootFolder(filePaths).isEmpty()`, and as `torrentName` otherwise. A single-file torrent's name carries its extension, and the stem is the folder qBittorrent itself nests a single file in, through `Path::addRootFolder(filePaths, filePaths.at(0).removedExtension())` in `addTorrent_impl()`, so the name form probes `/{string}/` for the file while the root form probes `{string}.{ext}`.
        *   `[X]` It builds `searchRoots` from `m_watchedFolderSavePaths` in order: an entry that is non-empty and relative becomes `savePath() / entry`; an empty entry passes through unchanged, and `candidateRoots()` gives it `savePath()`. Every entry is resolved against the session default save path. `initLoadTorrentParams()` resolves a watched folder's torrents against `suggestedSavePath(category, useAutoTMM)`, and a folder adding its torrents in automatic mode places them under its category's save path; the search location is the watched folder's configured save path, so neither resolution is applied here.
        *   `[X]` It calls `candidateRoots(torrentSavePath, torrentDownloadPath, searchRoots, savePath(), nameFormName, sourceFileName)` as `candidates`.
        *   `[X]` It reads `const bool appendExtension = isAppendExtensionEnabled();` on the calling thread, then dispatches `m_fileSearcher->searchRoots(filePaths, torrentSavePath, torrentDownloadPath, candidates, appendExtension, promise)` onto the I/O thread through a `QPromise<SearchRootsResult>`, `promise.start()`, `QMetaObject::invokeMethod` and `promise.finish()`, as `findIncompleteFiles()` dispatches `search()`, with `appendExtension` captured by value. It returns `future.then(this, ...)` with the continuation below, capturing `searchRoots`, `savePath()` as `defaultSavePath`, `nameFormName`, `torrentName` and `sourceFileName` by value.
        *   `[X]` The continuation with `result.foundAtOwnPath` false and `result.matchCount` above 0 takes as `root` the first entry of `searchRoots`, with an empty entry read as `defaultSavePath`, for which `candidateRoots({}, {}, {entry}, defaultSavePath, nameFormName, sourceFileName).contains(result.savePath)`, and calls `LogMsg(tr("Found existing torrent content. Torrent: \"%1\". Location: \"%2\". Found in: %3. Files found: %4").arg(torrentName, result.savePath.toString(), tr("watched folder save path \"%1\"").arg(root.toString()), QString::number(result.matchCount)), Log::INFO)`.
        *   `[X]` The continuation with `result.matchCount` equal to 0 and `result.searchedCandidates` true calls `LogMsg(tr("Existing torrent content not found. Torrent: \"%1\". Location: \"%2\"").arg(torrentName, result.savePath.toString()), Log::INFO)`. With `result.foundAtOwnPath` true, or with `result.matchCount` equal to 0 and `result.searchedCandidates` false, it logs nothing.
        *   `[X]` Either continuation returns `FileSearchResult {.savePath = result.savePath, .fileNames = result.fileNames}`.
        *   `[X]` `addTorrent_impl()`, inside `resolveFileNames`: `return findIncompleteFiles(actualSavePath, actualDownloadPath, filePaths);` is taken when `loadTorrentParams.useAutoTMM` is true. Otherwise it returns `findExistingContent(actualSavePath, actualDownloadPath, filePaths, torrentDescr.info()->name(), sourceFileName)`, where `const QString source = torrentDescr.source();`, `const Path sourcePath {source};` and `sourceFileName` is `sourcePath.filename()` when `!source.startsWith(u"magnet:"_s, Qt::CaseInsensitive)` and `sourcePath.hasExtension(TORRENT_FILE_EXTENSION)`, and an empty `QString` otherwise. That line runs only when `needFindIncompleteFiles` is set, which happens only with metadata and without a finished status, so `torrentDescr.info()` holds a value. A `.torrent` added in seed mode keeps the existing bypass through `hasFinishedStatus` and does not enter discovery; widening that gate would change the established skip-check path and is outside this additive node. `TorrentDescriptor::loadFromFile()` sets `source()` to the `.torrent` file's path; `TorrentDescriptor::parse()` sets it to a magnet URI, converting a bare info hash into one, and `AddNewTorrentDialog` and `TorrentsController` attach metadata to such a descriptor through `setTorrentInfo()`, so the scheme test keeps a magnet URI from producing a source form, including a magnet URI whose last parameter ends in `.torrent`. The source form is available only when the existing descriptor source names a local `.torrent` file. Descriptors loaded from an in-memory upload or downloaded byte buffer carry no local source filename and contribute no source form; changing those input pipelines is outside this node.
        *   `[X]` `addTorrent_impl()`, in the continuation, after `p.save_path = result.savePath.toString().toStdString();`: when `loadTorrentParams.useAutoTMM` is false, `result.savePath != loadTorrentParams.savePath` and `result.savePath != loadTorrentParams.downloadPath`, `loadTorrentParams.savePath = result.savePath;` then `loadTorrentParams.downloadPath = Path();`. With the feature off, when the torrent's save path or download path wins, or when nothing is found, the result is the torrent's own save path or its download path, so both assignments are skipped.
        *   `[X]` A manual-mode torrent resolved to a search-only candidate has an empty download path, so `adjustStorageLocation()` targets `savePath()`, the adopted location, before and after the torrent completes, and moves nothing. Discovery creates, moves, renames and deletes nothing.

    *   `[X]` src/base/bittorrent/`sessionimpl.cpp`
        *   `[X]` In the constructor's initialiser list, after `m_isUnwantedFolderEnabled(BITTORRENT_SESSION_KEY(u"UseUnwantedFolder"_s), false)`, initialise `m_isFindLocationEnabled` then `m_isFindLocationOnAddEnabled` per the interaction spec, matching the declaration order.
        *   `[X]` After `SessionImpl::setUnwantedFolderEnabled()`, define the four accessors per the interaction spec, each setter's parameter declared `const bool enabled`.
        *   `[X]` After `SessionImpl::setDownloadPathEnabled()`, define `SessionImpl::setWatchedFolderSavePaths()` per the interaction spec.
        *   `[X]` After `SessionImpl::findIncompleteFiles()`, define `SessionImpl::findExistingContent()` per the interaction spec.
        *   `[X]` In `SessionImpl::addTorrent_impl()`, change the return inside `resolveFileNames` and add the save path and download path assignments in the continuation, per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `sessionimpl` depends on `filesearcher` beside it and on `settingvalue`, `global` and `logger` beneath it, all through includes it already carries. Watched folder save paths arrive through `Session::setWatchedFolderSavePaths()` from `torrentfileswatcher`, which already depends on `Session`, so no dependency on `torrentfileswatcher` is introduced.

    *   `[X]` `requirements`
        *   `[X]` ST-1: the four accessors exist on `Session`, each getter defaulting to `true`, each setter assigning only on a changed value.
        *   `[X]` ST-2: `isFindLocationEnabled()` is its own key, `FindLocation/Enabled` under `BITTORRENT_SESSION_KEY`.
        *   `[X]` ST-3: `FindLocation/OnAddEnabled` is a separate key that no gate accessor writes.
        *   `[X]` CR-9: `findExistingContent()` composes the search roots from the watched folder save paths held through `setWatchedFolderSavePaths()`, the session default save path and the settings, and passes the composed list to `candidateRoots()` for both call sites.
        *   `[X]` CR-7: `searchRoots()` takes the destination as the download path when set and the save path otherwise, from the two paths this node passes it.
        *   `[X]` ST-4, CN-5: with either setting off the result is `findIncompleteFiles()`'s and both the save path and download path assignments are skipped, so torrent add behaves as it does without the feature; a profile written by an earlier version reads both new settings as `true`.
        *   `[X]` A torrent in automatic mode resolves through `findIncompleteFiles()` whatever the settings, and its `LoadTorrentParams` is untouched.
        *   `[X]` RL-1: the resolved save path is assigned in the existing continuation before `async_add_torrent()`, to `p.save_path` and, for a manual-mode torrent resolved to a search-only candidate, to `loadTorrentParams.savePath`, with `loadTorrentParams.downloadPath` emptied.
        *   `[X]` RL-3, RL-4: `searchRoots()` runs on `m_fileSearcher`'s thread, and `findExistingContent()` returns a future.
        *   `[X]` RL-6: nothing in this node creates, moves, renames or deletes a file, and a torrent adopted at a discovered location reports that location as its save path and has no download path, so `adjustStorageLocation()` has nothing to move.
        *   `[X]` PS-10: the continuation assigns the discovered location whatever `matchCount` it carries, with no threshold.
        *   `[X]` LG-1, LG-2, LG-3: one `LogMsg()` per torrent won by a search-only candidate or missed after its search-only candidates were probed, naming the torrent, location, originating folder and count, or recording the miss; none when the torrent's save path or download path wins. Manual verification observes the execution log for one hit, one miss and one add won by the torrent's own save path, checks every required field, and confirms exactly one outcome message for the hit and for the miss, rather than one message per candidate, and none for the add won by the torrent's own save path.
        *   `[X]` A single-file torrent without a root folder is found either as `{string}.{ext}` through the root form or inside `/{string}/` through the name form. Under the Subfolder content layout, `filePaths` carries the stem folder `addTorrent_impl()` adds, so such a torrent's file is found inside `/{string}/` under a candidate and not loose beside it.
        *   `[X]` A magnet-sourced descriptor carrying metadata contributes no source form, including one whose magnet URI ends in `.torrent`.
        *   `[X]` CN-2, CN-3, CN-4: `findIncompleteFiles()`'s declaration and its `search()` call are unchanged, and `actualSavePath` is derived as before.
        *   `[X]` `SessionImpl` cannot be constructed in a `qbt_base` test, so this node is verified by the dependency map's manual cases for steps 6 and 8, including: a manual-mode torrent whose content sits under a watched folder save path keeping that location after its check completes; the same case with a configured download path keeping that location after its check, reporting an empty download path and moving nothing; a manual-mode torrent whose save path holds part of its content while a watched folder save path holds all of it resolving to the watched folder save path; with both settings on and no watched folder configured, torrent add resolving and logging exactly as it does with the feature off; an automatic-mode torrent resolving as it does without the feature; a seed-mode `.torrent` retaining its existing bypass; a local `.torrent` filename contributing its source form while a magnet, including one whose last parameter ends in `.torrent`, and an in-memory upload do not; and the hit and miss logging checks above. On Windows, include a configured root whose traversal is denied and confirm that the remaining roots still resolve normally.

*   `[X]` [BE] src/base/`torrentfileswatcher` — push the configured save path of every watched folder into `Session::setWatchedFolderSavePaths()` whenever the watched folder set changes. T5

    *   `[X]` `objective`
        *   `[X]` Problem: the session composes its candidate root list from the watched folder save paths, and cannot read them from `TorrentFilesWatcher` without depending on a component that already depends on it.
        *   `[X]` Functional: whenever a watched folder is set, replaced or removed, hand the session the configured save path of every watched folder, ordered by watched folder path.
        *   `[X]` Non-functional: the order is deterministic across runs, so ties between watched folders resolve the same way every time.

    *   `[X]` `role`
        *   `[X]` Owner of the watched folder set in `src/base`, already a client of `BitTorrent::Session` through `Session::instance()->addTorrent()` in `TorrentFilesWatcher::onTorrentFound()`.
        *   `[X]` Out of scope: resolving an empty or relative save path, which `SessionImpl::findExistingContent()` and `candidateRoots()` do against the session default save path; watching, scanning or adding torrents; persistence.

    *   `[X]` `module`
        *   `[X]` Inside: turning `m_watchedFolders` into an ordered list of save paths and handing it over. Outside: what the session does with the list.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — `BitTorrent::Session::instance()` and `Session::setWatchedFolderSavePaths()`. Already included by `torrentfileswatcher.cpp`; the direction already exists.
        *   `[X]` `<algorithm>` — `std::ranges::sort`.

    *   `[X]` `context_slice`
        *   `[X]` `Application` initialises `TorrentFilesWatcher` inside its `Session::restored` handler, after `BitTorrent::Session` exists, and frees it before `BitTorrent::Session::freeInstance()`, so `Session::instance()` is valid for every call this node makes. The constructor's `load()` reaches `doSetWatchedFolder()` for each stored folder, as does `loadLegacy()`, so the startup set is pushed by the same path as later changes.
        *   `[X]` `Worker::processFolder()` adds a folder's torrents with `useAutoTMM.value_or(!isAutoTMMDisabledByDefault())` and the folder's category, so a folder in automatic mode places its torrents under its category's save path and carries an empty or unused `addTorrentParams.savePath`.

    *   `[X]` src/base/`torrentfileswatcher.h`
        *   `[X]` In the `private` section, after `void doSetWatchedFolder(const Path &path, const WatchedFolderOptions &options);`, add `void updateSessionWatchedFolderSavePaths() const;`.

    *   `[X]` `interaction.spec`
        *   `[X]` `updateSessionWatchedFolderSavePaths()`: takes `m_watchedFolders.keys()`, sorts them with `std::ranges::sort(folders, {}, &Path::data)`, builds a `PathList` of `m_watchedFolders.value(folder).addTorrentParams.savePath` in that order, empty entries included, and passes it to `BitTorrent::Session::instance()->setWatchedFolderSavePaths()`.
        *   `[X]` The pushed value is each folder's configured save path as the user set it. A folder in automatic mode, or with a category, contributes that configured value and not the category save path its torrents are placed under, and an empty value becomes the session default save path in `candidateRoots()`. The watched folder's save path is the search location the feature names.
        *   `[X]` `doSetWatchedFolder()`: after `m_watchedFolders[path] = options;` and the `invokeMethod` to the worker, and before `emit watchedFolderSet(path, options);`, calls `updateSessionWatchedFolderSavePaths()`. The two `InvalidArgument` throws above it return before any update.
        *   `[X]` `removeWatchedFolder()`: inside the `if (m_watchedFolders.remove(path))` branch, before `emit watchedFolderRemoved(path);`, calls `updateSessionWatchedFolderSavePaths()`. A path that was not watched changes nothing and pushes nothing.

    *   `[X]` src/base/`torrentfileswatcher.cpp`
        *   `[X]` Add `#include <algorithm>` to the standard library group, before `#include <chrono>`.
        *   `[X]` Define `TorrentFilesWatcher::updateSessionWatchedFolderSavePaths()` after `TorrentFilesWatcher::removeWatchedFolder()`, and add its two calls, per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `torrentfileswatcher` depends on `base/bittorrent/session.h` as it already does, and `sessionimpl` gains no dependency on `torrentfileswatcher`. No cycle is introduced.

    *   `[X]` `requirements`
        *   `[X]` CR-9: the session receives the watched folder save paths it composes from, without reading `TorrentFilesWatcher`.
        *   `[X]` Setting, replacing or removing a watched folder in the options dialog changes the set the next torrent add searches. Manual verification performs all three mutations and confirms the next add uses the new set without restarting. It also configures two watched folders whose save paths tie, restarts once, and confirms that sorting by watched folder path makes the same folder win before and after the restart.
        *   `[X]` `TorrentFilesWatcher` constructs a `QFileSystemWatcher`, a worker thread and a `Profile`-relative configuration path, and calls `Session::instance()`, none of which a `qbt_base` test provides, so this node is verified manually.

*   `[X]` [BE] src/base/bittorrent/`torrentimpl` — when metadata arrives, resolve a manual-mode torrent through `SessionImpl::findExistingContent()` and record the resolved location through `setSavePath()`, with its download path cleared, before `endReceivedMetadataHandling()`, so a torrent added from a magnet link reaches discovery before content pieces are requested and keeps the location discovery found. T7

    *   `[X]` `objective`
        *   `[X]` Problem: a torrent added from a magnet link has no file list at add, so the add-time search passes it by; when its metadata arrives, `handleSaveResumeData()` resolves through `findIncompleteFiles()`, which searches the torrent's save path and download path alone.
        *   `[X]` Problem: `endReceivedMetadataHandling()` assigns the resolved location to `p.save_path` only, leaving `m_savePath` as configured, so `adjustStorageLocation()` would move discovered content back to the configured save path after the check.
        *   `[X]` Functional: when metadata arrives, resolve a manual-mode torrent through the same discovery the add path uses, with the torrent's name and no source file name; resolve an automatic-mode torrent through `findIncompleteFiles()` as before.
        *   `[X]` Functional: a manual-mode torrent resolved to a search-only candidate records that location as its save path and has no download path.
        *   `[X]` Non-functional: resolution completes before `endReceivedMetadataHandling()` calls `reload()`, so no content piece is requested against an undiscovered location; recording the save path touches no file.

    *   `[X]` `role`
        *   `[X]` Torrent lifecycle in `src/base/bittorrent`.
        *   `[X]` Out of scope: composition, settings and logging, which `SessionImpl::findExistingContent()` performs in the `sessionimpl` node; the bodies of `setSavePath()`, `moveStorage()`, `adjustStorageLocation()` and `endReceivedMetadataHandling()`.

    *   `[X]` `module`
        *   `[X]` Inside: the metadata-received call site and its continuation. Outside: everything `findExistingContent()` does, and what `setSavePath()` does with the path it receives.

    *   `[X]` `deps`
        *   `[X]` `sessionimpl.h` — `SessionImpl::findExistingContent()` and `SessionImpl::findIncompleteFiles()`, reached through `SessionImpl *const m_session`, already included by `torrentimpl.cpp`.
        *   `[X]` `filesearcher.h` — `FileSearchResult`, already included by `torrentimpl.cpp`.
        *   `[X]` `TorrentImpl::isAutoTMMEnabled()`, `TorrentImpl::downloadPath()` and `TorrentImpl::setSavePath()`, public members used as they stand, and the private member `m_downloadPath`, assigned directly in the continuation — the same class.

    *   `[X]` `context_slice`
        *   `[X]` The call site is in `TorrentImpl::handleSaveResumeData()`, in the branch taken when `m_maintenanceJob` is `MaintenanceJob::HandleMetadata` and `params.ti` is set. There `metadata` is `TorrentInfo(*m_ltAddTorrentParams.ti)`, and `filePaths` carries the content layout and the renamed files applied just before the call.
        *   `[X]` `hasMetadata()` returns `m_torrentInfo.isValid()`, and `m_torrentInfo` is assigned in `endReceivedMetadataHandling()`, so it is false while the continuation runs. `setSavePath()` asserts manual mode and returns when the resolved path equals `savePath()`. Otherwise, a torrent whose download path is empty reaches `moveStorage()` with `MoveStorageContext::ChangeSavePath`, whose `!hasMetadata()` branch assigns `m_savePath` and calls `handleTorrentSavePathChanged()` with no storage job. `setDownloadPath()` does not clear the download path here: with an empty path and no metadata it passes `savePath()` to `moveStorage()` with `MoveStorageContext::ChangeDownloadPath`, whose `!hasMetadata()` branch stores that save path as the download path.

    *   `[X]` `interaction.spec`
        *   `[X]` In that branch, `m_session->findIncompleteFiles(savePath(), downloadPath(), filePaths)` becomes `(isAutoTMMEnabled() ? m_session->findIncompleteFiles(savePath(), downloadPath(), filePaths) : m_session->findExistingContent(savePath(), downloadPath(), filePaths, metadata.name(), {}))`, whose `.then(this, ...)` takes the existing continuation. The source file name is empty because a torrent reaching this path came from a magnet link and has no `.torrent` file, so its source form is omitted.
        *   `[X]` In the continuation, inside `if (m_maintenanceJob == MaintenanceJob::HandleMetadata)` and before `endReceivedMetadataHandling(result.savePath, result.fileNames);`: when `isAutoTMMEnabled()` is false, `result.savePath != savePath()` and `result.savePath != downloadPath()`, `m_downloadPath = Path();` then `setSavePath(result.savePath);`. With the download path empty, `setSavePath()` reaches `moveStorage()` with `MoveStorageContext::ChangeSavePath`, which assigns `m_savePath` with no storage job. With the feature off, when the torrent's save path or download path wins, or when nothing is found, the result is the torrent's own save path or its download path, so neither assignment is made.

    *   `[X]` src/base/bittorrent/`torrentimpl.cpp`
        *   `[X]` Change the call and its continuation per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `torrentimpl` depends on `sessionimpl` and `filesearcher` beside it, through includes it already carries.

    *   `[X]` `requirements`
        *   `[X]` RL-2: a torrent without metadata at add is resolved when metadata arrives, and `endReceivedMetadataHandling()` assigns `p.save_path` before `reload()` resumes it.
        *   `[X]` RL-6: `setSavePath()` runs while `hasMetadata()` is false and schedules no storage job, and the adopted torrent reports the discovered location as its save path and has no download path, so `adjustStorageLocation()` has nothing to move.
        *   `[X]` ST-4: with either setting off, `findExistingContent()` returns `findIncompleteFiles()`'s result, so this path behaves as before.
        *   `[X]` An automatic-mode torrent resolves through `findIncompleteFiles()` and `setSavePath()` is not called for it.
        *   `[X]` PS-10: `endReceivedMetadataHandling()` assigns the discovered location whatever proportion is present, with no threshold.
        *   `[X]` `TorrentImpl` requires a live session, so this node is verified by the dependency map's manual case for step 7, including a magnet-added manual-mode torrent whose content sits under a watched folder save path keeping that location after its check completes, and the same case with a configured download path keeping that location after its check, reporting an empty download path and moving nothing.

*   `[X]` [UI] src/gui/`optionsdialog` — add the checkable **Find location** group box `groupFindLocation` to the Downloads page, holding the **Find location automatically** checkbox `checkFindLocationOnAdd`, each loaded from and saved to `BitTorrent::Session`. T8

    *   `[X]` `objective`
        *   `[X]` Problem: the two settings the session consults have no desktop control.
        *   `[X]` Functional: a checkable group box whose checked state is `Session::isFindLocationEnabled()`, holding a checkbox whose state is `Session::isFindLocationOnAddEnabled()`. A change to either enables **Apply**, and both are written when the page is applied.
        *   `[X]` Functional: unchecking the group disables the checkbox it holds and leaves that checkbox's value as the user set it.
        *   `[X]` Non-functional: both labels are translatable, and both controls are standard widgets reachable by keyboard.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/gui`.
        *   `[X]` Out of scope: the web interface, which is the `appcontroller` and `preferences.html` nodes; the second epic's checkboxes.

    *   `[X]` `module`
        *   `[X]` Inside: two widgets, their loading, their saving and their enabling of **Apply**. Outside: what either setting causes.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — `isFindLocationEnabled()`, `setFindLocationEnabled()`, `isFindLocationOnAddEnabled()` and `setFindLocationOnAddEnabled()` from the `sessionimpl` node. A lower layer, depended on inward, already included by `optionsdialog.cpp`.
        *   `[X]` `ui_optionsdialog.h` — generated from `optionsdialog.ui`, providing `m_ui->groupFindLocation` and `m_ui->checkFindLocationOnAdd`.

    *   `[X]` `context_slice`
        *   `[X]` `OptionsDialog::loadDownloadsTabOptions()` reads through `const auto *session = BitTorrent::Session::instance();` and connects `ThisType::enableApplyButton` in the same function; `OptionsDialog::saveDownloadsTabOptions()` writes through `auto *session = BitTorrent::Session::instance();`. `checkUnwantedFolder` is loaded and saved through `session` in those functions. `groupExcludedFileNames` is the checkable group box precedent, loaded from and saved to `session`, and connected with `&QGroupBox::toggled`.

    *   `[X]` src/gui/`optionsdialog.ui`
        *   `[X]` In `verticalLayout`, the `QVBoxLayout` of `scrollAreaWidgetContents_2`, add an `<item>` after the item holding `checkRecursiveDownload` and before the item holding `groupSavingManagement`.
        *   `[X]` The item holds `<widget class="QGroupBox" name="groupFindLocation">` with property `title` set to `Find location`, property `checkable` set to `true` and property `checked` set to `true`, laid out by a `QVBoxLayout` named `groupFindLocationLayout`, as `groupExcludedFileNames` is built.
        *   `[X]` That layout holds one item, `<widget class="QCheckBox" name="checkFindLocationOnAdd">`, with property `text` set to `Find location automatically` and property `toolTip` set to `When a torrent is added in Manual mode, or its metadata is received, look for its files in its save path, its download path and the save paths of the watched folders, and use the location holding the most of them.`

    *   `[X]` `interaction.spec`
        *   `[X]` Load: after `m_ui->checkRecursiveDownload->setChecked(pref->isRecursiveDownloadEnabled());`, `m_ui->groupFindLocation->setChecked(session->isFindLocationEnabled());` then `m_ui->checkFindLocationOnAdd->setChecked(session->isFindLocationOnAddEnabled());`.
        *   `[X]` Enable **Apply**: after the `checkRecursiveDownload` connection, `connect(m_ui->groupFindLocation, &QGroupBox::toggled, this, &ThisType::enableApplyButton);` then `connect(m_ui->checkFindLocationOnAdd, &QAbstractButton::toggled, this, &ThisType::enableApplyButton);`.
        *   `[X]` Save: after `pref->setRecursiveDownloadEnabled(m_ui->checkRecursiveDownload->isChecked());`, `session->setFindLocationEnabled(m_ui->groupFindLocation->isChecked());` then `session->setFindLocationOnAddEnabled(m_ui->checkFindLocationOnAdd->isChecked());`.
        *   `[X]` An unchecked checkable `QGroupBox` disables its children with no code. The checkbox keeps its checked state while disabled, and saving writes that state, so the group gate never overwrites the add setting.

    *   `[X]` src/gui/`optionsdialog.cpp`
        *   `[X]` Add the six statements per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `optionsdialog` in `src/gui` depends on `BitTorrent::Session` in `src/base`, downward, as it already does.

    *   `[X]` `requirements`
        *   `[X]` IF-1: `groupFindLocation` is a checkable group box titled **Find location**, placed among the download options.
        *   `[X]` IF-12: `uic` routes both `.ui` strings through `tr()`.
        *   `[X]` ST-3: unchecking `groupFindLocation` disables `checkFindLocationOnAdd` without changing its state, and applying writes both states unchanged.
        *   `[X]` The dialog is verified by the dependency map's manual case for step 8: unchecking the group returns torrent add to its two-directory behaviour.

*   `[X]` [API] src/webui/api/`appcontroller` — expose `find_location_enabled` and `find_location_on_add_enabled` on `app/preferences` and `app/setPreferences`, read from and written to `BitTorrent::Session`. T9

    *   `[X]` `objective`
        *   `[X]` Problem: the two settings are reachable only from the desktop dialog, so the headless daemon and remote clients can neither read nor change them.
        *   `[X]` Functional: `app/preferences` returns both as booleans; `app/setPreferences` sets each when its key is present and leaves it untouched when its key is absent.

    *   `[X]` `role`
        *   `[X]` WebAPI controller in `src/webui/api`.
        *   `[X]` Out of scope: the web page, which is the `preferences.html` node; the changelog, which is the `WebAPI_Changelog` node; `API_VERSION`, which the maintainers set.

    *   `[X]` `module`
        *   `[X]` Inside: two keys in two actions. Outside: what either setting causes, and how either is presented.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — the four accessors from the `sessionimpl` node, already included by `appcontroller.cpp`.

    *   `[X]` `context_slice`
        *   `[X]` `AppController::preferencesAction()` reads through `const auto *session`. `AppController::setPreferencesAction()` writes through `auto *session` and tests each key with the `hasKey` lambda, which positions `it` on the key it finds. `use_unwanted_folder`, read and written through `session`, closes the group of adding options in each action, immediately before the `// Saving Management` comment.

    *   `[X]` `interaction.spec`
        *   `[X]` `preferencesAction()`: both keys are always present in the returned object, each holding the stored boolean.
        *   `[X]` `setPreferencesAction()`: a present key → the matching setter with `it.value().toBool()`; an absent key → no call, so the stored value stands.

    *   `[X]` src/webui/api/`appcontroller.cpp`
        *   `[X]` In `preferencesAction()`, after `data[u"use_unwanted_folder"_s] = session->isUnwantedFolderEnabled();`, add `data[u"find_location_enabled"_s] = session->isFindLocationEnabled();` then `data[u"find_location_on_add_enabled"_s] = session->isFindLocationOnAddEnabled();`.
        *   `[X]` In `setPreferencesAction()`, after the `use_unwanted_folder` branch, add `if (hasKey(u"find_location_enabled"_s))` followed by `session->setFindLocationEnabled(it.value().toBool());`, then `if (hasKey(u"find_location_on_add_enabled"_s))` followed by `session->setFindLocationOnAddEnabled(it.value().toBool());`, each setter call on the line after its condition, as the neighbouring branches are written.

    *   `[X]` `directionality`
        *   `[X]` `appcontroller` in `src/webui` depends on `BitTorrent::Session` in `src/base`, downward, as it already does.

    *   `[X]` `requirements`
        *   `[X]` IF-4: both settings are exposed on `app/preferences` and `app/setPreferences`.
        *   `[X]` ST-3: each key writes only its own setting, so setting `find_location_enabled` never writes `find_location_on_add_enabled`.
        *   `[X]` CN-5: a request omitting either key leaves that setting unchanged.
        *   `[X]` The endpoints are verified by the dependency map's manual case for step 9.

*   `[X]` [UI] src/webui/www/private/views/`preferences.html` — add the **Find location** fieldset to the Downloads tab, its legend checkbox `findLocationCheckbox` gating the `findLocationOnAddCheckbox` it holds, each loaded from and saved to `find_location_enabled` and `find_location_on_add_enabled`. T9

    *   `[X]` `objective`
        *   `[X]` Problem: the two WebAPI keys have no control in the web interface, so a user of the headless daemon can set them only by request.
        *   `[X]` Functional: a fieldset whose legend checkbox presents `find_location_enabled` and which holds a checkbox presenting `find_location_on_add_enabled`, both loaded when the page loads and both sent when it is saved.
        *   `[X]` Functional: unchecking the legend checkbox disables the inner checkbox, whose value is still sent as the user set it.
        *   `[X]` Non-functional: both labels pass through `QBT_TR` in the `OptionsDialog` context with the same source strings as the desktop dialog, as the neighbouring labels on the page do. The web interface translates them through its own `src/webui/www/translations/webui_*.ts` files, which are separate from the desktop's `src/lang` files, so each label is translated there independently of the desktop translation.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/webui/www`.
        *   `[X]` Out of scope: the keys and their endpoints, which are the `appcontroller` node; the translation files, which the project's translation process maintains.

    *   `[X]` `module`
        *   `[X]` Inside: the fieldset markup, one enablement handler and its export, and the load and save statements. Outside: what either setting causes.

    *   `[X]` `deps`
        *   `[X]` `app/preferences`, returning `find_location_enabled` and `find_location_on_add_enabled` from the `appcontroller` node, read as `pref.find_location_enabled` and `pref.find_location_on_add_enabled`.
        *   `[X]` `app/setPreferences`, receiving `settings["find_location_enabled"]` and `settings["find_location_on_add_enabled"]`.

    *   `[X]` `context_slice`
        *   `[X]` The Downloads tab is `<div id="DownloadsTab" class="PrefTab invisible">`. The fieldset holding `excludedFileNamesCheckbox` is the legend checkbox precedent. `updateTorrentBackupControls`, called in the load function after its checkbox is loaded, is the precedent for refreshing enablement on load. The object returned by `exports` publishes handlers named in `onclick` attributes.

    *   `[X]` `interaction.spec`
        *   `[X]` `updateFindLocationEnabled()` sets `document.getElementById("findLocationOnAddCheckbox").disabled` to the negation of `document.getElementById("findLocationCheckbox").checked`. It runs from the legend checkbox's `onclick`, and once after both checkboxes are loaded.

    *   `[X]` src/webui/www/private/views/`preferences.html`
        *   `[X]` Markup: after the `formRow` holding `unwantedfolder_checkbox` and its blank line, and before the **Saving Management** fieldset, add a `<fieldset class="settings">` whose `<legend>` holds `<input type="checkbox" id="findLocationCheckbox" onclick="qBittorrent.Preferences.updateFindLocationEnabled();">` and `<label id="findLocationLabel" for="findLocationCheckbox">QBT_TR(Find location)QBT_TR[CONTEXT=OptionsDialog]</label>`, followed by a `<div class="formRow">` holding `<input type="checkbox" id="findLocationOnAddCheckbox" title="QBT_TR(When a torrent is added in Manual mode, or its metadata is received, look for its files in its save path, its download path and the save paths of the watched folders, and use the location holding the most of them.)QBT_TR[CONTEXT=OptionsDialog]">` and `<label for="findLocationOnAddCheckbox">QBT_TR(Find location automatically)QBT_TR[CONTEXT=OptionsDialog]</label>`, then a blank line. Indentation follows the `excludedFileNamesCheckbox` fieldset.
        *   `[X]` Handler: after the definition of `updateExcludedFileNamesEnabled`, define `const updateFindLocationEnabled = () => { ... };` per the interaction spec, in the form of `updateExcludedFileNamesEnabled`.
        *   `[X]` Export: after `updateExcludedFileNamesEnabled: updateExcludedFileNamesEnabled,` in the object `exports` returns, add `updateFindLocationEnabled: updateFindLocationEnabled,`.
        *   `[X]` Load: after `document.getElementById("unwantedfolder_checkbox").checked = pref.use_unwanted_folder;`, add `document.getElementById("findLocationCheckbox").checked = pref.find_location_enabled;`, then `document.getElementById("findLocationOnAddCheckbox").checked = pref.find_location_on_add_enabled;`, then `updateFindLocationEnabled();`.
        *   `[X]` Save: after `settings["use_unwanted_folder"] = document.getElementById("unwantedfolder_checkbox").checked;`, add `settings["find_location_enabled"] = document.getElementById("findLocationCheckbox").checked;`, then `settings["find_location_on_add_enabled"] = document.getElementById("findLocationOnAddCheckbox").checked;`.

    *   `[X]` `directionality`
        *   `[X]` The page consumes the WebAPI and adds no dependency beyond the two keys the `appcontroller` node provides.

    *   `[X]` `requirements`
        *   `[X]` IF-5: both settings have a control on the web interface preferences page.
        *   `[X]` IF-12: both labels are wrapped in `QBT_TR` in the `OptionsDialog` context.
        *   `[X]` ST-3: a disabled `findLocationOnAddCheckbox` keeps its `checked` state and is sent unchanged.
        *   `[X]` The `CI - WebUI` workflow's `npm run lint` passes, and `npm run format` in `src/webui/www` leaves the file unchanged.
        *   `[X]` The page is verified by the dependency map's manual case for step 9.

*   `[X]` [DOCS] `WebAPI_Changelog` — record `find_location_enabled` and `find_location_on_add_enabled` on `app/preferences` and `app/setPreferences` under the version heading at the top of the file. T9

    *   `[X]` `objective`
        *   `[X]` Problem: WebAPI clients learn of new preference keys from this file, and two keys join both preference endpoints.
        *   `[X]` Functional: one entry, under the version heading at the top of the file when the branch is rebased, linking this submission's pull request and naming both keys on both endpoints. `API_VERSION` and the version headings are the maintainers' to set.

    *   `[X]` `context_slice`
        *   `[X]` The `#24870` entry, adding `i2p_pex_enabled`, added a preference key to both endpoints without changing `API_VERSION` or opening a heading, and is the wording model. Entries under a heading run newest first.

    *   `[X]` `WebAPI_Changelog.md`
        *   `[X]` Under the heading at the top of the file, before its first entry, insert the entry below, separated from its neighbours as the existing entries are.
        *   `[X]` The entry is a top-level bullet linking the pull request, `* [#<number>](https://github.com/qbittorrent/qBittorrent/pull/<number>)`, where `<number>` is the number GitHub assigns when this submission's pull request is opened.
        *   `[X]` Under it, the indented bullet ``* `app/preferences` endpoint includes `find_location_enabled` (bool) and `find_location_on_add_enabled` (bool) options``.
        *   `[X]` Under it, the indented bullet ``* `app/setPreferences` endpoint allows to set `find_location_enabled` (bool) and `find_location_on_add_enabled` (bool) options``.

    *   `[X]` `requirements`
        *   `[X]` IF-6: the submission adds one `WebAPI_Changelog.md` entry naming the keys it introduces, under the heading at the top of the file, and leaves `API_VERSION` unchanged.
        *   `[X]` The file passes the `rumdl` pre-commit hook.

*   `[X]` **Commit** `Find existing torrent content before downloading`
    *   `[X]` Structural: `SearchRootsResult`, `FileSearcher::searchRoots()` and `candidateRoots()` in `filesearcher`; four `Session` accessors backed by session settings, `Session::setWatchedFolderSavePaths()` and `SessionImpl::findExistingContent()`; `TorrentFilesWatcher::updateSessionWatchedFolderSavePaths()`; `groupFindLocation` and `checkFindLocationOnAdd` in the options dialog; the web interface **Find location** fieldset; three test executables and their fixtures.
    *   `[X]` Behavioural: while both settings are enabled, torrent add and metadata arrival resolve a manual-mode torrent by scoring its save path, its download path and the watched folder save paths together, choosing the location holding the most files with its own paths winning a tie, adopting a watched folder location as the torrent's save path with no download path, and logging the outcome with its originating folder; every other torrent resolves and logs as before.
    *   `[X]` Contract: `FileSearcher::search()` and `SessionImpl::findIncompleteFiles()` keep their signatures and behaviour; `FileSearcher::searchRoots()` scores the torrent's own paths alongside the search-only candidates; `app/preferences` and `app/setPreferences` gain two keys, recorded in `WebAPI_Changelog.md` with `API_VERSION` unchanged.
    *   `[X]` The pull request description states that `searchRoots()` probes the winning directory twice, once through `countInDir()` and once through the unchanged `findInDir()`, so that `findInDir()` is untouched, and offers a single pass that records the winner's names while counting as an alternative for the maintainers to choose.
    *   `[X]` The contributor commits once the manual cases for steps 6 through 9 have passed and the full build and suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows.

## Epic 2 — Start-triggered, manual, batch and assignment

*   `[X]` [BE] src/base/bittorrent/`torrentimpl`, `session` and `sessionimpl` — preserve the existing Start, Stop, metadata, movement and check bodies; add one Start interception call and one private Stop-origin overload in `TorrentImpl`; add the fourth Find Location setting and the manual discovery operations to `Session`; factor `searchExistingContent()`; and extend one `SessionImpl` assignment map with the exact state needed to hold, cancel and release a pending Start. T10, T11, T12

    *   `[X]` `objective`
        *   `[X]` Problem: discovery is reachable only from `SessionImpl`, which `src/gui` does not hold, so a torrent already in the session cannot be located on demand; assigning a location through the `Torrent` interface queues a storage move that a recheck issued at once would overtake; and `TorrentImpl::start()` currently clears an error and immediately leaves the stopped state or reloads missing files without giving Find Location an interception point.
        *   `[X]` Functional: for one torrent named by its ID, run discovery through the same composition the add path uses, and report the location it selected and whether any file was found there through a signal carrying that torrent's ID.
        *   `[X]` Functional: for one torrent named by its ID and a location, call the existing `setAutoTMMEnabled(false)` and `setSavePath(location)` operations in that order, as **Set location...** does; do not change either operation.
        *   `[X]` Functional: **Find location when starting stopped torrents**, **Recheck automatically**, **Seed automatically** and **Leech automatically** are session settings, each defaulting to enabled, persisting under its own key and writing only on a changed value.
        *   `[X]` Functional: while **Recheck automatically** is enabled, stop the torrent and recheck it at the assigned location, after its storage move where one is queued and at once where none is.
        *   `[X]` Functional: when a torrent held as awaiting a start decision reports its check, stop holding it and start it in auto-managed mode where its content is complete and **Seed automatically** is enabled, or where its content is incomplete and **Leech automatically** is enabled; otherwise leave it stopped at its location.
        *   `[X]` Functional: a second assignment to the location already held in the feature-owned assignment map is coalesced into the active assignment and performs no second location change or recheck; a second assignment to a different location replaces the active assignment.
        *   `[X]` Functional: an eligible explicit Start is held before `m_hasMissingFiles`, `m_isStopped` or native pause/resume state changes; a match is assigned and successfully rechecked before the saved normal or forced Start is released; a miss releases that Start without assignment; repeated Start updates the saved mode without duplicate work; Stop, removal, shutdown and every feature failure erase the pending Start before a late callback can act.
        *   `[X]` Non-functional: discovery probes on the session I/O thread and never blocks its caller; no feature branch creates, allocates or writes a payload file before a successful miss or successful feature recheck; assignment changes no implementation of the torrent location operations or the storage move queue; and every added lifecycle branch is guarded by an entry in `m_locationAssignments`.

    *   `[X]` `role`
        *   `[X]` `SessionImpl` is the feature owner because it already owns the torrent registry, the File Searcher, settings and the existing callbacks from `TorrentImpl`. `TorrentImpl` supplies only the two narrow entry points that cannot be implemented after the fact: pre-payload Start interception and distinguishing public Stop from its own stop-after-check call.
        *   `[X]` Out of scope: choosing which torrents to locate, presenting an outcome, and asking the user for a location, which are the `transferlistwidget`, `unmatchedtorrentsdialog`, `torrentscontroller`, `mocha-init.js`, `setlocation.html` and `unmatchedtorrents.html` nodes; changing `TorrentImpl::setAutoTMMEnabled()`, `setSavePath()`, `setDownloadPath()`, `moveStorage()`, `forceRecheck()` or the move queue; adding a general lifecycle coordinator; and repairing force recheck outside a feature-owned transaction.

    *   `[X]` `module`
        *   `[X]` Inside: four persisted booleans and their accessors; two `TorrentImpl` calls into `SessionImpl`; discovery for an existing torrent; one map entry holding phase, target, optional Start mode, one-shot Start pass and search token; exact transitions in the existing metadata, movement, check, error, removal and shutdown hooks. Outside: selection and presentation, candidate construction and probing, the move queue, and general torrent lifecycle behavior.

    *   `[X]` `deps`
        *   `[X]` `filesearcher.h` — `SearchRootsResult`, `FileSearcher::searchRoots()`, `candidateRoots()` and `FileSearchResult`. Beside it in `src/base/bittorrent`, already included by `sessionimpl.cpp`.
        *   `[X]` `torrentimpl.h` — the public methods `TorrentImpl::setAutoTMMEnabled()`, `setSavePath()`, `setDownloadPath()`, `downloadPath()`, `actualStorageLocation()`, `stop()`, `start()`, `forceRecheck()`, `nativeHandle()`, `isFinished()`, `isChecking()`, `hasMetadata()`, `savePath()`, `filePaths()`, `info()` and `id()`. Beside it, already included by `sessionimpl.cpp`. Do not call private `TorrentImpl::isMoveInProgress()` from `SessionImpl`.
        *   `[X]` `base/settingvalue.h` — `CachedSettingValue`, already included by `sessionimpl.h`, with `BITTORRENT_SESSION_KEY` defined in `sessionimpl.cpp`.

    *   `[X]` `context_slice`
        *   `[X]` `TorrentImpl::setSavePath()` and `TorrentImpl::setDownloadPath()` use the existing storage move queue. `moveStorage()` marks the torrent moving when it enqueues a job, and `TorrentImpl::handleMoveStorageJobFinished()` calls `SessionImpl::handleTorrentStorageMovingStateChanged()` once no move is outstanding. A failed move reports the torrent's current location through the same completion path.
        *   `[X]` The existing queue handles overlapping location operations without a feature-specific path: it cancels an inactive queued job for the same torrent before appending the new job, lets an active job finish before a different destination queued behind it, and refuses a duplicate of the active destination. The feature calls the existing torrent setters and never mutates `m_moveStorageQueue`. It reads the queue only through `isTorrentStorageMoving()`, because `TorrentImpl::isMoving()` reports the derived state, in which `CheckingResumeData` takes precedence over `Moving`, so a torrent reloading after its metadata arrives reports no move while its move runs.
        *   `[X]` `TorrentImpl::forceRecheck()` starts a stopped torrent with `StopCondition::FilesChecked`, so `TorrentImpl::handleTorrentChecked()` stops it again when the check completes and then, in its status-updated trigger, calls `SessionImpl::handleTorrentChecked()`, which emits `torrentFinishedChecking`.
        *   `[X]` `forceRecheck()` sets `m_nativeStatus.state` to `checking_resume_data` before its internal `start()` call. The plan nevertheless uses an explicit `LocationStartPass::Recheck` because that pass distinguishes the internal call from a repeated explicit Start and lets TX-6 update the latter's saved mode.
        *   `[X]` `TorrentImpl::setSavePath()` on an incomplete torrent with a non-empty download path records the save path without moving storage. `setDownloadPath({})` then moves that torrent's active storage to its save path. Start-triggered assignment uses this exact pair when necessary; manual assignment retains the existing single-`setSavePath()` semantics.
        *   `[X]` `findTorrentLocation()` and `assignTorrentLocation()` are called on the main thread by the `transferlistwidget`, `unmatchedtorrentsdialog` and `torrentscontroller` nodes through `BitTorrent::Session::instance()`, and `torrentLocationFound` is emitted on the main thread from a continuation whose context is `this`.
        *   `[X]` A metadata-less Start first stores its entry under the current ID. Under libtorrent 2, `TorrentImpl::handleMetadataReceived()` calls `SessionImpl::handleTorrentInfoHashChanged()` before the metadata pipeline later calls `handleTorrentMetadataReceived()`; the former must therefore re-key the waiting entry before the latter starts its search.

    *   `[X]` src/base/bittorrent/`torrentimpl.h`
        *   `[X]` In `private`, after `void reload();`, add `void stop(bool cancelPendingFindLocationStart);`. Do not change the public `void stop() override;` or `void start(TorrentOperatingMode mode = TorrentOperatingMode::AutoManaged) override;` declarations.

    *   `[X]` src/base/bittorrent/`torrentimpl.cpp`
        *   `[X]` Replace the current `TorrentImpl::stop()` body with `stop(true);`. Define `TorrentImpl::stop(const bool cancelPendingFindLocationStart)` immediately after it. At the top of that overload, when the argument is true, call `m_session->cancelFindLocationStart(this);`; after that call, copy the former public `stop()` body without changing its statements or order.
        *   `[X]` In `TorrentImpl::handleTorrentChecked()`, change only the `stop();` under `if (stopCondition() == StopCondition::FilesChecked)` to `stop(false);`. This preserves the existing stop-after-check behavior without cancelling the feature entry immediately before `SessionImpl::handleTorrentChecked()` consumes it.
        *   `[X]` In `TorrentImpl::start(const TorrentOperatingMode mode)`, leave the existing `hasError()` block and `m_operatingMode = mode;` unchanged. Immediately after `m_operatingMode = mode;` and before `if (m_hasMissingFiles)`, insert `if (m_session->interceptFindLocationStart(this, mode))` followed by `return;`. Do not move or rewrite the missing-files, stopped-state, auto-managed or forced-resume blocks.

    *   `[X]` src/base/bittorrent/`session.h`
        *   `[X]` After `virtual void setFindLocationOnAddEnabled(bool enabled) = 0;`, add in order `virtual bool isFindLocationOnStartEnabled() const = 0;`, `virtual void setFindLocationOnStartEnabled(bool enabled) = 0;`, `virtual bool isFindLocationRecheckEnabled() const = 0;`, `virtual void setFindLocationRecheckEnabled(bool enabled) = 0;`, `virtual bool isFindLocationSeedEnabled() const = 0;`, `virtual void setFindLocationSeedEnabled(bool enabled) = 0;`, `virtual bool isFindLocationLeechEnabled() const = 0;` and `virtual void setFindLocationLeechEnabled(bool enabled) = 0;`.
        *   `[X]` After `virtual void bottomTorrentsQueuePos(const QList<TorrentID> &ids) = 0;`, add `virtual void findTorrentLocation(const TorrentID &id) = 0;` then `virtual void assignTorrentLocation(const TorrentID &id, const Path &location) = 0;`.
        *   `[X]` In `signals`, after `void torrentFinishedChecking(Torrent *torrent);`, add `void torrentLocationFound(const TorrentID &id, const Path &location, bool found);`.

    *   `[X]` src/base/bittorrent/`sessionimpl.h`
        *   `[X]` After `struct FileSearchResult;`, add `struct SearchRootsResult;`.
        *   `[X]` After `void setFindLocationOnAddEnabled(bool enabled) override;`, add in order `bool isFindLocationOnStartEnabled() const override;`, `void setFindLocationOnStartEnabled(bool enabled) override;`, `bool isFindLocationRecheckEnabled() const override;`, `void setFindLocationRecheckEnabled(bool enabled) override;`, `bool isFindLocationSeedEnabled() const override;`, `void setFindLocationSeedEnabled(bool enabled) override;`, `bool isFindLocationLeechEnabled() const override;` and `void setFindLocationLeechEnabled(bool enabled) override;`.
        *   `[X]` After `void bottomTorrentsQueuePos(const QList<TorrentID> &ids) override;`, add `void findTorrentLocation(const TorrentID &id) override;` then `void assignTorrentLocation(const TorrentID &id, const Path &location) override;`.
        *   `[X]` In the public `// Torrent interface` block, immediately before `handleTorrentResumeDataRequested`, add `bool interceptFindLocationStart(TorrentImpl *torrent, TorrentOperatingMode mode);` and `void cancelFindLocationStart(TorrentImpl *torrent);`. These are concrete `SessionImpl` hooks used only by `TorrentImpl`; do not add them to `Session`.
        *   `[X]` In the `private` section, after `void handleTorrentFinishedAlert(const lt::torrent_finished_alert *alert);`, add `QFuture<SearchRootsResult> searchExistingContent(const Path &torrentSavePath, const Path &torrentDownloadPath, const PathList &filePaths, const QString &torrentName, const QString &sourceFileName);`.
        *   `[X]` After `searchExistingContent()`, declare, in order, `void searchStartedTorrentLocation(TorrentImpl *torrent, quint64 token);`, `void assignStartedTorrentLocation(TorrentImpl *torrent, const Path &location);`, `void forceLocationAssignmentRecheck(TorrentImpl *torrent);`, `void releaseFindLocationStart(TorrentImpl *torrent, bool matched);` and `void failFindLocationStart(TorrentImpl *torrent, const QString &phase, const QString &reason);` and `bool isTorrentStorageMoving(const TorrentImpl *torrent) const;`.
        *   `[X]` After `CachedSettingValue<bool> m_isFindLocationOnAddEnabled;`, add `CachedSettingValue<bool> m_isFindLocationOnStartEnabled;`, `CachedSettingValue<bool> m_isFindLocationRecheckEnabled;`, `CachedSettingValue<bool> m_isFindLocationSeedEnabled;` and `CachedSettingValue<bool> m_isFindLocationLeechEnabled;` in that order.
        *   `[X]` Immediately before `struct MoveStorageJob`, add `enum class LocationAssignmentPhase { WaitingForMetadata, Searching, WaitingForMove, WaitingForCheck };`, `enum class LocationStartPass { None, MetadataOnly, Recheck, Terminal };`, then `struct LocationAssignmentState { Path location; LocationAssignmentPhase phase = LocationAssignmentPhase::Searching; std::optional<TorrentOperatingMode> startMode; LocationStartPass startPass = LocationStartPass::None; quint64 token = 0; };`. Keep one declaration per line in the actual header.
        *   `[X]` After `PathList m_watchedFolderSavePaths;`, add `QHash<TorrentID, LocationAssignmentState> m_locationAssignments;` and `quint64 m_nextLocationAssignmentToken = 0;`.

    *   `[X]` `interaction.spec`
        *   `[X]` Settings: initialise, in order, `m_isFindLocationOnStartEnabled(BITTORRENT_SESSION_KEY(u"FindLocation/OnStartEnabled"_s), true)`, `m_isFindLocationRecheckEnabled(BITTORRENT_SESSION_KEY(u"FindLocation/RecheckEnabled"_s), true)`, `m_isFindLocationSeedEnabled(BITTORRENT_SESSION_KEY(u"FindLocation/SeedEnabled"_s), true)` and `m_isFindLocationLeechEnabled(BITTORRENT_SESSION_KEY(u"FindLocation/LeechEnabled"_s), true)`. Each getter returns its named member. Each setter returns when its argument equals that getter and otherwise assigns the argument only to the named member.
        *   `[X]` `searchExistingContent()` carries what `findExistingContent()` performs while both settings hold, as the first epic's `sessionimpl` node specifies: the name form's name `nameFormName`; `searchRoots` built from `m_watchedFolderSavePaths`; `candidateRoots(torrentSavePath, torrentDownloadPath, searchRoots, savePath(), nameFormName, sourceFileName)` as `candidates`; `const bool appendExtension = isAppendExtensionEnabled();` read on the calling thread; `m_fileSearcher->searchRoots(filePaths, torrentSavePath, torrentDownloadPath, candidates, appendExtension, promise)` dispatched onto the I/O thread; and the continuation logging the outcome. Its continuation returns the `SearchRootsResult` unchanged, and it returns that continuation's future.
        *   `[X]` `findExistingContent()` with either setting off returns `findIncompleteFiles(torrentSavePath, torrentDownloadPath, filePaths)`; with both on it returns `searchExistingContent(torrentSavePath, torrentDownloadPath, filePaths, torrentName, sourceFileName).then(this, ...)`, whose continuation returns `FileSearchResult {.savePath = result.savePath, .fileNames = result.fileNames}`.
        *   `[X]` `findTorrentLocation(id)` with no torrent in `m_torrents` under `id`, or a torrent without metadata → `emit torrentLocationFound(id, {}, false)`.
        *   `[X]` `findTorrentLocation(id)` with a torrent holding metadata → `searchExistingContent(torrent->savePath(), torrent->downloadPath(), torrent->filePaths(), torrent->info().name(), {})`, continued with `.then(this, ...)` into `emit torrentLocationFound(id, result.savePath, ((result.matchCount > 0) || result.foundAtOwnPath))`.
        *   `[X]` `interceptFindLocationStart(torrent, mode)` first looks up `torrent->id()`. If the entry has a non-`None` `startPass`, save the pass, set it to `None`, erase the entry only when the saved pass is `Terminal`, and return `false`; this is the only bypass of interception.
        *   `[X]` With no pass and an existing entry whose `startMode` is present, set `entry.startMode = mode` and return `true` before testing stopped or checking state. Do not change its phase, target or token and issue no work; this is the repeated-Start coalescing branch during every transaction phase, including the internally running feature recheck.
        *   `[X]` Otherwise return `false` when `!isFindLocationEnabled()`, `!isFindLocationOnStartEnabled()`, `!torrent->isStopped()`, `torrent->isAutoTMMEnabled()` or `torrent->isChecking()`. `isChecking()` reads the native state, which `forceRecheck()` sets to `checking_resume_data` before its internal `start()`, so a recheck of a stopped torrent is never intercepted; a stopped torrent that has never been checked and holds files at its own path is in `checking_files` and also falls through, to libtorrent's own check of those files. No state is created or changed in these branches, so the remainder of the existing `TorrentImpl::start()` executes unchanged. When an eligible manual-assignment entry exists without `startMode`, set `startMode = mode` and return `true` without changing that entry's phase, target or token.
        *   `[X]` With an eligible torrent and no entry, increment `m_nextLocationAssignmentToken`, insert a state with `startMode = mode` and that token, and hold the outer Start by returning `true`. If `torrent->hasMetadata()`, set `phase = Searching` and call `searchStartedTorrentLocation(torrent, token)`. Otherwise set `phase = WaitingForMetadata`, set `startPass = MetadataOnly`, call `torrent->start(mode)`, then call `torrent->setStopCondition(Torrent::StopCondition::MetadataReceived)`; the nested Start consumes the pass and only the existing metadata-acquisition path runs.
        *   `[X]` `searchStartedTorrentLocation(torrent, token)` copies `const TorrentID id = torrent->id()` and calls `searchExistingContent(torrent->savePath(), torrent->downloadPath(), torrent->filePaths(), torrent->info().name(), {})`. Its success continuation captures only `id` and `token`, obtains `TorrentImpl *const currentTorrent = m_torrents.value(id)`, and returns unless that pointer is non-null and the map contains `id` with the same token, `phase == Searching` and a `startMode`.
        *   `[X]` In that valid continuation, `const bool found = result.foundAtOwnPath || (result.matchCount > 0);`. A false value calls `releaseFindLocationStart(currentTorrent, false)`. A true value stores `result.savePath` in the entry and calls `assignStartedTorrentLocation(currentTorrent, result.savePath)`. On the future returned by that `then(this, ...)`, attach a no-argument `onFailed(this, ...)` capturing only `id` and `token`; repeat the pointer, token, phase and Start-intent validation, then call `failFindLocationStart(currentTorrent, u"search"_s, tr("the asynchronous search did not complete"))`.
        *   `[X]` `assignStartedTorrentLocation(torrent, location)` never copies payload data itself. If `torrent->actualStorageLocation() == location`, call `forceLocationAssignmentRecheck(torrent)`. Otherwise call `torrent->setAutoTMMEnabled(false)`. When `location != torrent->savePath()`, next call `torrent->setSavePath(location)`. If `!torrent->isFinished() && !torrent->downloadPath().isEmpty() && (torrent->actualStorageLocation() != location)` after that call, call `torrent->setDownloadPath({})`; this makes the newly recorded save path the active storage location through the existing `ChangeDownloadPath` move. Then set `phase = WaitingForMove` when `isTorrentStorageMoving(torrent)`; if `!isTorrentStorageMoving(torrent)` and the actual location equals the target, call `forceLocationAssignmentRecheck(torrent)`; otherwise call `failFindLocationStart(torrent, u"assignment"_s, tr("the selected location did not become the torrent's storage location"))`.
        *   `[X]` `assignTorrentLocation(id, location)` with an entry for `id` in `m_locationAssignments` recording the same `location` → return. An entry for `id` recording a different location → erase it and continue.
        *   `[X]` `assignTorrentLocation(id, location)` with no torrent in `m_torrents` under `id`, a torrent without metadata, `location == torrent->savePath()` or `location == torrent->downloadPath()` → return.
        *   `[X]` `assignTorrentLocation(id, location)` with a torrent holding metadata → `torrent->setAutoTMMEnabled(false)`, then `torrent->setSavePath(location)`.
        *   `[X]` For manual assignment, call `torrent->stop()` before inserting a new map entry. With `isFindLocationRecheckEnabled()` false, return after the two location calls. With it true and `isTorrentStorageMoving(torrent)`, insert `{.location = location, .phase = LocationAssignmentPhase::WaitingForMove}`. With it true, `!isTorrentStorageMoving(torrent)` and `torrent->actualStorageLocation() != location`, return without a recheck as the pre-existing incomplete-download-path rule requires. Otherwise insert `{.location = location, .phase = LocationAssignmentPhase::WaitingForCheck}`, then call `forceLocationAssignmentRecheck(torrent)`.
        *   `[X]` `isTorrentStorageMoving(torrent)`: `true` exactly while `m_moveStorageQueue` holds a job whose `torrentHandle` equals `torrent->nativeHandle()`, tested with `std::ranges::any_of`. A job enters the queue in the call that marks the torrent moving and leaves it before `handleTorrentStorageMovingStateChanged()` runs, so the test matches the torrent's move flag whatever state the torrent reports.
        *   `[X]` `forceLocationAssignmentRecheck(torrent)`: require an existing entry; set its phase to `WaitingForCheck` and its `startPass` to `Recheck`, then call `torrent->forceRecheck()`. The internal `start()` consumes `Recheck`, retains the entry and follows the unchanged Start body. This function is used for both manual assignment and pending Start, so neither can recursively start a second search.
        *   `[X]` `handleTorrentStorageMovingStateChanged(torrent)`: after `emit torrentsUpdated({torrent});`, return unless the entry is `WaitingForMove` and `!isTorrentStorageMoving(torrent)`. When settled at `state.location`, call `forceLocationAssignmentRecheck(torrent)`. On a different actual location, call `failFindLocationStart(torrent, u"movement"_s, tr("the storage move did not reach the selected location"))` if `startMode` is present; otherwise erase the manual-assignment entry.
        *   `[X]` `handleTorrentChecked(torrent)`: before `emit torrentFinishedChecking(torrent);`, return from feature handling unless the entry is `WaitingForCheck` and `torrent->isStopped()`. The feature's recheck starts a stopped torrent under `StopCondition::FilesChecked`, and `TorrentImpl::handleTorrentChecked()` calls `stop(false)` before this callback runs, so the feature's own check always reports a stopped torrent. A check the feature did not issue, such as the one libtorrent runs when a magnet torrent reloads after its metadata arrives, completes while the torrent is running and does not advance the entry. For an entry with Start intent, `torrent->hasError()` calls `failFindLocationStart(torrent, u"recheck"_s, torrent->error())`; movement or actual-location mismatch calls it with `u"recheck"_s` and `tr("the completed check did not report the selected location")`. Otherwise call `releaseFindLocationStart(torrent, true)`. For an entry without Start intent, erase it on error or location mismatch; otherwise start in `TorrentOperatingMode::AutoManaged` only under the existing complete/Seed or incomplete/Leech decisions, arming `LocationStartPass::Terminal` before that `start()` call, and erase it without starting when neither decision holds.
        *   `[X]` `releaseFindLocationStart(torrent, matched)`: copy the required `startMode`; set the entry's `startPass = Terminal`; log exactly one INFO outcome, `Find location Start continued after a match. Torrent: "%1". Location: "%2"` when `matched` or `Find location Start continued after no match. Torrent: "%1"` otherwise; then call `torrent->start(copiedMode)`. The nested interceptor consumes `Terminal`, erases the entry and lets the unchanged Start body execute once.
        *   `[X]` `cancelFindLocationStart(torrent)`: when the entry exists and has `startMode`, erase it and log `Find location Start cancelled. Torrent: "%1"` at INFO; otherwise do nothing. Because public `stop()` calls this even for an already stopped torrent, Stop cancels searching and waiting-for-metadata states too.
        *   `[X]` `failFindLocationStart(torrent, phase, reason)`: when the entry has no `startMode`, erase it and return. Otherwise erase it first, log `Find location Start failed. Torrent: "%1". Phase: %2. Reason: "%3"` at WARNING, then call public `torrent->stop()`. Erasing first prevents that Stop's cancellation hook from producing a second terminal log and clears `StopCondition::FilesChecked` through the existing stop body when the failed feature recheck had internally started the torrent.
        *   `[X]` `handleTorrentMetadataReceived(torrent)`: immediately after `emit torrentMetadataReceived(torrent);` and before the torrent-backup early return, find the entry. Only `WaitingForMetadata` with a `startMode` advances: set `phase = Searching` and call `searchStartedTorrentLocation(torrent, state.token)`.
        *   `[X]` `handleTorrentInfoHashChanged(torrent, prevInfoHash)`: inside `if (currentID != prevID)`, after re-keying `m_torrents` and `m_changedTorrentIDs`, find `prevID` in `m_locationAssignments`; when present, move its value to `currentID` and erase the old key. No other phase changes.
        *   `[X]` `handleFileErrorAlert()`: immediately after `torrent->handleFileError(...)`, define `const QString errorMessage = QString::fromStdString(alert->message());`; use `errorMessage` in the existing log and `torrentIOError` emission instead of the block-local `msg`. After `m_recentErroredTorrentsTimer->start();`, if the entry is `WaitingForCheck` with Start intent, call `failFindLocationStart(torrent, u"recheck"_s, errorMessage)`.
        *   `[X]` `handleSaveResumeDataFailedAlert()`: inside the existing `alert->error != lt::errors::resume_data_not_modified` block, materialize the same error text the existing version-specific log expression uses as `const QString errorMessage`; retain the existing CRITICAL log with that variable, then call `failFindLocationStart(torrent, u"metadata"_s, errorMessage)` only for a `WaitingForMetadata` entry with Start intent.
        *   `[X]` `handleStorageMovedFailedAlert()`: after the existing WARNING log and before `handleMoveTorrentStorageJobFinished(currentLocation)`, call `failFindLocationStart(torrent, u"movement"_s, errorMessage)` only when `torrent` is non-null and its entry is `WaitingForMove` with Start intent. Manual entries retain the existing logging and are erased when `handleMoveTorrentStorageJobFinished(currentLocation)` reaches the guarded movement callback with the mismatched path.
        *   `[X]` `removeTorrent(id, deleteOption)`: after `const TorrentID torrentID = torrent->id();`, call `cancelFindLocationStart(torrent);` then `m_locationAssignments.remove(torrentID)`. The first call supplies the pending Start's terminal cancellation log; the second also removes a manual-assignment entry.
        *   `[X]` `SessionImpl::~SessionImpl()`: before `m_nativeSession->pause();`, iterate the entries with `startMode`, obtain each still-live torrent from `m_torrents`, and log `Find location Start cancelled. Torrent: "%1"` at INFO once for each; then call `m_locationAssignments.clear();`. Leave the existing native-session pause, saving, worker draining and teardown sequence unchanged. Search continuations are context-bound to `this` and their ID/token guard makes results already queued before teardown inert.

    *   `[X]` src/base/bittorrent/`sessionimpl.cpp`
        *   `[X]` In the constructor's initialiser list, after `m_isFindLocationOnAddEnabled(BITTORRENT_SESSION_KEY(u"FindLocation/OnAddEnabled"_s), true)`, add the four complete initializers written in `interaction.spec`, in OnStart, recheck, seed and leech order.
        *   `[X]` After `SessionImpl::setFindLocationOnAddEnabled()`, define `isFindLocationOnStartEnabled()`/`setFindLocationOnStartEnabled()`, `isFindLocationRecheckEnabled()`/`setFindLocationRecheckEnabled()`, `isFindLocationSeedEnabled()`/`setFindLocationSeedEnabled()` and `isFindLocationLeechEnabled()`/`setFindLocationLeechEnabled()` in that order. Each getter returns its same-named cached member; each setter takes `const bool enabled`, returns when `enabled == getter()`, and otherwise assigns `enabled` to that member.
        *   `[X]` After `SessionImpl::findExistingContent()`, define `SessionImpl::searchExistingContent()`, moving into it the enabled branch of `findExistingContent()` per the interaction spec, and reduce `findExistingContent()` to its two branches per the interaction spec.
        *   `[X]` After `SessionImpl::bottomTorrentsQueuePos()`, define `SessionImpl::findTorrentLocation()`, `SessionImpl::assignTorrentLocation()` and `SessionImpl::forceLocationAssignmentRecheck()` per the interaction spec, each public operation finding its torrent through `m_torrents.value(id)`.
        *   `[X]` Define the five private transaction helpers after the three public location operations. Extend `handleTorrentMetadataReceived()`, `handleTorrentInfoHashChanged()`, `handleTorrentStorageMovingStateChanged()`, `handleTorrentChecked()`, the three named failure handlers, `removeTorrent()` and `~SessionImpl()` at the exact anchors and with the exact guards above.

    *   `[X]` `directionality`
        *   `[X]` `sessionimpl` depends on `filesearcher` and `torrentimpl` beside it and on `settingvalue` beneath it, through includes it already carries. `src/gui` reaches it only through `Session`; `TorrentImpl` already holds `SessionImpl *const m_session` and adds only the two concrete calls named above. No GUI or WebUI type enters `src/base`, and no new component owns torrent lifecycle state.

    *   `[X]` `requirements`
        *   `[X]` ST-1, ST-3, ST-9, CN-5: the eight Epic 2 accessors exist on `Session`; their four separate `FindLocation/...Enabled` keys default to `true`; no existing key, resume datum or save path is renamed or re-read.
        *   `[X]` IF-11: `findTorrentLocation()` and `assignTorrentLocation()` are declared on `Session`.
        *   `[X]` IF-13: `findTorrentLocation()` returns `void` and reports one torrent's outcome through `torrentLocationFound`, emitted as that torrent's search completes.
        *   `[X]` CR-9: `findExistingContent()` and `findTorrentLocation()` both compose through `searchExistingContent()`.
        *   `[X]` AS-1: `setAutoTMMEnabled(false)` precedes `setSavePath()`.
        *   `[X]` AS-2: for assignment without pending Start, `forceRecheck()` is issued only when `isFindLocationRecheckEnabled()` holds; for pending Start it is mandatory. Both paths issue it only at the target actual location.
        *   `[X]` AS-3: assignment invokes the existing torrent location operations without changing their implementations or the move queue, and a failed move erases the feature state without rechecking the prior location.
        *   `[X]` AS-4, AS-8: the start decision is taken in `handleTorrentChecked()` only for a `WaitingForCheck` state installed immediately before the feature's own `forceRecheck()`, keyed by `TorrentID` and carrying the assigned location.
        *   `[X]` AS-5, AS-6, AS-7: `start()` is called in its default auto-managed mode, under **Seed automatically** for complete content and **Leech automatically** for incomplete content.
        *   `[X]` TX-1, TX-2: the sole Start insertion is after error clearing and mode recording and before missing-files reload or stopped-state changes; every disabled and ineligible branch returns false without state, and an eligible outer Start returns before the original body can perform payload work.
        *   `[X]` TX-3, TX-4, TX-5: a match reaches release only from `handleTorrentChecked()` in `WaitingForCheck`; a miss reaches release only from the successful search continuation; both use the saved explicit mode through `LocationStartPass::Terminal`.
        *   `[X]` TX-6: an entry and token are created once; another eligible Start changes only `startMode`; the `MetadataOnly`, `Recheck` and `Terminal` passes identify all feature-owned recursive Start calls and are consumed once.
        *   `[X]` TX-7, TX-8: public Stop, removal, teardown, stale-token search results and the named failure hooks cannot release the pending Start; the recheck error path erases before ordinary Stop and accepts only `handleTorrentChecked()` as success.
        *   `[X]` LG-4: exactly one terminal transaction log is emitted by release, cancellation or failure; stale callbacks emit none.
        *   `[X]` PS-10, CN-7: the location is kept at any completion, no threshold is applied, and starting and checking pass through the session's queueing and `MaxActiveCheckingTorrents` limits.
        *   `[X]` RL-3, RL-4, RL-6: discovery probes on `m_fileSearcher`'s thread, `findTorrentLocation()` returns before the probe completes, and discovery itself creates, moves, renames and deletes nothing.
        *   `[X]` LG-1, LG-2: `searchExistingContent()` logs as the first epic's `sessionimpl` node specifies, from both call sites.
        *   `[X]` `SessionImpl` cannot be constructed in a `qbt_base` test without a new application-session fixture. Verify every branch above through the individually enumerated Epic 2 integration scenarios at the commit boundary; do not add a test-only session abstraction in this submission.

*   `[X]` [UI] src/gui/`unmatchedtorrentsdialog` — add `UnmatchedTorrentsDialog`, listing torrents that matched nothing and walking them one at a time into a **Choose save path** dialog that assigns each chosen location through `Session::assignTorrentLocation()`, or closing to abandon the rest. T14

    *   `[X]` `objective`
        *   `[X]` Problem: a batch that matches nothing for several torrents would otherwise open one file dialog per torrent in succession, with no way to see the set, choose the order, or stop.
        *   `[X]` Functional: list the unmatched torrents by name; for the selected entry, open a directory dialog starting at its save path, and on an existing directory assign that location and remove the entry; close once no entry remains.
        *   `[X]` Functional: closing the dialog abandons the remaining entries, assigning nothing further.
        *   `[X]` Non-functional: every string is translatable, and the list and its buttons are reachable and operable by keyboard.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/gui`, a window-modal dialog opened with `open()` by the `transferlistwidget` node.
        *   `[X]` Out of scope: discovery, assignment and the start decision, which the `sessionimpl` node performs behind `assignTorrentLocation()`; deciding which torrents are unmatched.

    *   `[X]` `module`
        *   `[X]` Inside: the list of unmatched torrent IDs, their rows, the directory dialog and the hand-off of a chosen location. Outside: what assignment does.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — `BitTorrent::Session::instance()`, `Session::getTorrent()`, `Session::assignTorrentLocation()` and the existing `torrentAboutToBeRemoved` signal. A lower layer, depended on inward.
        *   `[X]` `base/bittorrent/torrent.h` — `Torrent::name()` and `Torrent::savePath()`.
        *   `[X]` `base/bittorrent/infohash.h` — `BitTorrent::TorrentID`.
        *   `[X]` `base/path.h` — `Path`, `Path::exists()` and `Path::data()`.
        *   `[X]` `ui_unmatchedtorrentsdialog.h` — generated from `unmatchedtorrentsdialog.ui`.

    *   `[X]` `context_slice`
        *   `[X]` `TransferListWidget::setSelectedTorrentsLocation()` is the model for the directory dialog: a heap `QFileDialog` titled `tr("Choose save path")` with `Qt::WA_DeleteOnClose`, `QFileDialog::Directory`, `QFileDialog::DontConfirmOverwrite | QFileDialog::ShowDirsOnly | QFileDialog::HideNameFilterDetails`, `&QDialog::accepted` reading `selectedFiles().constFirst()` into a `Path` that must exist, and `open()`. `TorrentCategoryDialog` is the model for the class: a forward-declared `Ui` class, a raw `m_ui` created in the constructor's initialiser list and deleted in the destructor, and the generated header included by the `.cpp`.

    *   `[X]` src/gui/`unmatchedtorrentsdialog.h`
        *   `[X]` New file: the licence header of `torrentcategorydialog.h` naming the contributor, `#pragma once`, `#include <QDialog>`, `#include <QList>` and `#include "base/bittorrent/infohash.h"`, then `namespace Ui { class UnmatchedTorrentsDialog; }`.
        *   `[X]` `class UnmatchedTorrentsDialog final : public QDialog` with `Q_OBJECT` and `Q_DISABLE_COPY_MOVE(UnmatchedTorrentsDialog)`, declaring public `UnmatchedTorrentsDialog(QWidget *parent, const QList<BitTorrent::TorrentID> &torrentIDs);`, `~UnmatchedTorrentsDialog() override;` and `bool isEmpty() const;`, private `void setCurrentTorrentLocation();` and `void removeTorrent(const BitTorrent::TorrentID &id);`, and private members `Ui::UnmatchedTorrentsDialog *m_ui = nullptr;` and `QList<BitTorrent::TorrentID> m_torrentIDs;`.

    *   `[X]` src/gui/`unmatchedtorrentsdialog.ui`
        *   `[X]` New file in the form of `deletionconfirmationdialog.ui`: class `UnmatchedTorrentsDialog`, a `QDialog` named `UnmatchedTorrentsDialog` with `windowTitle` `Find location`, laid out by a `QVBoxLayout` named `verticalLayout` holding three items in order.
        *   `[X]` A `QLabel` named `labelUnmatched` with `text` `No existing content was found for these torrents. Set a location for each, or close to leave them where they are.` and `wordWrap` `true`.
        *   `[X]` A `QListWidget` named `listTorrents`.
        *   `[X]` A `QDialogButtonBox` named `buttonBox` with `orientation` `Qt::Orientation::Horizontal` and `standardButtons` `QDialogButtonBox::StandardButton::Close`.

    *   `[X]` `construction`
        *   `[X]` The constructor takes the parent and the unmatched IDs and leaves the dialog complete: every row populated, the first row current, and every connection made. The caller sets `Qt::WA_DeleteOnClose` and calls `open()`.

    *   `[X]` `interaction.spec`
        *   `[X]` Constructor: `m_ui->setupUi(this)`; for each ID in order, `BitTorrent::Session::instance()->getTorrent(id)` non-null → append the ID to `m_torrentIDs` and add a row to `listTorrents` holding `torrent->name()`; null → skip it. Then `listTorrents->setCurrentRow(0)`.
        *   `[X]` Constructor: store `m_ui->buttonBox->addButton(tr("Set location..."), QDialogButtonBox::ActionRole)` in a local `QPushButton *setLocationButton`; connect its `&QAbstractButton::clicked` to `setCurrentTorrentLocation`, `&QListWidget::itemDoubleClicked` to `setCurrentTorrentLocation`, and `&QDialogButtonBox::rejected` to `&QDialog::reject`. Connect `&QListWidget::currentRowChanged` to a lambda with `this` as its context that sets `setLocationButton` enabled exactly when the row is at least 0, then set its initial enabled state from `listTorrents->currentRow()`.
        *   `[X]` Constructor: connect `BitTorrent::Session::torrentAboutToBeRemoved` to a lambda with `this` as its context that calls `removeTorrent(torrent->id())`. This removes a live dialog row before its torrent becomes invalid and closes the dialog when that was its final row.
        *   `[X]` `isEmpty()` returns `m_torrentIDs.isEmpty()`.
        *   `[X]` `setCurrentTorrentLocation()`: `currentRow()` below 0 → return. The ID at that row with no torrent under it in the session → `removeTorrent(id)` and return. Otherwise open the directory dialog per the context slice, starting at `torrent->savePath().data()`.
        *   `[X]` Connect the directory dialog's `&QDialog::accepted` signal with `this` as the context and capture the dialog pointer and ID. In the handler, the selected `Path` does not exist → return, the entry kept. It exists and `m_torrentIDs` still holds the ID → `BitTorrent::Session::instance()->assignTorrentLocation(id, newLocation)` then `removeTorrent(id)`.
        *   `[X]` `removeTorrent(id)`: `m_torrentIDs.indexOf(id)` below 0 → return. Otherwise remove and delete that row from `listTorrents` with `takeItem()`, remove the ID from `m_torrentIDs`, and call `accept()` when the list becomes empty.
        *   `[X]` `&QDialog::rejected`, from **Close**, the window's close control or Escape, closes the dialog with no further assignment.

    *   `[X]` src/gui/`unmatchedtorrentsdialog.cpp`
        *   `[X]` New file: the licence header of `torrentcategorydialog.cpp` naming the contributor, `#include "unmatchedtorrentsdialog.h"`, then `#include <QFileDialog>`, `#include <QListWidget>` and `#include <QPushButton>`, then `#include "base/bittorrent/session.h"`, `#include "base/bittorrent/torrent.h"`, `#include "base/path.h"` and `#include "ui_unmatchedtorrentsdialog.h"`.
        *   `[X]` Define the constructor, the destructor deleting `m_ui`, `isEmpty()`, `setCurrentTorrentLocation()` and `removeTorrent()` per the interaction spec.

    *   `[X]` src/gui/`CMakeLists.txt`
        *   `[X]` Add `unmatchedtorrentsdialog.ui` to `qt_wrap_ui(UI_HEADERS ...)` after `uithemedialog.ui`, `unmatchedtorrentsdialog.h` to the headers of `add_library(qbt_gui ...)` after `uithemesource.h`, and `unmatchedtorrentsdialog.cpp` to its sources after `uithemesource.cpp`.

    *   `[X]` `directionality`
        *   `[X]` `unmatchedtorrentsdialog` in `src/gui` depends downward on `session`, `torrent` and `path` in `src/base`. Only `transferlistwidget` depends on it.

    *   `[X]` `requirements`
        *   `[X]` IF-9: unmatched torrents are presented as a list the user walks one entry at a time or abandons with **Close**.
        *   `[X]` IF-11: the dialog assigns through `Session::assignTorrentLocation()`.
        *   `[X]` IF-12: `uic` routes the `.ui` strings through `tr()`, and the button text is wrapped in `tr()`.
        *   `[X]` BT-2: the three files are registered in `src/gui/CMakeLists.txt`.
        *   `[X]` `listTorrents` activates a row with a double click, and every control, including the **Set location...** button, is reachable by Tab.
        *   `[X]` No empty dialog is opened or left open: the transfer-list node filters removed torrents before choosing the fallback and deletes a newly constructed `UnmatchedTorrentsDialog` without opening it when `isEmpty()` is true; the dialog removes a row on `torrentAboutToBeRemoved` and accepts itself when no row remains.
        *   `[X]` Verify the dialog through Epic 2 integration scenario 5 at the commit boundary.

*   `[X]` [UI] src/gui/`transferlistwidget` — add **Find location** beside **Set location...** while the **Find location** group is enabled, run `Session::findTorrentLocation()` for each selected torrent holding metadata, assign each match through `Session::assignTorrentLocation()` as its outcome arrives, and, once every outcome has arrived, open **Choose save path** for a single miss or `UnmatchedTorrentsDialog` for several. T13, T14

    *   `[X]` `objective`
        *   `[X]` Problem: a torrent already in the session reaches discovery only through the add path, so a user holding torrents placed at the wrong location repairs each by hand.
        *   `[X]` Functional: a context menu action over one or many selected torrents that runs discovery for each torrent holding metadata and assigns every location found, as each outcome arrives.
        *   `[X]` Functional: while an operation is active, another invocation adds only previously unseen torrent IDs to it; an ID already pending, matched or unmatched is not searched again. Once no submitted torrent remains pending, one valid torrent without a match opens the **Choose save path** dialog for it, more than one opens `UnmatchedTorrentsDialog`, and zero opens nothing.
        *   `[X]` Non-functional: the transfer list stays interactive while outcomes arrive; the action is absent while the **Find location** group is disabled.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/gui`, the transfer list and its context menu.
        *   `[X]` Out of scope: discovery, assignment, rechecking and the start decision, which are the `sessionimpl` node; the unmatched list, which is the `unmatchedtorrentsdialog` node.

    *   `[X]` `module`
        *   `[X]` Inside: the action, one operation map holding each submitted torrent's result state, merging another invocation into that operation, and dispatching misses. Outside: how a location is found and what assignment does.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — `Session::findTorrentLocation()`, `Session::assignTorrentLocation()`, `Session::getTorrent()` and the `torrentLocationFound` signal, from the `sessionimpl` node. Already included by `transferlistwidget.cpp`.
        *   `[X]` `unmatchedtorrentsdialog.h` — `UnmatchedTorrentsDialog`, from the `unmatchedtorrentsdialog` node.

    *   `[X]` `context_slice`
        *   `[X]` `TransferListWidget::displayListMenu()` constructs every action before deciding which to add, computing `oneHasMetadata` over the selection, and adds `actionSetTorrentPath` after the separator that follows `actionDelete`. `setSelectedTorrentsLocation()` is the model for the directory dialog. The mnemonic letters of the menu's other top-level entries leave `i` free.

    *   `[X]` src/gui/`transferlistwidget.h`
        *   `[X]` Add `#include <QHash>` after `#include <QtContainerFwd>`.
        *   `[X]` In `public slots`, after `void setSelectedTorrentsLocation();`, add `void findSelectedTorrentsLocation();`.
        *   `[X]` In the `private` section, after `void exportTorrent();`, add `void handleTorrentLocationFound(const BitTorrent::TorrentID &id, const Path &location, bool found);` and `void askTorrentLocation(const BitTorrent::TorrentID &id);`.
        *   `[X]` Find the `private:` label directly above `void dragEnterEvent(QDragEnterEvent *event) override;`. On the line directly after that label, insert `enum class FindLocationState { Pending, Matched, Unmatched };`, followed by one blank line. The existing declarations follow unchanged.
        *   `[X]` After `TransferListSortModel *m_sortFilterModel = nullptr;`, add `QHash<BitTorrent::TorrentID, FindLocationState> m_findLocationOperation;`. An empty map is no active operation; non-empty states remain in the map until that operation finalises.

    *   `[X]` `interaction.spec`
        *   `[X]` Constructor: after the `sortIndicatorChanged` connection, `connect(BitTorrent::Session::instance(), &BitTorrent::Session::torrentLocationFound, this, &TransferListWidget::handleTorrentLocationFound);`.
        *   `[X]` `displayListMenu()`: `actionFindLocation` is a `QAction` with `UIThemeManager::instance()->getIcon(u"edit-find"_s)` and `tr("F&ind location")`, constructed after `actionSetTorrentPath` and connected to `findSelectedTorrentsLocation`. After `listMenu->addAction(actionSetTorrentPath);`, `oneHasMetadata` true and `BitTorrent::Session::instance()->isFindLocationEnabled()` true → `listMenu->addAction(actionFindLocation)`; otherwise it is not added.
        *   `[X]` `findSelectedTorrentsLocation()`: create a local `QList<TorrentID> submitted`. In a first pass over `getSelectedTorrents()`, no metadata → skip; its ID already in `m_findLocationOperation` → skip; otherwise insert the ID into `m_findLocationOperation` with `FindLocationState::Pending` and append it to `submitted`. In a second pass over `submitted`, call `BitTorrent::Session::instance()->findTorrentLocation(id)` for each ID. Register the complete invocation before the first call so a synchronous invalid-torrent outcome cannot finalise and clear the operation while later IDs from the same invocation remain unregistered. A call made while the map is non-empty adds its unseen IDs to that operation.
        *   `[X]` `handleTorrentLocationFound(id, location, found)`: no map entry for `id`, or its state is not `Pending` → return. `found` true → set the state to `Matched` then call `BitTorrent::Session::instance()->assignTorrentLocation(id, location)`; `found` false → set the state to `Unmatched`.
        *   `[X]` If any mapped state remains `Pending`, return. Otherwise build `QList<TorrentID> unmatched` from the entries whose state is `Unmatched` and whose torrent still exists in the session, then clear `m_findLocationOperation`. Zero unmatched entries → return. One → call `askTorrentLocation()` for it. More than one → construct `UnmatchedTorrentsDialog(this, unmatched)`; `dialog->isEmpty()` true → delete it and return; otherwise set `Qt::WA_DeleteOnClose` and call `open()`.
        *   `[X]` `askTorrentLocation(id)`: no torrent under `id` in the session → return. Otherwise create the directory dialog of `setSelectedTorrentsLocation()`, starting at that torrent's `savePath().data()`. Connect its `&QDialog::accepted` signal with `this` as the context and capture the dialog pointer and ID; on an existing selected `Path`, call `BitTorrent::Session::instance()->assignTorrentLocation(id, newLocation)`.

    *   `[X]` src/gui/`transferlistwidget.cpp`
        *   `[X]` Add `#include "unmatchedtorrentsdialog.h"` after `#include "uithememanager.h"`.
        *   `[X]` Add the constructor connection and the `displayListMenu()` action per the interaction spec.
        *   `[X]` Define `TransferListWidget::findSelectedTorrentsLocation()` after `TransferListWidget::setSelectedTorrentsLocation()`, and `handleTorrentLocationFound()` and `askTorrentLocation()` after it, per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `transferlistwidget` depends downward on `Session` in `src/base`, as it already does, and beside it on `unmatchedtorrentsdialog`. It never names `SessionImpl`; its pre-existing uses of `Preferences` are unchanged.

    *   `[X]` `requirements`
        *   `[X]` IF-7: **Find location** sits directly after **Set location...** over single and multiple selections.
        *   `[X]` IF-8: a single torrent without a match opens the **Choose save path** dialog that **Set location...** opens.
        *   `[X]` IF-9: several torrents without a match open `UnmatchedTorrentsDialog`.
        *   `[X]` IF-11: every call reaches discovery and assignment through `BitTorrent::Session`.
        *   `[X]` IF-13: each match is assigned as its own outcome arrives, before the batch completes.
        *   `[X]` Repeated invocations during an active operation merge unseen IDs into it; an ID already marked `Pending`, `Matched` or `Unmatched` is not searched twice in that operation.
        *   `[X]` ST-2: the action is absent while `isFindLocationEnabled()` is false.
        *   `[X]` IF-12: the action text is wrapped in `tr()`.
        *   `[X]` Verify the action through Epic 2 integration scenarios 4 and 5 at the commit boundary.

*   `[X]` [UI] src/gui/`optionsdialog` — add `checkFindLocationOnStart`, `checkFindLocationRecheck`, `checkFindLocationSeed` and `checkFindLocationLeech` to `groupFindLocation`, each loaded from and saved to `BitTorrent::Session`, with the seed and leech checkboxes enabled only while `checkFindLocationRecheck` is checked. T15

    *   `[X]` `objective`
        *   `[X]` Problem: the pending-Start gate and the three assignment settings have no desktop control.
        *   `[X]` Functional: four checkboxes within the **Find location** group presenting **Find location when starting stopped torrents**, **Recheck automatically**, **Seed automatically** and **Leech automatically**. A change to any enables **Apply**, and all four are written when the page is applied.
        *   `[X]` Functional: **Seed automatically** and **Leech automatically** are enabled only while **Recheck automatically** is checked, and unchecking the group disables all four; neither change alters a checkbox's value.
        *   `[X]` Non-functional: every label and tooltip is translatable, and every control is a standard widget reachable by keyboard.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/gui`.
        *   `[X]` Out of scope: the web interface, which is the `appcontroller` and `preferences.html` nodes; what the settings cause.

    *   `[X]` `module`
        *   `[X]` Inside: four widgets, their loading, saving, enablement and enabling of **Apply**. Outside: what each setting causes.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — the eight Epic 2 setting accessors from the `sessionimpl` node, already included by `optionsdialog.cpp`.
        *   `[X]` `ui_optionsdialog.h` — generated from `optionsdialog.ui`, providing the four checkboxes.

    *   `[X]` `context_slice`
        *   `[X]` `groupFindLocation` and its `QVBoxLayout` `groupFindLocationLayout`, holding `checkFindLocationOnAdd`, and the load, connection and save statements for both, are those the first epic's `optionsdialog` node adds to `optionsdialog.ui` and to `OptionsDialog::loadDownloadsTabOptions()` and `OptionsDialog::saveDownloadsTabOptions()`. `connect(m_ui->backupDirCheckBox, &QAbstractButton::toggled, m_ui->backupDirPathEdit, &QWidget::setEnabled);`, with `backupDirPathEdit->setEnabled(backupDirCheckBox->isChecked())` at load, is the precedent for a checkbox enabling another control.
        *   `[X]` A checkable `QGroupBox` re-enabling its children on being checked leaves a child disabled through `setEnabled(false)` disabled, so the recheck dependency survives the group being unchecked and checked again.

    *   `[X]` src/gui/`optionsdialog.ui`
        *   `[X]` In `groupFindLocationLayout`, after the item holding `checkFindLocationOnAdd`, add four items in order, each a `QCheckBox`.
        *   `[X]` `checkFindLocationOnStart`, with `text` `Find location when starting stopped torrents` and `toolTip` `Search for existing content before starting a stopped torrent.`
        *   `[X]` `checkFindLocationRecheck`, with `text` `Recheck automatically` and `toolTip` `When a location is assigned by Find location, recheck the torrent's files there.`
        *   `[X]` `checkFindLocationSeed`, with `text` `Seed automatically` and `toolTip` `When that recheck finds the torrent's content complete, start the torrent.`
        *   `[X]` `checkFindLocationLeech`, with `text` `Leech automatically` and `toolTip` `When that recheck finds the torrent's content incomplete, start the torrent.`

    *   `[X]` `interaction.spec`
        *   `[X]` Load: immediately before `m_ui->groupFindLocation->setChecked(session->isFindLocationEnabled());`, call `m_ui->checkFindLocationOnStart->setChecked(session->isFindLocationOnStartEnabled());`, `m_ui->checkFindLocationRecheck->setChecked(session->isFindLocationRecheckEnabled());`, `m_ui->checkFindLocationSeed->setChecked(session->isFindLocationSeedEnabled());` and `m_ui->checkFindLocationLeech->setChecked(session->isFindLocationLeechEnabled());` in that order, then call `m_ui->checkFindLocationSeed->setEnabled(m_ui->checkFindLocationRecheck->isChecked());` and `m_ui->checkFindLocationLeech->setEnabled(m_ui->checkFindLocationRecheck->isChecked());`.
        *   `[X]` Connect: after `connect(m_ui->checkFindLocationOnAdd, &QAbstractButton::toggled, this, &ThisType::enableApplyButton);`, add `connect(m_ui->checkFindLocationOnStart, &QAbstractButton::toggled, this, &ThisType::enableApplyButton);`, `connect(m_ui->checkFindLocationRecheck, &QAbstractButton::toggled, m_ui->checkFindLocationSeed, &QWidget::setEnabled);`, `connect(m_ui->checkFindLocationRecheck, &QAbstractButton::toggled, m_ui->checkFindLocationLeech, &QWidget::setEnabled);`, then one connection from each of `checkFindLocationRecheck`, `checkFindLocationSeed` and `checkFindLocationLeech` to `this, &ThisType::enableApplyButton` using `&QAbstractButton::toggled`.
        *   `[X]` Save: after `session->setFindLocationOnAddEnabled(m_ui->checkFindLocationOnAdd->isChecked());`, call `session->setFindLocationOnStartEnabled(m_ui->checkFindLocationOnStart->isChecked());`, `session->setFindLocationRecheckEnabled(m_ui->checkFindLocationRecheck->isChecked());`, `session->setFindLocationSeedEnabled(m_ui->checkFindLocationSeed->isChecked());` and `session->setFindLocationLeechEnabled(m_ui->checkFindLocationLeech->isChecked());` in that order. A disabled checkbox keeps its checked state and is saved with it.

    *   `[X]` src/gui/`optionsdialog.cpp`
        *   `[X]` Add the load, connection and save statements per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `optionsdialog` in `src/gui` depends on `BitTorrent::Session` in `src/base`, downward, as it already does.

    *   `[X]` `requirements`
        *   `[X]` IF-2: `checkFindLocationSeed` and `checkFindLocationLeech` are enabled exactly while `checkFindLocationRecheck` is checked and the group is checked.
        *   `[X]` ST-3: unchecking the group or `checkFindLocationRecheck` changes no checkbox's value, and applying writes each value unchanged.
        *   `[X]` IF-14: `checkFindLocationOnStart` carries the exact accepted label and persists independently of the other four settings in the group.
        *   `[X]` IF-12: `uic` routes the eight `.ui` strings through `tr()`.
        *   `[X]` Verify the dialog through Epic 2 integration scenario 6 at the commit boundary.

*   `[X]` [API] src/webui/api/`appcontroller` — expose `find_location_on_start_enabled`, `find_location_recheck_enabled`, `find_location_seed_enabled` and `find_location_leech_enabled` on `app/preferences` and `app/setPreferences`, read from and written to `BitTorrent::Session`. T15

    *   `[X]` `objective`
        *   `[X]` Problem: the pending-Start gate and the three assignment settings are reachable only from the desktop dialog, so the headless daemon and remote clients can neither read nor change them.
        *   `[X]` Functional: `app/preferences` returns all four as booleans; `app/setPreferences` sets each when its key is present and leaves it untouched when its key is absent.

    *   `[X]` `role`
        *   `[X]` WebAPI controller in `src/webui/api`.
        *   `[X]` Out of scope: the web page, which is the `preferences.html` node; the changelog, which is the `WebAPI_Changelog` node; `API_VERSION`, which the maintainers set.

    *   `[X]` `module`
        *   `[X]` Inside: four keys in two actions. Outside: what each setting causes, and how each is presented.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — the eight Epic 2 setting accessors from the `sessionimpl` node, already included by `appcontroller.cpp`.

    *   `[X]` `context_slice`
        *   `[X]` `find_location_enabled` and `find_location_on_add_enabled`, added by the first epic's `appcontroller` node after `use_unwanted_folder` in `AppController::preferencesAction()` and `AppController::setPreferencesAction()`, are the model and the anchor.

    *   `[X]` `interaction.spec`
        *   `[X]` `preferencesAction()`: after `find_location_on_add_enabled`, add `data[u"find_location_on_start_enabled"_s] = session->isFindLocationOnStartEnabled();`, `data[u"find_location_recheck_enabled"_s] = session->isFindLocationRecheckEnabled();`, `data[u"find_location_seed_enabled"_s] = session->isFindLocationSeedEnabled();` and `data[u"find_location_leech_enabled"_s] = session->isFindLocationLeechEnabled();` in that order.
        *   `[X]` `setPreferencesAction()`: after the `find_location_on_add_enabled` branch, add `if (hasKey(u"find_location_on_start_enabled"_s))` followed by `session->setFindLocationOnStartEnabled(it.value().toBool());`; then the same two lines using `find_location_recheck_enabled`/`setFindLocationRecheckEnabled`, `find_location_seed_enabled`/`setFindLocationSeedEnabled` and `find_location_leech_enabled`/`setFindLocationLeechEnabled`, in that order and with the existing indentation. An absent key makes no setter call.

    *   `[X]` src/webui/api/`appcontroller.cpp`
        *   `[X]` In `preferencesAction()`, after `data[u"find_location_on_add_enabled"_s] = session->isFindLocationOnAddEnabled();`, add the four assignments from the interaction spec in OnStart, recheck, seed and leech order.
        *   `[X]` In `setPreferencesAction()`, after the `find_location_on_add_enabled` branch, add the four `hasKey()` branches from the interaction spec in OnStart, recheck, seed and leech order.

    *   `[X]` `directionality`
        *   `[X]` `appcontroller` in `src/webui` depends on `BitTorrent::Session` in `src/base`, downward, as it already does.

    *   `[X]` `requirements`
        *   `[X]` IF-4, IF-14: all four settings are exposed on `app/preferences` and `app/setPreferences`.
        *   `[X]` ST-3: each key writes only its own setting.
        *   `[X]` CN-5: a request omitting any of the keys leaves that setting unchanged.
        *   `[X]` Verify the endpoints through Epic 2 integration scenario 6 at the commit boundary.

*   `[X]` [UI] src/webui/www/private/views/`preferences.html` — add `findLocationOnStartCheckbox`, `findLocationRecheckCheckbox`, `findLocationSeedCheckbox` and `findLocationLeechCheckbox` to the **Find location** fieldset, loaded from and saved to the four Epic 2 keys, with `updateFindLocationEnabled()` enforcing the group and recheck dependencies. T15

    *   `[X]` `objective`
        *   `[X]` Problem: the pending-Start and three assignment keys have no control in the web interface, so a user of the headless daemon can set them only by request.
        *   `[X]` Functional: four checkboxes within the **Find location** fieldset presenting the four Epic 2 keys, loaded when the page loads and sent when it is saved.
        *   `[X]` Functional: OnStart and recheck are disabled while the legend checkbox is unchecked; seed and leech are disabled while either the legend or recheck is unchecked; disabled values are still sent unchanged.
        *   `[X]` Non-functional: every label passes through `QBT_TR` in the `OptionsDialog` context with the same source string as the corresponding desktop label. The web interface extracts the strings into `src/webui/www/translations/webui_*.ts`; the desktop extracts them separately into `src/lang`.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/webui/www`.
        *   `[X]` Out of scope: the keys and their endpoints, which are the `appcontroller` node; the translation files, which the project's translation process maintains.

    *   `[X]` `module`
        *   `[X]` Inside: four form rows, the enablement handler, and the load and save statements. Outside: what each setting causes.

    *   `[X]` `deps`
        *   `[X]` `app/preferences`, returning `find_location_on_start_enabled`, `find_location_recheck_enabled`, `find_location_seed_enabled` and `find_location_leech_enabled`, read from `pref` with those exact property names.
        *   `[X]` `app/setPreferences`, receiving the same four exact keys in `settings`.

    *   `[X]` `context_slice`
        *   `[X]` The **Find location** fieldset holding `findLocationCheckbox` and `findLocationOnAddCheckbox`, the handler `updateFindLocationEnabled` exported through `exports`, its load and save statements, and its call after loading are those the first epic's `preferences.html` node adds.

    *   `[X]` `interaction.spec`
        *   `[X]` `updateFindLocationEnabled()` reads `findLocationCheckbox.checked` as `groupEnabled` and `findLocationRecheckCheckbox.checked` as `recheckEnabled`; it sets `findLocationOnAddCheckbox.disabled`, `findLocationOnStartCheckbox.disabled` and `findLocationRecheckCheckbox.disabled` to `!groupEnabled`, and sets seed and leech disabled to `!(groupEnabled && recheckEnabled)`.
        *   `[X]` It runs from the `onclick` of `findLocationCheckbox` and `findLocationRecheckCheckbox`, and once after all six checkboxes are loaded.

    *   `[X]` src/webui/www/private/views/`preferences.html`
        *   `[X]` Markup: after the `findLocationOnAddCheckbox` row, add four `<div class="formRow">` elements. The first holds `<input type="checkbox" id="findLocationOnStartCheckbox" title="QBT_TR(Search for existing content before starting a stopped torrent.)QBT_TR[CONTEXT=OptionsDialog]">` and `<label for="findLocationOnStartCheckbox">QBT_TR(Find location when starting stopped torrents)QBT_TR[CONTEXT=OptionsDialog]</label>`.
        *   `[X]` The second row holds `<input type="checkbox" id="findLocationRecheckCheckbox" title="QBT_TR(When a location is assigned by Find location, recheck the torrent's files there.)QBT_TR[CONTEXT=OptionsDialog]" onclick="qBittorrent.Preferences.updateFindLocationEnabled();">` and `<label for="findLocationRecheckCheckbox">QBT_TR(Recheck automatically)QBT_TR[CONTEXT=OptionsDialog]</label>`.
        *   `[X]` The third row holds `<input type="checkbox" id="findLocationSeedCheckbox" title="QBT_TR(When that recheck finds the torrent's content complete, start the torrent.)QBT_TR[CONTEXT=OptionsDialog]">` and `<label for="findLocationSeedCheckbox">QBT_TR(Seed automatically)QBT_TR[CONTEXT=OptionsDialog]</label>`.
        *   `[X]` The fourth row holds `<input type="checkbox" id="findLocationLeechCheckbox" title="QBT_TR(When that recheck finds the torrent's content incomplete, start the torrent.)QBT_TR[CONTEXT=OptionsDialog]">` and `<label for="findLocationLeechCheckbox">QBT_TR(Leech automatically)QBT_TR[CONTEXT=OptionsDialog]</label>`. Indent all four rows exactly as the existing OnAdd row.
        *   `[X]` Handler: rewrite the body of `updateFindLocationEnabled` per the interaction spec.
        *   `[X]` Load: after `document.getElementById("findLocationOnAddCheckbox").checked = pref.find_location_on_add_enabled;` and before `updateFindLocationEnabled();`, assign `pref.find_location_on_start_enabled`, `pref.find_location_recheck_enabled`, `pref.find_location_seed_enabled` and `pref.find_location_leech_enabled` to the checked properties of `findLocationOnStartCheckbox`, `findLocationRecheckCheckbox`, `findLocationSeedCheckbox` and `findLocationLeechCheckbox`, respectively and in that order.
        *   `[X]` Save: after `settings["find_location_on_add_enabled"] = document.getElementById("findLocationOnAddCheckbox").checked;`, assign the checked properties of the same four checkbox IDs to `settings["find_location_on_start_enabled"]`, `settings["find_location_recheck_enabled"]`, `settings["find_location_seed_enabled"]` and `settings["find_location_leech_enabled"]`, respectively and in that order.

    *   `[X]` `directionality`
        *   `[X]` The page consumes the WebAPI and adds no dependency beyond the four keys the `appcontroller` node provides.

    *   `[X]` `requirements`
        *   `[X]` IF-5, IF-14: all four settings have a control on the web interface preferences page.
        *   `[X]` IF-2: the web controls honour the recheck dependency the options dialog enforces.
        *   `[X]` IF-12: every label is wrapped in `QBT_TR` in the `OptionsDialog` context and is extracted into the WebUI translation catalogs independently of the desktop catalogs.
        *   `[X]` ST-3: a disabled checkbox keeps its `checked` state and is sent unchanged.
        *   `[X]` The `CI - WebUI` workflow's `npm run lint` passes, and `npm run format` in `src/webui/www` leaves the file unchanged.
        *   `[X]` Verify the page through Epic 2 integration scenario 6 at the commit boundary.

*   `[X]` [API] src/webui/api/`torrentscontroller` — add `torrents/findLocation`, running discovery for the named torrents holding metadata through `Session::findTorrentLocation()`, assigning each match through `Session::assignTorrentLocation()` as its outcome arrives, and answering HTTP 202 until no torrent of the web session's operation is pending; and `torrents/assignLocation`, assigning a named existing directory through `Session::assignTorrentLocation()`. T23

    *   `[X]` `objective`
        *   `[X]` Problem: discovery and assignment for torrents already in the session are reachable only from the desktop transfer list, so a user of the web interface or the headless daemon can neither run **Find location** nor assign a location as **Find location** assigns one.
        *   `[X]` Functional: for the torrents a request names, run discovery for each holding metadata, assign every match as its outcome arrives, and answer with the operation pending while any torrent is, then with the torrents assigned and those that matched nothing once none is.
        *   `[X]` Functional: a request naming torrents already in the web session's active operation searches none of them again, a request naming new torrents merges them into it, and a request naming none reports the operation without starting anything.
        *   `[X]` Functional: assign a location a request names to the torrents it names, through the assignment a found location receives.
        *   `[X]` Non-functional: neither action waits for a probe, and both are accepted by POST alone.

    *   `[X]` `role`
        *   `[X]` WebAPI controller in `src/webui/api`, one instance per web session.
        *   `[X]` Out of scope: discovery, assignment, recheck and the start decision, which are the `sessionimpl` node; presentation, which is the `mocha-init.js`, `setlocation.html` and `unmatchedtorrents.html` nodes; the changelog, which is the `WebAPI_Changelog` node.

    *   `[X]` `module`
        *   `[X]` Inside: the two actions, one operation map holding each registered torrent's result, merging a request into the active operation, and the answer. Outside: how a location is found and what assignment does.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — `Session::findTorrentLocation()`, `Session::assignTorrentLocation()`, `Session::getTorrent()`, `Session::isFindLocationEnabled()` and the `torrentLocationFound` signal, from the `sessionimpl` nodes. Already included by `torrentscontroller.cpp`.
        *   `[X]` `base/path.h` — `Path`, `Path::isEmpty()` and `Path::toString()`, held by value in the operation map.
        *   `[X]` `base/utils/fs.h` — `Utils::Fs::isDir()`. Already included by `torrentscontroller.cpp`.
        *   `[X]` `webapplication.h` — `WebApplication::m_allowedMethod`.

    *   `[X]` `context_slice`
        *   `[X]` `WebApplication` registers a factory creating `TorrentsController` for each `WebSession`, which creates it on first use, so its members live as long as the web session. The constructor connects `Session::metadataDownloaded` to `onMetadataDownloaded()`, which fills `m_torrentMetadataCache`; `fetchMetadataAction()` answers from that cache and calls `setStatus(APIStatus::Async)` while metadata is outstanding, and `WebApplication` answers `APIStatus::Async` with HTTP 202.
        *   `[X]` `applyToTorrents()` takes the `hashes` parameter split on `|`, treats a single `all` as every torrent, and skips an ID naming no torrent. `setLocationAction()` requires `hashes` and `location` and throws `APIError(APIErrorType::BadParams, tr("Save path cannot be empty"))` for an empty location. `APIErrorType::BadParams` answers HTTP 400 and `APIErrorType::Conflict` HTTP 409.
        *   `[X]` `m_allowedMethod` in `webapplication.h` lists the POST-only actions by scope and then action name, in alphabetical order.
        *   `[X]` `TransferListWidget::handleTorrentLocationFound()` acts only on IDs its own operation holds as pending, so a desktop operation and a web session operation searching one torrent each act on their own result, and `SessionImpl::assignTorrentLocation()` coalesces an assignment of the location already being assigned.

    *   `[X]` src/webui/api/`torrentscontroller.h`
        *   `[X]` After `#include "base/bittorrent/torrentdescriptor.h"`, add `#include "base/path.h"`.
        *   `[X]` In `private slots`, after `void setLocationAction();`, add `void findLocationAction();` then `void assignLocationAction();`.
        *   `[X]` In `private`, before `void onDownloadFinished(const Net::DownloadResult &result);`, add `enum class FindLocationState { Pending, Matched, Unmatched };` and `struct FindLocationEntry { FindLocationState state = FindLocationState::Pending; Path location; };`, one declaration per line, followed by one blank line.
        *   `[X]` After `void onSearchPluginTorrentDownloaded(const QString &source, const QString &data);`, add `void onTorrentLocationFound(const BitTorrent::TorrentID &id, const Path &location, bool found);`.
        *   `[X]` After `QSet<QString> m_invalidTorrentSource;`, add `QHash<BitTorrent::TorrentID, FindLocationEntry> m_findLocationOperation;`. An empty map is no active operation.

    *   `[X]` `interaction.spec`
        *   `[X]` Constructor: after the `metadataDownloaded` connection, `connect(BitTorrent::Session::instance(), &BitTorrent::Session::torrentLocationFound, this, &TorrentsController::onTorrentLocationFound);`.
        *   `[X]` `findLocationAction()`: `!BitTorrent::Session::instance()->isFindLocationEnabled()` → `throw APIError(APIErrorType::Conflict, tr("Find location is disabled"))`. Create a local `QList<BitTorrent::TorrentID> submitted`. When `params()[u"hashes"_s]` is non-empty, `applyToTorrents()` over it split on `|`: a torrent without metadata → skip; its ID already in `m_findLocationOperation` → skip; otherwise insert `FindLocationEntry {}` under its ID and append the ID to `submitted`. Then call `BitTorrent::Session::instance()->findTorrentLocation(id)` for each ID in `submitted`, so every ID of the request is registered before the first search is issued.
        *   `[X]` `findLocationAction()` answer: iterate `m_findLocationOperation` into three `QJsonArray`s. A `Pending` entry appends `id.toString()` to `pending`; a `Matched` entry appends `QJsonObject {{u"hash"_s, id.toString()}, {u"location"_s, entry.location.toString()}}` to `matched`; an `Unmatched` entry appends `id.toString()` to `unmatched` only while `BitTorrent::Session::instance()->getTorrent(id)` is non-null. `setResult(QJsonObject {{u"pending"_s, pending}, {u"matched"_s, matched}, {u"unmatched"_s, unmatched}})`. A non-empty `pending` → `setStatus(APIStatus::Async)`; otherwise `m_findLocationOperation.clear()`, ending the operation.
        *   `[X]` `onTorrentLocationFound(id, location, found)`: no entry for `id`, or an entry whose state is not `Pending` → return. `found` true → set the entry's state to `Matched` and its location to `location`, then call `BitTorrent::Session::instance()->assignTorrentLocation(id, location)`; `found` false → set its state to `Unmatched`.
        *   `[X]` `assignLocationAction()`: `requireParams({u"hashes"_s, u"location"_s})`; `const Path location {params()[u"location"_s].trimmed()};`. An empty location → `throw APIError(APIErrorType::BadParams, tr("Save path cannot be empty"))`; `!Utils::Fs::isDir(location)` → `throw APIError(APIErrorType::Conflict, tr("Location does not exist"))`. Otherwise `applyToTorrents()` over `params()[u"hashes"_s].split(u'|')` calls `BitTorrent::Session::instance()->assignTorrentLocation(torrent->id(), location)` for each torrent, then `setResult(QString())`. Nothing is created.

    *   `[X]` src/webui/api/`torrentscontroller.cpp`
        *   `[X]` Add the constructor connection per the interaction spec.
        *   `[X]` Define `TorrentsController::findLocationAction()` and `TorrentsController::assignLocationAction()` after `TorrentsController::setLocationAction()`, and `TorrentsController::onTorrentLocationFound()` after `TorrentsController::onMetadataDownloaded()`, per the interaction spec.

    *   `[X]` src/webui/`webapplication.h`
        *   `[X]` In `m_allowedMethod`, add `{{u"torrents"_s, u"assignLocation"_s}, Http::HEADER_REQUEST_METHOD_POST},` after the `addWebSeeds` entry and `{{u"torrents"_s, u"findLocation"_s}, Http::HEADER_REQUEST_METHOD_POST},` after the `filePrio` entry.

    *   `[X]` `directionality`
        *   `[X]` `torrentscontroller` in `src/webui` depends downward on `Session` in `src/base`, as it already does. It never names `SessionImpl` or anything in `src/gui`.

    *   `[X]` `requirements`
        *   `[X]` IF-17: `torrents/findLocation` registers, searches, assigns and answers as specified, including HTTP 409 while the group is disabled and a request without `hashes` registering nothing.
        *   `[X]` IF-18: `torrents/assignLocation` answers HTTP 400 for an empty location and HTTP 409 for one that is not an existing directory, and creates nothing.
        *   `[X]` IF-11: both actions reach discovery and assignment through `BitTorrent::Session`.
        *   `[X]` IF-13: each outcome is handled as its `torrentLocationFound` signal arrives, so each match is assigned before the operation completes.
        *   `[X]` RL-4: neither action waits for a search.
        *   `[X]` BT-10: both actions are registered as POST-only.
        *   `[X]` IF-12: both new messages pass through `tr()`.
        *   `[X]` Verify the actions through Epic 2 integration scenario 18 at the commit boundary.

*   `[X]` [UI] src/webui/www/private/`setlocation.html` — post the entered location to `torrents/assignLocation` when the window is opened with `assign=true`, and to `torrents/setLocation` otherwise. T24

    *   `[X]` `objective`
        *   `[X]` Problem: the window posts to `torrents/setLocation`, which creates a missing directory and calls the location setters directly, so a torrent that matched nothing in the web interface would be placed by different rules than one placed from the desktop interface's **Choose save path** dialog.
        *   `[X]` Functional: opened with `assign=true`, post the entered location to `torrents/assignLocation`; opened without it, post to `torrents/setLocation` exactly as the window does for **Set location...**.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/webui/www`.
        *   `[X]` Out of scope: deciding when the window opens in its assigning mode, which is the `mocha-init.js` node.

    *   `[X]` `module`
        *   `[X]` Inside: the choice of action. Outside: what either action does.

    *   `[X]` `deps`
        *   `[X]` `torrents/assignLocation` from the `torrentscontroller` node.

    *   `[X]` `context_slice`
        *   `[X]` The window's script reads `hashes` and `path` from `new URLSearchParams(window.location.search)` as `searchParams`, and its **Save** click handler posts `hashes` and `location` to the literal `"api/v2/torrents/setLocation"`, writes the response text into `error_div` when the response is not ok, and otherwise calls `window.parent.qBittorrent.Client.closeFrameWindow(window)`.

    *   `[X]` src/webui/www/private/`setlocation.html`
        *   `[X]` In the click handler, replace `"api/v2/torrents/setLocation"` with `((searchParams.get("assign") === "true") ? "api/v2/torrents/assignLocation" : "api/v2/torrents/setLocation")`. Nothing else changes.

    *   `[X]` `directionality`
        *   `[X]` The page consumes the WebAPI and adds no dependency beyond `torrents/assignLocation`.

    *   `[X]` `requirements`
        *   `[X]` IF-19: one unmatched torrent's location is assigned through `torrents/assignLocation`.
        *   `[X]` **Set location...** behaves as before, posting to `torrents/setLocation`.
        *   `[X]` BT-9: `npm run lint` passes, and `npm run format` in `src/webui/www` leaves the file unchanged.
        *   `[X]` Verify the page through Epic 2 integration scenario 19 at the commit boundary.

*   `[X]` [UI] src/webui/www/private/`unmatchedtorrents.html` — add the window listing torrents that matched nothing, assigning a location entered for each entry through `torrents/assignLocation`, and closing when abandoned or empty. T24

    *   `[X]` `objective`
        *   `[X]` Problem: a web interface batch matching nothing for several torrents would otherwise open one **Set location** window per torrent, with no view of the set and no way to stop.
        *   `[X]` Functional: list the unmatched torrents still in the session by name; for the selected entry, take a location in a field holding its save path, assign it through `torrents/assignLocation` and remove the entry; close once no entry remains.
        *   `[X]` Functional: **Close** and Escape close the window, assigning nothing further.
        *   `[X]` Non-functional: every string passes through `QBT_TR` with the desktop dialog's source strings where they exist, and the list, the location field and both buttons are native elements reachable by Tab.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/webui/www`, an iframe window opened by the `mocha-init.js` node.
        *   `[X]` Out of scope: discovery, assignment and deciding which torrents are unmatched.

    *   `[X]` `module`
        *   `[X]` Inside: the list, the location field and the hand-off of an entered location. Outside: what assignment does.

    *   `[X]` `deps`
        *   `[X]` `torrents/info`, returning objects carrying `hash`, `name` and `save_path` for the torrents named by `hashes`; `torrents/assignLocation` from the `torrentscontroller` node.
        *   `[X]` `scripts/localpreferences.js`, `scripts/color-scheme.js` and `scripts/pathAutofill.js`, loaded as `setlocation.html` loads them.

    *   `[X]` `context_slice`
        *   `[X]` `setlocation.html` is the form model: `css/style.css`, the three scripts loaded with `defer` and `?v=${CACHEID}`, a text input with class `pathDirectory` and `autocorrect="off" autocapitalize="none"`, `window.qBittorrent.pathAutofill.attachPathAutofill()`, a `keydown` handler taking Enter and Escape, an `error_div`, the `QBT_TR(Save path is empty)QBT_TR[CONTEXT=TorrentsController]` message and `window.parent.qBittorrent.Client.closeFrameWindow(window)`. `unmatchedtorrentsdialog.ui` carries the desktop strings `Find location` and `No existing content was found for these torrents. Set a location for each, or close to leave them where they are.`, and `UnmatchedTorrentsDialog` carries `Set location...`.
        *   `[X]` `src/webui/www/webui.qrc` lists every page under `private/` in path order.

    *   `[X]` `interaction.spec`
        *   `[X]` Load: read `hashes` from the window's search parameters and fetch `api/v2/torrents/info` with `hashes` as a query parameter and `cache: "no-store"`. For each returned torrent, append to `unmatchedTorrents` an `option` whose `value` is `hash`, whose `textContent` is `name` and whose `dataset.savePath` is `save_path`. No option → close the window. Otherwise select the first option and write its `dataset.savePath` into `unmatchedLocation.value`.
        *   `[X]` A `change` on `unmatchedTorrents` writes the selected option's `dataset.savePath` into `unmatchedLocation.value` and clears `error_div`.
        *   `[X]` **Set location...**: no selected option → return. An empty trimmed location → write the empty-path message into `error_div` and return. Otherwise post `hashes` holding the option's `value` and `location` to `api/v2/torrents/assignLocation`. A response that is not ok → write its text into `error_div`. An ok response → remove the option, clear `error_div`, and close the window when no option remains, or else select the first option and write its save path into the field.
        *   `[X]` **Close** and Escape close the window. Enter on a focused button activates that button; Enter elsewhere clicks **Set location...**.
        *   `[X]` A torrent removed after the window loads is still listed; `torrents/assignLocation` skips an ID naming no torrent and answers ok, so assigning it removes the entry without effect.

    *   `[X]` src/webui/www/private/`unmatchedtorrents.html`
        *   `[X]` New file following `setlocation.html`: `<!DOCTYPE html>`, `<html lang="${LANG}" class="dark">`, the same `<head>` with `<title>QBT_TR(Find location)QBT_TR[CONTEXT=UnmatchedTorrentsDialog]</title>`, and the script per the interaction spec inside `window.addEventListener("DOMContentLoaded", ...)` under `"use strict";`.
        *   `[X]` Body: a `<div style="padding: 10px 10px 0px 10px;">` holding `<p id="unmatchedLabel">QBT_TR(No existing content was found for these torrents. Set a location for each, or close to leave them where they are.)QBT_TR[CONTEXT=UnmatchedTorrentsDialog]</p>`, `<select id="unmatchedTorrents" size="6" style="width: 99%;" aria-labelledby="unmatchedLabel"></select>`, `<label for="unmatchedLocation" style="font-weight: bold;">QBT_TR(Location:)QBT_TR[CONTEXT=TransferListWidget]</label>`, `<input type="text" id="unmatchedLocation" class="pathDirectory" autocorrect="off" autocapitalize="none" style="width: 99%;">`, `<div style="float: none; width: 99%;" id="error_div">&nbsp;</div>`, and a `<div style="text-align: center; padding-top: 10px;">` holding `<input type="button" value="QBT_TR(Set location...)QBT_TR[CONTEXT=UnmatchedTorrentsDialog]" id="setLocationButton">` and `<input type="button" value="QBT_TR(Close)QBT_TR[CONTEXT=UnmatchedTorrentsDialog]" id="closeButton">`.

    *   `[X]` src/webui/www/`webui.qrc`
        *   `[X]` Add `<file>private/unmatchedtorrents.html</file>` after `<file>private/speedlimit.html</file>`.

    *   `[X]` `directionality`
        *   `[X]` The page consumes the WebAPI alone and is opened only by the `mocha-init.js` node.

    *   `[X]` `requirements`
        *   `[X]` IF-19: several unmatched torrents are listed and walked or abandoned, each entered location assigned through `torrents/assignLocation`.
        *   `[X]` IF-12: every label passes through `QBT_TR`.
        *   `[X]` BT-10: the page is registered in `webui.qrc`.
        *   `[X]` Names are written through `textContent` and locations through `value`, never into markup.
        *   `[X]` BT-9: `npm run lint` passes, and `npm run format` in `src/webui/www` leaves the file unchanged.
        *   `[X]` Verify the page through Epic 2 integration scenario 19 at the commit boundary.

*   `[X]` [UI] src/webui/www/private/scripts/`mocha-init.js` — add **Find location** to the web interface transfer list context menu, shown while the selection holds a torrent with metadata and `find_location_enabled` holds, polling `torrents/findLocation` until no torrent is pending and then opening `setlocation.html` in its assigning mode for one unmatched torrent or `unmatchedtorrents.html` for several. T24

    *   `[X]` `objective`
        *   `[X]` Problem: the web interface has no way to run discovery for torrents already in the session.
        *   `[X]` Functional: a context menu action over the selection that runs `torrents/findLocation` for the selected torrents holding metadata and, once no torrent is pending, opens **Set location** in its assigning mode for one unmatched torrent, the unmatched list for several, and nothing for none.
        *   `[X]` Functional: one poll runs per page; invoking the action while a poll is scheduled cancels that poll and merges the new selection into the operation through its own request.
        *   `[X]` Non-functional: the action is absent while the **Find location** group is disabled, and the transfer list stays interactive while the operation runs.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/webui/www`, the transfer list and its context menu.
        *   `[X]` Out of scope: discovery and assignment, which are the `torrentscontroller` and `sessionimpl` nodes; the windows it opens, which are the `setlocation.html` and `unmatchedtorrents.html` nodes.

    *   `[X]` `module`
        *   `[X]` Inside: the menu item, its visibility, the request and poll, and choosing which window opens. Outside: how a location is found and what assignment does.

    *   `[X]` `deps`
        *   `[X]` `torrents/findLocation` from the `torrentscontroller` node; `find_location_enabled` from `app/preferences`, read through `window.qBittorrent.Cache.preferences.get()`.
        *   `[X]` `images/edit-find.svg`, already in `webui.qrc`.

    *   `[X]` `context_slice`
        *   `[X]` `index.html` lists `torrentsTableMenu` items as `<li>` elements whose anchors name an action by `href`; **Set location...** is the `<li class="separator">` holding `#setLocation`. `views/transferlist.html` binds each action name to a function in the object it passes to `TorrentsTableContextMenu`, with `setLocation: (element, ref) => { setLocationFN(); }`.
        *   `[X]` `TorrentsTableContextMenu::updateMenuItems()` in `scripts/contextmenu.js` walks the selected rows' `full_data`, declaring its flags before the walk and ending the flag declarations with `let all_are_auto_tmm = true;`, then shows and hides items through `showItem()` and `hideItem()` and calls `this.setItemChecked("autoTorrentManagement", all_are_auto_tmm);`. Each row's `full_data` carries `has_metadata` and `save_path`.
        *   `[X]` `mocha-init.js` declares each action as `let setLocationFN = () => {};` near the top and assigns it later; `setLocationFN` opens `setlocation.html` in a `MochaUI.Window` with id `setLocationPage`, passing `hashes` and the first row's `save_path`. `addtorrent.js` polls `torrents/fetchMetadata` on a one-second delay while the answer is HTTP 202.

    *   `[X]` `interaction.spec`
        *   `[X]` `openSetLocationWindow(hashes, assign)`, a `const` arrow function defined immediately before the `setLocationFN` assignment, holds the current body of `setLocationFN` after its empty-selection test, with `hashes` as a parameter and, when `assign` is true, `assign: "true"` added to the window's search parameters. `setLocationFN` keeps its empty-selection test and then calls `openSetLocationWindow(hashes, false)`.
        *   `[X]` `findLocationFN`: take the selected row IDs whose row `full_data.has_metadata` is true; none → return. Clear the scheduled poll, then post `hashes` joined by `|` to `api/v2/torrents/findLocation` and pass the response to `handleFindLocationResponse`.
        *   `[X]` `scheduleFindLocationPoll()` stores the timer of a post, one second later, to `api/v2/torrents/findLocation` without `hashes`, whose response is passed to `handleFindLocationResponse` and whose failure to complete calls `scheduleFindLocationPoll()` again; the request `findLocationFN` posts is handled the same way. `handleFindLocationResponse(response)`: not ok → `alert()` the response text, ending the poll. HTTP 202 → `scheduleFindLocationPoll()`. HTTP 200 → read `unmatched` from the JSON; one ID → `openSetLocationWindow([id], true)`; more than one → open `unmatchedtorrents.html` with `v` set to `${CACHEID}` and `hashes` joined by `|` in a `MochaUI.Window` with id `unmatchedTorrentsPage`, icon `images/qbittorrent-tray.svg`, title `QBT_TR(Find location)QBT_TR[CONTEXT=UnmatchedTorrentsDialog]`, `loadMethod: "iframe"`, `scrollbars: false`, `resizable: true`, `maximizable: false`, both paddings 0, width `window.qBittorrent.Dialog.limitWidthToViewport(480)` and height 320; none → nothing.
        *   `[X]` `updateMenuItems()`: after `let all_are_auto_tmm = true;`, declare `let there_are_metadata = false;`; in the walk, set it true for a row whose `data["has_metadata"]` is true; after `this.setItemChecked("autoTorrentManagement", all_are_auto_tmm);`, show `findLocation` when `there_are_metadata` and `window.qBittorrent.Cache.preferences.get().find_location_enabled` both hold, and hide it otherwise.

    *   `[X]` src/webui/www/private/`index.html`
        *   `[X]` After the `<li class="separator">` holding `#setLocation`, add a `<li>` holding `<a href="#findLocation"><img src="images/edit-find.svg" alt="QBT_TR(Find location)QBT_TR[CONTEXT=TransferListWidget]"> QBT_TR(Find location)QBT_TR[CONTEXT=TransferListWidget]</a>`, indented as the neighbouring items.

    *   `[X]` src/webui/www/private/views/`transferlist.html`
        *   `[X]` After the `setLocation` binding and its blank line, add `findLocation: (element, ref) => { findLocationFN(); },` in the multi-line form of `setLocation`, followed by a blank line.

    *   `[X]` src/webui/www/private/scripts/`contextmenu.js`
        *   `[X]` Add the flag, its test and the visibility statement per the interaction spec.

    *   `[X]` src/webui/www/private/scripts/`mocha-init.js`
        *   `[X]` After `let setLocationFN = () => {};`, add `let findLocationFN = () => {};`.
        *   `[X]` Define `openSetLocationWindow` before the `setLocationFN` assignment and reduce `setLocationFN` to it; assign `findLocationFN`, with `handleFindLocationResponse` and its timer defined before it, after the `setLocationFN` assignment, per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` The web interface consumes the WebAPI and `app/preferences` alone, and depends beside itself on `setlocation.html` and `unmatchedtorrents.html`.

    *   `[X]` `requirements`
        *   `[X]` IF-19: **Find location** sits directly after **Set location...**, shown while the selection holds a torrent with metadata and `find_location_enabled` holds, opening the assigning **Set location** window for one miss and the unmatched list for several.
        *   `[X]` IF-12: the menu label and window title pass through `QBT_TR`.
        *   `[X]` ST-2: the action is absent while `find_location_enabled` is false.
        *   `[X]` BT-9: `npm run lint` passes, and `npm run format` in `src/webui/www` leaves every edited file unchanged.
        *   `[X]` Verify the action through Epic 2 integration scenario 19 at the commit boundary.

*   `[X]` [DOCS] `WebAPI_Changelog` — record `find_location_on_start_enabled`, `find_location_recheck_enabled`, `find_location_seed_enabled` and `find_location_leech_enabled` on `app/preferences` and `app/setPreferences`, and the `torrents/findLocation` and `torrents/assignLocation` endpoints, under the version heading at the top of the file. T15, T23

    *   `[X]` `objective`
        *   `[X]` Problem: WebAPI clients learn of new preference keys and endpoints from this file, and four keys join both preference endpoints while two endpoints join the torrent actions.
        *   `[X]` Functional: add one entry under the version heading at the top of the file when the branch is rebased, linking this submission's pull request, naming all four keys on both preference endpoints and adding both torrent endpoints. Leave `API_VERSION` and the version headings unchanged.

    *   `[X]` `context_slice`
        *   `[X]` The entry added by the first epic's `WebAPI_Changelog` node is the wording model. Entries under a heading run newest first.
        *   `[X]` The entries adding `torrents/fetchMetadata` and `torrents/setComment` are the wording model for an added endpoint, in the form ``Add `torrents/<action>` endpoint with ... for ...``.

    *   `[X]` `WebAPI_Changelog.md`
        *   `[X]` Under the heading at the top of the file, before its first entry, insert the entry below, separated from its neighbours as the existing entries are.
        *   `[X]` The entry is a top-level bullet linking the pull request, `* [#<number>](https://github.com/qbittorrent/qBittorrent/pull/<number>)`, where `<number>` is the number GitHub assigns when this submission's pull request is opened.
        *   `[X]` Under it, the indented bullet ``* `app/preferences` endpoint includes `find_location_on_start_enabled` (bool), `find_location_recheck_enabled` (bool), `find_location_seed_enabled` (bool) and `find_location_leech_enabled` (bool) options``.
        *   `[X]` Under it, the indented bullet ``* `app/setPreferences` endpoint allows to set `find_location_on_start_enabled` (bool), `find_location_recheck_enabled` (bool), `find_location_seed_enabled` (bool) and `find_location_leech_enabled` (bool) options``.
        *   `[X]` Under it, the indented bullet ``* Add `torrents/findLocation` endpoint with optional parameter `hashes` for finding the existing content of torrents and assigning each location found, answering HTTP 202 with `pending`, `matched` and `unmatched` while any torrent is pending and HTTP 200 once none is``.
        *   `[X]` Under it, the indented bullet ``* Add `torrents/assignLocation` endpoint with parameters `hashes` and `location` for assigning an existing directory as the location of torrents``.

    *   `[X]` `requirements`
        *   `[X]` IF-6: the submission adds one `WebAPI_Changelog.md` entry naming the keys and actions it introduces under the heading at the top of the file and leaves `API_VERSION` unchanged.
        *   `[X]` The file passes the `rumdl` pre-commit hook.

*   `[X]` **Commit** `Find location before starting existing torrents`
    *   `[X]` Structural: `Session::findTorrentLocation()`, `Session::assignTorrentLocation()`, the eight Epic 2 setting accessors and `torrentLocationFound`; `TorrentImpl::stop(bool)` plus the single `interceptFindLocationStart()` call; `SessionImpl::searchExistingContent()`, the five named transaction helpers and `m_locationAssignments`; `UnmatchedTorrentsDialog`; `m_findLocationOperation` and the transfer-list action; `TorrentsController::findLocationAction()`, `TorrentsController::assignLocationAction()` and the controller's `m_findLocationOperation`; the web interface **Find location** action, `unmatchedtorrents.html` and the assigning mode of `setlocation.html`; four new checkboxes in each preferences interface.
    *   `[X]` Behavioural: the manual action retains its exact discovery, assignment and fallback flow, and the web interface and the WebAPI run the same discovery, assignment and fallback for a web session's operation, answering HTTP 202 until no torrent is pending; additionally, an eligible explicit Start is held, searched, assigned when needed, rechecked after the actual location settles, and released in its saved normal or forced mode only after a successful match check, while a successful miss releases the unchanged Start workflow immediately.
    *   `[X]` Contract: `findExistingContent()` keeps its signature and add-time behavior; public `Torrent::start()` and `Torrent::stop()` signatures, `forceRecheck()`, the location setters and the storage move queue are unchanged; when either Start gate is disabled or the torrent is ineligible, the original Start body runs unchanged; `app/preferences` and `app/setPreferences` gain four keys, and `torrents/findLocation` and `torrents/assignLocation` are added, all recorded in `WebAPI_Changelog.md` with `API_VERSION` unchanged; `torrents/setLocation` and **Set location...** in the web interface are unchanged.
    *   `[X]` **Integration scenario 1 — assignment movement and overlap:** assign one torrent already at the chosen location and one requiring a storage move; confirm each recheck begins only after it reports the chosen location. While another feature assignment is waiting for movement, use the existing **Set location...** action to choose a different destination; confirm the existing move queue reaches that destination and the feature clears its assignment state without issuing a recheck or start for the superseded target.
    *   `[X]` **Integration scenario 2 — movement failure:** make an assignment move fail; confirm the existing warning is logged, the torrent remains stopped at its prior location, and no feature recheck or start follows.
    *   `[X]` **Integration scenario 3 — check and start decisions:** with a complete torrent and an incomplete torrent, verify **Recheck automatically** off, then on; with it on, verify the complete torrent under each **Seed automatically** value and the incomplete torrent under each **Leech automatically** value. While an assignment waits for movement, complete a separately initiated check for that torrent and confirm it causes no feature start decision. Run a batch larger than both configured checking and active-torrent limits and confirm both limits hold.
    *   `[X]` **Integration scenario 4 — operation coalescing:** while a mixed batch remains pending, invoke **Find location** over a selection containing one pending ID, one ID already matched in that operation and one new ID; confirm the first two are not searched or assigned again, the new ID joins the operation, and each match is assigned as its result arrives.
    *   `[X]` **Integration scenario 5 — unmatched handling:** verify that one valid miss opens **Choose save path**, several valid misses open `UnmatchedTorrentsDialog`, zero valid misses open nothing, **Close** and Escape assign no remaining entry, and removing torrents before and after the dialog opens leaves no empty dialog.
    *   `[X]` **Integration scenario 6 — settings surfaces:** verify `find_location_on_start_enabled`, `find_location_recheck_enabled`, `find_location_seed_enabled` and `find_location_leech_enabled` are always returned by `app/preferences`; each present key independently changes only its session setting through `app/setPreferences`; each absent key leaves its setting unchanged; desktop and WebUI controls reflect all four values; disabling the group preserves every child value; and disabling recheck preserves the seed and leech values.
    *   `[X]` **Integration scenario 7 — own paths:** run **Find location** for a torrent whose content is at its own save path and for one whose content is at its own download path; confirm each is reported as matched, no assignment, recheck or start follows, and neither opens a dialog.
    *   `[X]` **Integration scenario 8 — existing location semantics:** assign an automatic-mode torrent and confirm it switches to manual mode as **Set location...** does; assign an incomplete torrent with a download path and confirm its save path is recorded, its storage stays in the download path, and no feature recheck or start follows; assign a torrent while it is being checked and confirm the feature's recheck follows its move.
    *   `[X]` **Integration scenario 9 — disabled and ineligible Start fall-through (TX-1):** capture the state and native behavior of a stopped manual torrent started normally and forcibly with the feature absent. Repeat with the master gate off, with OnStart off, with automatic torrent management enabled, and while the torrent is checking. In every case confirm no `m_locationAssignments` entry or Find Location terminal log is produced and the same stopped-state, mode and resume behavior occurs as in the baseline.
    *   `[X]` **Integration scenario 10 — Start match ordering and data safety (TX-2, TX-3, TX-5):** for complete and partial stopped manual torrents whose configured save and download paths contain none of their files, place content at another watched-folder save path. Record target-directory creation time, file size and modification time and monitor payload peer traffic. Invoke normal Start and forced Start separately. Confirm no configured-destination payload file is created, allocated, truncated or written and no content block is requested before search completion, actual-location settlement and successful check; confirm the found location becomes `actualStorageLocation()`, checking makes progress there, and release uses the originally requested mode.
    *   `[X]` **Integration scenario 11 — own-path matches (TX-3):** test content at the current active download path, at the save path with no separate download path, and at the save path while a separate download directory containing none of the torrent's files is configured. Confirm each path is searched once, no external assignment occurs, the last case clears the separate download path through the specified existing setter sequence so the save path becomes actual, every case completes the mandatory feature recheck even with **Recheck automatically** off, and Start is released only from `torrentFinishedChecking`.
    *   `[X]` **Integration scenario 12 — successful miss (TX-4):** with all candidate paths containing no torrent file, start normally and forcibly. Confirm search completes and logs a miss, no location setter or recheck is called, exactly one terminal `continued after no match` log is written, and the original Start behavior creates/downloads at the configured destination in the requested mode.
    *   `[X]` **Integration scenario 13 — coalescing and saved intent (TX-6):** hold one torrent separately in `WaitingForMetadata`, `Searching`, `WaitingForMove` and `WaitingForCheck`; issue at least two additional Start requests, ending with the opposite mode. Confirm the map retains one entry and one token, search/assignment/recheck each occur at most once, and the terminal Start uses only the last explicit mode.
    *   `[X]` **Integration scenario 14 — metadata-only acquisition (TX-2, TX-3):** start a stopped manual magnet without metadata. Confirm the metadata-only pass invokes the existing Start body once with `StopCondition::MetadataReceived`, payload files and content requests remain absent, a libtorrent-2 hybrid ID change re-keys the one map entry, `handleTorrentMetadataReceived()` starts one search after file names are ready, and match and miss then follow scenarios 10 and 12 without a second transaction.
    *   `[X]` **Integration scenario 15 — cancellation and late callbacks (TX-7):** cancel once with public Stop in each of `WaitingForMetadata`, `Searching`, `WaitingForMove` and `WaitingForCheck`; remove another torrent in each phase; and begin session shutdown with searches outstanding. Confirm state is erased, Stop works even while the torrent already reports Stopped, Stop, removal and shutdown each emit one cancellation log per pending Start, and delayed search, movement and check callbacks perform no assignment, recheck or Start. Also confirm `TorrentImpl::handleTorrentChecked()` uses `stop(false)` and therefore does not cancel a successful feature check before the session callback releases it.
    *   `[X]` **Integration scenario 16 — failure cleanup and force-recheck boundary (TX-8):** independently provoke asynchronous search failure, an assignment that cannot make the selected path actual, storage-move failure, metadata resume-data failure and file I/O error during the feature check. Confirm each pending Start is erased before ordinary Stop, produces one `failed` log naming the exact phase and reason, never produces a continuation log, leaves no live `FilesChecked` stop condition, and can be retried by explicit Start without restarting qBittorrent. Then provoke the known force-recheck bug outside Find Location and confirm its existing behavior is unchanged.
    *   `[X]` **Integration scenario 17 — responsiveness and isolation:** run a search over slow or unavailable volumes while repeatedly opening menus and issuing Stop; confirm the GUI thread never blocks. While feature transactions are active, complete unrelated storage moves, checks and file-error alerts for torrents absent from `m_locationAssignments`; confirm those torrents retain their baseline behavior and produce no Find Location terminal log.
    *   `[X]` **Integration scenario 18 — WebAPI actions:** with **Find location** enabled, post `torrents/findLocation` naming a mixed set of torrents: two whose content sits under a watched folder save path, one without content anywhere, one without metadata and one unknown ID. Confirm the answer is HTTP 202 listing the three torrents with metadata as pending and omitting the others, and that each match is assigned as its result arrives, before polling, following **Recheck automatically**, **Seed automatically** and **Leech automatically**. While one remains pending, post again naming a pending torrent, a matched torrent and a new torrent; confirm only the new torrent is searched. Poll without `hashes` until HTTP 200, and confirm `matched` carries each assigned torrent with its location in the form `save_path` uses and `unmatched` carries the miss. Poll again and confirm HTTP 200 with empty lists and no search. Remove an unmatched torrent before the final answer and confirm it is omitted. Run the desktop **Find location** on a torrent while a web session operation searches it, and confirm it receives one assignment. Post `torrents/assignLocation` with an empty location, a missing directory and an existing directory; confirm HTTP 400, HTTP 409 with no directory created, and an assignment following the same rules as a found location. Disable the group and confirm `torrents/findLocation` answers HTTP 409 and searches nothing. Confirm both actions refuse GET.
    *   `[X]` **Integration scenario 19 — web interface action and list:** confirm **Find location** follows **Set location...** in the web interface transfer list context menu over a selection holding a torrent with metadata, and is absent for a selection of torrents without metadata and while the group is disabled, including after the group is disabled from the web interface preferences page. Run it over one torrent matching nothing and confirm the **Set location** window opens with its save path, that saving an existing directory assigns it through `torrents/assignLocation`, and that saving a missing one shows the error and creates nothing. Run it over several torrents matching nothing and confirm `unmatchedtorrents.html` lists them by name, assigns an entered location to the selected entry and removes it, closes when the last entry is assigned, and assigns nothing further after **Close** or Escape. Remove a listed torrent before opening the list and confirm it is not listed. Invoke **Find location** again while a first invocation is polling and confirm one set of windows opens for the combined operation. Disable the group from another client while an invocation is polling and confirm the web interface reports the refusal; interrupt the connection during a poll and confirm polling resumes and the windows open once it completes. Operate the list by keyboard alone, and confirm Enter on **Close** closes the window without assigning. Confirm **Set location...** still moves a torrent to a directory it creates, as before.
    *   `[X]` The body carries `Closes #8261.` Commit after all nineteen integration scenarios pass, the full build and suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows, the WebUI lint and format checks pass, and the changelog passes `rumdl`.

## Epic 3 — discovery roots

*   `[X]` [BE] src/base/`discoveryroots` — add `DiscoveryRootOptions`, `DiscoveryRoot`, the `DiscoveryRoots` singleton holding an ordered list of roots, empty by default and persisted to `discovery_roots.json` as a JSON array holding one object per root, and the `parseDiscoveryRoots()` and `serializeDiscoveryRoots()` pair. T16

    *   `[X]` `objective`
        *   `[X]` Problem: the search reaches only paths the application already knows, so content held under a directory the user knows but has configured nowhere is never found.
        *   `[X]` Functional: hold an ordered list of discovery roots, each a directory with its own options, of which recursion is one, preserving the order the user configured.
        *   `[X]` Functional: persist the list across sessions as a JSON array of objects, each carrying `path` and `recursive`; load it at construction and write it when the list changes; start empty where no file exists.
        *   `[X]` Functional: convert between the list and its JSON form in one place, dropping an entry that is not an object, whose path is empty or relative, or whose path repeats an earlier entry.
        *   `[X]` Non-functional: the conversion reads no singleton and writes no log, so it is covered by a `qbt_base` test.

    *   `[X]` `role`
        *   `[X]` Settings storage in `src/base`, a singleton whose lifetime `Application` owns, read by `sessionimpl` and presented by `discoveryrootsmodel` and `appcontroller`.
        *   `[X]` Out of scope: enumerating a root, which is the `filesearcher` node; composing roots into a search, which is the `sessionimpl` node; checking that a directory exists or is readable, which the desktop model does before accepting an entry.

    *   `[X]` `module`
        *   `[X]` Inside: the root types, the ordered list, its JSON form, and its file. Outside: what the roots are searched for, and how they are edited.

    *   `[X]` `deps`
        *   `[X]` `base/path.h` — `Path`, `Path::isEmpty()`, `Path::isRelative()`, `Path::data()` and `operator==`. A lower layer, depended on inward.
        *   `[X]` `base/profile.h` — `specialFolderLocation(SpecialFolder::Config)`.
        *   `[X]` `base/utils/io.h` — `Utils::IO::readFile()`, `Utils::IO::ReadError::NotExist` and `Utils::IO::saveToFile()`.
        *   `[X]` `base/logger.h` — `LogMsg()` and `Log::WARNING`, from `load()` and `store()` only.
        *   `[X]` `base/global.h` — the `_s` literal.

    *   `[X]` `context_slice`
        *   `[X]` `TorrentFilesWatcher` is the model: `initInstance()`, `freeInstance()` and `instance()` over a static `m_instance`; a private constructor calling `load()`; `load()` reading `specialFolderLocation(SpecialFolder::Config) / Path(CONF_FILE_NAME)` through `Utils::IO::readFile()` with a 10 MiB limit and logging a warning on a read error other than `NotExist`, a parse error or a wrong document type; `store()` writing `QJsonDocument(...).toJson()` through `Utils::IO::saveToFile()` and logging a warning on failure; and the option key constant `OPTION_RECURSIVE`.
        *   `[X]` `TorrentFilesWatcher` keys its JSON object by folder path, which `QJsonObject` holds in sorted order; the discovery root list is ordered by the user, so it is held as a JSON array.

    *   `[X]` src/base/`discoveryroots.h`
        *   `[X]` New file: the licence header of `torrentfileswatcher.h` naming the contributor, `#pragma once`, `#include <QList>` and `#include <QObject>`, then `#include "base/path.h"`, with `class QJsonArray;` forward-declared.
        *   `[X]` `struct DiscoveryRootOptions` declaring `bool recursive = false;` and a defaulted equality operator.
        *   `[X]` `struct DiscoveryRoot` declaring in order `Path path;` and `DiscoveryRootOptions options;`, and a defaulted equality operator. Equality exists so an unchanged list can be rejected before persistence or notification.
        *   `[X]` `class DiscoveryRoots final : public QObject` with `Q_OBJECT` and `Q_DISABLE_COPY_MOVE(DiscoveryRoots)`, declaring public `static void initInstance();`, `static void freeInstance();`, `static DiscoveryRoots *instance();`, `QList<DiscoveryRoot> roots() const;` and `void setRoots(const QList<DiscoveryRoot> &roots);`; signal `void rootsChanged();`; private `explicit DiscoveryRoots(QObject *parent = nullptr);`, `void load();`, `void store() const;`, `static DiscoveryRoots *m_instance;` and `QList<DiscoveryRoot> m_roots;`.
        *   `[X]` After the class: `QList<DiscoveryRoot> parseDiscoveryRoots(const QJsonArray &jsonArray);` and `QJsonArray serializeDiscoveryRoots(const QList<DiscoveryRoot> &roots);`.

    *   `[X]` `construction`
        *   `[X]` `initInstance()` constructs the one instance when none exists, and the constructor calls `load()`, so `instance()` returns a complete list from its first call. `freeInstance()` deletes it and resets `m_instance`.

    *   `[X]` `interaction.spec`
        *   `[X]` `parseDiscoveryRoots(jsonArray)`: for each value in order, not an object → dropped; `Path(object.value(OPTION_PATH).toString())` empty or relative → dropped; a path already in the result by `operator==` → dropped; otherwise appended as `DiscoveryRoot {.path = path, .options = {.recursive = object.value(OPTION_RECURSIVE).toBool()}}`.
        *   `[X]` `serializeDiscoveryRoots(roots)`: one object per root in order, `{{OPTION_PATH, root.path.data()}, {OPTION_RECURSIVE, root.options.recursive}}`.
        *   `[X]` `load()`: the file absent → `m_roots` stays empty, silently; any other read error → a warning naming the error; a parse error → a warning naming the path and the error; a document that is not an array → a warning naming the path; otherwise `m_roots = parseDiscoveryRoots(document.array())`.
        *   `[X]` `setRoots(roots)`: `roots == m_roots` → return without writing or emitting; otherwise `m_roots = roots`, then `store()`, then `emit rootsChanged()`.
        *   `[X]` `store()`: writes `QJsonDocument(serializeDiscoveryRoots(m_roots)).toJson()`; a failed write → a warning naming the path and the error.

    *   `[X]` test/`testdiscoveryroots.cpp`
        *   `[X]` New file proving `parseDiscoveryRoots()` and `serializeDiscoveryRoots()`. Until the implementation element exists it fails to link with an unresolved symbol for each, and that link failure is its red state.
        *   `[X]` Form as the first epic's `testbittorrentfilesearcher.cpp`, with `class TestDiscoveryRoots`, includes `<QJsonArray>`, `<QJsonObject>`, `<QObject>`, `<QTest>`, `"base/discoveryroots.h"`, `"base/global.h"` and `"base/path.h"`, and no fixture. Paths are `/music` and `/video`.
        *   `[X]` `testRoundTripPreservesOrderAndOptions` — roots `/video` recursive, `/music` not recursive → serialised then parsed, the same two roots in the same order with the same flags.
        *   `[X]` `testSerializedShape` — `/music` recursive → a one-element array whose object holds `path` `/music` and `recursive` `true` and no other key.
        *   `[X]` `testRecursiveDefaultsFalse` — an object holding `path` `/music` alone → one root, not recursive.
        *   `[X]` `testEmptyPathDropped` — objects with `path` empty and `/music` → `/music` alone.
        *   `[X]` `testRelativePathDropped` — objects with `path` `music` and `/video` → `/video` alone.
        *   `[X]` `testNonObjectDropped` — the string `/music` then an object with `path` `/video` → `/video` alone.
        *   `[X]` `testDuplicateKeepsFirst` — `/music` recursive then `/music` not recursive → one root, recursive.
        *   `[X]` `testValueEquality` — two roots with the same path and options compare equal, and changing either the path or recursion flag makes them unequal. The storage-level no-op is verified manually because constructing `DiscoveryRoots` also reaches the profile and filesystem.

    *   `[X]` test/`CMakeLists.txt`
        *   `[X]` Add `testdiscoveryroots.cpp` to `testFiles` after `testconceptsstringable.cpp` and before `testglobal.cpp`.

    *   `[X]` src/base/`discoveryroots.cpp`
        *   `[X]` New file: the licence header of `torrentfileswatcher.cpp` naming the contributor, `#include "discoveryroots.h"`, then `#include <QJsonArray>`, `#include <QJsonDocument>`, `#include <QJsonObject>` and `#include <QJsonParseError>`, then `#include "base/global.h"`, `#include "base/logger.h"`, `#include "base/profile.h"` and `#include "base/utils/io.h"`.
        *   `[X]` File-scope constants `const QString CONF_FILE_NAME = u"discovery_roots.json"_s;`, `const QString OPTION_PATH = u"path"_s;` and `const QString OPTION_RECURSIVE = u"recursive"_s;`, then `DiscoveryRoots *DiscoveryRoots::m_instance = nullptr;`.
        *   `[X]` Define the singleton functions, the constructor, `roots()`, `setRoots()`, `load()`, `store()`, `parseDiscoveryRoots()` and `serializeDiscoveryRoots()` per the construction element and the interaction spec, each warning written with `tr()` and `LogMsg(..., Log::WARNING)`.

    *   `[X]` src/base/`CMakeLists.txt`
        *   `[X]` Add `discoveryroots.h` to the headers of `add_library(qbt_base ...)` after `digest32.h`, and `discoveryroots.cpp` to its sources after `bittorrent/trackerentrystatus.cpp` and before `exceptions.cpp`.

    *   `[X]` `directionality`
        *   `[X]` `discoveryroots` depends inward on `path`, `profile`, `utils/io`, `logger` and `global`, and on nothing in `src/base/bittorrent`, so `sessionimpl` depends on it without a cycle.

    *   `[X]` `requirements`
        *   `[X]` EN-1, ST-5: each root is a JSON object carrying `recursive`, proven by `testSerializedShape` and `testRoundTripPreservesOrderAndOptions`.
        *   `[X]` ST-6: with no file, `roots()` is empty.
        *   `[X]` ST-7: `DiscoveryRoots` is a singleton with `initInstance()` and `freeInstance()`, called by the `application` node.
        *   `[X]` CR-3: the configured order survives persistence, proven by `testRoundTripPreservesOrderAndOptions`.
        *   `[X]` BT-1: both files are registered in `src/base/CMakeLists.txt`.
        *   `[X]` BT-3: `testdiscoveryroots.cpp` is registered in `testFiles`.
        *   `[X]` `load()` and `store()` reach `specialFolderLocation()` and `LogMsg()`, which no `qbt_base` test initialises, so persistence to disk is verified by the explicit restart and malformed-file cases at the Epic 3 commit boundary.
        *   `[X]` Applying another Downloads setting while the discovery root list is unchanged performs no discovery-root write and emits no `rootsChanged()` signal, verified at the Epic 3 commit boundary.

*   `[X]` [BE] src/app/`application` — call `DiscoveryRoots::initInstance()` before `BitTorrent::Session::initInstance()` and `DiscoveryRoots::freeInstance()` after `BitTorrent::Session::freeInstance()`, so the list outlives every session read of it. T16

    *   `[X]` `objective`
        *   `[X]` Problem: `SessionImpl` reads the discovery root list whenever it composes a search, so the list must exist for the session's whole lifetime.
        *   `[X]` Functional: the singleton is constructed before the session and destroyed after it.

    *   `[X]` `context_slice`
        *   `[X]` `Application` calls `BitTorrent::Session::initInstance()` directly after `Net::DownloadManager::initInstance()`, and `TorrentFilesWatcher::initInstance()` later, inside the handler connected to `Session::restored`; it calls `BitTorrent::Session::freeInstance()` after `TorrentFilesWatcher::freeInstance()`. `SessionImpl` can reach `findExistingContent()` from `TorrentImpl::handleSaveResumeData()` for any torrent it holds, from construction to destruction, so the discovery root list is constructed before the session and destroyed after it rather than beside the watcher.

    *   `[X]` src/app/`application.cpp`
        *   `[X]` Add `#include "base/discoveryroots.h"` after `#include "base/bittorrent/torrent.h"`.
        *   `[X]` Add `DiscoveryRoots::initInstance();` after `Net::DownloadManager::initInstance();` and before `BitTorrent::Session::initInstance();`.
        *   `[X]` Add `DiscoveryRoots::freeInstance();` after `BitTorrent::Session::freeInstance();`.

    *   `[X]` `requirements`
        *   `[X]` ST-7: `Application` owns the singleton's lifetime.
        *   `[X]` `DiscoveryRoots::instance()` is non-null throughout the session's lifetime, verified by the dependency map's manual case for step 22.

*   `[X]` [BE] src/base/bittorrent/`filesearcher` — add `SubdirectoryMap` and `enumerateSubdirectories()`, walking every directory beneath a root into a map from case-folded name to every directory bearing it, and give `candidateRoots()` a trailing list of optional maps, so a search root with a map contributes, for each directory named for the torrent or its `.torrent` file at any depth, that directory's parent then the directory. T17, T18

    *   `[X]` `objective`
        *   `[X]` Problem: the root form, name form and source form reach only content folders directly beneath a search root or one folder below it. On a volume structured as `volume/someDeterminant/someOtherDeterminant/contentFolder/content`, every probe stops at `volume/someDeterminant`, and searching such a tree by probing would need a probe of every directory for every torrent.
        *   `[X]` Functional: walk every directory beneath a root once, at any depth, into a map from subdirectory name to every directory bearing that name, ordered by `Path::data()`, folding case where `Path::CASE_SENSITIVITY` is `Qt::CaseInsensitive`. Hidden directories, symbolic links and junctions are neither listed nor entered.
        *   `[X]` Functional: a search root supplied with a map contributes the root form; then, for each directory the map holds under the torrent's name and then under the source name, the directory's parent followed by the directory. The parent is where a torrent whose declared paths begin with its own folder is found, and the directory is where a single-file torrent or content nested in a folder of the torrent's name is found. A search root supplied without a map contributes the three forms it does now.
        *   `[X]` Non-functional: a root that cannot be listed yields an empty map and fails nothing, and a directory beneath it that cannot be listed contributes nothing beneath it while its siblings are walked; calls supplying no maps behave as they do now.

    *   `[X]` `role`
        *   `[X]` Domain logic in `src/base/bittorrent`, run on the session I/O thread.
        *   `[X]` Out of scope: deciding which roots are enumerated, and holding or discarding a map, which the `sessionimpl` node does.

    *   `[X]` `module`
        *   `[X]` Inside: walking a directory tree, folding names, and serving parent and directory candidates from a map. Outside: where roots and maps come from.

    *   `[X]` `deps`
        *   `[X]` `<QDir>`, `<QDirIterator>` and `<QFileInfo>` — a non-recursive `QDirIterator` per directory over `QDir::Dirs | QDir::NoDotAndDotDot | QDir::NoSymLinks`, `QDirIterator::nextFileInfo()` and `QFileInfo::isJunction()`, available from the Qt 6.6 minimum the build requires. `TorrentFilesWatcher::Worker::processFolder()` recurses into every subdirectory of a recursive watched folder, and this walk reaches the same depth.
        *   `[X]` `<algorithm>` — `std::ranges::sort`, ordering each map value by `Path::data()`.
        *   `[X]` `base/path.h` — `Path::filename()`, `Path::parentPath()` and `Path::CASE_SENSITIVITY`.
        *   `[X]` `<QHash>` and `<optional>` — the map and its per-root presence.

    *   `[X]` `context_slice`
        *   `[X]` The first epic's `filesearcher` node defines `candidateRoots()`, its containment check, its deduplication through `QList::contains`, and `testbittorrentcandidateroots.cpp`. `SubdirectoryMap` is keyed on `QString` because `qHash(const Path &, std::size_t)` hashes the unfolded string while `operator==` compares through `Path::CASE_SENSITIVITY`.

    *   `[X]` src/base/bittorrent/`filesearcher.h`
        *   `[X]` Add `#include <optional>` to a standard library group ahead of the Qt group, and `#include <QHash>` to the Qt group before `#include <QObject>`.
        *   `[X]` Before `struct FileSearchResult`, add `using SubdirectoryMap = QHash<QString, PathList>;`.
        *   `[X]` The `candidateRoots()` declaration gains the trailing parameter `const QList<std::optional<SubdirectoryMap>> &subdirectoryMaps = {}`.
        *   `[X]` After it, declare `SubdirectoryMap enumerateSubdirectories(const Path &root);`.

    *   `[X]` `interaction.spec`
        *   `[X]` `foldedName(name)`, new in the anonymous namespace: `name.toCaseFolded()` where `Path::CASE_SENSITIVITY` is `Qt::CaseInsensitive`, and `name` otherwise.
        *   `[X]` `enumerateSubdirectories(root)`: `root` empty → an empty map, since `QDirIterator` over an empty path lists the working directory. Otherwise a breadth-first walk over `PathList pending {root}`, indexed by `for (qsizetype i = 0; i < pending.size(); ++i)` with `const Path directory = pending.at(i);` copied before any append. Each `directory` is listed by `QDirIterator iter {directory.data(), (QDir::Dirs | QDir::NoDotAndDotDot | QDir::NoSymLinks)}`, without `QDirIterator::Subdirectories`. For each `const QFileInfo info = iter.nextFileInfo();`, an entry for which `info.isJunction()` holds is skipped; otherwise `const Path subdirectory {info.filePath()}` is appended to `map[foldedName(subdirectory.filename())]` and to `pending`. After the walk, each value of the map is sorted with `std::ranges::sort(paths, {}, &Path::data)`.
        *   `[X]` The filter omits `QDir::Hidden` and carries `QDir::NoSymLinks`, and the junction test runs on the same entry, so a hidden directory, a symbolic link and a junction are neither listed nor appended to `pending`, and the walk never enters them. The walk is explicit so that these tests govern descent as well as listing. A root or a directory beneath it that does not exist or cannot be read lists no entry, and the walk continues with the rest of `pending`. The root itself is never an entry.
        *   `[X]` `candidateRoots()`: the search root at index `i` of `searchRoots` takes `subdirectoryMaps.value(i)`; empty-entry substitution of `defaultSavePath` is unchanged. Without a map it contributes the root form, name form and source form as before.
        *   `[X]` With a map it contributes the root form; then, when `hasName`, for each `hit` of `map->value(foldedName(nameForm.data()))` in order, `hit.parentPath()` then `hit`; then, when `hasSource`, for each `hit` of `map->value(foldedName(sourceStem.data()))` in order, `hit.parentPath()` then `hit`. It contributes neither `effectiveRoot / nameForm` nor `effectiveRoot / sourceStem`. Each candidate passes `tryAppend`, the same exclusion and deduplication as every other, so a hit directly beneath the root collapses its parent into the root form and a parent shared by several hits is contributed once, at its first position.

    *   `[X]` test/testdata/`filesearcher`
        *   `[X]` test/testdata/filesearcher/library/Album One/`alpha.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/library/Album One/Disc 1/`alpha.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/library/album two/`alpha.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/library/`loose.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/library/Genre/Artist/Album Three/`alpha.txt` — the single line `fixture` followed by a line feed.
        *   `[X]` test/testdata/filesearcher/library/Genre/Other/Disc 1/`alpha.txt` — the single line `fixture` followed by a line feed.

    *   `[X]` test/`testbittorrentcandidateroots.cpp`
        *   `[X]` Add `#include <optional>`. Slots below pass search roots `/library` with the map list stated, over the file's default inputs unless stated otherwise. Map keys are formed by a local lambda folding case where `Path::CASE_SENSITIVITY` is `Qt::CaseInsensitive`, written below as the unfolded name.
        *   `[X]` `testDeepHitContributesParentThenDirectory` — maps holding `{Album → [/library/a/b/Album]}` → `/library`, `/library/a/b`, `/library/a/b/Album`.
        *   `[X]` `testShallowHitParentCollapsesIntoRootForm` — maps holding `{Album → [/library/Album]}` → `/library`, `/library/Album`.
        *   `[X]` `testEveryHitOfRepeatedNameContributes` — maps holding `{Album → [/library/a/Album, /library/b/Album]}` → `/library`, `/library/a`, `/library/a/Album`, `/library/b`, `/library/b/Album`.
        *   `[X]` `testSourceHitsFollowNameHits` — maps holding `{Album → [/library/p/Album], Album Rip → [/library/q/Album Rip]}` → `/library`, `/library/p`, `/library/p/Album`, `/library/q`, `/library/q/Album Rip`.
        *   `[X]` `testSharedParentContributesOnce` — maps holding `{Album → [/library/x/Album], Album Rip → [/library/x/Album Rip]}` → `/library`, `/library/x`, `/library/x/Album`, `/library/x/Album Rip`.
        *   `[X]` `testHitParentEqualToOwnPathIsExcluded` — save `/library/p`, maps holding `{Album → [/library/p/Album]}` → `/library`, `/library/p/Album`.
        *   `[X]` `testUncontainedNameIsNotLookedUp` — torrent name `..`, source file name empty, maps holding `{.. → [/elsewhere/inner]}` → `/library`.
        *   `[X]` `testEnumeratedRootContributesRootFormOnMiss` — maps holding one empty map → `/library`.
        *   `[X]` `testRootsBeyondMapsAreProbed` — search roots `/library`, `/watch`, maps holding one empty map → `/library`, `/watch`, `/watch/Album`, `/watch/Album Rip`.
        *   `[X]` `testNulloptMapIsProbed` — maps holding `std::nullopt` → `/library`, `/library/Album`, `/library/Album Rip`.
        *   `[X]` `testLookupFoldsCaseOnWindows`, compiled only under `#ifdef Q_OS_WIN` — torrent name `ALBUM`, maps holding `{album → [C:/library/x/Album]}`, search roots `C:/library` → `C:/library`, `C:/library/x`, `C:/library/x/Album`.

    *   `[X]` test/`testbittorrentsubdirectories.cpp`
        *   `[X]` New file proving `enumerateSubdirectories()`. Until the implementation element exists it fails to link with an unresolved symbol `enumerateSubdirectories`, and that link failure is its red state.
        *   `[X]` Form, includes and fixture root as `testbittorrentfilesearcher.cpp`, with `class TestBittorrentSubdirectories` and without `<QPromise>` and `"base/bittorrent/common.h"`. Add `<QFile>`, `<QFileInfo>`, `<QScopeGuard>` and `<QTemporaryDir>` for the slots building trees at run time, and `<QtSystemDetection>` for their platform guards. Expected keys are formed by a local lambda folding case where `Path::CASE_SENSITIVITY` is `Qt::CaseInsensitive`, and each value is compared as a `PathList`.
        *   `[X]` `testListsEveryDirectoryInTree` — root `library` → exactly the keys `Album One`, `Disc 1`, `album two`, `Genre`, `Artist`, `Album Three` and `Other`, with `Album One` → [`library/Album One`], `album two` → [`library/album two`], `Genre` → [`library/Genre`], `Artist` → [`library/Genre/Artist`], `Album Three` → [`library/Genre/Artist/Album Three`] and `Other` → [`library/Genre/Other`].
        *   `[X]` `testRepeatedNameListsEachInPathOrder` — root `library` → `Disc 1` → [`library/Album One/Disc 1`, `library/Genre/Other/Disc 1`].
        *   `[X]` `testFilesAndRootAreNotListed` — root `library` → no entry for `loose.txt`, `alpha.txt` or `library`.
        *   `[X]` `testHiddenDirectoriesAreNotEntered`, its body guarded by `#ifdef Q_OS_UNIX` with an `#else` branch calling `QSKIP("Requires dot-prefixed hidden directories.")` — in a `QTemporaryDir`, create `visible` and `.hidden/inner`, `QVERIFY` each setup operation → exactly the key `visible`.
        *   `[X]` `testSymbolicLinksAreNotFollowed`, its body guarded by `#ifdef Q_OS_UNIX` with an `#else` branch calling `QSKIP("Requires symbolic links.")` — in a `QTemporaryDir` `t`, create `real/inner`, then `QFile::link(t, real/loop)` and `QFile::link(real, linked)`, `QVERIFY` each setup operation → the walk returns, with exactly the keys `real` and `inner`, `real` → [`t/real`].
        *   `[X]` `testUnreadableBranchIsSkipped`, its body guarded by `#ifdef Q_OS_UNIX` with an `#else` branch calling `QSKIP("Requires Unix permission semantics.")` — in a `QTemporaryDir`, create `open/inside` and `closed/child`, remove the permissions of `closed` with `QFile::setPermissions(closed, {})`, and register a `qScopeGuard` restoring owner read, write and execute before `QTemporaryDir` cleans up. `QVERIFY` each setup operation. When `QFileInfo::exists()` still reports `closed/child`, use `QSKIP`, because the running account can traverse `closed`. Otherwise → exactly the keys `open`, `inside` and `closed`.
        *   `[X]` `testAbsentRootYieldsEmptyMap` — root `absent` → empty.
        *   `[X]` `testEmptyRootYieldsEmptyMap` — root `Path()` → empty.
        *   `[X]` `testKeysFoldCaseOnWindows`, compiled only under `#ifdef Q_OS_WIN` — root `library` → the map holds the keys `album one` and `album three`.

    *   `[X]` test/`CMakeLists.txt`
        *   `[X]` Add `testbittorrentsubdirectories.cpp` to `testFiles` after `testbittorrentpeeraddress.cpp` and before `testbittorrenttorrentdescriptor.cpp`.

    *   `[X]` src/base/bittorrent/`filesearcher.cpp`
        *   `[X]` Add `#include <algorithm>` in a standard library group ahead of the Qt group, and `#include <QDir>`, `#include <QDirIterator>` and `#include <QFileInfo>` before `#include <QPromise>`.
        *   `[X]` Add `foldedName` to the anonymous namespace, extend `candidateRoots()`, and define `enumerateSubdirectories()` after it, per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `filesearcher` depends inward on `path` and Qt core alone, and on nothing that depends on it.

    *   `[X]` `requirements`
        *   `[X]` EN-2, EN-3: `testListsEveryDirectoryInTree`, `testRepeatedNameListsEachInPathOrder`, `testFilesAndRootAreNotListed`.
        *   `[X]` EN-4, EN-5: `testKeysFoldCaseOnWindows`, `testLookupFoldsCaseOnWindows`; `SubdirectoryMap` is keyed on `QString`.
        *   `[X]` EN-8: `testHiddenDirectoriesAreNotEntered` and `testSymbolicLinksAreNotFollowed` on Unix. Windows does not give an unprivileged unit test a stable way to create a junction, so the dependency map's manual case for step 22 verifies that a junction pointing back at its root is not entered.
        *   `[X]` CR-8: `testDeepHitContributesParentThenDirectory`, `testShallowHitParentCollapsesIntoRootForm`, `testEveryHitOfRepeatedNameContributes`, `testSourceHitsFollowNameHits`, `testSharedParentContributesOnce`, `testHitParentEqualToOwnPathIsExcluded`, `testUncontainedNameIsNotLookedUp`, `testEnumeratedRootContributesRootFormOnMiss`.
        *   `[X]` PS-12: `testAbsentRootYieldsEmptyMap`, `testEmptyRootYieldsEmptyMap`, and the Unix-only `testUnreadableBranchIsSkipped`.
        *   `[X]` CN-1, CR-2: every existing slot of `testbittorrentcandidateroots.cpp`, `testbittorrentfilesearcher.cpp` and `testbittorrentfilesearchermultiroot.cpp` passes unmodified, alongside `testRootsBeyondMapsAreProbed` and `testNulloptMapIsProbed`.
        *   `[X]` BT-3, BT-5, BT-6: `testbittorrentsubdirectories.cpp` is registered and uses fixtures under `test/testdata/filesearcher`; the new `testbittorrentcandidateroots.cpp` slots use none.

*   `[X]` [BE] src/base/bittorrent/`sessionimpl` — add `Session::findTorrentLocations()`, a batch operation taking torrent IDs and an optional pointed root; compose each operation's search roots as the pointed root, then the discovery roots in configured order, then the watched folder save paths; enumerate the pointed root and every recursive discovery root once on the I/O thread for that operation; reuse those immutable maps for every torrent in a batch and for automatic additions arriving while an unchanged add operation is in flight; and preserve the first epic's exact candidate-membership attribution when naming pointed and discovery roots in the log. T22, T20

    *   `[X]` `objective`
        *   `[X]` Problem: the composition behind every discovery searches only the torrent's own paths and the watched folder save paths, so neither a configured discovery root nor a directory the user points at is searched.
        *   `[X]` Functional: compose the search roots as a pointed root where one is supplied, then the discovery roots in the order configured, then the watched folder save paths, all after the torrent's own save path and download path.
        *   `[X]` Functional: walk the tree beneath the pointed root and every discovery root marked recursive once for the operation that needs it, and hand those maps to `candidateRoots()`; a batch reuses the same maps for every torrent it submits, and an automatic addition reuses the maps of an add operation still in flight whose composed roots and default save path are unchanged.
        *   `[X]` Functional: accept a list of torrent IDs and an optional pointed root through `findTorrentLocations()`, use the pointed root for that operation alone, writing nothing to the discovery root list, report each result through the existing per-torrent signal as it completes, and enumerate the operation's roots once rather than once per torrent.
        *   `[X]` Functional: `findTorrentLocation()` keeps its declaration and its outcome, and runs as a one-torrent batch.
        *   `[X]` Functional: record the exact search root and whether it was the pointed root, a discovery root or a watched folder root, without inferring provenance from ancestry.
        *   `[X]` Non-functional: listing is filesystem I/O and runs on the session I/O thread with the probing; the operation-local maps are immutable while its torrent searches use them and are discarded after its last search ends. No session-lived filesystem cache is added.

    *   `[X]` `role`
        *   `[X]` Application service in `src/base/bittorrent`, the one place candidate roots are composed.
        *   `[X]` Out of scope: storing or editing discovery roots, which are the `discoveryroots` and `discoveryrootsmodel` nodes; asking the user for a directory, which is the `unmatchedtorrentsdialog` node.

    *   `[X]` `module`
        *   `[X]` Inside: the order of search roots, which of them are enumerated, sharing their maps within one operation, and the exact origin named in the log. Outside: how a map is built and how candidates are derived from it.

    *   `[X]` `deps`
        *   `[X]` `base/discoveryroots.h` — `DiscoveryRoots::instance()`, `DiscoveryRoots::roots()` and `DiscoveryRoot`, from the `discoveryroots` node. A lower layer, depended on inward.
        *   `[X]` `filesearcher.h` — `SubdirectoryMap`, `enumerateSubdirectories()` and the extended `candidateRoots()`, from the `filesearcher` node. Already included by `sessionimpl.cpp`.
        *   `[X]` `<memory>` — `std::shared_ptr`, holding one operation's snapshot and maps across its searches, and `std::weak_ptr`, referring to the in-flight add operation without owning it.

    *   `[X]` `context_slice`
        *   `[X]` `searchExistingContent()`, `findExistingContent()`, `findTorrentLocation()` and the log continuation are those the first and second epics' `sessionimpl` nodes define. `DiscoveryRoots::roots()` is read on the main thread, where the operation snapshot is composed before dispatching. `findIncompleteFiles()` dispatches onto the I/O thread through `QMetaObject::invokeMethod(m_fileSearcher, ...)`; functors queued to that one thread run in the order they are queued. The first epic attributes a winning watched-folder candidate by rebuilding each root's exact candidate set in order and taking the first set containing the winning path; this node extends that rule and does not replace it with an ancestry test.

    *   `[X]` src/base/bittorrent/`session.h`
        *   `[X]` After `virtual void findTorrentLocation(const TorrentID &id) = 0;`, which is unchanged, add `virtual void findTorrentLocations(const QList<TorrentID> &ids, const Path &pointedRoot = {}) = 0;`, as `removeTorrent()` carries a trailing default. It is an operation boundary for sharing enumeration; it does not replace the per-torrent completion signal.

    *   `[X]` src/base/bittorrent/`sessionimpl.h`
        *   `[X]` Add `#include <memory>` after `#include <functional>`.
        *   `[X]` After `void findTorrentLocation(const TorrentID &id) override;`, which is unchanged, add `void findTorrentLocations(const QList<TorrentID> &ids, const Path &pointedRoot = {}) override;`.
        *   `[X]` In the `private` section, before the `searchExistingContent()` declaration, forward-declare the nested `struct SearchOperation;` and declare, in order, `std::shared_ptr<SearchOperation> composeSearchOperation(const Path &pointedRoot) const;`, `void enumerateSearchOperation(const std::shared_ptr<SearchOperation> &operation);` and `std::shared_ptr<SearchOperation> createSearchOperation(const Path &pointedRoot);`.
        *   `[X]` After `QHash<TorrentID, LocationAssignmentState> m_locationAssignments;`, add `std::weak_ptr<SearchOperation> m_inFlightAddOperation;`.
        *   `[X]` The `searchExistingContent()` declaration gains the trailing parameter `std::shared_ptr<const SearchOperation> operation`.

    *   `[X]` src/base/bittorrent/`sessionimpl.cpp` types
        *   `[X]` `struct SessionImpl::SearchOperation`, defined in `sessionimpl.cpp` before its first use, declares `enum class Origin { Pointed, Discovery, WatchedFolder };`, `struct Root { Path path; Origin origin; bool enumerated; bool operator==(const Root &) const = default; };`, and the members `QList<Root> roots;`, `PathList searchRoots;`, `Path defaultSavePath;` and `QList<std::optional<SubdirectoryMap>> subdirectoryMaps;`. `searchRoots` holds `roots`' paths in the same order, as `candidateRoots()` takes them.

    *   `[X]` `interaction.spec`
        *   `[X]` `composeSearchOperation(pointedRoot)`, on the main thread and touching no filesystem, builds a `std::make_shared<SearchOperation>()` whose `roots` hold, in order: `pointedRoot` where non-empty, as `Origin::Pointed` and enumerated; each `DiscoveryRoot` of `DiscoveryRoots::instance()->roots()`, as `Origin::Discovery` and enumerated exactly when `root.options.recursive`; then each entry of `m_watchedFolderSavePaths` resolved as the first epic resolves it, as `Origin::WatchedFolder` and not enumerated. It fills `searchRoots` from `roots`, sets `defaultSavePath` to `savePath()`, and returns the operation.
        *   `[X]` `enumerateSearchOperation(operation)`: when any root of `operation` is enumerated, queue one functor onto `m_fileSearcher` through `QMetaObject::invokeMethod`, capturing the operation, that fills `subdirectoryMaps` with `enumerateSubdirectories(roots[i].path)` at each enumerated index and `std::nullopt` at each other. When none is, `subdirectoryMaps` stays empty, `subdirectoryMaps.value(i)` yields `std::nullopt` for every index, and nothing is listed.
        *   `[X]` `createSearchOperation(pointedRoot)`: `composeSearchOperation(pointedRoot)`, then `enumerateSearchOperation()` on the result, then return it.
        *   `[X]` The enumeration functor is queued before every search of its operation, including every search of an automatic addition that reuses it, so on the one I/O thread it completes before any of them reads the maps; after it, nothing writes the operation. Each search reaches its main-thread continuation through its promise, so the continuation reads maps already written.
        *   `[X]` `searchExistingContent(..., operation)` keeps the first epic's derivation of `nameFormName`, and reads `const bool appendExtension = isAppendExtensionEnabled();` on the calling thread. Inside the functor it queues onto `m_fileSearcher`, it computes `candidates` as `candidateRoots(torrentSavePath, torrentDownloadPath, operation->searchRoots, operation->defaultSavePath, nameFormName, sourceFileName, operation->subdirectoryMaps)`, replacing the first epic's `candidateRoots()` call on the calling thread, because the maps exist only once the enumeration functor has run. It then calls `m_fileSearcher->searchRoots(filePaths, torrentSavePath, torrentDownloadPath, candidates, appendExtension, promise)`. The functor and the continuation each capture `operation` by value, so the operation is released when the last continuation of its last search ends. No session-lived filesystem cache is added.
        *   `[X]` The continuation with `result.foundAtOwnPath` false and `result.matchCount` above 0 takes as origin the first index `i` of `operation->roots` for which `candidateRoots({}, {}, {roots[i].path}, defaultSavePath, nameFormName, sourceFileName, {subdirectoryMaps.value(i)})` contains the winning location, with the entry read as the first epic reads it, and names it `tr("pointed root \"%1\"")`, `tr("discovery root \"%1\"")` or `tr("watched folder save path \"%1\"")` by that root's `Origin`, substituting that exact root, in the first epic's "Found existing torrent content" message. No origin is classified because the winning path has a root as an ancestor. The continuation with `result.matchCount` equal to 0 and `result.searchedCandidates` true logs the first epic's "Existing torrent content not found" message. Otherwise it logs nothing.
        *   `[X]` `findExistingContent()` returns `findIncompleteFiles(torrentSavePath, torrentDownloadPath, filePaths)` first when either setting is off, and obtains an operation only in the branch that follows. There it calls `composeSearchOperation({})` and locks `m_inFlightAddOperation`. A live operation whose `roots` and `defaultSavePath` equal the composed operation's is used. Otherwise `enumerateSearchOperation()` is called on the composed operation, `m_inFlightAddOperation` is assigned it, and it is used. The operation used is passed to `searchExistingContent()`. `m_inFlightAddOperation` owns nothing, so the operation and its maps are released when the last continuation holding it ends. **Find location** and **Search folder...** operations are never assigned to `m_inFlightAddOperation`.
        *   `[X]` `findTorrentLocation(id)` → `findTorrentLocations({id})`, keeping the second epic's outcomes.
        *   `[X]` `findTorrentLocations(ids, pointedRoot)`: for each ID in order, no torrent in `m_torrents` under it, or a torrent without metadata → `emit torrentLocationFound(id, {}, false)`; otherwise the torrent joins the valid list. An empty valid list → return, creating no operation. Otherwise one `createSearchOperation(pointedRoot)` is shared by every valid torrent's `searchExistingContent(torrent->savePath(), torrent->downloadPath(), torrent->filePaths(), torrent->info().name(), {}, operation)`, each continued with `.then(this, ...)` into `emit torrentLocationFound(id, result.savePath, ((result.matchCount > 0) || result.foundAtOwnPath))` as its own search completes.

    *   `[X]` src/base/bittorrent/`sessionimpl.cpp`
        *   `[X]` Add `#include "base/discoveryroots.h"` after `#include "base/algorithm.h"`.
        *   `[X]` Define `SessionImpl::SearchOperation`, `SessionImpl::composeSearchOperation()`, `SessionImpl::enumerateSearchOperation()` and `SessionImpl::createSearchOperation()` before `SessionImpl::searchExistingContent()`; revise `SessionImpl::searchExistingContent()`, `SessionImpl::findExistingContent()` and `SessionImpl::findTorrentLocation()`; and define `SessionImpl::findTorrentLocations()` after `SessionImpl::findTorrentLocation()`, per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `sessionimpl` gains a dependency on `discoveryroots` in `src/base`, which depends on nothing in `src/base/bittorrent`. `src/gui` supplies a pointed root only through `Session::findTorrentLocations()`.

    *   `[X]` `requirements`
        *   `[X]` CR-3: the pointed root precedes the discovery roots, which precede the watched folder save paths, in configured order.
        *   `[X]` CR-10: the discovery root list is read in `createSearchOperation()`, ahead of the watched folder save paths, and maps are supplied for recursive roots.
        *   `[X]` EN-2, EN-6, EN-7: the pointed root is enumerated on the terms of a recursive discovery root, each root is listed once per operation, automatic additions arriving while an unchanged add operation is in flight share its listing, and the maps live no longer than that operation.
        *   `[X]` ST-8: nothing in this node writes to `DiscoveryRoots`.
        *   `[X]` RL-3, RL-5: `enumerateSubdirectories()` runs on `m_fileSearcher`'s thread.
        *   `[X]` ST-4: with either setting off, `findExistingContent()` returns `findIncompleteFiles()`'s result, reading no discovery root.
        *   `[X]` IF-13: the batch operation retains the existing per-torrent completion signal, so each outcome is reported as its search completes.
        *   `[X]` LG-1: the origin identifies the exact candidate-producing root. Nested pointed, discovery and watched-folder roots cannot be misclassified through ancestry.
        *   `[X]` This node is verified by the dependency map's manual case for step 22 and the Epic 3 batch-enumeration and overlapping-origin cases at the commit boundary.

*   `[X]` [UI] src/gui/`transferlistwidget` — submit each **Find location** invocation's new IDs through one `Session::findTorrentLocations()` call, so a multi-torrent invocation enumerates each recursive root once. T22

    *   `[X]` `objective`
        *   `[X]` Problem: the second epic's second pass calls `findTorrentLocation()` once per ID, so each torrent would be its own operation and list every recursive discovery root again.
        *   `[X]` Functional: one invocation's newly registered IDs form one operation.

    *   `[X]` `context_slice`
        *   `[X]` `findSelectedTorrentsLocation()`, `submitted`, `m_findLocationOperation` and `handleTorrentLocationFound()` are those the second epic's `transferlistwidget` node defines.

    *   `[X]` src/gui/`transferlistwidget.cpp`
        *   `[X]` In `findSelectedTorrentsLocation()`, the second pass becomes one `BitTorrent::Session::instance()->findTorrentLocations(submitted);`, made only when `submitted` is non-empty. The first pass, registration of every ID before that call, the operation map, `handleTorrentLocationFound()` and assignment are unchanged.

    *   `[X]` `directionality`
        *   `[X]` `transferlistwidget` depends downward on `Session` as it already does.

    *   `[X]` `requirements`
        *   `[X]` EN-6: a multi-torrent invocation is one operation, verified by the batch-enumeration case at the Epic 3 commit boundary.
        *   `[X]` IF-13: each outcome still arrives through `torrentLocationFound` as its search completes; Epic 2 integration scenarios 4 and 5 still pass.

*   `[X]` [API] src/webui/api/`torrentscontroller` — submit each `torrents/findLocation` request's new IDs through one `Session::findTorrentLocations()` call, so a multi-torrent request enumerates each recursive root once, and accept an optional `root` as that operation's pointed root. T22, T25

    *   `[X]` `objective`
        *   `[X]` Problem: the second epic's `findLocationAction()` calls `findTorrentLocation()` once per registered ID, so each torrent would be its own operation and list every recursive discovery root again.
        *   `[X]` Problem: a client of the web interface or the headless daemon cannot point a search at a directory, which the desktop unmatched list does through **Search folder...**.
        *   `[X]` Functional: one request's newly registered IDs form one operation.
        *   `[X]` Functional: a request carrying `root` searches its newly registered IDs with that directory as the pointed root, and writes the directory nowhere.

    *   `[X]` `context_slice`
        *   `[X]` `findLocationAction()`, `submitted`, `m_findLocationOperation` and `onTorrentLocationFound()` are those the second epic's `torrentscontroller` node defines.

    *   `[X]` src/webui/api/`torrentscontroller.cpp`
        *   `[X]` In `findLocationAction()`, after the disabled test, read `const Path root {params()[u"root"_s].trimmed()};`. A non-empty `root` for which `!Utils::Fs::isDir(root)` → `throw APIError(APIErrorType::Conflict, tr("Folder does not exist"))`, before any ID is registered.
        *   `[X]` In `findLocationAction()`, the calls to `findTorrentLocation()` over `submitted` become one `BitTorrent::Session::instance()->findTorrentLocations(submitted, root);`, made only when `submitted` is non-empty, so an empty `root` is no pointed root. Registration of every ID before that call, the operation map, `onTorrentLocationFound()`, assignment and the answer are unchanged.

    *   `[X]` `directionality`
        *   `[X]` `torrentscontroller` depends downward on `Session` as it already does, and writes nothing to `DiscoveryRoots`.

    *   `[X]` `requirements`
        *   `[X]` EN-6: a multi-torrent request is one operation, verified by the batch-enumeration case at the Epic 3 commit boundary.
        *   `[X]` IF-20: a request carrying an existing `root` searches its registered torrents with it as the pointed root; a `root` that is not an existing directory answers HTTP 409 and registers nothing.
        *   `[X]` ST-8: `root` reaches only `findTorrentLocations()`.
        *   `[X]` IF-12: the new message passes through `tr()`.
        *   `[X]` Verify the parameter through Epic 3 integration scenario 10 at the commit boundary.
        *   `[X]` IF-13: each outcome still arrives through `torrentLocationFound` as its search completes; Epic 2 integration scenarios 18 and 19 still pass.

*   `[X]` [UI] src/gui/`discoveryrootsmodel` — add `DiscoveryRootsModel`, a `QAbstractListModel` over the discovery root list, following `WatchedFoldersModel`, holding edits until `apply()` writes the list through `DiscoveryRoots::setRoots()`. T19

    *   `[X]` `objective`
        *   `[X]` Problem: the options dialog needs an editable, ordered view of the discovery root list whose edits take effect only when the page is applied.
        *   `[X]` Functional: present each root by path in configured order; add a root at the end after checking it; remove rows; read and replace a row's options; write the whole list on `apply()`.
        *   `[X]` Functional: reload when the stored list changes by another route.

    *   `[X]` `role`
        *   `[X]` Presentation model in `src/gui`.
        *   `[X]` Out of scope: persistence, which is the `discoveryroots` node; the per-root dialog and the view, which are the `discoveryrootoptionsdialog` and `optionsdialog` nodes.

    *   `[X]` `module`
        *   `[X]` Inside: the pending list, its rows, its validation and its application. Outside: storage and presentation of options.

    *   `[X]` `deps`
        *   `[X]` `base/discoveryroots.h` — `DiscoveryRoots`, `DiscoveryRoot`, `DiscoveryRootOptions`, `roots()`, `setRoots()` and `rootsChanged`. A lower layer, depended on inward.
        *   `[X]` `base/exceptions.h` — `InvalidArgument` and `RuntimeError`.
        *   `[X]` `<QDir>` — `QDir::exists()` and `QDir::isReadable()`.

    *   `[X]` `context_slice`
        *   `[X]` `WatchedFoldersModel` is the model: constructed with the storage object and a parent, one column, `Qt::DisplayRole` returning `toString()` of the path, `headerData()` returning one translated title, `removeRows()` validating its range and bracketing the removal with `beginRemoveRows()` and `endRemoveRows()`, `addFolder()` throwing `InvalidArgument` for an empty or relative path and `RuntimeError` for a duplicate, missing or unreadable one, `folderOptions()` and `setFolderOptions()` asserting the row, and `apply()`.

    *   `[X]` src/gui/`discoveryrootsmodel.h`
        *   `[X]` New file: the licence header of `watchedfoldersmodel.h` naming the contributor, `#pragma once`, `#include <QAbstractListModel>` and `#include <QList>`, then `#include "base/discoveryroots.h"` and `#include "base/path.h"`.
        *   `[X]` `class DiscoveryRootsModel final : public QAbstractListModel` with `Q_OBJECT` and `Q_DISABLE_COPY_MOVE(DiscoveryRootsModel)`, declaring public `explicit DiscoveryRootsModel(DiscoveryRoots *discoveryRoots, QObject *parent = nullptr);`, the overrides `rowCount()`, `columnCount()`, `data()`, `headerData()` and `removeRows()` with `WatchedFoldersModel`'s signatures, `void addRoot(const Path &path, const DiscoveryRootOptions &options);`, `DiscoveryRootOptions rootOptions(int row) const;`, `void setRootOptions(int row, const DiscoveryRootOptions &options);` and `void apply();`; private `void onRootsChanged();`, `DiscoveryRoots *m_discoveryRoots = nullptr;` and `QList<DiscoveryRoot> m_roots;`.

    *   `[X]` `construction`
        *   `[X]` The constructor takes the storage object, copies `roots()` into `m_roots`, and connects `DiscoveryRoots::rootsChanged` to `onRootsChanged`, so the model is complete on return.

    *   `[X]` `interaction.spec`
        *   `[X]` `rowCount(parent)`: a valid parent → 0; otherwise `m_roots.size()`. `columnCount()` → 1. `data(index, role)`: an index out of range → `{}`; `Qt::DisplayRole` → `m_roots.at(row).path.toString()`; any other role → `{}`. `headerData()`: horizontal display role for section 0 → `tr("Discovery root")`; otherwise `{}`.
        *   `[X]` `removeRows(row, count, parent)`: a valid parent or a range outside the rows → `false`; otherwise the rows are removed between `beginRemoveRows()` and `endRemoveRows()`, and `true`.
        *   `[X]` `addRoot(path, options)`: empty → `InvalidArgument(tr("Discovery root path cannot be empty."))`; relative → `InvalidArgument(tr("Discovery root path cannot be relative."))`; already present by `operator==` → `RuntimeError(tr("Folder '%1' is already in the discovery root list.").arg(path.toString()))`; `QDir` not existing → `RuntimeError(tr("Folder '%1' doesn't exist.").arg(path.toString()))`; not readable → `RuntimeError(tr("Folder '%1' isn't readable.").arg(path.toString()))`; otherwise appended between `beginInsertRows()` and `endInsertRows()`.
        *   `[X]` `rootOptions(row)` and `setRootOptions(row, options)` assert `row` is in range, then read or replace `m_roots[row].options`.
        *   `[X]` `apply()` → `m_discoveryRoots->setRoots(m_roots)`.
        *   `[X]` `onRootsChanged()` → `beginResetModel()`, `m_roots = m_discoveryRoots->roots()`, `endResetModel()`.

    *   `[X]` src/gui/`discoveryrootsmodel.cpp`
        *   `[X]` New file: the licence header of `watchedfoldersmodel.cpp` naming the contributor, `#include "discoveryrootsmodel.h"`, then `#include <QDir>`, then `#include "base/exceptions.h"`, and every member per the interaction spec.

    *   `[X]` src/gui/`CMakeLists.txt`
        *   `[X]` Add `discoveryrootsmodel.h` to the headers of `add_library(qbt_gui ...)` after `desktopintegration.h`, and `discoveryrootsmodel.cpp` to its sources after `desktopintegration.cpp`.

    *   `[X]` `directionality`
        *   `[X]` `discoveryrootsmodel` in `src/gui` depends downward on `discoveryroots` and `exceptions` in `src/base`. Only `optionsdialog` depends on it.

    *   `[X]` `requirements`
        *   `[X]` CR-3: rows keep the configured order, new roots are appended, and `apply()` writes that order.
        *   `[X]` IF-12: every message and the header title are wrapped in `tr()`.
        *   `[X]` BT-2: both files are registered in `src/gui/CMakeLists.txt`.
        *   `[X]` The model is verified by the dependency map's manual case for step 19.

*   `[X]` [UI] src/gui/`discoveryrootoptionsdialog` — add `DiscoveryRootOptionsDialog`, following `WatchedFolderOptionsDialog`, carrying the **Recursive mode** checkbox for one root's `DiscoveryRootOptions`. T19

    *   `[X]` `objective`
        *   `[X]` Problem: a discovery root's options need a dialog for adding and editing, as a watched folder's do.
        *   `[X]` Functional: show one root's options, with recursion as a checkbox, and return the options as edited when accepted.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/gui`, opened by the `optionsdialog` node.
        *   `[X]` Out of scope: which root the options belong to, and applying them.

    *   `[X]` `module`
        *   `[X]` Inside: one checkbox and its options value. Outside: the root list.

    *   `[X]` `deps`
        *   `[X]` `base/discoveryroots.h` — `DiscoveryRootOptions`. A lower layer, depended on inward.
        *   `[X]` `ui_discoveryrootoptionsdialog.h` — generated from `discoveryrootoptionsdialog.ui`.

    *   `[X]` `context_slice`
        *   `[X]` `WatchedFolderOptionsDialog` is the model: a constructor taking the options and a parent, a forward-declared `Ui` class, a raw `m_ui`, `setupUi()`, `accepted` and `rejected` connected to `accept()` and `reject()`, an accessor building the options from the checkbox, and `checkBoxRecursive` with text `Recursive mode`.

    *   `[X]` src/gui/`discoveryrootoptionsdialog.h`
        *   `[X]` New file: the licence header of `watchedfolderoptionsdialog.h` naming the contributor, `#pragma once`, `#include <QDialog>`, `#include "base/discoveryroots.h"`, and `namespace Ui { class DiscoveryRootOptionsDialog; }`.
        *   `[X]` `class DiscoveryRootOptionsDialog final : public QDialog` with `Q_OBJECT` and `Q_DISABLE_COPY_MOVE(DiscoveryRootOptionsDialog)`, declaring public `explicit DiscoveryRootOptionsDialog(const DiscoveryRootOptions &options, QWidget *parent);`, `~DiscoveryRootOptionsDialog() override;` and `DiscoveryRootOptions discoveryRootOptions() const;`, and private `Ui::DiscoveryRootOptionsDialog *m_ui = nullptr;`.

    *   `[X]` src/gui/`discoveryrootoptionsdialog.ui`
        *   `[X]` New file in the form of `watchedfolderoptionsdialog.ui`: class `DiscoveryRootOptionsDialog`, a `QDialog` with `windowTitle` `Discovery Root Options`, laid out by a `QVBoxLayout` named `verticalLayout` holding, in order, a `QCheckBox` named `checkBoxRecursive` with `text` `Recursive mode` and `toolTip` `Search every folder beneath this folder, at any depth, for a folder named as the torrent or its .torrent file.`, a vertical spacer, and a `QDialogButtonBox` named `buttonBox` with `standardButtons` `QDialogButtonBox::StandardButton::Cancel|QDialogButtonBox::StandardButton::Ok`.

    *   `[X]` `interaction.spec`
        *   `[X]` Constructor: the initialiser list is `QDialog {parent}` then `m_ui {new Ui::DiscoveryRootOptionsDialog}`, as `WatchedFolderOptionsDialog` allocates its `m_ui`; the body calls `m_ui->setupUi(this)`, checks `checkBoxRecursive` as `options.recursive`, and connects `buttonBox` `accepted` and `rejected` to `accept()` and `reject()`. The destructor deletes `m_ui`.
        *   `[X]` `discoveryRootOptions()` → `DiscoveryRootOptions {.recursive = m_ui->checkBoxRecursive->isChecked()}`.

    *   `[X]` src/gui/`discoveryrootoptionsdialog.cpp`
        *   `[X]` New file: the licence header of `watchedfolderoptionsdialog.cpp` naming the contributor, `#include "discoveryrootoptionsdialog.h"`, `#include "ui_discoveryrootoptionsdialog.h"`, and the constructor, the destructor deleting `m_ui`, and `discoveryRootOptions()` per the interaction spec.

    *   `[X]` src/gui/`CMakeLists.txt`
        *   `[X]` Add `discoveryrootoptionsdialog.ui` to `qt_wrap_ui(UI_HEADERS ...)` after `deletionconfirmationdialog.ui`, `discoveryrootoptionsdialog.h` to the headers of `add_library(qbt_gui ...)` after `desktopintegration.h` and before `discoveryrootsmodel.h`, and `discoveryrootoptionsdialog.cpp` to its sources after `desktopintegration.cpp` and before `discoveryrootsmodel.cpp`.

    *   `[X]` `directionality`
        *   `[X]` `discoveryrootoptionsdialog` in `src/gui` depends downward on `discoveryroots`. Only `optionsdialog` depends on it.

    *   `[X]` `requirements`
        *   `[X]` IF-3: the per-root dialog carries the recursion flag.
        *   `[X]` IF-12: `uic` routes the `.ui` strings through `tr()`.
        *   `[X]` BT-2: the three files are registered in `src/gui/CMakeLists.txt`.
        *   `[X]` The dialog is verified by the dependency map's manual case for step 19.

*   `[X]` [UI] src/gui/`optionsdialog` — add the discovery root list `discoveryRootsView`, backed by `DiscoveryRootsModel`, with **Add...**, **Options...** and **Remove** buttons opening `DiscoveryRootOptionsDialog`, inside `groupFindLocation`, applied with the page. T19

    *   `[X]` `objective`
        *   `[X]` Problem: discovery roots have no desktop control.
        *   `[X]` Functional: a list of discovery roots within the **Find location** group, with buttons to add a root through a directory chooser and the per-root dialog, to edit the selected root's options, and to remove the selected root; any change enables **Apply**, and applying writes the list.
        *   `[X]` Non-functional: the list and its buttons follow the watched folder list's layout and keyboard behaviour, and are disabled with the group.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/gui`.
        *   `[X]` Out of scope: the web interface control, which is the `preferences.html` node.

    *   `[X]` `module`
        *   `[X]` Inside: the view, its three buttons, their handlers, and the model's application. Outside: storage and the model's rules.

    *   `[X]` `deps`
        *   `[X]` `base/discoveryroots.h` — `DiscoveryRoots::instance()`.
        *   `[X]` `discoveryrootsmodel.h` and `discoveryrootoptionsdialog.h` — from the `discoveryrootsmodel` and `discoveryrootoptionsdialog` nodes.

    *   `[X]` `context_slice`
        *   `[X]` The watched folder list is the model throughout: `scanFoldersView` with `addWatchedFolderButton`, `editWatchedFolderButton` and `removeWatchedFolderButton` in `optionsdialog.ui`; `WatchedFoldersModel` set on the view in `loadDownloadsTabOptions()` with its `dataChanged`, selection and `doubleClicked` connections; the add and remove buttons connected to `enableApplyButton`; `apply()` in `saveDownloadsTabOptions()`; and the auto-connected slots `on_addWatchedFolderButton_clicked()`, `on_editWatchedFolderButton_clicked()` and `on_removeWatchedFolderButton_clicked()` with `handleWatchedFolderViewSelectionChanged()` and `editWatchedFolderOptions()`.

    *   `[X]` src/gui/`optionsdialog.h`
        *   `[X]` In `private slots`, after `void editWatchedFolderOptions(const QModelIndex &index);`, add `void handleDiscoveryRootViewSelectionChanged();` and `void editDiscoveryRootOptions(const QModelIndex &index);`; after `void on_removeWatchedFolderButton_clicked();`, add `void on_addDiscoveryRootButton_clicked();`, `void on_editDiscoveryRootButton_clicked();` and `void on_removeDiscoveryRootButton_clicked();`.

    *   `[X]` src/gui/`optionsdialog.ui`
        *   `[X]` In `groupFindLocationLayout`, after the `<item>` holding `checkFindLocationLeech`, add two `<item>`s in order.
        *   `[X]` The first holds `<widget class="QLabel" name="labelDiscoveryRoots">` with property `text` set to `Also search these folders:`.
        *   `[X]` The second holds `<layout class="QHBoxLayout" name="discoveryRootsLayout">` containing two `<item>`s in order.
        *   `[X]` The first item of `discoveryRootsLayout` holds `<widget class="QTreeView" name="discoveryRootsView">` with, in order, property `sizePolicy` set to `<sizepolicy hsizetype="Expanding" vsizetype="Expanding">` with `horstretch` 0 and `verstretch` 1; property `minimumSize` set to width 250 and height 100; property `selectionMode` set to `QAbstractItemView::SelectionMode::SingleSelection`; property `selectionBehavior` set to `QAbstractItemView::SelectionBehavior::SelectRows`; property `textElideMode` set to `Qt::TextElideMode::ElideNone`; property `rootIsDecorated` set to `false`; attribute `headerDefaultSectionSize` set to 80; and attribute `headerStretchLastSection` set to `false`. It has no `editTriggers` property.
        *   `[X]` The second item of `discoveryRootsLayout` holds `<layout class="QVBoxLayout" name="discoveryRootsButtonsLayout">` containing four `<item>`s in order: `<widget class="QPushButton" name="addDiscoveryRootButton">` with `text` `Add...`; `<widget class="QPushButton" name="editDiscoveryRootButton">` with `enabled` `false` and `text` `Options..`; `<widget class="QPushButton" name="removeDiscoveryRootButton">` with `enabled` `false` and `text` `Remove`; and `<spacer name="discoveryRootsSpacer">` with `orientation` `Qt::Orientation::Vertical` and `sizeHint` width 20 and height 40.

    *   `[X]` `interaction.spec`
        *   `[X]` Load: after the `scanFoldersView` `doubleClicked` connection, a `DiscoveryRootsModel(DiscoveryRoots::instance(), this)` with `dataChanged` connected to `enableApplyButton`; `discoveryRootsView` header resized to contents and its model set; its selection model's `selectionChanged` connected to `handleDiscoveryRootViewSelectionChanged`; its `doubleClicked` connected to `editDiscoveryRootOptions`.
        *   `[X]` Connect: after the `addWatchedFolderButton` connection, `addDiscoveryRootButton` and `removeDiscoveryRootButton` `clicked` connected to `enableApplyButton`.
        *   `[X]` Save: after `watchedFoldersModel->apply();`, the `DiscoveryRootsModel` of `discoveryRootsView` → `apply()`.
        *   `[X]` `on_addDiscoveryRootButton_clicked()`: `QFileDialog::getExistingDirectory(this, tr("Select folder to search"))` empty → return; otherwise a heap `DiscoveryRootOptionsDialog({}, this)` with `Qt::WA_DeleteOnClose`, whose `accepted` calls `addRoot()` with the directory and `discoveryRootOptions()`, resizes the view's columns and enables **Apply**, a `RuntimeError` showing `QMessageBox::critical(this, tr("Adding entry failed"), err.message())`; then `open()`.
        *   `[X]` `on_editDiscoveryRootButton_clicked()` → `editDiscoveryRootOptions()` on the first selected index. `editDiscoveryRootOptions(index)`: invalid → return; otherwise a heap `DiscoveryRootOptionsDialog(model->rootOptions(index.row()), this)` with `Qt::WA_DeleteOnClose`, whose `accepted`, with the index still valid, calls `setRootOptions()` and enables **Apply**; then `open()`.
        *   `[X]` `on_removeDiscoveryRootButton_clicked()` → `removeRow()` for each selected index. `handleDiscoveryRootViewSelectionChanged()` → **Remove** enabled with any selection, **Options...** enabled with exactly one.

    *   `[X]` src/gui/`optionsdialog.cpp`
        *   `[X]` Add `#include "base/discoveryroots.h"` after `#include "base/bittorrent/sharelimits.h"`, and `#include "discoveryrootoptionsdialog.h"` then `#include "discoveryrootsmodel.h"` after `#include "banlistoptionsdialog.h"`.
        *   `[X]` Add the load, connection and save statements, and define the five slots after `OptionsDialog::editWatchedFolderOptions()`, per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` `optionsdialog` depends beside it on `discoveryrootsmodel` and `discoveryrootoptionsdialog`, and downward on `discoveryroots`.

    *   `[X]` `requirements`
        *   `[X]` IF-3: the list opens the per-root dialog for adding and editing.
        *   `[X]` IF-12: `uic` routes the `.ui` strings through `tr()`, and every handler string is wrapped in `tr()`.
        *   `[X]` ST-3: unchecking `groupFindLocation` disables the list and its buttons and leaves the list as configured.
        *   `[X]` The list is verified by the dependency map's manual case for step 19.

*   `[X]` [UI] src/gui/`unmatchedtorrentsdialog` — add **Search folder...**, taking a directory and running one `Session::findTorrentLocations()` operation over the listed torrents with it as the pointed root, assigning and removing each torrent that matches, repeatable while entries remain. T20

    *   `[X]` `objective`
        *   `[X]` Problem: a user looking at torrents that matched nothing, and knowing which directory holds them, can only set each location by hand.
        *   `[X]` Functional: a button taking one directory and searching it for every listed torrent whose previous search has reported; each torrent found there is assigned its location and leaves the list, each torrent not found stays.
        *   `[X]` Functional: the button stays available while entries remain, so a library across several disks is covered one directory at a time.
        *   `[X]` Non-functional: the chosen directory is used for that search alone and written nowhere.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/gui`, the dialog the second epic's `unmatchedtorrentsdialog` node adds.
        *   `[X]` Out of scope: enumeration and composition, which are the `sessionimpl` and `filesearcher` nodes.

    *   `[X]` `module`
        *   `[X]` Inside: the button, the directory chooser, the torrents awaiting a search outcome, and the hand-off of matches. Outside: discovery and assignment.

    *   `[X]` `deps`
        *   `[X]` `base/bittorrent/session.h` — `Session::findTorrentLocations()` with its pointed root, from the `sessionimpl` node, and the `torrentLocationFound` signal.

    *   `[X]` `context_slice`
        *   `[X]` The constructor, `m_torrentIDs`, `removeTorrent()` and the directory dialog are those the second epic's `unmatchedtorrentsdialog` node defines. `TransferListWidget` ignores an outcome for an ID outside its own pending set.

    *   `[X]` src/gui/`unmatchedtorrentsdialog.h`
        *   `[X]` Add `#include <QSet>` after `#include <QList>`, and `class Path;` before the `Ui` namespace.
        *   `[X]` Add private `void searchFolder();` and `void handleTorrentLocationFound(const BitTorrent::TorrentID &id, const Path &location, bool found);`, and the member `QSet<BitTorrent::TorrentID> m_pendingSearches;`.

    *   `[X]` src/gui/`unmatchedtorrentsdialog.ui`
        *   `[X]` `labelUnmatched`'s `text` becomes `No existing content was found for these torrents. Search a folder, set a location for each, or close to leave them where they are.`

    *   `[X]` `interaction.spec`
        *   `[X]` Constructor: before the **Set location...** button, `m_ui->buttonBox->addButton(tr("Search folder..."), QDialogButtonBox::ActionRole)` with `clicked` connected to `searchFolder`; and `BitTorrent::Session::torrentLocationFound` connected to `handleTorrentLocationFound`.
        *   `[X]` `searchFolder()`: a heap `QFileDialog` titled `tr("Choose a folder to search")` in the directory mode and options of the existing directory dialog, with `Qt::WA_DeleteOnClose`, starting at no path. On `accepted` with an existing `Path`, each ID of `m_torrentIDs` not in `m_pendingSearches` is inserted into it and appended to a local `QList<BitTorrent::TorrentID> submitted`; after every insertion, a non-empty `submitted` is passed once to `BitTorrent::Session::instance()->findTorrentLocations(submitted, folder)`, so the folder is listed once for the whole list and an outcome emitted during the call finds its ID already pending. Then `open()`.
        *   `[X]` `handleTorrentLocationFound(id, location, found)`: `m_pendingSearches.remove(id)` false → return. `found` true and `m_torrentIDs` holding `id` → `BitTorrent::Session::instance()->assignTorrentLocation(id, location)` then `removeTorrent(id)`; `found` false → the entry stays.

    *   `[X]` src/gui/`unmatchedtorrentsdialog.cpp`
        *   `[X]` Add the constructor statements and define `searchFolder()` and `handleTorrentLocationFound()` per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` The dialog depends downward on `Session` as it already does, and writes nothing to `DiscoveryRoots`.

    *   `[X]` `requirements`
        *   `[X]` IF-10: the list accepts a directory to search, and another while entries remain.
        *   `[X]` ST-8: the pointed root reaches only `findTorrentLocations()`.
        *   `[X]` EN-6: one **Search folder...** action is one operation, verified by the batch-enumeration case at the Epic 3 commit boundary.
        *   `[X]` IF-11: discovery and assignment are reached through `BitTorrent::Session`.
        *   `[X]` IF-12: the button and dialog titles are wrapped in `tr()`.
        *   `[X]` The action is verified by the dependency map's manual case for step 20.

*   `[X]` [UI] src/webui/www/private/`unmatchedtorrents.html` — add **Search folder...**, posting every listed torrent to `torrents/findLocation` with the directory in the path field as `root`, polling until the operation completes, and removing each torrent the answer reports matched, repeatable while entries remain. T25

    *   `[X]` `objective`
        *   `[X]` Problem: a web interface user looking at torrents that matched nothing, and knowing which directory holds them, can only set each location by hand, while the desktop list searches a chosen directory.
        *   `[X]` Functional: a button searching the directory entered in the path field for every listed torrent as one operation; each torrent found there is assigned its location and leaves the list, each torrent not found stays.
        *   `[X]` Functional: the button is available again once its search completes while entries remain, so a library across several disks is covered one directory at a time.
        *   `[X]` Non-functional: the directory is used for that search alone and written nowhere; the button is a native element reachable by Tab, and its label passes through `QBT_TR`.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/webui/www`, the window the second epic's `unmatchedtorrents.html` node adds.
        *   `[X]` Out of scope: enumeration, composition and assignment, which are the `sessionimpl` and `torrentscontroller` nodes.

    *   `[X]` `module`
        *   `[X]` Inside: the button, its request and poll, and removing matched entries. Outside: discovery and assignment.

    *   `[X]` `deps`
        *   `[X]` `torrents/findLocation` with `root`, from this epic's `torrentscontroller` node.

    *   `[X]` `context_slice`
        *   `[X]` The option list `unmatchedTorrents`, the path field `unmatchedLocation`, `error_div`, **Set location...**, the close behaviour and the label `unmatchedLabel` are those the second epic's `unmatchedtorrents.html` node defines. `torrents/findLocation` answers HTTP 202 while any torrent is pending and HTTP 200 with `matched` and `unmatched` once none is, and a request without `hashes` registers nothing.

    *   `[X]` `interaction.spec`
        *   `[X]` **Search folder...**: no option → return. An empty trimmed path → write the empty-path message into `error_div` and return. Otherwise disable the button, clear `error_div`, and post `hashes` holding every option's `value` joined by `|` and `root` holding the path to `api/v2/torrents/findLocation`.
        *   `[X]` Each response: not ok → write its text into `error_div` and enable the button. HTTP 202 → one second later, post to `api/v2/torrents/findLocation` without `hashes` and handle that response the same way. HTTP 200 → remove every option whose `value` the answer's `matched` names by `hash`, enable the button, and close the window when no option remains, or else select the first option and write its save path into the field. An option named by neither list stays.

    *   `[X]` src/webui/www/private/`unmatchedtorrents.html`
        *   `[X]` Before `setLocationButton`, add `<input type="button" value="QBT_TR(Search folder...)QBT_TR[CONTEXT=UnmatchedTorrentsDialog]" id="searchFolderButton">`.
        *   `[X]` `unmatchedLabel`'s text becomes `QBT_TR(No existing content was found for these torrents. Search a folder, set a location for each, or close to leave them where they are.)QBT_TR[CONTEXT=UnmatchedTorrentsDialog]`, the desktop dialog's string.
        *   `[X]` Add the handler per the interaction spec.

    *   `[X]` `directionality`
        *   `[X]` The page consumes the WebAPI alone and writes nothing to the discovery root list.

    *   `[X]` `requirements`
        *   `[X]` IF-10: the web interface list accepts a directory to search, and another while entries remain.
        *   `[X]` IF-20: **Search folder...** posts the listed torrents with the entered directory as `root`.
        *   `[X]` ST-8: the directory reaches only the request.
        *   `[X]` IF-12: the button and label pass through `QBT_TR`.
        *   `[X]` BT-9: `npm run lint` passes, and `npm run format` in `src/webui/www` leaves the file unchanged.
        *   `[X]` Verify the action through the dependency map's manual case for step 25 and Epic 3 integration scenario 10 at the commit boundary.

*   `[X]` [API] src/webui/api/`appcontroller` — expose `find_location_discovery_roots` on `app/preferences` and `app/setPreferences` as an array of objects carrying `path` and `recursive`, returned in the WebAPI's native path form and received through `parseDiscoveryRoots()`. T21

    *   `[X]` `objective`
        *   `[X]` Problem: the discovery root list is reachable only from the desktop dialog.
        *   `[X]` Functional: `app/preferences` returns the list in configured order; `app/setPreferences` replaces it when its key is present, dropping invalid entries, and leaves it untouched when absent.

    *   `[X]` `role`
        *   `[X]` WebAPI controller in `src/webui/api`.
        *   `[X]` Out of scope: the web page and the changelog, which are their own nodes; `API_VERSION`, which the maintainers set.

    *   `[X]` `module`
        *   `[X]` Inside: one key in two actions and the key's outgoing path form. Outside: the receiving JSON rules, which the `discoveryroots` node owns.

    *   `[X]` `deps`
        *   `[X]` `base/discoveryroots.h` — `DiscoveryRoots::instance()`, `roots()`, `setRoots()` and `parseDiscoveryRoots()`.

    *   `[X]` `context_slice`
        *   `[X]` `preferencesAction()` builds `QJsonObject data`, and `appcontroller.cpp` already includes `<QJsonArray>`. `setPreferencesAction()` reads the request through `QJsonDocument::fromJson(...).toVariant().toHash()`, so a JSON array arrives as a `QVariantList`, and tests keys with `hasKey`. The second epic's `find_location_leech_enabled` statements are the anchor.
        *   `[X]` WebAPI path values are written with `Path::toString()`, as `scan_dirs` writes each watched folder; persisted configuration is written with `Path::data()`, as `serializeDiscoveryRoots()` writes `discovery_roots.json`. The persistence serializer is therefore not the WebAPI serializer.

    *   `[X]` src/webui/api/`appcontroller.cpp`
        *   `[X]` Add `#include "base/discoveryroots.h"` after `#include "base/bittorrent/session.h"`.
        *   `[X]` In `preferencesAction()`, after the `find_location_leech_enabled` line, build a local `QJsonArray discoveryRoots` holding, for each `DiscoveryRoot` of `DiscoveryRoots::instance()->roots()` in order, `QJsonObject {{u"path"_s, root.path.toString()}, {u"recursive"_s, root.options.recursive}}`, then `data[u"find_location_discovery_roots"_s] = discoveryRoots;`.
        *   `[X]` In `setPreferencesAction()`, after the `find_location_leech_enabled` branch, add `if (hasKey(u"find_location_discovery_roots"_s))` followed by `DiscoveryRoots::instance()->setRoots(parseDiscoveryRoots(QJsonArray::fromVariantList(it.value().toList())));`. `Path` normalises each received string, so native and generic separators both parse; an unchanged list writes and emits nothing through `setRoots()`.

    *   `[X]` `directionality`
        *   `[X]` `appcontroller` depends downward on `discoveryroots` in `src/base`.

    *   `[X]` `requirements`
        *   `[X]` IF-4: the discovery root list is exposed on both endpoints.
        *   `[X]` CN-5: a request omitting the key leaves the list unchanged.
        *   `[X]` On Windows, `app/preferences` returns each discovery root path in the native form it returns for `save_path` and the `scan_dirs` keys, and `discovery_roots.json` holds the `Path::data()` form, verified at the Epic 3 commit boundary.
        *   `[X]` A root set through the WebAPI appears in the options dialog list through `rootsChanged`, verified by the dependency map's manual case for step 21.

*   `[X]` [UI] src/webui/www/private/views/`preferences.html` — add the `discovery_roots_tab` table and an **Add...** button to the **Find location** fieldset, one row per root holding its path, a recursion checkbox and a **Remove** button, loaded from and saved to `find_location_discovery_roots`. T21

    *   `[X]` `objective`
        *   `[X]` Problem: the discovery root list has no control in the web interface.
        *   `[X]` Functional: a table listing each root's path with its recursion flag in configured order; **Add...** appends an empty row and focuses its path; each row's **Remove** deletes that row and leaves the others in order; saving sends every row with a non-empty path, in row order.
        *   `[X]` Functional: the path inputs, checkboxes, **Remove** buttons and **Add...** are disabled while the legend checkbox is unchecked, and the rows' values are still sent, so disabling the group erases nothing.
        *   `[X]` Non-functional: every control is a native, keyboard-operable element with an accessible name, reached by Tab in row order with **Add...** after the table; the column titles use the translation contexts of the desktop model and dialog.

    *   `[X]` `role`
        *   `[X]` Presentation in `src/webui/www`.
        *   `[X]` Out of scope: validation of entries, which `parseDiscoveryRoots()` performs on receipt.

    *   `[X]` `module`
        *   `[X]` Inside: the table, the **Add...** button, the row builder, the reader, the enablement of the new controls, and the load and save statements. Outside: the list's rules.

    *   `[X]` `deps`
        *   `[X]` `app/preferences`, returning `find_location_discovery_roots` from the `appcontroller` node; `app/setPreferences`, receiving it.

    *   `[X]` `context_slice`
        *   `[X]` `watched_folders_tab` supplies the table's markup form, `<table style="border: 1px solid black;">` with a `thead`. Its script is not followed: `addWatchFolder()` interpolates the folder into markup, adds rows through a clickable image, and addresses rows by positional IDs that a removal would break. The discovery root rows are built with DOM elements and read by walking the `tbody` rows, so they need no `HtmlTable` and no positional ID. The earlier epics' `preferences.html` nodes define the fieldset and `updateFindLocationEnabled`, and the object `exports` returns publishes handlers named in `onclick` attributes.

    *   `[X]` `interaction.spec`
        *   `[X]` `addDiscoveryRoot(path = "", recursive = false)` appends to the `tbody` of `discovery_roots_tab` one `tr` of three `td`s built with `document.createElement`: an `input` with `type` `text` and `aria-label` `"QBT_TR(Discovery root)QBT_TR[CONTEXT=DiscoveryRootsModel]"`, whose `value` property is set to `path`; an `input` with `type` `checkbox` and `aria-label` `"QBT_TR(Recursive mode)QBT_TR[CONTEXT=DiscoveryRootOptionsDialog]"`, whose `checked` property is set to `recursive`; and a `button` with `type` `button` and `textContent` `"QBT_TR(Remove)QBT_TR[CONTEXT=OptionsDialog]"`, whose `click` listener removes its row and then focuses `addDiscoveryRootButton`. Each of the three is disabled unless `findLocationCheckbox` is checked. It returns the text input.
        *   `[X]` `addEmptyDiscoveryRoot()` calls `addDiscoveryRoot()` and focuses the returned input.
        *   `[X]` `getDiscoveryRoots()`: for each `tr` of the `tbody` in order whose text input's trimmed `value` is non-empty, `{ path, recursive }` from that value and the checkbox's `checked`, read whether or not the controls are disabled; returns the array.
        *   `[X]` `updateFindLocationEnabled()` additionally sets `disabled` to the negation of the group state on every element matched by `#discovery_roots_tab input, #discovery_roots_tab button, #addDiscoveryRootButton`.

    *   `[X]` src/webui/www/private/views/`preferences.html`
        *   `[X]` Markup: in the **Find location** fieldset, after the `formRow` holding `findLocationLeechCheckbox`, add `<table id="discovery_roots_tab" style="border: 1px solid black;">` with a `thead` row of `<th scope="col">QBT_TR(Discovery root)QBT_TR[CONTEXT=DiscoveryRootsModel]</th>` and `<th scope="col">QBT_TR(Recursive mode)QBT_TR[CONTEXT=DiscoveryRootOptionsDialog]</th>` and an empty `tbody`, then `<button type="button" id="addDiscoveryRootButton" onclick="qBittorrent.Preferences.addEmptyDiscoveryRoot();">QBT_TR(Add...)QBT_TR[CONTEXT=OptionsDialog]</button>`.
        *   `[X]` Script: after `getWatchedFolders`, define `addDiscoveryRoot`, `addEmptyDiscoveryRoot` and `getDiscoveryRoots` per the interaction spec; extend `updateFindLocationEnabled` per the interaction spec; after `addWatchFolder: addWatchFolder,` in `exports`, add `addEmptyDiscoveryRoot: addEmptyDiscoveryRoot,`.
        *   `[X]` Load: after the `findLocationLeechCheckbox` load and before the `updateFindLocationEnabled();` call, `for (const root of pref.find_location_discovery_roots)` calls `addDiscoveryRoot(root.path, root.recursive);`. No trailing empty row is added.
        *   `[X]` Save: after the `find_location_leech_enabled` statement, `settings["find_location_discovery_roots"] = getDiscoveryRoots();`.

    *   `[X]` `directionality`
        *   `[X]` The page consumes the WebAPI and adds no dependency beyond the key the `appcontroller` node provides.

    *   `[X]` `requirements`
        *   `[X]` IF-5: the discovery root list has a control on the web interface preferences page.
        *   `[X]` IF-12: both column titles, both accessible names and both button texts are wrapped in `QBT_TR`, in the contexts of the desktop strings they repeat.
        *   `[X]` A path is written to its input through the `value` property, never interpolated into markup.
        *   `[X]` ST-3: unchecking the legend checkbox disables every discovery root control and leaves each row's values in the saved settings.
        *   `[X]` Keyboard: Tab reaches each row's path, checkbox and **Remove** in row order and then **Add...**; Space and Enter operate both buttons; **Add...** moves focus to the new path and **Remove** moves it to **Add...**; disabled controls are skipped.
        *   `[X]` Adding, removing a middle row, and saving send the remaining rows in their displayed order.
        *   `[X]` The `CI - WebUI` workflow's `npm run lint` passes, and `npm run format` in `src/webui/www` leaves the file unchanged.
        *   `[X]` The page is verified by the dependency map's manual case for step 21 and the WebUI case at the Epic 3 commit boundary.

*   `[X]` [DOCS] `WebAPI_Changelog` — record `find_location_discovery_roots` on `app/preferences` and `app/setPreferences`, and the `root` parameter of `torrents/findLocation`, under the version heading at the top of the file. T21, T25

    *   `[X]` `objective`
        *   `[X]` Problem: WebAPI clients learn of new preference keys and parameters from this file, and one key joins both preference endpoints while one parameter joins `torrents/findLocation`.
        *   `[X]` Functional: add one entry under the version heading at the top of the file when the branch is rebased, linking this submission's pull request, naming the key and its shape on both endpoints, and naming the parameter. Leave `API_VERSION` and the version headings unchanged; any version decision is the maintainers'.

    *   `[X]` `context_slice`
        *   `[X]` The entries the first and second epics' `WebAPI_Changelog` nodes add are the wording model. Entries under a heading run newest first.

    *   `[X]` `WebAPI_Changelog.md`
        *   `[X]` Under the heading at the top of the file, before its first entry, insert the entry below, separated from its neighbours as the existing entries are.
        *   `[X]` The entry is a top-level bullet linking the pull request, `* [#<number>](https://github.com/qbittorrent/qBittorrent/pull/<number>)`, where `<number>` is the number GitHub assigns when this submission's pull request is opened.
        *   `[X]` Under it, the indented bullet ``* `app/preferences` endpoint includes `find_location_discovery_roots` (array of objects with `path` (string) and `recursive` (bool)) option``.
        *   `[X]` Under it, the indented bullet ``* `app/setPreferences` endpoint allows to set `find_location_discovery_roots` (array of objects with `path` (string) and `recursive` (bool)) option``.
        *   `[X]` Under it, the indented bullet ``* `torrents/findLocation` endpoint accepts optional parameter `root` (string), an existing directory searched ahead of every other root for the torrents that request registers``.

    *   `[X]` `requirements`
        *   `[X]` IF-6: the submission adds one `WebAPI_Changelog.md` entry naming the key and parameter it introduces under the heading at the top of the file and leaves `API_VERSION` unchanged.
        *   `[X]` The file passes the `rumdl` pre-commit hook.

*   `[X]` **Commit** `Search discovery roots and pointed folders`
    *   `[X]` Structural: `DiscoveryRootOptions`, `DiscoveryRoot`, `DiscoveryRoots`, `parseDiscoveryRoots()` and `serializeDiscoveryRoots()`; `SubdirectoryMap` and `enumerateSubdirectories()`; `candidateRoots()` taking optional subdirectory maps holding every directory per name; `Session::findTorrentLocations()` taking torrent IDs and a pointed root; `SessionImpl::SearchOperation` and `SessionImpl::createSearchOperation()`; `DiscoveryRootsModel` and `DiscoveryRootOptionsDialog`; the discovery root list in the options dialog and web interface; **Search folder...** in the desktop and web interface unmatched lists; the `root` parameter of `torrents/findLocation`; two test executables and their fixtures.
    *   `[X]` Behavioural: every operation composes a pointed root, then the discovery roots in configured order, then the watched folder save paths, walking every directory beneath recursive roots and the pointed root once per operation and matching each torrent's name and source name against that listing at any depth, each directory found contributing its parent then itself; automatic additions share an unchanged add operation while it is in flight and otherwise each form their own, and a transfer-list invocation, a `torrents/findLocation` request or a **Search folder...** action is one batch operation; the log names the exact root that produced each found location; the desktop and web interface unmatched lists search a chosen folder for their entries, and a `torrents/findLocation` request searches the folder its `root` names; every mode is offered by the desktop interface, the web interface and the WebAPI alike.
    *   `[X]` Contract: `candidateRoots()` callers supplying no maps behave as before; `findTorrentLocation()` keeps its declaration and outcomes; `app/preferences` and `app/setPreferences` gain `find_location_discovery_roots` and `torrents/findLocation` gains `root`, recorded in `WebAPI_Changelog.md` with `API_VERSION` unchanged; a request without `root` behaves as the second epic defines.
    *   `[X]` **Integration scenario 1 — batch enumeration:** configure one recursive discovery root holding one content folder per torrent, one directly beneath the root and the others beneath one and two determinant folders, and watch directory listings of that tree with `inotifywait -m -r -e open` on Linux or Process Monitor's `QueryDirectory` events on Windows. Run **Find location** over three torrents, post `torrents/findLocation` naming three torrents, then **Search folder...** on that root with three unmatched entries; confirm each operation lists each directory of the discovery root's tree once and each directory of the pointed root's tree once, and each torrent's outcome is still reported and assigned as its own search completes. Add one torrent; confirm that addition lists each directory of the root's tree once.
    *   `[X]` **Integration scenario 2 — overlapping origins:** configure a watched folder save path `W`, a discovery root `W/library` and a discovery root `W/library/sub`, and point **Search folder...** at `W/library`. With content placed where only the pointed root's candidates reach it, only a discovery root's candidates reach it, and only the watched folder's candidates reach it, confirm the log names the pointed root, that exact discovery root and the watched folder save path respectively, and never a nested root that merely contains the winner.
    *   `[X]` **Integration scenario 3 — persistence:** configure at least two roots in a meaningful order with different recursion flags; apply, exit normally, restart, and confirm the exact paths, order and flags in the options dialog and in `app/preferences`. Remove one root and change another's flag, restart again, and confirm the change persisted.
    *   `[X]` **Integration scenario 4 — absent and malformed files:** start with no `discovery_roots.json` and confirm the list is empty and no warning is logged. Then start with malformed JSON, and again with a JSON object at the top level; each time confirm the application remains usable, the list is empty, and the expected warning is logged.
    *   `[X]` **Integration scenario 5 — unchanged writes:** with roots configured and one selected in the options dialog, change and apply an unrelated Downloads setting; confirm `discovery_roots.json`'s modification time is unchanged and the list keeps its selection, so the model was not reset. Send `app/setPreferences` with an unchanged `find_location_discovery_roots` and confirm the same.
    *   `[X]` **Integration scenario 6 — path forms:** on Windows, confirm `app/preferences` returns each discovery root path with the same separators as `save_path` and the `scan_dirs` keys, that sending those values back leaves the list unchanged, and that `discovery_roots.json` holds each path in its `Path::data()` form.
    *   `[X]` **Integration scenario 7 — web interface:** add three roots, remove the middle one, and save; confirm the remaining two are returned in order. Uncheck the legend checkbox; confirm every discovery root control and **Add...** are disabled and skipped by Tab, and that saving keeps the list. Operate **Add...** and **Remove** by keyboard alone and confirm focus moves as specified.
    *   `[X]` **Integration scenario 8 — automatic addition burst:** configure one recursive discovery root holding one content folder per torrent at varying depths, watch its directory listings as in scenario 1, and drop a batch of `.torrent` files into a watched folder while that root is being walked; confirm each directory of the root's tree is listed once for the additions arriving during that walk, and that each of those torrents whose content folder sits in the tree is found through the listing.
    *   `[X]` **Integration scenario 9 — structured volume:** configure a recursive discovery root `V` holding a multi-file torrent's content at `V/someDeterminant/someOtherDeterminant/contentFolder/content`, a single-file torrent's file at `V/Category/Stem/Stem.ext` where `Stem.ext` is the torrent's file and the torrent is added under the Original content layout, and two copies of a third torrent's folder, named `Disc 1`, at `V/A/Disc 1` holding one of its files and `V/B/Disc 1` holding all of them. Add a hidden directory and a symbolic link on Linux, or a junction on Windows, inside `V` pointing at `V`. Add the three torrents in manual mode; confirm the first resolves to `V/someDeterminant/someOtherDeterminant`, the second to `V/Category/Stem`, and the third to `V/B`, each logged as found in the discovery root `V`, each completing its check without transferring content. Confirm with the listing watch from scenario 1 that the walk finishes, lists each real directory once, and neither lists nor enters the hidden directory, the link or the junction.
    *   `[X]` **Integration scenario 10 — pointed root through the WebAPI and web interface:** with three torrents whose content sits only beneath a directory `P` that is neither a discovery root nor a watched folder save path, run **Find location** in the web interface and confirm `unmatchedtorrents.html` lists all three. Enter a directory holding none of them and use **Search folder...**; confirm the button is disabled until the search completes and all three stay. Enter `P` and use **Search folder...**; confirm, with the listing watch from scenario 1, that `P`'s tree is listed once, that each torrent is assigned, logged as found in the pointed root `P` and removed, and that the window closes. Repeat with the content split across `P` and a second directory `Q`, searching `P` then `Q`, and confirm the list shrinks at each step. Post `torrents/findLocation` naming the three torrents with `root` set to `P` and confirm the same assignments; with `root` naming a missing directory, confirm HTTP 409 and no search. Confirm `P` appears in neither `discovery_roots.json` nor `app/preferences`.
    *   `[X]` Commit after the manual cases for steps 19 through 22 and 25 and the ten integration scenarios pass, the Epic 2 integration scenarios still pass, the full build and suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows, the WebUI lint and format checks pass, and the changelog passes `rumdl`.

# To Do

*   `[ ]` Investigate and fix the pre-existing force-recheck failure represented by [qBittorrent issue #14216](https://github.com/qbittorrent/qBittorrent/issues/14216) after the Find Location feature is submitted.
    *   A force recheck can make progress, fail with an I/O error, and leave the torrent unable to perform a later recheck. A retry can remain indefinitely in **Checking** without making progress; restarting qBittorrent permits another attempt, but the same failure can recur.
    *   This behavior is reproducible without Find Location and is not caused by the feature. Find Location exposed it because assignment will use force recheck automatically.
    *   Current code-level hypothesis: `TorrentImpl::forceRecheck()` starts a stopped torrent so it can check and relies on `TorrentImpl::handleTorrentChecked()` to satisfy `StopCondition::FilesChecked` and stop it again. If checking fails before `torrent_checked_alert`, that completion path does not run. A later retry may call libtorrent's `force_recheck()` while the previous error is still set, then update qBittorrent's cached state to checking even though no new check begins. Confirm this hypothesis against the alert sequence before changing code.
    *   Until the underlying defect is fixed, Find Location must handle recheck failure defensively: it must not honor a pending start request, must clear its own pending transaction state, and must leave the torrent stopped and able to retry. It must not treat the cached **Checking** state as evidence that a recheck began or succeeded.
    *   When returning to this issue, reproduce it against the then-current qBittorrent, libtorrent, Qt and operating-system versions; capture the execution-log filename, operation and error; review related issue [#21443](https://github.com/qbittorrent/qBittorrent/issues/21443); and have a human contributor update #14216 or submit the eventual fix in accordance with the repository contribution policy.

*   `[ ]` Fix the pre-existing **Recursive mode** display defect in the watched folder options dialog, as a separate submission independent of the Find Location feature.
    *   The `WatchedFolderOptionsDialog` constructor passes `watchedFolderOptions.addTorrentParams` to `AddTorrentParamsWidget` and never initialises `checkBoxRecursive` from `watchedFolderOptions.recursive`, so the checkbox opens unchecked whatever the stored value.
    *   `watchedFolderOptions()` reads the checkbox on OK, `WatchedFoldersModel::setFolderOptions()` holds it, and `WatchedFoldersModel::apply()` writes it through `TorrentFilesWatcher::setWatchedFolder()`, so a checked box followed by **Apply** persists `recursive: true` to `watched_folders.json`. Reopening that folder's options shows the box unchecked, and accepting the dialog then writes `recursive: false` on the next **Apply**.
    *   This behaviour is reproducible without Find Location. The feature's only edit to the watched folder path is `TorrentFilesWatcher::updateSessionWatchedFolderSavePaths()`, which reads save paths and touches neither the recursion flag, the dialog nor the model.
    *   Fix: in the `WatchedFolderOptionsDialog` constructor, directly after `m_ui->setupUi(this);`, add `m_ui->checkBoxRecursive->setChecked(watchedFolderOptions.recursive);`.
    *   Verification: check **Recursive mode**, accept, **Apply**, reopen the folder's options and confirm the box is checked; accept again, **Apply**, and confirm `watched_folders.json` still holds `"recursive": true`. Uncheck, accept, **Apply**, reopen and confirm the box is unchecked and the file holds `"recursive": false`.
    *   Before submitting, confirm the defect is still present on upstream `master`. The Epic 3 `DiscoveryRootOptionsDialog` copies this dialog's form; its constructor initialises `checkBoxRecursive` from `options.recursive`, and its review confirms that line is present.
