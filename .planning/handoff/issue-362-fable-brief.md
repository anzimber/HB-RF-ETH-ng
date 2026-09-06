# Issue #362 — compact audit brief (token-efficient mission: READ-ONLY code bug hunt)

Context in one paragraph: ESP32 firmware (this repo, ESP-IDF 6.1 + lwIP 2.2.0). Field failure: the lwIP `tcpip_thread` spontaneously stops progressing (device dead to ICMP, scheduler alive). Once it stalls, both relay tasks (UART task `main/radiomoduleconnector.cpp::_serialQueueHandler`, worker `main/rawuartudplistener.cpp:_udpQueueHandler`) freeze inside unbounded `tcpip_api_call` (`include/udphelper.h`) — that amplification is already understood; do NOT re-derive it. On IDF 5.5.3 the result is a silent hang; on 6.1 it additionally becomes an interrupt-watchdog reset, which requires some core at INTLEVEL ≥1 (e.g. spinning on a spinlock) for ≥300 ms — the concrete holder is unknown. Historical stability of the old app code (v2.1.10, tag in git) on the same hardware implies the defect entered with the 2.2.x application rewrite.

**Your mission — exactly two questions:**
1. **The initiator:** find the concrete defect (wild write, lifetime bug, overflow, non-returning handler) in the code paths that interact with lwIP structures, that can stop `tcpip_thread` or corrupt its state. Audit these suspects line by line (others only if implicated):
   - `main/rawuartudplistener.cpp` — `handlePacket()` (heap fallback + `pbuf_copy_partial` + CRC over pbuf data), `_udpReceivePacket`/queue lifecycle, `start()`/`stop()`/`_udpQueueHandler` teardown (pbuf ownership across `_stopRequested`/`_activeSenders` transitions, queue delete vs in-flight events).
   - `main/ntpserver.cpp` — same UDP-queue pattern (`_udpReceivePacket`, stop path).
   - `main/log_manager.cpp` — ring buffer arithmetic (`total_written`, `ring_start_offset`, wrap), subscriber snapshot logic.
   - `main/webui_storage.cpp` / `main/settings.cpp` — only insofar as memory writes could escape their buffers.
   - `main/ethernet.cpp` — DNS cache (hostname bounds, expiry math).
   - Compare against tag `v2.1.10` (`git show v2.1.10:src/<file>`) wherever behavior differs — the bug is in the delta.
2. **The 6.1 escalation:** identify a plausible ≥300 ms INTLEVEL holder: any code path (app or the IDF dirs `components/lwip`, `components/esp_netif`, `components/esp_eth`, `components/freertos` — no full IDF checkout needed, reason from the APIs used) where a spinlock/portMUX is held **across** a potentially blocking call (mutex/semaphore/mbox/`tcpip_api_call`) in paths this firmware exercises.

**Closed — do not revisit:** heap exhaustion, ECO3 silicon livelock (workaround active; its stage-0 "20 ms" is feed granularity, not the panic threshold), MQTT/syslog/logging features, flash-write cadence, flash guard, `ETH_IRAM_OPTIMIZATION`, lwIP/IDF version deltas, PM/tickless, app critical sections, DCF ISR, UART TX concurrency, EMAC TX blocking, LAN-flood-alone, `ccuIP`/security resets, SPIFFS mount noise, I2C/RTC, pbuf double-free in `sendMessage` (contract-verified), any app mutex blocking tcpip (impossible — they block only their caller), EMAC-RX-only death (contradicted: no "CCU timed out" line ever logged during hangs).

**Answer format (keep it terse — this brief is for an expensive model):**
Write `.planning/handoff/issue-362-fable-response.md`:
1. `## Findings` — numbered; each: what/where (`file:line`, quote ≤3 lines), why it can stop or corrupt tcpip (or hold INTLEVEL), trigger conditions, severity.
2. `## Best candidate` — your single most likely initiator, one paragraph.
3. `## Not found` — one line per checked-and-clean suspect.
No patches, no build/test plans, no summaries of this brief. Facts only; mark inference as such.
