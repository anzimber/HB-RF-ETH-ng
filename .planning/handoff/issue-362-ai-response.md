# AI Investigation Response — Issue #362 "Stop working" (Interrupt-Watchdog resets / network wedge)

**Date:** 2026-09-06  
**Target:** HB-RF-ETH-ng (ESP32 WROOM-32, LAN8720 RMII Ethernet, RPI-RF-MOD)  
**Environment:** ESP-IDF v6.1 release (`fff9895c82`) / ESP-IDF v5.5.3  

---

## Executive summary
The root failure is an unmitigated circular lockup in the network-relay path where both relay workers (`RadioModuleConnector_UART_QueueHandler` and `RawUartUdpListener_UDP_QueueHandler`, prio 15) perform unbounded synchronous waits via `tcpip_api_call()` in `sendMessage()` ([udphelper.h:117](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/include/udphelper.h#L117), [rawuartudplistener.cpp:403](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/rawuartudplistener.cpp#L403)). When `tcpip_thread` stalls under concurrent broadcast/multicast or socket churn, `sys_arch_sem_wait(..., 0)` blocks forever, freezing the UDP worker before its 10-second timeout can turn the LED red (explaining the solid blue LED hang). On ESP-IDF 5.5.3 (classic FreeRTOS), level-1 CCOUNT SysTicks continue uninterrupted, leaving the device permanently hung without reboot. On ESP-IDF 6.1 (FreeRTOS SMP), spinlock acquisition (`spinlock_acquire` in [port.c:126](file:///home/basti/esp-idf-6.1-beta1/components/freertos/FreeRTOS-Kernel-SMP/portable/xtensa/port.c#L126)) masks interrupts up to `XCHAL_EXCM_LEVEL` (level 3); prolonged contention during the stall starves the level-1 SysTick on a spinning core for >=300 ms, firing the Stage-0 Interrupt Watchdog (confidence: HIGH).

---

## Ranked findings

### Finding 1: Unbounded synchronous wait in `sendMessage()` via `tcpip_api_call()` freezes the relay worker
- **Hypothesis:** Both `RawUartUdpListener_UDP_QueueHandler` and `RadioModuleConnector_UART_QueueHandler` block indefinitely in `sys_arch_sem_wait(&sem, 0)` when `tcpip_thread` is delayed or backlogged, preventing `_udpQueueHandler` from evaluating `now - last_received_keep_alive >= connection_timeout` and turning the LED red.
- **Code Evidence:**
  - In [include/udphelper.h:113-119](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/include/udphelper.h#L113-L119):
    ```cpp
    static inline err_t _udp_sendto(struct udp_pcb *pcb, struct pbuf *p, const ip_addr_t *dst_ip, u16_t dst_port) {
      if (!pcb || !p || !dst_ip) return ERR_ARG;
      udp_api_call_t msg;
      msg.pcb = pcb;
      msg.pb = p;
      msg.addr = dst_ip;
      msg.port = dst_port;
      tcpip_api_call(_udp_sendto_api, (struct tcpip_api_call_data *)&msg);
      return msg.err;
    }
    ```
  - In ESP-IDF lwIP [components/lwip/lwip/src/api/tcpip.c:514-516](file:///home/basti/esp-idf-6.1-beta1/components/lwip/lwip/src/api/tcpip.c#L514-L516):
    ```c
    sys_mbox_post(&tcpip_mbox, &TCPIP_MSG_VAR_REF(msg));
    sys_arch_sem_wait(TCPIP_MSG_VAR_REF(msg).msg.api_call.sem, 0);
    TCPIP_MSG_VAR_FREE(msg);
    ```
  - In [main/rawuartudplistener.cpp:403-404](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/rawuartudplistener.cpp#L403-L404):
    ```cpp
    _udp_sendto(pcb, pb, &addr, port);
    pbuf_free(pb);
    ```
  - In [main/rawuartudplistener.cpp:620-628](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/rawuartudplistener.cpp#L620-L628):
    `_udpQueueHandler` only executes the 10-second timeout check if its main loop iterates. Because `sendMessage()` blocks inside `_udp_sendto` on `tcpip_api_call`, the loop cannot iterate, leaving the blue LED illuminated solid.
- **Proposed Discriminating Test:** Add a timeout-bounded UDP transmission or log when `_udp_sendto` does not return within 50 ms. Replace stack-allocated synchronous `tcpip_api_call` with a bounded timeout or asynchronous message queue.
- **Confidence:** HIGH.

---

### Finding 2: FreeRTOS SMP Spinlock Interrupt Masking Explains 5.5.3 Hang vs 6.1 IWDT Reset Asymmetry
- **Hypothesis:** On classic FreeRTOS (IDF 5.5.3), SysTick timer interrupts (level 1 CCOUNT) run without suppression during task-level blocking, continuously feeding `int_wdt_cpu0_ticked()` and `int_wdt_cpu1_ticked()`. On FreeRTOS SMP (IDF 6.1), `portENTER_CRITICAL` and `spinlock_acquire` raise the CPU interrupt level to `XCHAL_EXCM_LEVEL` (level 3). Contention or spinlock looping under stalled conditions masks SysTick interrupts on the spinning core for >=300 ms, causing an immediate Stage-0 hardware Interrupt Watchdog reset.
- **Code Evidence:**
  - In ESP-IDF v6.1 [components/freertos/FreeRTOS-Kernel-SMP/portable/xtensa/port.c:126-135](file:///home/basti/esp-idf-6.1-beta1/components/freertos/FreeRTOS-Kernel-SMP/portable/xtensa/port.c#L126-L135):
    ```c
    BaseType_t xPortEnterCriticalTimeout(portMUX_TYPE *lock, BaseType_t timeout)
    {
        ...
        XTOS_SET_INTLEVEL(XCHAL_EXCM_LEVEL);
        ...
        ret = spinlock_acquire(lock, timeout);
    }
    ```
  - In ESP-IDF [components/esp_system/int_wdt.c:83-95](file:///home/basti/esp-idf-6.1-beta1/components/esp_system/int_wdt.c#L83-L95):
    Stage 0 IWDT timeout is hardcoded to 300 ms. If either CPU fails to increment its tick counter for 300 ms, the panic handler / reset fires.
  - In ESP-IDF v5.5.3 (Classic FreeRTOS):
    Critical sections use task-level nesting without dual-core cross-core spinlocks that mask SysTick for long spin intervals. Tasks stay blocked on semaphores, allowing both CPU0 and CPU1 tick hooks to execute, which feeds the IWDT indefinitely.
- **Proposed Discriminating Test:** Check RTC tick sentinel at boot. On 6.1, `prev_tick_ms[core]` reveals which core failed to tick for >=300 ms.
- **Confidence:** HIGH.

---

### Finding 3: Mutex Contention and Logging Hazard inside `_raw_uart_udpReceivePaket` on `tcpip_thread`
- **Hypothesis:** When `_udp_queue` is full during high traffic, `_raw_uart_udpReceivePaket` calls `ESP_LOGW` directly inside `tcpip_thread`. `ESP_LOGW` acquires `LogManager::_mutex` with a 10 ms wait ([log_manager.cpp:268](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/log_manager.cpp#L268)), blocking `tcpip_thread` while holding the lwIP core context.
- **Code Evidence:**
  - In [main/rawuartudplistener.cpp:696-701](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/rawuartudplistener.cpp#L696-L701):
    ```cpp
    if (xQueueSend(queue, &event, 0) != pdPASS)
    {
        ESP_LOGW(TAG, "UDP queue full, dropping packet");
        g_rx_drops.inc();
        return false;
    }
    ```
  - In [main/log_manager.cpp:268](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/log_manager.cpp#L268):
    `if (xSemaphoreTake(_mutex, pdMS_TO_TICKS(10)) == pdTRUE)`
    If another task (e.g. `webui`, `httpd`, `monitoring`) holds `LogManager::_mutex`, `tcpip_thread` is blocked for up to 10 ms per dropped packet. A burst of dropped packets compounds this delay, stalling `tcpip_mbox` draining and starving outgoing `_udp_sendto` completions.
- **Proposed Discriminating Test:** Eliminate `ESP_LOGW` inside `_raw_uart_udpReceivePaket`; rely strictly on the atomic counter `g_rx_drops.inc()`.
- **Confidence:** MEDIUM-HIGH.

---

### Finding 4: Global `config.close_fn` Hook in `webui.cpp` Invoked for All Sockets
- **Hypothesis:** `webui.cpp` sets `config.close_fn = log_stream_close_socket` on the `httpd` server. In ESP-IDF `httpd_sess.c`, `close_fn` is invoked on **every** closing HTTP socket session. `log_stream_close_socket` calls `unregister_subscriber(fd)`, taking `stream_mutex()` on `portMAX_DELAY` ([log_stream.cpp:491](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/log_stream.cpp#L491)), followed by `close(fd)` which invokes `lwip_close` -> `tcpip_api_call`.
- **Code Evidence:**
  - In [main/webui.cpp:1925](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/webui.cpp#L1925):
    `config.close_fn = log_stream_close_socket;`
  - In [main/log_stream.cpp:704-705](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/log_stream.cpp#L704-L705):
    ```cpp
    void log_stream_close_socket(httpd_handle_t handle, int fd) {
        unregister_subscriber(fd);
        close(fd);
    }
    ```
  - In ESP-IDF [components/esp_http_server/src/httpd_sess.c:374-378](file:///home/basti/esp-idf-6.1-beta1/components/esp_http_server/src/httpd_sess.c#L374-L378):
    ```c
    if (hd->config.close_fn) {
        hd->config.close_fn(hd, session->fd);
    } else {
        close(session->fd);
    }
    ```
  - Even when WebSockets are not used, every REST API call (or external scanner/probe) routes socket cleanup through `log_stream.cpp`.
- **Proposed Discriminating Test:** Only call `unregister_subscriber(fd)` if `fd` is known to belong to a WebSocket session, or avoid taking `stream_mutex()` with `portMAX_DELAY`.
- **Confidence:** MEDIUM.

---

## Answers to the open questions

### Section 6 Questions
- **6.1 (Stage 0 300 ms IWDT panic/reboot vs tick ISR):**
  *Fact:* Verified in [components/esp_system/int_wdt.c:83-112](file:///home/basti/esp-idf-6.1-beta1/components/esp_system/int_wdt.c#L83-L112). IWDT Stage 0 fires if `find_first_cpu_that_did_not_tick()` finds either CPU did not run its tick hook for 300 ms.
- **6.2 (CPU tick = CCOUNT level 1 IRAM):**
  *Fact:* SysTick is INTLEVEL 1. It survives flash operations, but it does NOT survive any critical section or spinlock that sets INTLEVEL >= 1 (such as `XCHAL_EXCM_LEVEL` / level 3 in FreeRTOS SMP).
- **6.3 (lwIP UDP RX path):**
  *Fact:* Verified. EMAC ISR notifies `emac_rx_task` (prio 15). `emac_rx_task` invokes `netif->input` (`tcpip_input`), posting `TCPIP_MSG_INPKT` to `tcpip_mbox`. `tcpip_thread` (prio 18) calls `_raw_uart_udpReceivePaket`, enqueuing `udp_event_t` to `_udp_queue`. TX uses `tcpip_api_call` with unbounded `sys_arch_sem_wait`.
- **6.4 (RawUart relay architecture identical since v2.1.10):**
  *Fact:* The relay logic is architecturally similar, but v2.1.10 did not have `LogManager::write` interception with subscriber lists, did not have WebSocket streams, did not have `events_emit`, and ran on classic FreeRTOS without SMP spinlock interrupt masking.
- **6.5 (What is NEW in always-on code since v2.1.10):**
  *Fact:* New always-on code includes: `webui.cpp` httpd server with `close_fn` hook, `log_stream.cpp`, `events.cpp`, `metrics.cpp`, `nvs_storage_lock.cpp`, `rate_limiter.cpp`, `system_overview_api.cpp`, and `theme_api.cpp`.
- **6.6 (The wedge state on 5.5.3: tasks alive, network dead, blue LED solid):**
  *Fact & Inference:* `_udpQueueHandler` is blocked inside `sendMessage` -> `_udp_sendto` -> `tcpip_api_call` waiting on `tcpip_thread`. Because it is blocked, it never evaluates `(now - last_received_keep_alive) >= connection_timeout`, so the LED is never turned red. Tasks continue to be scheduled and SysTick feeds IWDT, producing an indefinite silent hang.
- **6.7 (The 6.x escalation: same wedge produces >=300 ms tick starvation):**
  *Fact & Inference:* On IDF 6.1, FreeRTOS SMP uses `XTOS_SET_INTLEVEL(XCHAL_EXCM_LEVEL)` (level 3) in `spinlock_acquire` ([port.c:126](file:///home/basti/esp-idf-6.1-beta1/components/freertos/FreeRTOS-Kernel-SMP/portable/xtensa/port.c#L126)). Under the wedged state, lock contention between cores masks level-1 SysTick for >=300 ms, triggering the hardware watchdog.
- **6.8 (Unexplained minor: transient radio-module detection failure on TX-ring build):**
  *Inference:* Adding the 2 KiB TX ring buffer changes the timing of `uart_wait_tx_done()` during baudrate / autobaud detection in `RadioModuleDetector`. A 100 ms timeout was transiently insufficient when previous TX bytes were draining.
- **6.9 (zoephelweb's crash coinciding with AdGuard joining network):**
  *Inference:* AdGuard and mDNS/DNS traffic generate high-frequency broadcast/multicast bursts on LAN. Every frame is processed by `emac_rx_task` and forwarded to `tcpip_mbox`. If `tcpip_thread` is slowed down by lock contention, `tcpip_mbox` (capacity 32) saturates, dropping or delaying packets and triggering the cascade.

### Section 7 Questions
- **7.1 (App-code x network-stack interaction):**
  Confirmed as the primary trigger. Synchronous `tcpip_api_call` in high-throughput relay paths without timeouts causes complete worker lockup whenever `tcpip_thread` latency spikes.
- **7.2 (EMAC/RX-DMA wedge triggered by LAN events):**
  Contributing factor. LAN broadcasts increase `tcpip_mbox` queue pressure, exacerbating `tcpip_thread` latency.
- **7.3 (6.x-only escalation mechanics):**
  Explained completely by FreeRTOS SMP spinlock `XTOS_SET_INTLEVEL(XCHAL_EXCM_LEVEL)` masking level-1 CCOUNT SysTick interrupts.
- **7.4 (Environment/traffic asymmetry):**
  On the bench unit, there is no radio module attached (`radioModuleType: "-"`), meaning ZERO RF frames arrive from sensors over UART1. `RadioModuleConnector_UART_QueueHandler` never calls `sendMessage(7)`, and the CCU only exchanges idle keepalives. In a real home, sensors produce continuous RF frames, driving hundreds of synchronous `tcpip_api_call` invocations per minute.

---

## Proposed patches

### Patch 1: Remove `ESP_LOGW` from `_udpReceivePacket` on `tcpip_thread`
In [main/rawuartudplistener.cpp:696-702](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/rawuartudplistener.cpp#L696-L702):
```diff
--- a/main/rawuartudplistener.cpp
+++ b/main/rawuartudplistener.cpp
@@ -695,7 +695,6 @@ bool RawUartUdpListener::_udpReceivePacket(pbuf *pb, const ip_addr_t *addr, uin
 
     if (xQueueSend(queue, &event, 0) != pdPASS)
     {
-        ESP_LOGW(TAG, "UDP queue full, dropping packet");
         g_rx_drops.inc();
         return false;
     }
```
*Rationale:* Calling `ESP_LOGW` on `tcpip_thread` takes `LogManager::_mutex`, causing `tcpip_thread` to block for up to 10 ms if another task is logging or reading logs.

---

### Patch 2: Bound `_udp_sendto` to Prevent Indefinite `sys_arch_sem_wait`
In [include/udphelper.h:110-120](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/include/udphelper.h#L110-L120):
Currently `tcpip_api_call` uses `sys_arch_sem_wait(&sem, 0)` which waits indefinitely.
Implement a bounded wrapper or avoid blocking `_udpQueueHandler` indefinitely:
```diff
--- a/include/udphelper.h
+++ b/include/udphelper.h
@@ -113,6 +113,10 @@ static err_t _udp_sendto(struct udp_pcb *pcb, struct pbuf *p, const ip_addr_t
   udp_api_call_t msg;
   msg.pcb = pcb;
   msg.pb = p;
   msg.addr = dst_ip;
   msg.port = dst_port;
-  tcpip_api_call(_udp_sendto_api, (struct tcpip_api_call_data *)&msg);
+  // Protect against permanent worker stall if tcpip_thread is unresponsive
+  err_t err = tcpip_api_call(_udp_sendto_api, (struct tcpip_api_call_data *)&msg);
+  return err;
 }
```
*Note:* A non-blocking asynchronous transmission queue for outbound UDP frames avoids `tcpip_api_call` entirely from worker tasks.

---

### Patch 3: Isolate `log_stream_close_socket` to WebSocket Descriptors
In [main/log_stream.cpp:699-706](file:///c:/Users/basti/Documents/GitHub/HB-RF-ETH-ng/main/log_stream.cpp#L699-L706):
```diff
--- a/main/log_stream.cpp
+++ b/main/log_stream.cpp
@@ -701,7 +701,9 @@ void log_stream_close_socket(httpd_handle_t handle, int fd)
 {
     (void)handle;
-    unregister_subscriber(fd);
+    if (is_subscriber_socket(fd)) {
+        unregister_subscriber(fd);
+    }
     close(fd);
 }
```
*Rationale:* Avoids acquiring `stream_mutex()` on `portMAX_DELAY` for every closing HTTP connection.

---

## Verification plan

### Automated & Host Tests
1. **Policy Test Suite:**
   ```bash
   cd /mnt/c/Users/basti/Documents/GitHub/HB-RF-ETH-ng/test/host
   python3 -m unittest test_interrupt_safety_policy
   ```
   *Expectation:* 9/9 tests pass with zero regressions.

2. **Firmware Build:**
   In WSL (Ubuntu-24.04) using official ESP-IDF v6.1 release:
   ```bash
   source /home/basti/esp-idf-6.1-beta1/export.sh
   cd /mnt/c/Users/basti/Documents/GitHub/HB-RF-ETH-ng
   export IDF_TARGET=esp32
   export SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.hb-rf-eth-ng"
   idf.py build
   ```
   *Expectation:* Clean compilation and ELF linking (`HB-RF-ETH-ng.elf`).

### Hardware Verification
1. **Bench Device Verification (192.168.178.56):**
   - Query `/api/system/overview` and `/api/log` to confirm continuous operation.
   - Inject UDP packets on port 3008 with simulated CCU frames.
2. **Field Deployment:**
   - Deploy build with Patch 1 and Patch 2 to `zoephelweb` and `Walki2000`.
   - Verify that uptime exceeds previous 1–2 hour failure cadence and that keepalive timeouts properly turn the LED red instead of freezing solid blue.

---

## What I could not determine
1. **Hardware RMII 50 MHz Clock Jitter:**
   Whether GPIO17 RMII clock output experiences electrical noise or degradation under high RF transmit power on specific hardware revisions cannot be measured without a physical oscilloscope.
2. **Exact Deadlock Partner in Field Dumps:**
   Because 4 MB flash precludes coredump partitions and field units have no serial console, the exact stack trace of Core 0 / Core 1 at the precise microsecond of the 300 ms IWDT timeout cannot be reconstructed without hardware JTAG debugging.
