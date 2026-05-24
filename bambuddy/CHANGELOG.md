## 0.2.4.3

**Bambuddy v0.2.4.3**

**⚠ Upgrade Notes — Read Before Updating**

Almost everyone is upgrading from 0.2.4.2. 0.2.4.3 is a patch release on the same 0.2.4 code base — no schema breaks, no Docker entrypoint changes, no Vite/proxy quirks. The in-app Apply Update button in Settings → System → Updates works for Docker and for any native install already on 0.2.4.1 or later.

Make a backup before upgrading via Settings → Backup → Create Backup. Native install with update.sh snapshots the database automatically and rolls back on failure. Docker and fully-manual paths don't.

### Docker

docker compose pull
docker compose up -d

docker-compose.yml doesn't need refreshing — none of the entrypoint, volume, or env-var conventions changed since 0.2.4.

### Native install — recommended path

sudo BRANCH=main /opt/bambuddy/install/update.sh

Snapshots the database first and rolls back on failure.

### Native install — manual path

sudo systemctl stop bambuddy
cd /opt/bambuddy
sudo -u bambuddy git fetch --prune --tags --force origin
sudo -u bambuddy git checkout main
sudo -u bambuddy git reset --hard origin/main
sudo /opt/bambuddy/venv/bin/pip install -r requirements.txt
sudo systemctl start bambuddy

**Behaviour changes to know about**

- "Prefer Lowest Remaining Filament" now reads from your inventory, not the printer's RFID counter. Slots bound to a Bambuddy or Spoolman spool sort by your remaining-grams figure first; MQTT-only slots fall back to the previous logic. If you use the preference and have bound spools, the dispatch will start picking a different slot than before — the one that actually has less filament left in your inventory. (#1508)

- Camera live view: ffmpeg's RTSP socket-timeout flag is now probed at runtime. Native-Ubuntu installs with a transitional-era ffmpeg were hitting Address already in use because that ffmpeg repurposed -timeout into a listen-mode option. No user action — but if you'd manually patched your install to drop the flag, you can revert. (#1504)

- Cloudflare-fronted installs: CSP script-src is now nonce-based. If you'd added 'unsafe-inline' to your reverse-proxy CSP as a workaround for Cloudflare's auto-injected Rocket Loader / email-obfuscation script, remove it — it's no longer needed. (#1460 follow-up)

- The 0.2.4 feature-freeze still holds for 0.2.5-bound work. A handful of feat: items shipped in this release because they were already in flight before the freeze and would have stranded the original reporters until the 0.2.5 cycle (Spanish locale #1243, the slicer-filter series #1325, cross-printer re-slicing). Everything else from now on is fix-only on the 0.2.4 train.

---

**Highlights**

0.2.4.3 closes two long-running issue clusters and lands the self-service triage infrastructure the support queue has been asking for. The #1325 slicer-filter series is finally complete — Process / Filament dropdowns in the Slice dialog now filter by your selected printer and nozzle diameter, with the right answer whether you've uploaded a Slicer Bundle or are relying on the @BBL preset-name fallback. The #1395 camera reliability series picks up P2S stream stability, ffmpeg stderr capture on   stalls, and the cross-ffmpeg-version RTSP timeout flag that was breaking live view on native Ubuntu (#1504). Around them: a new Connection Diagnostic for "printer won't connect" triage, a Log Health Scanner that surfaces self-fixable issues before they hit the support queue, and a Virtual Printer Setup Diagnostic with one-click slicer-cert export — all three are now auto-attached (sanitized) to support bundles and bug reports.

Plus the largest fix tally of any 0.2.4.x patch — re-slicing correctness across nozzle classes and multi-plate sources (#1493 + the slice-all toggle), "Prefer Lowest Filament" now reads from your inventory instead of the printer's RFID counter (#1508), timelapse re-attaches after a backend restart mid-print (#1485 follow-up), and a full Spanish locale (#1243). Security: idna >= 3.15 for CVE-2026-45409 (ReDoS) and starlette >= 1.0.1 for PYSEC-2026-161 (malformed Host header).

  ---

**New Features**

- Connection Diagnostic — self-service triage for "printer won't connect / won't print" — Triage review of recently-closed issues found roughly a third were user-side setup errors (wrong IP after DHCP renew, LAN-mode toggle off, access code mismatch, port-block by a router-level guest network). The new per-printer diagnostic runs the full layer-1-through-MQTT check in one click — TCP reachability, MQTT TLS handshake, status-message receipt, IP sanity, SSDP visibility — and renders a ranked layer-8 root-cause list with a "Copy to clipboard for bug report" affordance.

- Log Health Scanner + Add/Edit-Printer setup pre-flight — Passive scanner that walks your last N hours of logs against a curated catalog of known-issue signatures (#1480 FTP loop, #1395 ffmpeg drops, #1462 drying-power-off, …) and surfaces matches on the System page. Add/Edit-Printer also runs a pre-save pre-flight that warns before you commit a misconfiguration (e.g. trailing whitespace in the access code, swapped IP/serial).

- Virtual Printer Setup Diagnostic + one-click slicer-certificate export — VP card now exposes a per-VP "Diagnose" button that walks the slicer-side misconfig list (cert not installed in BambuStudio's trust root, FTPS port collision with another service on the bind IP, MQTT-mode mismatch with the target printer). The matching "Export slicer certificate" button writes the right .crt for whichever VP you're testing, eliminating the manual openssl invocation that ate half the VP-related support    tickets.

- Slicer: process & filament profiles now filter by the selected printer (#1325, requested by @IndividualGhost1905) — Picking an X1C in the server-side Slice dialog no longer floods the Process / Filament dropdowns with H2D-only presets. When you've uploaded a Slicer Bundle for that printer, filtering is exact-match against the bundle's contents; without a bundle, falls back to a name-based @BBL heuristic on model + nozzle.

- Slicer: cross-printer re-slicing across nozzle classes + multi-plate slice-all toggle — Re-slicing an X1C archive for an H2D, or re-slicing a parted-statue multi-plate 3MF in one click. The slice-all toggle produces a single output 3MF whose Metadata/plate_N.gcode entries cover every plate, with the filament dropdowns showing the union of slot usage across plates so paint slots from plate 2 aren't invisible.

- Spanish (es) locale (#1243, requested by @MiguelAngelLV) — Full European Spanish translation across all 4899 keys. Joins the existing en / de / fr / it / ja / pt-BR / zh-CN / zh-TW locales; parity check enforces no English-leak.

- Currency: Belize Dollars (BZD) (#1454, requested by @PLGuerraDesigns) — Added to the Settings → Cost currency dropdown.

- Event-loop stall watchdog — makes a frozen backend self-diagnose (#1486 groundwork) — Several "container hangs after adding a printer" reports share a signature that leaves no actionable evidence in the logs. The watchdog logs a full traceback when the asyncio event loop is blocked for longer than the threshold, so the next time it hangs we get a real stacktrace instead of "it just stopped responding".

---

**Changes**

- Support bundle + bug-report submission now include the live diagnostic snapshot. Connection Diagnostic / Virtual Printer Setup Diagnostic / Log Health Scanner results are now persisted into the downloadable support ZIP and the submitted GitHub issue, with IPs / printer names / serials / access codes sanitized via the existing log-sanitizer plus an IPv4 regex fallback. A 4-step progress checklist replaces the spinner so the longer wait is communicated honestly.

- Settings → SpoolBuddy: CPU load tile on the device card. Pulled from the existing system_stats.load_avg heartbeat — no new round-trip.

- Filament inventory: grouped rows now show group totals (#1368) — Collapsed group row now shows the rolled-up Remaining / Total values for the group instead of the values from a single member spool.

- Bug-report panel: connection diagnostic collapses for multi-printer setups so the panel doesn't overflow when you have eight printers.

- Color Catalog sync now identifies itself as Bambuddy to filamentcolors.xyz. Honest User-Agent — Bambuddy/<version> instead of the bare python-httpx/x.y.z default. Same compliance-audit pass that produced today's Bambu Lab outreach.

---

**Fixed**

Slicer / slicing

- Slicer process / filament dropdowns now filter by nozzle diameter too (#1325 follow-up #2, reported by @IndividualGhost1905) — With the @BBL name fallback in place, an X2D 0.4 selection was still admitting 0.2 / 0.6 / 0.8 nozzle process variants. The regex now parses both model AND nozzle from preset names; differing nozzles fall into the "Other printers" group.

- Slicer process / filament dropdowns now filter for users without uploaded Slicer Bundles (#1325 follow-up, reported by @IndividualGhost1905) — Original #1325 fix relied on the user having uploaded a .bbscfg; reporters running without one saw no filtering at all. The name-based @BBL fallback now produces sensible filtering even when no bundle is present.

- Slicer: process / filament dropdowns now filter by printer using uploaded Slicer Bundles (#1325, reported by @IndividualGhost1905) — Root fix for #1325 — replaces the preset-name guessing heuristic with bundle-backed exact match.

- Re-slicing across the single-nozzle / dual-nozzle boundary now actually works (#1493) — Re-slicing a model sliced for a single-nozzle printer (X1C, P1S, A1, P2S, …) onto a dual-nozzle printer (H2D, X2D) — or vice versa — now produces a correct output 3MF. Per-plate loop handles the bed-bound consolidation that BS CLI's --arrange would otherwise reject.

- Failed slice now opens an error modal instead of a toast that vanishes before it can be read.

- Re-slicing for a different printer no longer silently produces a file for the original printer — Cross-model re-slice was carrying the source's printer_id through, so the resulting archive looked like it belonged to the wrong printer in the UI.

- Re-sliced archive now shows the printer it was sliced for, not the source's printer.

- Sliced files no longer report "0 g" filament usage — Archive card and slice-result both showed filament_used_g: 0 for real multi-hour prints.

- Flow Calibration now actually runs when the print option is enabled (#1478, reported by @andreirusu99) — H2S reporter saw poor extrusion around corners; root cause was the Flow Calibration option being silently skipped because the wrong project_file field was being passed to the printer.

- File Manager: list-view actions no longer clipped; preview modal Slice button now respects the Slicer API setting.

Camera

- X1/H2/P2 live camera no longer fails with Address already in use on transitional ffmpeg builds (#1504, reported by @rage03usa, confirmed by @PawseHaxor) — Native-Ubuntu reporter was retrying indefinitely with Unable to open RTSP for listening … Address already in use because that ffmpeg version repurposed -timeout into a listen-mode option. Runtime probe now picks -stimeout or -timeout based on what ffmpeg advertises, covering the full pre-deprecation / transitional / modern range without regressing either end.

- Camera: P2S RTSP stream no longer drops every frame after the first (#1395, reported by @Tschipel)

- Camera: ffmpeg stderr is now captured when an RTSP stream stalls (#1395, reported by @Tschipel) — Previously stderr was only captured when ffmpeg actually crashed, so silent stalls left no diagnostic trace.

- Camera diagnostic (stethoscope) is now visible in the pop-out camera window (#1395, reported by @Tschipel)

Inventory / scheduler

- "Prefer Lowest Remaining Filament" now uses Bambuddy's inventory weight, not just the printer's RFID counter (#1508, reported by @kleinwareio) — Reporter had a cloned spool in slot 1 (950 g) and the original in slot 4 (50 g), the preference enabled, and the dispatch consistently picked slot 1. Root cause: scheduler was reading the printer's tray.remain (which is -1 for non-RFID spools and ties every slot at the sentinel). Now bound-spool slots sort by Bambuddy / Spoolman remaining-grams first; MQTT-only slots fall through to the previous logic.

- Scheduler: queue items with force_color_match filament overrides now produce a correct AMS mapping at dispatch (#1437, fixed by external PR #1440 from @Person2099) — When a queue item had force_color_match overrides but no specific plate (plate_id=None), the scheduler was dispatching with ams_mapping: null and the printer was picking AMS slots by filament type only, ignoring the required colour entirely. Contributor fix walks <plate> elements first in filament_requirements.py and falls back to legacy _collect_filaments only when the modern format is absent.

- Archive filament colour now reflects the assigned inventory spool, not the slicer's 3MF (#1494, reported by @IndividualGhost1905) — Adding a #000000 black filament was being overridden by the 3MF's embedded colour.

- Insufficient-filament pre-print warning now fires on every dispatch path (#1496, reported by @needo37) — Pre-print check was only firing for one of the three dispatch entry points.

Spoolman

- OpenSpoolman-tagged spools are now selectable in the AMS-slot assignment picker (#1122, reported by @mithkr) — Reporter runs Bambuddy alongside OpenSpoolman; OpenSpoolman writes a different tag shape into spool.extra, which the assignability check was rejecting. Now decided from the slot-assignment ledger instead of the tag field.

- Missing-spool-assignment notification no longer false-fires on every Spoolman-mode print (#1473, reported and root-caused by @ojimpo) — Check was only querying one of the two assignment tables.

- Per-print weight reporting now works for tag-less spools assigned via the Bambuddy UI (#1459, reported by @Moskito99 — follow-up to #1119) — Postgres + Spoolman reporter saw weight updates skip every non-tag spool assigned through the UI rather than via NFC scan.

- AMS hover card and SpoolBuddy fill-bar no longer surface a stale spool after re-assigning a non-RFID slot (#1457, reported by @Menthe11) — P1S reporter with generic non-RFID spools saw the old assignment hang around in both UI surfaces after switching the slot to a different spool.

Archives / timelapse

- Timelapse now attaches to the archive after a backend restart mid-print (#1485 follow-up, reported by @pwostran) — Companion fix to the duplicate-archive guard from #1485. The same first-push guard that prevents the duplicate also prevented the timelapse-baseline capture, so the post-reboot scan couldn't find new files. Baseline is now captured on the first observed RUNNING state when the start callback is suppressed.

- A backend restart mid-print no longer duplicates the job in the archive (#1485, reported by @pwostran)

- Timelapse auto-attach works for VP-queue / dispatch prints (#1403 follow-up, reported by @pwostran) — The snapshot-diff strategy that picks the right MP4 was incorrectly attributing files to the wrong archive for dispatched prints.

File Manager / Library

- "All Files" view now shows files inside subfolders (#1499) — Was previously returning only top-level files.

- Library files now display the filename, not the embedded 3MF Title (#1489, reported by @needo37) — File Manager cards / search / sort were keying off file_metadata.print_name, which mismatched what users actually search by.

- 3D preview no longer freezes the page on complex multi-part 3MFs (#1412, reported by @anthonyma94) — Opening the 3D preview on a multi-colour parted MakerWorld statue hung the browser tab.

- STL thumbnail generation failures now log a full traceback — Surfaced by the #1480 support bundle.

Printers / connectivity

- File Manager no longer polls the printer over FTPS every 30 seconds while open (#1480, reported by @OscarsWorldTech) — P1S reporter's printer was churning through MQTT disconnect/reconnect cycles every 30s because the File Manager was opening idle FTPS sessions to the printer for no functional reason.

- Printer serial numbers normalized on input; stale connection that never receives a status report now logs an actionable hint (#1465, reported by @jmneely94)

- Smart-plug "Auto Off after Drying" no longer kills the printer seconds into a drying cycle (#1462, reported by @Kyobinoyo) — X2D reporter set a 1-hour dry cycle and the smart plug cut power 8 seconds in. False "drying complete" detection on power-up state.

- AMS drying popover now scrolls instead of clipping the Start button on short viewports (#1458, reported by @kleinweby)

- Add Printer no longer hangs the container on P1S (#1445, reported by @psybernoid, confirmed by @thomassjogren) — Regression introduced in 0.2.4.2 by the fix(printers) cleanup.

- FTP: P2S upload truncates / 426 "Failure reading network stream" on Python 3.13 (#1401, reported and root-caused by @iitazz) — P2S firmware 01.02.00.00 reporter on Python 3.13 saw uploads truncate at exactly 1.8 MB.

SpoolBuddy

- Spool ID surfaced everywhere a spool's identity is rendered + Write-Tag page honours Spoolman mode (#1439, reported + partially prototyped by @flom89)

Stats

- Print Activity heatmap buckets prints by local date, not UTC date (#1446, reported and root-caused by @needo37) — CDT (UTC-5) reporter noticed prints finished in the evening appeared on the wrong calendar day.

- Failure Analysis widget no longer shows "Unknown" for archives classified after the fact (#1444, reported and root-caused by @needo37)

UI / PWA

- PWA: in-app install button + self-host the Inter font (#1460, reported by @Soopahfly) — Reporter could install Bambuddy as a PWA on desktop but not on Android, and the font was being fetched from rsms.me on every page load.

- Self-hosted Inter font now actually loads — /fonts/*.woff2 was not served (#1460 follow-up) — The browser console logged downloadable font: rejected by sanitizer because the static mount didn't include the fonts subdirectory.

- Cloudflare-fronted Bambuddy no longer needs an unsafe-inline override to load (#1460 follow-up, reported by @Soopahfly) — CSP script-src switched to nonce-based so Cloudflare's auto-injected email-obfuscation / Rocket Loader script passes without the override.

- Local Profiles: search bar no longer disappears when a query matches nothing (#1470, reported by @pwostran)

- Failure Detection: Status panel Low / High thresholds now reflect the selected sensitivity (#1469, reported by @JohnMacOB)

Security

- idna >= 3.15 — clears CVE-2026-45409 (ReDoS in idna.encode() with crafted Unicode payloads).

- starlette >= 1.0.1 — clears PYSEC-2026-161 (malformed Host header crash in request.url construction).

- PyJWT CVE-2025-45768 permanently ignored in pip-audit — Advisory is disputed by the PyJWT maintainers; Bambuddy auto-generates 86-char secrets via secrets.token_urlsafe(64), file-loaded path rejects secrets shorter than 32 chars.

---

**Thanks**
  
**@Person2099** ([PR #1440](https://github.com/maziggy/bambuddy/pull/1440) — `force_color_match` AMS-mapping fix). Thanks for the upstream patch.

