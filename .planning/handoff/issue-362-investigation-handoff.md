# AI Investigation Handoff — Issue #362 "Stop working" (Interrupt-Watchdog resets / network wedge)

**Date:** 2026-09-05
**Status:** OPEN — root cause not yet identified. Root failure class identified and instrumented.
**Prepared by:** prior AI investigation session(s). This document is the complete state of the hunt.
**Purpose:** enable a fresh AI (or human) investigator to continue with full context, no repetition of eliminated paths, and a clear reporting contract.

---

## 1. Mission statement for the receiving AI

Find the root cause of spontaneous device failures in this ESP32 (WROOM-32, no PSRAM, LAN8720 RMII Ethernet) firmware: the HB-RF-ETH-ng. Your deliverable is described in section 8. Read this whole document before touching code. Do **not** re-investigate the eliminated paths in section 5 without new evidence — they are closed for the reasons given.

---

## 2. The device and the failure

HB-RF-ETH-ng: ESP32-based HomeMatic RF-to-Ethernet bridge. A CCU (HomeMatic central, e.g. RaspberryMatic/OpenCCU) holds a persistent UDP session ("raw-uart protocol", port 3008) that bridges to an RPI-RF-MOD radio module over UART1 @115200. The device also runs an httpd-based WebUI, optional MQTT/syslog/Prometheus, NTP client/server, I2C RTC (RX8130).

**Failure signature (field, firmware 2.2.x, both IDF generations):**
- Device dies spontaneously: WebUI unreachable, CCU reports radio loss.
- On ESP-IDF 6.x builds: reset reason **"Watchdog Reset (Interrupt Watchdog)"** (ESP_RST_INT_WDT), device auto-reboots. No log line precedes it (syslog verified running). Heap healthy at every crash (crash blackbox: 82–104 KB free, low_streak=0).
- On an ESP-IDF 5.5.3 build of the SAME current code (experiment, see 4.3): device **hangs completely instead** — not even pingable, no watchdog fires, power-cycle required. LED state frozen.
- Uptimes before failure: 66 s … 32 h, many in the 1–1.5 h range; clustering during/right after CCU reconnect bursts.
- Load correlation: units in real homes with continuous RF traffic crash; the maintainer's bench unit (same firmware, low RF traffic) ran 12+ days stable.

---

## 3. Timeline of the investigation and all changes made

Everything below is committed on `main` unless noted.

1. **ESP-IDF pinned v6.1-rc1 → v6.1 final** (workflows/docs/test; unrelated to the bug, maintainer request).
2. **ECO3 cache-lock livelock workaround enabled** (`scripts/patch_idf_eco3_fix.sh` + step in all 5 firmware workflows). Removes the `SPIRAM` gate from `CONFIG_ESP32_ECO3_CACHE_LOCK_FIX` in the freshly cloned IDF Kconfig. No-op on silicon ≤ v2. Kept (correct for rev3 silicon) even though it did not fix #362.
3. **Tick sentinel in the crash blackbox** (`main/crash_blackbox.cpp`, `include/crash_blackbox.h`): IRAM FreeRTOS tick hook per CPU, writes per-core last-tick ms into RTC-noinit; latched at next boot into `prev_tick_ms[]` before re-arming; boot prints `Tick sentinel (pre-reset): cpu0 … cpu1 …` for watchdog-class resets. Beta.4 shipped a broken version (read live slots ~7 s after boot → only new-session data); Beta.5 fixed the latch.
4. **UART1 TX ring buffer 2 KiB** + `uart_wait_tx_done()` before module reset (`main/radiomoduleconnector.cpp`). One known one-off: a single boot on the TX-ring build showed "Radio module could not be detected" (transient, relay still worked). Unexplained — see leads, 6.8.
5. **RawUart worker priority restored 12 → 15** (the stable v2.1.10 value; level with the UART task).
6. **Releases:** v2.2.7-Beta.4, v2.2.7-Beta.5 (regular, IDF 6.1) and `experiment-idf55exp-b5` (branch `experiment/idf-v5.5.3`, tag not starting with `v` deliberately, version `2.2.7-Beta.5-IDF55EXP`, built on ESP-IDF v5.5.3 = the generation of stable v2.1.10, classic (non-SMP) FreeRTOS kernel verified). 55EXP compiles with tiny version guards (lan87xx header/RMII enum/log-stream-WS compiled out on <6.0 — note: **the 55EXP build has NO live-log WebSocket**, WebUI falls back to `/api/log` polling).
7. **Network wedge self-healing + event blackbox snapshots** (commit ec542c9, pending release as Beta.6): heap_watchdog (60 s cycle) pings the default gateway; link-up + network-was-alive + 5 consecutive fails → clean restart with `net watchdog:` diag, crash tail, blackbox snapshot. Event-driven snapshots also on CCU keepalive timeout and Ethernet link-down (`crash_blackbox_snapshot_now()`). Live-validated 12 min on the reference device (no false positives, correct behavior across a CCU-initiated disconnect).
8. Live experiments on the maintainer's bench device (see 5.16): 67 Mbit/s unicast UDP flood 25 s (CPU ~49%, one ~10 s HTTP stall of unclear origin — possibly sender congestion), ~10 Mbit/s broadcast 30 s: **no reset**.

---

## 4. Field data (what the humans reported — sources: issue #362 comments)

### 4.1 Testers
| Tester | CCU | Chip rev | Network | History |
|---|---|---|---|---|
| zoephelweb | CCU3-class, 192.168.0.100 | **301** | generic | crashes since 2.2.3 (~1 h cadence typical); board REPLACED (new PCB, same crash) |
| Walki2000 | OpenCCU, 192.168.178.101 (FritzBox net) | **100** | FritzBox | crashes since 2.2.x (66 s–5 h); 9 h 15 min stable once (build version NOT confirmed — question pending) |
| ChristophA | OpenCCU VM, 192.168.188.x | unknown | FritzBox-ish | crashes; syslog+MQTT on |
| M-Schoeler | — | unknown | — | crashes "couple of hours" |
| maintainer bench (.56) | 192.168.178.53 | **100** | FritzBox | **stable** (12 d on Beta.2; also stable on Beta.4/5-dev since) — low RF traffic |

### 4.2 Key events
- Beta.4 crashes: Walki2000 up=3786 s (rev100), zoephelweb up=5286 s (rev301) — both WITH ECO3 workaround ⇒ silicon livelock falsified as sole cause.
- 55EXP: zoephelweb one crash ~2 h after install (coinciding with integrating **AdGuard** into his network — possible network-event trigger), then 6 h stable, then overnight **total hang** (ping dead, power-cycle needed, blue LED solid). ⇒ IDF 5.5.3 does NOT fix it; symptom changes from reset to hang.
- Walki2000: 9 h 15 min stable (build unconfirmed) but 19 devices reported disturbed communication after update (his known legacy issue, possibly unrelated).
- zoephelweb: on older FW versions crashes always auto-rebooted; only 55EXP hangs completely.

### 4.3 Important inconsistency to remember
Users historically reported **v2.1.10 (IDF 5.5.3, old app code) as stable on the same hardware** — yet 55EXP (same IDF 5.5.3, CURRENT app code) hangs. This is a strong hint that the **root cause lives in the 2.2.x application rewrite's interaction with the network stack**, not (only) in the IDF generation. See leads 6.5.

---

## 5. ELIMINATED hypotheses (do NOT re-investigate without new evidence)

1. **Heap exhaustion/fragmentation** — blackbox healthy at every crash; heap stable over 32 h in the field.
2. **ESP32 ECO3 silicon livelock (errata WDT-3.15) as sole cause** — rev100 + rev301 crash with the IDF workaround active. (Workaround stays enabled; it is correct for rev3 regardless.)
3. **Optional features (MQTT/syslog/log capture)** — Walki2000 crashes with everything disabled.
4. **Stuck NVS/net-fetch operation** — op-tag flight recorder (see `crash_blackbox_describe_stuck_op`) never showed an in-flight op at crash.
5. **Periodic flash writes** — full audit: all NVS/SPIFFS writers are user-action triggered; no periodic writer. Crash times don't align with any flash op.
6. **Flash-op guard starving the tick** — verified in both IDF 5.x and 6.x source: other CPU stalls in TASK context (`spi_flash_op_block_func`), non-IRAM interrupts masked only; tick (internal CPU timer, IRAM) keeps running. Also `spi_flash_erase` yields. Dead end.
7. **`CONFIG_ETH_IRAM_OPTIMIZATION`** — crashed in both states (on: 2.2.4–2.2.6B3; off: 2.1.10, 2.2.7B.x).
8. **lwIP version delta** — 5.5.3 and 6.1 both ship lwIP **2.2.0**.
9. **Power management / tickless idle** — `CONFIG_PM_ENABLE` unset in both generations' fragments (verified in generated sdkconfig).
10. **sdkconfig deltas app-relevant** — full diff of the two generated configs (identical app code): nothing functional besides renames/defaults of disabled subsystems.
11. **App-level critical sections** — every `portENTER_CRITICAL` in the app audited: trivial fixed-size struct copies.
12. **App-level ISRs** — only DCF77 pin-change ISR (hardware absent on crashing units).
13. **UART TX concurrency** — exactly one live writer task (RawUart worker); RadioModuleDetector writes only at boot.
14. **EMAC TX blocking** — 6.1 `emac_esp_dma_transmit_frame` returns 0 on busy (non-blocking); `esp_eth_transmit` mutex path disabled.
15. **Kernel-spinlock starvation of the IWDT feed** — in the 6.1 SMP tick handler the IWDT feed (tick hook) runs BEFORE `taskENTER_CRITICAL_FROM_ISR`, so kernel-lock contention cannot starve the feed. The feed only dies if a core's tick ISR does not run for 300 ms (interrupts masked ≥ tick level, or a looping/re-entering same-or-higher-level ISR).
16. **Generic LAN flood** — 67 Mbit/s unicast / 10 Mbit/s broadcast against the bench device: no reset (one 10 s stall of unclear origin under unicast flood).
17. **Upstream IDF regression (known issue)** — esp-idf tracker searched (ESP_RST_INT_WDT / watchdog / SMP / xtensa queries): no match; v6.1→master contains no relevant fix (6 commits, none watchdog/xtensa/freertos).
18. **lwIP TCPIP core-locking mode difference** — neither build defines `CONFIG_LWIP_TCPIP_CORE_LOCKING`; `tcpip_api_call` behaves identically (mbox + semaphore).
19. **"CCU IP not entered" / security-reset theories** — `ccuIP` setting only whitelists the WebUI login rate limiter; no code path resets on network input; foreign UDP sources are logged+dropped.
20. **SPIFFS mount failure (E SPIFFS: mount failed, -10025)** — `format_if_mount_failed=false`, benign fallback to embedded WebUI; no erase path.
21. **I2C/RTC (RX8130) hourly writes** — new `i2c_master` driver, 100 ms bounded timeouts, task context; crash times uncorrelated with the hourly write phase.
22. **esp_timer ISR-dispatch mode** — `ESP_TIMER_ISR_DISPATCH` unused (BT-only in IDF).
23. **ccuIP/OTA/update-check/supporter-key era theories** — those features were removed entirely across 2.2.5/2.2.6; crashes continued.

---

## 6. Proven mechanical facts (verified in IDF source, not assumed)

1. **IWDT feed logic** (`components/esp_system/int_wdt.c`, identical 5.5.3 & 6.1): fed only by CPU0's tick hook and only if CPU1 ticked since. Stage 0 = 300 ms interrupt → panic/reboot; stage 1 = 600 ms hard reset. ⇒ every IWDT reset means **a core's tick ISR did not run for ≥300 ms** (interrupts masked at/above tick level, or a same/higher-level ISR looping/re-entering continuously).
2. ESP32 FreeRTOS tick = **internal CPU timer interrupt (CCOUNT, level 1, IRAM)** — survives flash ops and `esp_intr_noniram_disable()`.
3. lwIP UDP RX path on this app: EMAC ISR (6.1: lean, notifies `emac_rx` task) → esp_netif → tcpip task → our UDP recv callback enqueues pbuf to RawUart worker queue. TX: worker/UART task → `tcpip_api_call` (synchronous mbox+sem).
4. RawUart relay architecture is **essentially identical since v2.1.10** (same task shape, same `tcpip_api_call` pattern); priorities: UART task 15, worker 15 (restored in Beta.5).
5. **What's NEW in always-on code since v2.1.10** (this is the prime suspect surface — v2.1.10 stable in field, 55EXP with new code on same IDF hangs): httpd-based WebUI (`webui.cpp` + `webui_ota/backup/storage.cpp`), WebSocket support (`CONFIG_HTTPD_WS_SUPPORT`), `log_stream.cpp` (WS live log), `events.cpp`, `metrics.cpp`/`prometheus.cpp`, `syslog.cpp` (optional), `nvs_storage_lock.cpp`, `crash_blackbox.cpp`, monitoring heap/net watchdogs, rate limiter, `system_overview_api.cpp`, `theme_api.cpp`. v2.1.10 already had: monitoring, MQTT handler, log_manager.
6. The wedge state (5.5.3 hang): **tasks/scheduler alive (no WDT), network completely dead incl. ICMP** ⇒ EMAC RX → tcpip input chain is dead somewhere while ticks continue.
7. The 6.x escalation: same wedge ALSO produces ≥300 ms tick starvation (mechanism unknown — candidate: a portMUX held/spun, or an ISR storm; note EMAC 6.1 ISR is lean).
8. Unexplained minor: one boot with TX-ring build missed radio-module detection once (transient).
9. zoephelweb's 55EXP crash coincided with AdGuard joining the network (DNS/multicast/broadcast pattern change?) — single observation, unverified.

---

## 7. Ranked open hypotheses

1. **App-code×network-stack interaction wedge (strongest).** Something in the 2.2.x-only always-on network-consuming code wedges the RX path (httpd/log_stream/metrics/events under specific traffic), on any IDF. Discriminator: with Beta.6 deployed, the next field wedge produces a net-watchdog restart with blackbox onset data.
2. **EMAC/RX-DMA wedge triggered by LAN events/traffic patterns** (link renegotiation, multicast bursts, PHY clock glitch). RMII REF_CLK is output by GPIO17 at 50 MHz to the LAN8720; board-level clock instability is untested (needs hardware scope/logic analyzer — maintainer considering buying a board for bench reproduction).
3. **6.x-only escalation mechanics** (how the wedge becomes tick starvation): SMP port critical sections / spin behavior under the wedged state. Secondary: only matters after (1)/(2).
4. Environment/traffic asymmetry (why bench stable, homes crash): RF-traffic volume via UART relay, LAN noise composition (FritzBox IGMP/multicast, AdGuard…), device count.

---

## 8. YOUR TASK and the response contract

### 8.1 Task
Primary: identify the root cause of the network-stack wedge (hypothesis class 1 and/or 2) by static code analysis of THIS repository plus targeted reasoning over ESP-IDF v6.1 sources (a full 6.1 checkout exists at `~/esp-idf-6.1-beta1` in the maintainer's WSL; see 9). Secondary: explain the 6.x tick-starvation escalation.

Concretely:
- Audit the 2.2.x-only network-facing always-on code (section 6.5 list) for paths that can wedge/deadlock/starve the tcpip task or EMAC RX under sustained UDP-relay load + occasional HTTP/WS access. Trace every callback that runs on the tcpip thread.
- Cross-check the esp_netif/lwIP/httpd glue in IDF 6.1 for known deadlock shapes with our call pattern (`tcpip_api_call` from prio-15 tasks per relay frame, httpd sockets, WS).
- Explain why the same code on 5.5.3 hangs silently while on 6.x the tick starves.
- Propose the MINIMAL discriminating experiment/instrumentation for each surviving hypothesis.

### 8.2 Constraints (hard)
- No partition-table changes, no coredump partition (4 MB flash is full; maintainer vetoed).
- Field devices have no serial console; evidence must come through WebUI logs / crash blackbox / behavior.
- Every change must build on ESP-IDF v6.1 (regular line) — ideally still on 5.5.3 (experiment line).
- Do not weaken the interrupt watchdog (300 ms is policy; `test/host/test_interrupt_safety_policy.py` enforces related invariants — keep 9/9 green).
- Keep the maintainer's style: heavy comments explaining WHY, no drive-by refactors.

### 8.3 How to respond (the contract)
Write your findings to **`.planning/handoff/issue-362-ai-response.md`** (create it) with EXACTLY these sections:

1. `## Executive summary` — ≤10 lines: your single best root-cause candidate + confidence.
2. `## Ranked findings` — numbered, each with: hypothesis, code evidence (file:line, quote the lines), proposed discriminating test, confidence (high/medium/low).
3. `## Answers to the open questions` — for each item in section 6/7 you can answer.
4. `## Proposed patches` — concrete diffs or file/line-precise change descriptions (do not apply without building).
5. `## Verification plan` — commands/steps to build (see section 9) and what to observe.
6. `## What I could not determine` — honest gaps.

Rules: every claim about code MUST cite `path:line`. Separate FACT (verified in source/field) from INFERENCE. If you re-open an eliminated hypothesis (section 5), you MUST cite new evidence. Do not modify firmware behavior outside your proposed-patches section.

---

## 9. Build & repo environment (maintainer's machine)

- Repo: this one (`main` = regular IDF 6.1 line; `experiment/idf-v5.5.3` = 5.5.3 line; tags `experiment-idf55exp-b5`, `v2.2.7-Beta.4/5`).
- Builds run in **WSL Ubuntu-24.04**: `. ~/venvs/dev/bin/activate; . /home/basti/esp-idf-6.1-beta1/export.sh` (6.1 line) or `/home/basti/esp-idf-5.5.3/export.sh` (experiment line); then in repo root: `export IDF_TARGET=esp32 SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.hb-rf-eth-ng"; idf.py build` (use `-B build-idf55` for the 5.5 line). Apply `bash scripts/patch_idf_eco3_fix.sh <idf_path>` once per IDF checkout.
- Embedded WebUI assets must exist (`webui/dist/*.gz`; regenerate with `python3 rename_webui_files.py` if missing).
- Releases are created via `workflow_dispatch` on `.github/workflows/release.yml` with input `tag=vX.Y.Z-Beta.N` (the workflow bumps version.txt and manifests itself; pushing a `v*` tag directly FAILS validation unless version.txt matches). Experiment builds: non-`v` tags + manual `gh release create` to avoid the pipeline.
- Reference bench device (maintainer's, rev100, low traffic, historically stable) is on the LAN for live experiments; access credentials are NOT in this repo — ask the maintainer.
- Host policy tests: `cd test/host && python -m unittest test_interrupt_safety_policy` (must stay 9/9 OK).

## 10. Companion documents
- `.planning/debug/eco3-livelock-rootcause.md` — investigation log incl. decision tree for field results.
- `.planning/debug/resolved/*.md` — earlier, superseded investigations (ping UAF #393, MQTT, update search…).
- Issue #362 (GitHub) — all field logs; the comments are the ground truth for symptoms.
