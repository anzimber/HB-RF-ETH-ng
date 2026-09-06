# AI Investigation Response #2 — Issue #362 "Stop working" (network wedge / IWDT resets)

**Date:** 2026-09-06
**Target:** HB-RF-ETH-ng, `main` @ `699261c` (IDF 6.1 line) and `experiment/idf-v5.5.3` (5.5.3 line)
**IDF sources verified against:** `esp-idf-6.1-beta1` and `esp-idf-5.5.3` checkouts in WSL (`/home/basti/…`), plus the issue-#362 field logs (ground truth).

---

## Executive summary

The wedge is **not** a lock the app takes on `tcpip_thread`: the only app callback that runs there (`_raw_uart_udpReceivePaket`) is non-blocking, and every app lock is a FreeRTOS mutex that blocks its *caller*, never tcpip. Verified mechanics narrow the initiator to one class: **`tcpip_thread` stops progressing inside lwIP core machinery (a non-returning handler or corrupted lwIP structures) driven by the 2.2.x traffic profile** — the relay amplification topology (already established) then freezes the UART/worker tasks inside `tcpip_api_call()`, which explains the silent 5.5.3 hang exactly (tcpip has priority 18, is **not** task-WDT-watched in this build, and its mbox/timer machinery is bounded — so only a stuck handler or corruption stops it). On 6.1 the same state is *converted* into the IWDT reset by an SMP-specific mechanism: a core spinning ≥300 ms in `spinlock_acquire()` at INTLEVEL 3 (the only 6.1-only INTLEVEL holder that exists), most plausibly on a lock whose holder is itself frozen in a `tcpip_api_call()`. Confidence: initiator class HIGH, exact first instruction MEDIUM. Two concrete defects found on the way are independent of that: (1) the Beta.6 net-watchdog **cannot fire in the wedge state** because `esp_ping` session create/delete are blocking `tcpip_api_call`s; (2) `RadioModuleConnector::stop()` can `vTaskDelete` the UART task while it blocks inside `tcpip_api_call` with a stack-backed call descriptor.

---

## Ranked findings

### Finding 1 — Amplification topology confirmed; the pbuf "double-free" theory is definitively closed
- **Hypothesis:** The silent presentation (no "CCU timed out" log, frozen LED) is caused by both relay tasks blocking inside `sendMessage()` → `tcpip_api_call()` while `tcpip_thread` stalls; the keepalive timeout check is unreachable in that state. Additionally, a plausible *initiator* theory ("`pbuf_free(pb)` after `_udp_sendto` is a double free") is **falsified** by the lwIP 2.2.0 contract.
- **Code evidence (FACT):**
  - `main/rawuartudplistener.cpp:403-404` — the send path:
    ```cpp
        _udp_sendto(pcb, pb, &addr, port);
        pbuf_free(pb);
    ```
  - lwIP `udp.c:447` (IDF 6.1 checkout): `" * Sends the pbuf p using UDP. The pbuf is not deallocated."` and `udp.c:916` (in `udp_sendto_if_src_chksum`): `"    /* p is still referenced by the caller, and will live on */"` — the caller **keeps ownership** and *must* free the pbuf. `sendMessage()` therefore does exactly what lwIP demands. (Also identical to the stable v2.1.10 pattern — `git diff v2.1.10 HEAD -- include/udphelper.h` shows only helper additions.)
  - `main/rawuartudplistener.cpp:610-628`: the 10 s timeout check `if ((TickType_t)(now - last_received_keep_alive) >= connection_timeout)` runs only if the loop iterates; with `sendMessage` (line 634, keepalive; line 403) blocked inside `tcpip_api_call`, it never does. `main/radiomoduleconnector.cpp:207-214` (`_handleFrame` → `frameHandler->handleFrame`) funnels every module frame through the same blocking call (`rawuartudplistener.cpp:418 sendMessage(7, buffer, len)`).
  - `include/udphelper.h:144-153`: `_udp_sendto` → `tcpip_api_call(_udp_sendto_api, …)`; lwIP `tcpip.c:508`: `sys_arch_sem_wait(TCPIP_MSG_VAR_REF(msg).msg.api_call.sem, 0);` — **unbounded wait** (FACT).
- **Discriminating test (field-producible):** Beta.6's intended `net watchdog:` restart line appearing in the boot log after a field wedge — *if* the watchdog could fire (see Finding 3, which shows it cannot). The actionable instrumentation is the tcpip-liveness sentinel proposed in Patch 2.
- **Confidence:** HIGH (mechanism fully verified; the double-free alternative eliminated with cited lwIP source).

### Finding 2 — The initiator must be a `tcpip_thread`-internal stall; no app lock can block it
- **Hypothesis:** `tcpip_thread` stops progressing because a handler never returns or lwIP core structures are corrupted. Everything else that could "kill" tcpip is verified away.
- **Code evidence (FACT):**
  - Priority: `sdkconfig.hb-rf-eth-ng` → `CONFIG_LWIP_TCPIP_TASK_PRIO=18` (highest app-facing task; only kernel/IPC tasks sit above). No starvation from the prio-15 relay tasks is possible.
  - No task-WDT coverage: `LWIP_TCPIP_THREAD_ALIVE()` is the no-op default (`lwip/src/include/lwip/opt.h:1868-1869`: `#define LWIP_TCPIP_THREAD_ALIVE()`), so a busy loop inside `tcpip_thread` hangs the device **silently** — matching the 5.5.3 field signature (no panic, no TWDT backtrace ever reported).
  - The loop itself is bounded: `tcpip.c:127-153` (`tcpip_thread`) with `TCPIP_MBOX_FETCH` → `tcpip_timeouts_mbox_fetch` (`tcpip.c:75-114`): bounded `sys_arch_mbox_fetch(…, sleeptime)` + `sys_check_timeouts()`. `timeouts.c`: `sys_timeouts_sleeptime()` returns `SYS_TIMEOUTS_SLEEPTIME_INFINITE` / 0 / a positive bounded `ret` (asserted `ret <= LWIP_MAX_TIMEOUT`). The EMAC side is bounded too: 6.1 `emac_isr_default_handler` (`esp_eth_mac_esp.c:758-799`) only clears interrupts and does `vTaskNotifyGiveFromISR`; `emac_esp32_rx_task` (`:600-649`) and the DMA descriptor loops (`esp_eth_mac_esp_dma.c:323,358`: `while (… && (used_descs < CONFIG_ETH_DMA_RX_BUFFER_NUM))`) are all bounded; `emac_esp32_transmit` (`:548-557`) returns `ESP_ERR_NO_MEM` when busy (non-blocking).
  - The only app callback on tcpip is non-blocking: `main/rawuartudplistener.cpp:678-703` (`_udpReceivePacket`: `xQueueSend(queue, &event, 0)` + a 10 ms-bounded `ESP_LOGW` via `log_manager.cpp:268`). `ntpserver.cpp`'s `_ntp_udpReceivePaket` likewise only enqueues (comment at `ntpserver.cpp:191`: "never block here"). All other new 2.2.x network code runs on its own tasks (httpd prio 5, `events` prio 3, `log_stream` prio 4, `heap_watchdog` prio 2) and only uses FreeRTOS mutexes — a task blocked on a mutex cannot stop tcpip.
- **INFERENCE:** With the loop bounded and no app lock reachable from tcpip, a permanent stall requires either (a) a handler (`udp_input`/`tcp_input`/`ip_input`/reassembly/`sys_check_timeouts` callback) failing to return, e.g. via a corrupted lwIP list (timer chain, PCB list, pbuf chain), or (b) corruption of the FreeRTOS queue backing `tcpip_mbox`. I could **not** identify the concrete wild write; the 2.2.x-specific candidates are enumerated in Section "Answers — H1".
- **Discriminating test:** the Patch-2 liveness sentinel (keepalive-driven timestamp updated *on* tcpip, checked by `heap_watchdog` without touching the network). It turns "tcpip stopped" into a visible, field-diagnosable event with a blackbox snapshot, on **both** IDF generations (no serial needed).
- **Confidence:** HIGH for the class; MEDIUM for the specific first instruction.

### Finding 3 — The Beta.6 network watchdog is structurally unable to detect the wedge it was built for
- **Hypothesis:** `heap_watchdog_task` → `ping_service_ping()` → `esp_ping_new_session()` blocks inside `tcpip_api_call()` when tcpip is dead, so the streak counter never increments and the self-healing restart never fires in exactly the failure state it targets.
- **Code evidence (FACT):**
  - `main/monitoring.cpp:1296-1431`: `heap_watchdog_task` (60 s cycle) calls `ping_service_ping(gw_str, NET_WATCHDOG_PING_TIMEOUT_MS)` at line 1392, and only on 5 consecutive failures triggers `ResetInfo::storeResetReason(RESET_REASON_WATCHDOG, diag)` + restart (lines 1417-1430).
  - `main/ping_service.cpp:99-139`: `esp_ping_new_session(&config, &cbs, &ping)` and `esp_ping_delete_session(ping)` — both wrap lwIP raw-PCB operations executed via the mbox/`tcpip_api_call` machinery (the raw pcb is created/removed on the tcpip thread). `esp_ping_start` hands the session to the IDF ping task whose recv path also depends on tcpip RX.
  - `tcpip.c:508` (again): the API-call wait is unbounded. A wedged tcpip therefore freezes `heap_watchdog` inside `ping_service_ping` **before any timeout logic can run**. The same applies to `crash_blackbox_snapshot_now()` calls on CCU-timeout/link-down — those run on the wedge's victims, not the wedge itself.
- **INFERENCE:** This is not the historical initiator (commit `ec542c9` postdates all field data), but it defeats the Beta.6 mitigation and must be fixed before Beta.6 ships (see Patch 2/3).
- **Discriminating test:** bench: manually wedge tcpip (e.g. hold the module in reset + flood CCU frames, or use the Patch-2 sentinel test mode) and observe that no `net watchdog:` restart ever appears while the sentinel shows tcpip stale.
- **Confidence:** HIGH (both halves cited; mechanism follows directly).

### Finding 4 — The 6.1 escalation: verified requirement (≥300 ms INTLEVEL) and the only SMP-specific holder
- **Hypothesis:** The 6.1 IWDT reset requires a core with INTLEVEL ≥1 for ≥300 ms; the only 6.1-specific INTLEVEL holder is the `spinlock_acquire` spin. No concrete ≥300 ms holder was found in app or IDF code — the remaining consistent scenarios are (i) a core spinning on a lock whose holder is frozen in `tcpip_api_call`, or (ii) corruption of a spinlock/kernel object by the same root cause as Finding 2.
- **Code evidence (FACT) — all verified in the 6.1 checkout:**
  - Tick = CCOUNT level-1: `sdkconfig.hb-rf-eth-ng` → `CONFIG_FREERTOS_SYSTICK_USES_CCOUNT=y`; IWDT 300 ms / stage1 600 ms HW reset: `components/esp_system/int_wdt.c:96-103` (`reconfigure_ticks(…, IWDT_STAGE0_TIMEOUT_US, IWDT_STAGE1_TIMEOUT_US)` with `WDT_STAGE_ACTION_INT` / `WDT_STAGE_ACTION_RESET_SYSTEM`).
  - Feed gating: `int_wdt.c:119-140` — `tick_hook` feeds only on CPU0 **and only if `int_wdt_cpu1_ticked`** (`:127-133: if (int_wdt_cpu1_ticked) {…} else { return; }`); `CONFIG_ESP_INT_WDT_CHECK_CPU1=y`. So if **either** core stops ticking ≥300 ms, the reset fires.
  - Feed runs **before** the kernel spinlock: `components/freertos/port_systick.c:191-216` — `xPortSysTickHandler`: `esp_vApplicationTickHook();` (line 199-200) precedes `taskENTER_CRITICAL_FROM_ISR()` (line 205). Kernel-lock contention in the tick path can therefore not starve the feed (confirms handoff fact 6).
  - SMP spinlocks mask the tick while spinning: `components/esp_hw_support/include/spinlock.h:86` `irq_status = XTOS_SET_INTLEVEL(XCHAL_EXCM_LEVEL);` then the spin loop `:131-137` (`do { lock_set = esp_cpu_compare_and_set(…); } while ((timeout == SPINLOCK_WAIT_FOREVER) || …);`), restoring interrupts only after acquisition (`:149-150`). The **holder**, by contrast, restores interrupts after acquiring (`:108-113, :150`) — so the holder can be descheduled while the other core spins. IDF portMUX critical sections additionally keep INTLEVEL 3 for the *whole held region*: `freertos/FreeRTOS-Kernel-SMP/portable/xtensa/port.c:126` (`xPortEnterCriticalTimeout`: `XTOS_SET_INTLEVEL(XCHAL_EXCM_LEVEL)` before `spinlock_acquire`) with restore only in `vPortExitCriticalIDF` (`:144-166`).
  - Classic 5.5.3 masks ticks identically while inside a critical section (`portable/xtensa/include/freertos/portmacro.h:418`: `#define portDISABLE_INTERRUPTS() do { XTOS_SET_INTLEVEL(XCHAL_EXCM_LEVEL); …`) — **but has no cross-core spinlocks**, so there is no "wait for the other core" spin on 5.5.3.
  - ECO3 side effect (note, not a reopening): with the project's patched `CONFIG_ESP32_ECO3_CACHE_LOCK_FIX=y`, on rev-3 silicon the effective stage-0 window shrinks to **20 ms**: `eco3_livelock_workaround.c:18` (`#define IWDT_LIVELOCK_TIMEOUT_MS (20)`), `:36-38` (`_lx_intr_livelock_max = timeout_ms / IWDT_LIVELOCK_TIMEOUT_MS - 1` ⇒ 14 for rev301) and `:41-44` (`return IWDT_STAGE0_TIMEOUT_US / (_lx_intr_livelock_max + 1);` ⇒ 300 ms/15). On rev100 `soc_has_cache_lock_bug()` is false ⇒ full 300 ms. This predicts rev301 units (zoephelweb) reset on much shorter stalls than rev100 units — consistent with the field distribution, but the **historical** pre-Beta.4 resets still require a true ≥300 ms window.
- **INFERENCE:** For the historical 6.1 resets, (i) is the best structural fit: once tcpip wedges (Finding 2), any core spinning on an IDF-internal spinlock whose holder is one of the tasks blocked in `tcpip_api_call` starves its tick ≥300 ms → IWDT, no log, healthy heap. I could not name the specific lock from static analysis (candidates: the sys_arch/lwIP-glue locks, log, heap, UART — none demonstrably held across a blocking tcpip call in the paths exercised here). (ii) remains possible but unproven.
- **Discriminating test:** the next **valid** Beta.5+ tick-sentinel line (`Tick sentinel (pre-reset): cpu0 … cpu1 …`) names the core that stopped ticking first and the millisecond it stopped. cpu1-first ⇒ ISR/lock holder on CPU1 (or CPU1 spinning); cpu0-first ⇒ holder/spinner on CPU0. Correlating that timestamp with the last syslog line and the last blackbox snapshot time brackets the onset. On the bench: a level-3-ISR GPIO-toggle trace can directly measure longest INTLEVEL windows under reconnect-burst load.
- **Confidence:** HIGH that the requirement is INTLEVEL ≥300 ms (code-verified end to end); MEDIUM that a spinlock spin is the holder; LOW on the concrete lock/path.

### Finding 5 — Two concrete relay-path hazards (one pre-existing, one new with Beta.6)
- **Hypothesis:** (a) `_serialQueueHandler` can block forever on `uart_read_bytes(…, portMAX_DELAY)`, killing the relay silently; (b) `RadioModuleConnector::stop()` deletes the UART task without the rendezvous the UDP worker uses, allowing deletion *while blocked inside `tcpip_api_call`* with a stack-allocated call descriptor still referenced by tcpip → potential lwIP/`tcpip_mbox` corruption on the **restart path** (Beta.6 makes restarts automatic, raising exposure).
- **Code evidence (FACT):**
  - `main/radiomoduleconnector.cpp:185`: `int read = uart_read_bytes(UART_NUM_1, buffer, event.size, portMAX_DELAY);` — unbounded.
  - `main/radiomoduleconnector.cpp:115-124`: `stop()` does `vTaskDelete(_tHandle);` immediately — contrast with the worker's careful `_activeSenders` rendezvous (`rawuartudplistener.cpp:353-366, 639-650` and the comment at `:535-540` explaining why a blocked `tcpip_api_call` task must not be aborted).
  - `include/udphelper.h:52-53, 87-90`: `udp_api_call_t msg = {};` is **stack-allocated**; `tcpip.c:508` shows tcpip signals that stack-resident semaphore — if the waiting task dies first, tcpip writes into dead/reused stack memory.
- **INFERENCE:** (a) is relay-fatal but does not kill ping/tcpip (not the initiator). (b) is a genuine corruption vector on restarts; it cannot explain historical crashes (restarts were manual then) but must be fixed alongside Beta.6.
- **Confidence:** HIGH for the code facts; MEDIUM that (b) has ever fired in the field.

### Finding 6 — httpd `close_fn` global hook (retained from AI#1, re-verified)
- **Hypothesis:** `log_stream_close_socket` runs for every closed HTTP socket; `close(fd)` inside it is an lwIP close that can block for the full TCP close sequence; the stream mutex is taken with `portMAX_DELAY`.
- **Code evidence (FACT):** `main/webui.cpp:1925` `config.close_fn = log_stream_close_socket;` (global, `lru_purge_enable = true` at `:1921` makes idle-session purges also pass through it); `main/log_stream.cpp:699-706`:
  ```cpp
  void log_stream_close_socket(httpd_handle_t handle, int fd)
  {
      (void)handle;
      unregister_subscriber(fd);   // takes stream_mutex() portMAX_DELAY (:491)
      close(fd);                   // lwIP close, can block for FIN/retransmit timeouts
  }
  ```
- **INFERENCE:** This stalls the **httpd task** (WebUI), not tcpip — not the initiator, but it contributes the observed "WebUI unreachable" phase and should be scoped to WS descriptors (first AI's Patch 3, still unapplied).
- **Confidence:** MEDIUM (impact), HIGH (code facts).

---

## Answers to section 7

**H1 (2.2.x-only code blocking/killing `tcpip_thread` under load):** PARTIALLY CONFIRMED as the initiator class, with an important negative result: after auditing every path that runs on tcpip (recv callbacks `rawuartudplistener.cpp:110-118/678-703`, `ntpserver.cpp:36-40/176-193`; lwIP input/timer machinery; esp_netif glue `esp_netif_receive` → `ethernetif_input` → `tcpip_input` mbox trypost) and every new always-on subsystem (httpd/WS `webui.cpp:1908-1976`, `log_stream.cpp`, `events.cpp`, `metrics.cpp`, `rate_limiter`, `nvs_storage_lock`, DNS cache `ethernet.cpp:48-165`, blackbox `crash_blackbox.cpp`) — **no 2.2.x code blocks tcpip via a lock or an unbounded queue**. The remaining H1 mechanisms are (a) a corrupted lwIP structure causing a non-returning list walk (timer chain in `sys_check_timeouts`, PCB lists in `udp_input`/`tcp_input`, pbuf chains) or (b) corruption of the `tcpip_mbox` FreeRTOS queue. Concrete corruption suspects in 2.2.x-only code that I could *not* rule out and that deserve targeted review: `handlePacket`'s heap fallback + `pbuf_copy_partial` (`rawuartudplistener.cpp:142-175`), `log_manager.cpp` ring + subscriber iteration under the 10 ms-bounded lock, `webui_storage.cpp` SPIFFS mount/layout, `settings.cpp` NVS key handling, and the `udp_event_t` queue lifetime during `RawUartUdpListener::start/stop` transitions (`rawuartudplistener.cpp:421-556`). Status: initiator class CONFIRMED as tcpip-internal; exact first instruction UNKNOWN.

**H2 (EMAC/RX-DMA input death):** NO EVIDENCE. Both generations use the same bounded RX-task design (5.5.3 `esp_eth_mac_esp.c:448-491`; 6.1 `:600-649`), bounded descriptor walks, a lean ISR (6.1 `:758-799`), and non-blocking TX (6.1 `:548-557`). Driver-level wedge cannot explain the 6.1 IWDT (task context cannot mask ticks) and would leave tcpip alive on 5.5.3 (ARP/keepalives would keep the wire alive and the 10 s timeout would fire → LED red — not observed). The bench/field asymmetry is better explained by traffic *composition* feeding H1 (see H4), not by PHY/clock state. Keep H2 dormant.

**H3 (the 6.1-only INTLEVEL holder):** Requirement now proven end-to-end (see Finding 4): a core's CCOUNT tick must not fire for ≥300 ms; kernel-spinlock contention in the tick path is excluded (hook before spinlock, `port_systick.c:199-205`); the only SMP-only holder class is the `spinlock_acquire` spin at EXCM_LEVEL (`spinlock.h:86,131-137`). The strongest remaining scenario: wedge (H1) → some task holds an IDF-internal spinlock while blocked in `tcpip_api_call` → another core spins ≥300 ms → IWDT. No concrete lock/path identified — this is the honest gap. Secondary note: on rev3 + ECO3-fix builds the effective threshold is 20 ms (`eco3_livelock_workaround.c:18,36-44`), making rev301 units more reset-prone than rev100 *since Beta.4* (cannot explain pre-Beta.4 history).

**H4 (network-event trigger surface / tcpip_mbox saturation):** PLAUSIBLE TRIGGER, not root cause. The `tcpip_mbox` (32 slots) is drained by tcpip at prio 18 — saturation only causes `tcpip_input` drops (`sys_mbox_trypost` path), not a stall. But reconnect bursts (field logs: "CCU 3 disconnected" → "reconnected" clusters before several crashes) plus AdGuard-style LAN churn (mDNS/IGMP/ARP/DNS bursts) maximize the rate of (i) `tcpip_api_call` round-trips, (ii) pbuf/memp churn through `g_lwip_protect_mutex` (`sys_arch.c:30,458-483`), and (iii) the log_stream/metrics/event paths — i.e., they maximize exposure of whatever H1 defect exists. This matches "bench with low RF traffic is stable, real homes crash" without needing a hardware story.

**H5 (one-off radio-module detection miss):** MOST LIKELY ANSWER: `resetModule()` now calls `uart_wait_tx_done(UART_NUM_1, pdMS_TO_TICKS(250))` (`radiomoduleconnector.cpp:139-151`) and the detector re-probes at `115200` after a module reset; a TX ring that still drains bytes from the *previous* (boot) session can delay the detector's first command/response window beyond the detector's internal timeout, producing a transient "could not be detected" that self-heals on the next boot. No code defect found in the detector itself (all waits bounded). One-off field evidence only; not the initiator.

---

## Proposed patches

*(Not applied. All build on 6.1 and, where noted, 5.5.3.)*

### Patch 1 — Bound the relay sends; let the 10 s timeout stay alive (both IDF generations)
`main/rawuartudplistener.cpp` — replace the blocking `sendMessage` for **keepalives** (and optionally frame type 7) with a timeout: either
- (a) a bounded wait via `tcpip_api_call` + `sys_arch_sem_wait(…, timeout)` variant (new helper in `include/udphelper.h`), or
- (b) an outbound `udp_event_t`-style queue drained by the worker with `xQueueSend(…, 0)` and drop+count on overflow (matches the existing RX design, keeps protocol bytes identical).
Rationale: the worker loop then always iterates; the 10 s timeout, LED states and `crash_blackbox_snapshot_now(0)` (`rawuartudplistener.cpp:610-628`) keep functioning when tcpip stalls — converting today's silent wedge into a logged, self-diagnosing event and enabling Patch 2/3. Do **not** use `xTaskAbortDelay` (see the comment at `:535-540`).

### Patch 2 — `tcpip_thread` liveness sentinel (field evidence generator, both generations)
`main/rawuartudplistener.cpp` — in `_udpReceivePacket` (runs on tcpip): on every CCU keepalive (or any frame) store `esp_timer_get_time()` into an RTC-noinit slot via the existing blackbox (`crash_blackbox.h`); in `heap_watchdog_task` (`monitoring.cpp:1296+`): compare that timestamp with `now` — **without any network call** — and if tcpip is stale > 60 s while `_isConnected`, log `tcpip stalled: last rx %u ms ago`, `crash_blackbox_snapshot_now()`, save crash tail, and restart. This is the minimal change that answers "did tcpip stop first?" on the next field device, on both IDF lines, with no serial/coredump.

### Patch 3 — Fix the net-watchdog blindness (Finding 3)
Gate `ping_service_ping` behind the Patch-2 sentinel: if tcpip is already stale, skip the ping entirely and go straight to the snapshot + restart path (`monitoring.cpp:1387-1431`). Optionally additionally bound the ping by moving it into a disposable task whose non-completion the watchdog detects — keeping `esp_ping` session create/delete out of the watchdog task itself.

### Patch 4 — UART task hardening (Finding 5)
`main/radiomoduleconnector.cpp:185` — bound `uart_read_bytes` (e.g. 100 ms), and on timeout `uart_flush_input` + `xQueueReset` + `_streamParser->flush()` (the same recovery the OVF cases at `:189-194` already use). `:115-124` — replace the immediate `vTaskDelete` with a short bounded quiesce (mirror the `_activeSenders` rendezvous) so the task is never deleted while inside `tcpip_api_call` with stack-resident `udp_api_call_t` state (`udphelper.h:52-53,87-90`).

### Patch 5 — Carry-over hygiene from AI#1 (still unapplied, still valid)
- Drop `ESP_LOGW` in `_udpReceivePacket` (`rawuartudplistener.cpp:698`) — keep `g_rx_drops.inc()`.
- Scope `close_fn` to WS subscriber fds (`log_stream.cpp:699-706`): skip `unregister_subscriber` when the fd is not a registered subscriber; never take `stream_mutex()` with `portMAX_DELAY` from the httpd close path — use the 20 ms bounded take already used in `log_stream_subscriber_count` (`log_stream.cpp:387`).

### Patch 6 (diagnostics only, optional) — INTLEVEL profiler for the bench
Instrument a level-3 IRAM ISR (e.g. the EMAC ISR path or a spare timer) to toggle a GPIO while ISRs/critical sections run; logic-analyze under reconnect-burst load to confirm the ≥300 ms INTLEVEL window hypothesis on 6.1 vs 5.5.3 before investing in more firmware changes.

---

## Verification plan

```bash
# Host policy tests (must stay 9/9):
cd test/host && python -m unittest test_interrupt_safety_policy

# WSL builds (both lines), per handoff section 9:
. ~/venvs/dev/bin/activate && . /home/basti/esp-idf-6.1-beta1/export.sh
export IDF_TARGET=esp32 SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.hb-rf-eth-ng"
idf.py build                      # 6.1 line (main)
# and: . /home/basti/esp-idf-5.5.3/export.sh; idf.py build -B build-idf55   # 5.5 line
```

Bench (maintainer's device, rev100, CCU live):
1. Flash Patch 1+2+3 build; verify boot log, 9/9 policy tests, relay round-trip (OpenCCU session must stay green for ≥24 h).
2. Inject a CCU reconnect burst (stop/start CCU session repeatedly) while a browser holds the live-log WS — confirm no wedges and that `hbrfeth_udp_queue_wait_*` metrics stay sane.
3. Force the failure: with Patch 2 build, temporarily stall tcpip on the bench (e.g. debug hook or module-reset + frame flood) — expected: `tcpip stalled:` log + snapshot + clean restart within one watchdog cycle (this validates the instrument end-to-end).
4. Flash the same build on 55EXP (5.5.3) — expected behavior on stall: log + restart instead of today's silent hang.
5. Observe next field samples: `Tick sentinel (pre-reset)` (which core, when) + `tcpip stalled:` + blackbox snapshot → these three lines decide between H1/H3 branches.

---

## What I could not determine

1. **The exact first instruction of the initiator.** I proved the class (tcpip-internal stall; no app lock can block tcpip) but did not find the concrete non-returning path or wild write. The corruption suspects listed under H1 need a targeted line-by-line audit I could not complete within this session.
2. **The concrete ≥300 ms INTLEVEL holder on 6.1.** Verified the requirement and the mechanism family (spinlock spin), but no specific lock/path demonstrably holds a spinlock across a blocking `tcpip_api_call` in the code paths exercised here. A valid field tick-sentinel sample or bench logic-analyzer trace is required to close this.
3. **Whether the 6.1 reset and the 5.5.3 hang share one root cause or are two independent defects** triggered by the same traffic profile (both orders are consistent with all evidence).
4. **The role of the AdGuard event** beyond generic broadcast/multicast pressure (single observation, no packet capture).
5. **Any contribution of the RPI-RF-MOD's own behavior** (its UART framing, keepalive cadence, and the "solid blue" LED report may belong to the module, not the device RGB — I could not disambiguate from the issue text).
