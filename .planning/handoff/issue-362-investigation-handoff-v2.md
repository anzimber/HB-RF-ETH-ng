# AI Investigation Handoff v2 — Issue #362 "Stop working" (network wedge / IWDT resets)

**Date:** 2026-09-06 (supersedes `issue-362-investigation-handoff.md`; that file and `issue-362-ai-response.md` — the first external AI's answer — remain in this folder as history)
**Status:** OPEN. Amplification topology of the failure is now mapped and code-verified. The **initiator** — what first stops the lwIP `tcpip_thread` / EMAC input path while everything else keeps running — is still unidentified. That is your mission.
**How to use this document:** it is self-contained. Read it fully before touching code. Section 5 lists eliminated paths — do not reopen them without new evidence.

---

## 1. Mission for the receiving AI

Identify the **initiator** of the network-stack death in this ESP32 (WROOM-32, no PSRAM, LAN8720 RMII, RPI-RF-MOD on UART1) firmware: the exact code path or mechanism that first stops `tcpip_thread` progress (or the EMAC→lwIP input chain) under field conditions, producing a device that is dead to ICMP while the scheduler and non-network tasks keep running. Secondary: explain what on ESP-IDF 6.1 turns that state into a ≥300 ms tick starvation (Interrupt-Watchdog reset) while 5.5.3 merely hangs.

---

## 2. Failure signature (ground truth from field logs, issue #362)

- Device dies spontaneously. WebUI gone, CCU loses the radio.
- IDF 6.x builds: reset reason **Interrupt Watchdog** (ESP_RST_INT_WDT), auto-reboot, **no preceding log line** (syslog verified running), heap healthy at every crash (crash blackbox: 82–104 KB free, low_streak=0, no stuck NVS/net op).
- IDF 5.5.3 build of the same current code (experiment `2.2.7-Beta.5-IDF55EXP`): device **hangs completely** — not even ICMP ping answers, no watchdog fires, power-cycle required, LED state frozen (one report: solid blue).
- Uptimes to failure: 66 s … 32 h; many around 1–1.5 h; clustering during/right after CCU reconnect bursts.
- Load correlation: real homes with continuous RF traffic crash; maintainer's bench unit (same firmware, **with** RPI-RF-MOD attached and live CCU session, but few/no real RF devices) ran 12+ days stable.
- One crash coincided with **AdGuard** being integrated into the tester's LAN (single observation).
- Historical: users report **v2.1.10 as stable on the same hardware** (same IDF 5.5.3 generation, OLD application code) — while the 55EXP build (same IDF, CURRENT app code) hangs. ⇒ the root lives in the 2.2.x application rewrite's interaction with the network stack, not in the IDF generation alone.

---

## 3. Everything done so far (chronological, all on `main` unless noted)

1. ESP-IDF pinned v6.1-rc1 → v6.1 final (housekeeping).
2. **ECO3 cache-lock livelock workaround enabled** (`scripts/patch_idf_eco3_fix.sh` + step in all 5 firmware workflows; removes the `SPIRAM` gate from `CONFIG_ESP32_ECO3_CACHE_LOCK_FIX`). No-op on silicon ≤ v2. Kept — falsified as the fix for #362 (rev100 and rev301 both crash with it active) but correct for rev3 silicon regardless.
3. **Tick sentinel** in the crash blackbox (`main/crash_blackbox.cpp`): IRAM per-CPU tick hook → RTC-noinit `last_tick_ms[]`; next boot latches into `prev_tick_ms[]` before re-arming; prints `Tick sentinel (pre-reset): cpu0 … cpu1 …` for watchdog-class resets. Beta.4 shipped a broken variant (read live slots after re-arm); Beta.5 fixed the latch. **No field crash has yet produced a valid pre-reset sentinel line** (the one Beta.4-era report predated the fix; a later hang was power-cycled, wiping RTC).
4. **UART1 TX ring buffer 2 KiB** + `uart_wait_tx_done()` before module reset (`main/radiomoduleconnector.cpp`). One unexplained one-off: a single boot missed radio-module detection once (transient, relay still worked).
5. **RawUart worker priority restored 12 → 15** (v2.1.10 value).
6. **Releases:** v2.2.7-Beta.4, v2.2.7-Beta.5 (IDF 6.1); experiment `experiment-idf55exp-b5` on branch `experiment/idf-v5.5.3` (IDF v5.5.3, classic FreeRTOS kernel verified, **live-log WebSocket compiled out** — IDF 5.x lacks `ws_post_handshake_cb`; WebUI falls back to `/api/log` polling).
7. **Network wedge self-healing + event blackbox snapshots** (commit `ec542c9`, on main, **not yet released** — intended for Beta.6): heap_watchdog pings the default gateway once per 60 s cycle; link-up + network-was-alive + 5 consecutive fails → clean restart with `net watchdog:` diag + crash tail + blackbox snapshot. Event-driven `crash_blackbox_snapshot_now()` also on CCU keepalive timeout and Ethernet link-down. Live-validated 12 min on the bench device (no false positives; correct non-reaction to a CCU-initiated disconnect).
8. Bench experiments: 67 Mbit/s unicast UDP flood 25 s (CPU ~49 %, one ~10 s HTTP stall of unclear origin), ~10 Mbit/s broadcast 30 s — **no reset**.
9. Handoff v1 written; first external AI ("Gemini") answered (`issue-362-ai-response.md`); its claims were audited against the code (results in section 6).

---

## 4. Proven mechanical facts (verified in source / live — rely on these)

1. **IWDT feed logic** (identical 5.5.3 & 6.1, `components/esp_system/int_wdt.c`): fed only by CPU0's tick hook and only when CPU1 ticked since; stage 0 = 300 ms → interrupt/panic → reboot; stage 1 = 600 ms → hard reset. The mechanism is the `int_wdt_cpu1_ticked` flag pair (there is **no** `find_first_cpu_that_did_not_tick()` — a hallucinated name from the first external AI).
2. ESP32 tick = internal CPU timer interrupt (CCOUNT, level 1, IRAM). Survives flash ops and `esp_intr_noniram_disable()`. **Any critical section or spin at INTLEVEL ≥1 masks it while held/spun** — on BOTH the classic (5.5.3) and SMP (6.1) ports (both do `RSIL XCHAL_EXCM_LEVEL` ≈ level 3 in `portENTER_CRITICAL`/mux acquisition). The claim "classic FreeRTOS never suppresses ticks" is FALSE.
3. Flash guard stalls the other CPU in **task context** (both generations) — ticks keep running. Flash ops cannot starve the IWDT.
4. **Amplification topology (code-verified, from the first external AI — its main valid contribution):** every outbound relay datagram goes through `sendMessage()` → `_udp_sendto()` (`include/udphelper.h`) → `tcpip_api_call()` → `sys_mbox_post` + `sys_arch_sem_wait(sem, 0)` = **unbounded**. The **UART task (prio 15) blocks inside this call for every module→CCU frame**, and the RawUart worker blocks for every keepalive (1 s cadence). If `tcpip_thread` stops progressing, both tasks freeze **inside** `sendMessage` — therefore the 10-s keepalive timeout check never runs, no "CCU timed out" is ever logged, LEDs freeze. This explains the silent presentation of the hang.
5. Consequently: once `tcpip_thread` is dead, every network-touching task progressively freezes (socket ops, esp_ping, httpd), while non-network tasks and ticks survive — matching the 5.5.3 hang (ping dead, no watchdog).
6. On 6.1 the same state additionally produces a ≥300 ms tick starvation ⇒ IWDT. **The mechanism (what holds/spins INTLEVEL that long) is unknown.** Note: the IWDT feed (tick hook) runs BEFORE the kernel spinlock in the SMP tick handler, so kernel-lock contention alone cannot starve the feed — a core's tick must not fire at all.
7. lwIP UDP RX path: EMAC ISR (6.1: lean, notifies `emac_rx` task) → esp_netif → `tcpip_thread` → our recv callback enqueues pbuf to the worker queue (depth 32, non-blocking).
8. `LogManager::write` (vprintf hook) takes its mutex with a **10 ms timeout** (`main/log_manager.cpp:268`) — bounded. httpd's `config.close_fn` is `log_stream_close_socket` **globally for every HTTP socket** (`main/webui.cpp:1925`), routing each connection close through WS-subscriber bookkeeping (stream mutex, `portMAX_DELAY`).
9. Field chip revisions: crashing units include rev100 AND rev301; the stable bench unit is rev100.
10. lwIP 2.2.0 on both 5.5.3 and 6.1; PM/tickless off in both; full sdkconfig diff (same app code) shows no app-relevant delta.

---

## 5. Eliminated paths (do NOT reopen without new evidence)

Heap exhaustion · ECO3 livelock as sole cause (both revs crash with workaround) · MQTT/syslog/log features (crash with all off) · stuck NVS/net-fetch ops (flight recorder) · periodic flash writes (none exist) · flash guard tick starvation (mechanics exclude it) · `ETH_IRAM_OPTIMIZATION` (both states crashed) · lwIP version (identical) · PM/tickless (off) · config deltas (none relevant) · app critical sections (trivial) · app ISRs (DCF only, absent) · UART TX concurrency (single writer) · EMAC TX blocking (6.1 non-blocking) · generic LAN flood (bench: no reset) · known upstream IDF regression (none found; v6.1→master has nothing) · lwIP core-locking mode (off in both) · `ccuIP`/security-reset theories (no such path) · SPIFFS mount failure (benign fallback) · I2C/RTC (bounded, uncorrelated) · esp_timer ISR dispatch (unused) · removed-feature theories (update-check/supporter-key/CRL — removed, crashes continued).

---

## 6. First external AI response — audit result

Valid (kept): the amplification topology (fact 4); `ESP_LOGW` on `tcpip_thread` from the queue-full path (bounded 10 ms — hygiene issue); the global `close_fn` hazard; several IDF citations.

Invalid (discarded): claim that the bench unit has **no radio module** (false — RPI-RF-MOD attached, CCU session live, verified via `/api/system/overview`); the 5.5.3-vs-6.1 asymmetry explanation ("classic never suppresses ticks" — false, see fact 2); `find_first_cpu_that_did_not_tick()` (does not exist); its "Patch 2" is a literal no-op diff. Its patches 1 (drop `ESP_LOGW` in recv callback) and 3 (guard `close_fn` to subscriber fds) are sensible and **planned for Beta.6** but NOT yet applied on main.

**Neither we nor the first external AI identified the initiator.** That is the open problem.

---

## 7. Where to dig — ranked hypotheses and concrete audit targets for you

**H1 (prime): something in the 2.2.x-only always-on code blocks or kills `tcpip_thread` under load.** New-since-v2.1.10 network-facing surface: httpd WebUI (`webui.cpp`, `webui_ota.cpp`, `webui_backup.cpp`, `webui_storage.cpp`), `log_stream.cpp` (WS live log + its publish worker and `close_fn` hook), `events.cpp`, `metrics.cpp`/`prometheus.cpp`, `syslog.cpp` (optional), `system_overview_api.cpp`, `theme_api.cpp`, `rate_limiter.cpp`, `nvs_storage_lock.cpp`. v2.1.10 already had monitoring/MQTT/log_manager basics. Audit every callback that runs on `tcpip_thread` (our UDP recv callback and anything lwIP/esp_netif invokes) for blocking calls, mutexes, logging, or unbounded queues. Trace the `log_stream` publish worker and its interaction with httpd socket sends. Check whether `log_stream_close_socket`'s `portMAX_DELAY` mutex can be held long by any path (WS send while network is degraded?).
**H2: EMAC/RX input death** (descriptor ring stall, PHY/link state machine, RMII REF_CLK on GPIO17 — 50 MHz out — board-level instability, needs hardware to measure). Explain why the bench (same silicon class, same PHY wiring) stays alive: traffic composition.
**H3: the 6.1-only INTLEVEL holder** — find any path that can hold/spin a portMUX ≥300 ms under the wedged state (driver bug holding a spinlock across a slow operation; a spinning loop in an ISR; `xt_highint` paths). This only matters after H1/H2 fire.
**H4: network-event trigger surface** (AdGuard observation): bursts of broadcast/multicast/DNS from LAN events interacting with H1 paths. The `tcpip_mbox` (capacity 32) saturation behavior is a candidate pressure point.
**H5: minor mystery** — the one-off radio-module detection miss on the TX-ring build (`uart_wait_tx_done` interaction with detector timing?).

Also evaluate (not yet analyzed in depth): the interaction of the new net-watchdog's per-minute `ping_service_ping` (fresh `esp_ping` session each call, raw PCB, from heap_watchdog task) with the tcpip path — introduced AFTER the field data, so it cannot explain history, but must not add new deadlock surface.

---

## 8. YOUR deliverable and response contract

Write `.planning/handoff/issue-362-ai-response-2.md` with EXACTLY these sections:

1. `## Executive summary` — ≤10 lines: your best initiator candidate + confidence.
2. `## Ranked findings` — numbered; each with hypothesis, **code evidence (file:line, quote the lines)**, a discriminating test that field devices can actually produce (no serial, no coredump), and confidence.
3. `## Answers to section 7` — per hypothesis H1–H5.
4. `## Proposed patches` — precise diffs; must build on IDF 6.1 and ideally 5.5.3. Do NOT apply them.
5. `## Verification plan` — build commands (section 9) + what to observe on the bench device.
6. `## What I could not determine` — honest gaps.

Rules: every code claim cites `path:line`; separate FACT from INFERENCE; reopening a section-5 item requires new cited evidence; no firmware changes outside the patches section.

---

## 9. Environment

- Repo `Xerolux/HB-RF-ETH-ng` on GitHub. `main` = IDF 6.1 regular line (HEAD includes net-watchdog, commit `ec542c9`+). Branch `experiment/idf-v5.5.3` = 5.5.3 line. Releases: `v2.2.7-Beta.4/5`, experiment `experiment-idf55exp-b5`. Issue #362 comments = field ground truth.
- Builds in WSL Ubuntu-24.04: `. ~/venvs/dev/bin/activate; . /home/basti/esp-idf-6.1-beta1/export.sh` (or `/home/basti/esp-idf-5.5.3/export.sh`), then in repo root: `export IDF_TARGET=esp32 SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.hb-rf-eth-ng"; idf.py build` (`-B build-idf55` for the 5.5 line). Apply `bash scripts/patch_idf_eco3_fix.sh <idf_path>` once per IDF checkout. `webui/dist/*.gz` must exist (`python3 rename_webui_files.py` regenerates).
- Host policy tests: `cd test/host && python -m unittest test_interrupt_safety_policy` — must stay 9/9.
- Bench device (rev100, RPI-RF-MOD attached, CCU session live, historically stable) exists on the maintainer's LAN for experiments — credentials are NOT in this repo; ask the maintainer.
- Hard constraints: no partition-table changes / no coredump partition (4 MB flash full, maintainer veto); field has no serial; keep the 300 ms IWDT (policy test enforces); changes must not weaken the relay protocol.

## 10. Companion documents
- `issue-362-investigation-handoff.md` (v1) and `issue-362-ai-response.md` (first external answer) — same folder, kept as history.
- `.planning/debug/eco3-livelock-rootcause.md` — investigation log + decision tree.
