---
title: A new comparsion about esp32c6 and linkit7697 on Linux as coprocessor
date: 2026-08-26T13:05:09.581Z
---

Test by LAN iperf3 server:

| Test | ESP32-C6 | LinkIt 7697 |
|---|---:|---:|
| TCP upload, MTU 576 | 2.16 Mbit/s | 0.01 Mbit/s |
| TCP download, MTU 576 | 0.25 Mbit/s | 0.05 Mbit/s |
| TCP upload, MTU 1500 | 2.31 Mbit/s | 0.02 Mbit/s |
| Sustainable UDP upload, MTU 576 | 2.00 Mbit/s | 0.02 Mbit/s |
| Sustainable UDP download, MTU 576 | 1.00 Mbit/s | Below threshold |
| Idle RTT, 56-byte packets | 2.87 ms | 37.36 ms |
| Idle RTT, 512-byte packets | 6.21 ms | 10,026.57 ms |

**Conclusion:** ESP32-C6 provides substantially higher throughput, lower latency, and better stability than LinkIt 7697.