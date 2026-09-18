# Fix Recheck Error — Workplan

Nodes for the fix of [qBittorrent issue #14216](https://github.com/qbittorrent/qBittorrent/issues/14216): a force recheck that fails with a file I/O error leaves the torrent unable to perform a later recheck, and a retry remains in **Checking** without making progress until qBittorrent restarts.

Each node addresses one source file and the support that file requires.

## Force recheck after a check I/O error

*   `[ ]` [BE] src/base/bittorrent/`torrentimpl` — when `TorrentImpl::forceRecheck()` runs on a torrent that holds an error and that qBittorrent holds as not stopped, reapply its auto-managed flag, or resume it in Forced mode, as the missing-files branch does, so the check libtorrent starts for the recheck is not left paused by the earlier failure.

    *   `[ ]` `objective`
        *   `[ ]` Problem: a check that meets a file error other than a missing, short or truncated file makes libtorrent clear the torrent's auto-managed flag, pause it and set its error, and post no `torrent_checked_alert`. qBittorrent's `m_isStopped` stays false, and a torrent that was stopped before the recheck keeps `StopCondition::FilesChecked`. A retried force recheck clears the error in libtorrent, but the torrent remains paused and not auto-managed, so libtorrent never begins hashing, and `TorrentImpl::updateState()` reports it as checking with no progress.
        *   `[ ]` Functional: a force recheck issued after a check failed with a file I/O error hashes the torrent's files and completes, without restarting qBittorrent and without an explicit Start.
        *   `[ ]` Functional: a torrent that was stopped when the failed recheck was issued is stopped again when the retried check completes, through its pending `StopCondition::FilesChecked`.
        *   `[ ]` Functional: a torrent that was running when the failed recheck was issued runs again in its operating mode when the retried check completes.
        *   `[ ]` Non-functional: a force recheck of a torrent holding no error behaves exactly as it does now, whether the torrent is stopped or running.

    *   `[ ]` `role`
        *   `[ ]` Torrent lifecycle in `src/base/bittorrent`.
        *   `[ ]` Out of scope: libtorrent; `SessionImpl::handleFileErrorAlert()` and every other alert handler; the bodies of `TorrentImpl::start()`, `TorrentImpl::stop()` and `TorrentImpl::handleTorrentChecked()`; how an error is reported or displayed.

    *   `[ ]` `module`
        *   `[ ]` Inside: which native calls `forceRecheck()` makes for a torrent qBittorrent holds as not stopped. Outside: how libtorrent checks, pauses and reports, and what a completed check does.

    *   `[ ]` `deps`
        *   `[ ]` `TorrentImpl::hasError()`, `TorrentImpl::isStopped()`, `TorrentImpl::setAutoManaged()`, `m_operatingMode` and `m_nativeHandle` — the same class, used as they stand.
        *   `[ ]` `lt::torrent_handle::force_recheck()` and `lt::torrent_handle::resume()` — libtorrent, already called by `torrentimpl.cpp`.

    *   `[ ]` `context_slice`
        *   `[ ]` `forceRecheck()` returns without metadata; otherwise it calls `m_nativeHandle.force_recheck()`, resets the cached state to `checking_resume_data` with no pieces, then, inside `if (m_hasMissingFiles)`, clears `m_hasMissingFiles` and, for a torrent not stopped, calls `setAutoManaged(m_operatingMode == TorrentOperatingMode::AutoManaged)` and, in Forced mode, `m_nativeHandle.resume()`; finally, a stopped torrent is started with `start()` and given `StopCondition::FilesChecked`.
        *   `[ ]` `hasError()` reads the cached `m_nativeStatus`: a set `errc` or the `upload_mode` flag. `forceRecheck()` does not update either before its branches run.
        *   `[ ]` `start()` clears a native error and upload mode, clears `m_isStopped`, and reapplies the auto-managed flag or resumes in Forced mode. `handleTorrentChecked()` calls `stop()` when `stopCondition()` is `StopCondition::FilesChecked`, and otherwise resumes the torrent in its operating mode. `stop()` logs `Torrent stopped. Torrent: "%1"` through `SessionImpl::handleTorrentStopped()`; `start()` logs `Torrent resumed. Torrent: "%1"` through `SessionImpl::handleTorrentStarted()`.
        *   `[ ]` `SessionImpl::handleFileErrorAlert()` records the error through `TorrentImpl::handleFileError()`, logs `File error alert. Torrent: "%1". File: "%2". Reason: "%3"` at WARNING at most once per torrent within one second, and emits `torrentIOError`. It changes no stopped state.
        *   `[ ]` `updateState()` reports `CheckingUploading` or `CheckingDownloading` while the native state is `checking_files` and the torrent is not stopped, below the missing-files and error states.
        *   `[ ]` libtorrent 2.0: `torrent::on_piece_hashed()`, on an error other than `no_such_file_or_directory`, end of file or `file_too_short`, resets its checking counters, posts `file_error_alert`, calls `auto_managed(false)`, `pause()` and `set_error()`, and returns without completing the check. `torrent::force_recheck()` returns when `should_check_files()` holds or the state is `checking_resume_data`; otherwise it calls `clear_error()`, sets `checking_resume_data` and queues the file check, whose completion `torrent::on_force_recheck()` sets `checking_files`, pauses gracefully when auto-managed, and calls `start_checking()` only when `should_check_files()` holds, which requires the torrent to be unpaused and without error. `session_impl::auto_manage_checking_torrents()` resumes and starts checking each auto-managed torrent in `checking_files` within the active checking limit. A call through `m_nativeHandle` is queued to the libtorrent network thread in the order it is made.

    *   `[ ]` `interaction.spec`
        *   `[ ]` `forceRecheck()` without metadata → return; nothing changes.
        *   `[ ]` Otherwise `const bool hadError = hasError();` is read first, then `m_nativeHandle.force_recheck();` and the cached state reset run unchanged.
        *   `[ ]` `m_hasMissingFiles` or `hadError` holding → `m_hasMissingFiles = false;`; then, when `isStopped()` is false, `setAutoManaged(m_operatingMode == TorrentOperatingMode::AutoManaged)` and, when `m_operatingMode` is `TorrentOperatingMode::Forced`, `m_nativeHandle.resume()`. The auto-managed flag is queued after `force_recheck()`, so libtorrent's auto-manager resumes the torrent and starts its check once `on_force_recheck()` sets `checking_files`; the forced resume unpauses it before `on_force_recheck()`, which then starts the check itself.
        *   `[ ]` Neither holding → no native call beyond `force_recheck()` for a torrent not stopped; libtorrent checks it under its current flags, as it does now.
        *   `[ ]` `isStopped()` true → `start();` then `m_stopCondition = StopCondition::FilesChecked;`, unchanged. `start()` clears the error and reapplies the flags itself.
        *   `[ ]` A retried check that completes posts `torrent_checked_alert`, and `handleTorrentChecked()` stops a torrent whose `StopCondition::FilesChecked` survived the failed check, or resumes a torrent without it.

    *   `[ ]` FindLocationLab/`labdefs.py`
        *   `[ ]` In `CATALOG`, after the last Epic 3 entry, add `_t("s30-recheck-error", "large")`, a large-shape torrent whose check takes long enough to make progress before reaching its second file.
        *   `[ ]` In `PRESETS`, after `"s29"`, add `"s30": [("s30-recheck-error", "save", False)]`, placing complete content at `saves\s30\s30-recheck-error`.
        *   `[ ]` Run `python build_lab.py` without `--reset-profiles`, which writes `torrents\s30-recheck-error.torrent`, its magnet, its `lab.json` entry and `saves\s30`, keeping both profiles and the existing payloads.

    *   `[ ]` FindLocationLab/`integration.py`
        *   `[ ]` Add `def s30(sc: Scenario):` after `s29`, with the docstring `"""Force recheck after a check I/O error (qBittorrent issue #14216)."""`, built from the module's existing helpers: `prepare()`, `clear_main()`, `main.add_checked()`, `main.recheck()`, `main.start_torrent()`, `main.stop_torrent()`, `restart_main()`, `lock()`, `Sampler`, `settle()`, `logs_matching()`, `main.mark_log()`, `main.log_messages()`, `reset_content()` and the state sets `STOPPED`, `CHECKING`, `STARTED_UP`.
        *   `[ ]` Setup: `prepare("s30")`; `main.add_checked("s30-recheck-error", lab.SAVES / "s30")`, timing that initial check as `baseline`; the stall window is `max(30, 5 * baseline)` seconds. The file locked is `lab.SAVES / "s30" / "s30-recheck-error" / "part2.bin"`.
        *   `[ ]` Stopped torrent, failed recheck: with the torrent stopped, `main.mark_log()`, take `lock(part2, 600)`, then, inside a `Sampler`, `main.recheck(tid)` and wait for the log line `File error alert` naming `"s30-recheck-error"` and `part2.bin`; hold ten seconds more, then release the lock by killing the locker. Checks: `Torrent resumed` is logged for the torrent; the file error alert is logged once, naming `part2.bin`; the highest progress sampled while the state is in `CHECKING` is above 0 and below 1, so the check hashed `part1.bin` before failing; the state after the alert is `error`; no `Torrent stopped` is logged for the torrent after the alert.
        *   `[ ]` Stopped torrent, retry: `main.mark_log()`, then, inside a `Sampler`, `main.recheck(tid)` and wait up to the stall window for the state to leave `CHECKING`. The headline check, `retry after the check I/O error completes and the torrent ends stopped`, requires a state in `STOPPED`, progress 1 and one `Torrent stopped` log for the torrent. An evidence check records the distinct states sampled, the highest progress sampled and whether a new file error alert was logged during the retry.
        *   `[ ]` Stopped torrent, recovery by Start: when the retry did not complete, `main.start_torrent(tid)` and wait up to the stall window for a state outside `CHECKING`. Checks: the check completes with progress 1; the torrent ends in `STOPPED` with one `Torrent stopped` log, which proves `StopCondition::FilesChecked` survived the failed check. When the retry completed, record it with `sc.not_run("recovery by Start", "the retry completed")`.
        *   `[ ]` Running torrent, failed recheck and retry: `clear_main()`; `main.add_checked()` again; `main.start_torrent(tid)` and `settle()` in `STARTED_UP`; then the same lock, recheck, file error alert and release as the stopped torrent. Checks: the file error alert is logged; the state after it is `error`. Retry as above; the headline check, `retry after the check I/O error completes and the torrent runs again`, requires a state in `STARTED_UP` and progress 1 within the stall window, with the same evidence check.
        *   `[ ]` Running torrent, recovery by restart: when the retry did not complete, `restart_main()` and wait up to the stall window for a state in `STARTED_UP` with progress 1. When the retry completed, record it with `sc.not_run("recovery by restart", "the retry completed")`.
        *   `[ ]` Recheck without an error: `clear_main()`; `main.add_checked()`; `main.recheck(tid)` on the stopped torrent → it ends in `STOPPED` with progress 1 and one `Torrent stopped` log; `main.start_torrent(tid)`, `settle()` in `STARTED_UP`, `main.recheck(tid)` → it ends in `STARTED_UP` with progress 1 and no `Torrent stopped` log.
        *   `[ ]` Cleanup: `clear_main()`, `base_preferences()`, `reset_content("s30")`.

    *   `[ ]` FindLocationLab/`README.md`
        *   `[ ]` After the Epic 3 scenario table, add a `## Recheck error scenario` section holding one table row in the form of the scenario tables: `30`, preset `s30`, and the setup "Add `s30-recheck-error` with save path `saves\s30` and check it once. Lock `saves\s30\s30-recheck-error\part2.bin`, force a recheck, and release the lock after the file error alert; force a recheck again. Repeat with the torrent started. `integration.py s30` drives it against qBittorrent issue #14216."

    *   `[ ]` RED proof
        *   `[ ]` Build the `qbittorrent` target into `build`, the binary `qbt.py` and `run.ps1` run, from this branch with `torrentimpl.cpp` unchanged, and run `python integration.py s30` in FindLocationLab.
        *   `[ ]` The run is RED when both headline checks fail and every other check of the failed rechecks, both recoveries and the recheck without an error passes, with each evidence check recording a retry held in `CHECKING` at progress 0 for the whole stall window and no new file error alert.
        *   `[ ]` Any other outcome is a discovery: report the `results.jsonl` record and halt before `torrentimpl.cpp` is edited.

    *   `[ ]` src/base/bittorrent/`torrentimpl.cpp`
        *   `[ ]` In `TorrentImpl::forceRecheck()`, add `const bool hadError = hasError();` after the `hasMetadata()` return and before `m_nativeHandle.force_recheck();`, and change `if (m_hasMissingFiles)` to `if (m_hasMissingFiles || hadError)`, per the interaction spec. Every other statement of the function is unchanged.

    *   `[ ]` GREEN proof
        *   `[ ]` Rebuild the `qbittorrent` target into `build` and run `python integration.py s30` unchanged. Every check passes, and both recoveries are recorded `NOT RUN` because each retry completed.

    *   `[ ]` `directionality`
        *   `[ ]` `torrentimpl` gains no dependency; the change calls only members of its own class and the native handle it already holds.

    *   `[ ]` `requirements`
        *   `[ ]` A retried force recheck after a check I/O error completes without a restart or an explicit Start: the two headline checks of `s30`.
        *   `[ ]` A torrent stopped before the failed recheck ends stopped through `StopCondition::FilesChecked`: the stopped torrent's headline check, requiring one `Torrent stopped` log.
        *   `[ ]` A torrent running before the failed recheck runs again: the running torrent's headline check.
        *   `[ ]` A recheck without an error is unchanged for stopped and running torrents: the recheck-without-an-error checks of `s30`.
        *   `[ ]` `TorrentImpl` requires a live session and cannot be constructed in a `qbt_base` test, so `s30` in FindLocationLab is this node's source test.

    *   `[ ]` **Commit** `Resume torrent when rechecking after a file error`
        *   `[ ]` Structural: `TorrentImpl::forceRecheck()` reads `hasError()` before issuing the native recheck and extends its missing-files condition to it.
        *   `[ ]` Behavioural: a force recheck of a torrent that qBittorrent holds as not stopped and that holds an error reapplies its auto-managed flag, or resumes it in Forced mode, so a check that failed with a file I/O error can be retried and completes, ending stopped or running as the torrent was before the failed recheck.
        *   `[ ]` Contract: `forceRecheck()` keeps its declaration; a recheck of a torrent holding no error, and every recheck of a stopped torrent, is unchanged; no WebAPI key, setting or resume datum changes.
        *   `[ ]` The body carries `Closes #14216.` Commit after `integration.py s30` passes against this node's build and the full build and suite pass under `-DTESTING=ON` on Ubuntu, macOS and Windows.
