# 0002 · 內嵌容器選型

> **這則展示**：區分 **「代價太高」與「結構上做不到」**；也是這份選集裡唯一有客觀通過門檻的 Confirmation。
> 選集索引見 [README](../README.md)。

---

## Context and Problem Statement

面向連線的服務要握住大量長生命週期的 WebSocket 連線，每條連線持續送心跳／ACK，且 ACK 有一個明確的毫秒級時間預算。

## Decision Drivers

在這個前提下，thread-per-request 模型**不是「比較慢」，是給出一道硬上限**：連線數被綁在執行緒數上。

這個區分決定了後面怎麼談：**「太慢」可以靠加機器談判，「結構上到不了」不行。**

## Considered Options

| 選項 | 判定 |
| --- | --- |
| Tomcat（既有預設容器）+ 虛擬執行緒 | 否決——最熟悉、文件最齊，虛擬執行緒也確實緩解了一部分壓力；但在測試中持續高訊息率下**開始掉 frame**，突發負載下迅速崩潰，目標量級到不了 |
| Netty／reactive stack | 否決——它為大規模併發而生，但**強制 reactive 程式模型並放棄 `jakarta.websocket`**，改寫與學習成本對這個 workload 不划算，因此直接排除在對打測試之外。**排除的理由本身被寫下來，而不是省略這個選項** |
| Undertow（非阻塞 I/O）+ 虛擬執行緒 | **採用**——非阻塞 I/O 讓連線數與執行緒數脫鉤，同時保留團隊已經熟悉的 `jakarta.websocket` 模型 |

## Decision Outcome

採用 **Undertow（非阻塞 I/O）+ 虛擬執行緒**。

非阻塞 I/O 讓連線數與執行緒數脫鉤——移除的是那道硬上限本身，不是把上限往上推一格；同時保留團隊已經熟悉的 `jakarta.websocket` 模型，不必為此換掉程式模型。

範圍刻意限定在**面向連線的服務**，不推廣到一般 request/response 服務（見下節）。

## Consequences

- **Undertow 社群比 Tomcat 小，深度除錯可能得直接讀原始碼。** 緩解方式也是一種判斷習慣：**保留退回 Tomcat 的路（embedded JAR 或 WAR），讓這個選擇是可逆的**
- 範圍限制（Neutral）：**只標準化在「面向連線的服務」上**，一般 request/response 服務可以留在預設容器——選型沒有被無差別推廣到整個平台

## Confirmation

面向連線的服務上線前，必須有**重現心跳／ACK 模式的負載測試**，在約定的每副本連線數與訊息率下**零掉幀**。

這是整份選集裡**唯一有客觀通過條件**的 Confirmation。

## 誠實補充

- **基準測試是 directional 的，原文即如此標註。** 它跑在開發機等級硬體、有明確的並行上限，超過該上限之後 I/O 執行緒飽和、比較會失真。它支撐的是「這個容器在這個模式下比預設容器撐得住」，**不是任何生產容量保證**。
- **沒有端到端的合併驗證。** 這一則論證的是「連線這一層的天花板被移除」，但「容器＋執行緒模型＋序列化＋外部化連線狀態疊起來之後，端到端延遲是多少」不在任何一份紀錄裡。

---

## English summary

**Context** A connection-facing service holds a large number of long-lived WebSocket connections, each sending continuous heartbeats and ACKs against an explicit millisecond-level budget, at C10K scale and above.

**Drivers** Under those conditions thread-per-request is not *slower* — it is a hard ceiling, because connection count is bound to thread count. The distinction sets the terms of the whole discussion: **"too expensive" can be negotiated with more hardware; "structurally unreachable" cannot.**

**Options** Tomcat with virtual threads was rejected: virtual threads did relieve part of the pressure, but under sustained high message rates it began dropping frames and collapsed quickly under bursts, so the target scale was out of reach. Netty was excluded without a head-to-head test — it is built for exactly this concurrency, but it forces a reactive programming model and gives up `jakarta.websocket`, a rewrite and learning cost this workload does not justify; **the reason for excluding it is written down rather than the option being silently omitted**. Undertow with virtual threads was **adopted**: connection count decouples from thread count while the team keeps the `jakarta.websocket` model it already knows.

**Outcome** Undertow with virtual threads, standardised for connection-facing services only — the ceiling is removed rather than raised a notch, and `jakarta.websocket` stays.

**Consequences** Undertow has a smaller community than Tomcat, so deep debugging may mean reading source. The mitigation is itself a habit: keep the path back to Tomcat open, so the choice stays reversible. Scope is also limited deliberately — it is standardised only for connection-facing services; ordinary request/response services stay on the default, so the selection is not generalised across the platform.

**Confirmation** Before a connection-facing service ships, a load test reproducing the heartbeat/ACK pattern must show **zero dropped frames** at the agreed per-replica connection count and message rate. This is the only objective pass condition in the set.

**Honest note** The benchmark is directional, as the original states: developer-grade hardware with an explicit concurrency cap beyond which I/O threads saturate and the comparison distorts. It supports "Undertow holds up better than Tomcat under this pattern", not any production capacity guarantee. Nor is there an end-to-end measurement combining this layer with the others.
