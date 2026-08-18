# 0019 / 0024 · 併發模型

> **這則展示**：同一條判準套在兩種 workload 上，**得到相反的答案**。
> 選集索引見 [README](../README.md)。

---

## Context and Problem Statement

一則否決 reactive（`0019`），另一則採用 reactive（`0024`）。這看起來矛盾，實際上兩份紀錄處理的是**不同的組件**，而且互相引用、明講這是刻意的分界。

**分界線是：這個組件有沒有領域模型。**

## Decision Drivers

- **判準只有一條：付出非同步複雜度，換到的東西值不值得。** 兩則用的是同一條，答案相反是因為 workload 不同，不是判準不同
- **這個組件有沒有領域模型**——有分層領域模型時，reactive 改寫是架構性的：分層呼叫堆疊上每一個做 I/O 的方法都要改回傳型別與組合方式；沒有領域模型時，這筆成本不存在
- **底層有沒有一流的非阻塞 driver**——沒有的話，reactive 改寫只是加了語法，底下的阻塞呼叫並沒有被移除

## Considered Options

兩則各自的候選比較並列在下，判準相同、判定相反。

### `0019` 業務服務——否決 reactive

業務服務建立在分層的 imperative 領域模型上。

| 選項 | 判定 |
| --- | --- |
| 續用平台執行緒，靠加執行緒擴容 | 否決——每條平台執行緒帶固定大小的堆疊，光是撐住閒置連線就需要數 GB 堆疊記憶體，加上 OS 排程器負擔，在目標連線數之前就先撞牆 |
| 把 I/O 路徑改寫成 reactive | 否決——改寫是**架構性的，不是語法性的**：分層呼叫堆疊上**每一個做 I/O 的方法都要改回傳型別與組合方式**，這會使建立在該分層模型方法簽章之上的結構性規則失效；而且所使用的關聯式持久層**沒有一流的 reactive driver**，改寫後仍要包一層阻塞呼叫的橋接——**等於加了 reactive 語法卻沒有移除底下的阻塞呼叫** |
| 採用虛擬執行緒 | **採用**——虛擬執行緒在等 I/O 時是 park 而不是 block，把底層 carrier thread 還給一個約略等於 CPU 核心數的小池；既有程式碼不必改程式模型，只換執行它的 executor |

**一個容易搞混的釐清**：內嵌容器（Undertow）自己的 I/O event-loop 執行緒不會、也不該被換成虛擬執行緒——那個迴圈本來就非阻塞、設計上永不 park。虛擬執行緒是在上面一層引入的：frame 從 event loop 交給應用程式碼之後，那一段跑在虛擬執行緒上，在那裡阻塞 I/O 是預期且安全的。**兩種執行緒模型在不同層互補。**

### `0024` 邊緣 gateway——採用 reactive

它的 workload 形狀完全不同：**純 I/O proxy**，沒有領域模型、沒有交易性持久層、沒有要保護的分層結構。filter chain 上的每一步都是「查一次快取，然後轉發」——正好是 reactive 組合擅長的非阻塞 I/O 模式。而且該 reactive 框架的 filter-chain 模型**本來就沒有 blocking 版本可選**。

## Decision Outcome

**同一個判準（付出非同步複雜度換到的東西值不值得）套在兩種 workload 上，得到相反的答案。**

`0024` 把這件事寫成 Neutral 而不是 Bad——平台上恰好有一個組件使用不同的程式模型，這是**一個範圍明確的例外，不是逐漸擴散的不一致**，而讓它成為例外的正是它的範圍（純 I/O proxy、無領域邏輯）。

## Consequences

- **虛擬執行緒不會自動繼承產生它的執行緒的 tracing context**，每個虛擬執行緒 executor 都要顯式包裝
- **CPU-bound 工作被刻意排除在虛擬執行緒之外**，用另一個有界的平台執行緒池——虛擬執行緒解的是「等 I/O」，不是「CPU 工作太多、核心太少」，混在同一個池會有餓死 carrier thread 的風險

## Confirmation

- 新的 I/O-bound 併發路徑預設使用虛擬執行緒 executor；真正 CPU-bound 的路徑使用另一個有界平台執行緒池，並**明確標示**
- 新的邊緣層關注點（認證、限流、路由）實作在 gateway filter chain；新的業務能力實作在業務服務——兩者不是可互換的落點

兩條都**沒有工具，靠 review**。而 CPU-bound 那條是**最容易靜默出錯的一條**：把 CPU-bound 工作誤放到虛擬執行緒上不會報錯，只會在負載上來時餓死 carrier thread，而分類判斷完全依賴寫的人自己認定。

## 誠實補充

- **`0024` 的 gateway 設計細節是結構性論證。** 例如「每請求查共享快取的延遲成本乘上全部流量」是推論，沒有附上實測的每請求節省。
- **「虛擬執行緒不繼承 tracing context」這條代價不是事後發現的意外**，是在採用決策裡就寫下來的——後來才變成一個共用函式庫（見 [相關作品](../README.md)）。

---

## English summary

**Context** One record rejects reactive, the other adopts it. They look contradictory and are not: they concern different components, cross-reference each other, and state the boundary explicitly. **The dividing line is whether the component has a domain model.**

**Drivers** One criterion only — is the asynchronous complexity worth what it buys — applied to both. What moves the answer is whether the component has a domain model (with one, a reactive rewrite is architectural: every I/O-performing method in a layered call stack changes signature) and whether the layer underneath has a first-class non-blocking driver (without one, reactive syntax is added while the blocking calls stay).

**`0019`, business services** Built on a layered imperative domain model. Adding platform threads was rejected: each carries a fixed-size stack, so gigabytes are consumed just holding idle connections, and the OS scheduler load hits a wall well before the target connection count. Rewriting the I/O path as reactive was rejected as **architectural rather than syntactic** — every I/O-performing method in a layered call stack changes its return type and composition, invalidating the structural rules built on those signatures, and the relational persistence layer has no first-class reactive driver, so blocking calls would still need bridging: reactive syntax added without the blocking calls underneath being removed. Virtual threads were **adopted**: they park rather than block while waiting on I/O, returning the carrier thread to a small pool roughly the size of the core count, and existing code changes executor rather than programming model.

One clarification worth keeping: Undertow's own non-blocking I/O event loop is deliberately *not* converted — that loop never parks by design. Virtual threads are introduced one layer up, where application code is expected to block, and the two thread models complement each other at different layers.

**`0024`, the edge gateway** A pure I/O proxy: no domain model, no transactional persistence, no layered structure to protect. Every step in its filter chain is "check a fast cache, then forward", exactly what reactive composition is for, and its filter-chain model has no blocking-mode equivalent. **Reactive adopted.**

**Outcome** The same criterion — is the asynchronous complexity worth what it buys — produces opposite answers, because the workloads genuinely differ. The second record files this as Neutral rather than Bad: exactly one component uses a different programming model, and that is a **scoped exception rather than creeping inconsistency**, precisely because of how narrow the scope is.

**Consequences** Virtual threads do not inherit the tracing context of the thread that spawned them, so every executor must be wrapped explicitly. CPU-bound work is deliberately kept off them, on a separate bounded platform pool: virtual threads solve waiting on I/O, not too much CPU work for too few cores, and mixing the two risks starving carrier threads.

**Confirmation** New I/O-bound paths default to a virtual-thread executor while genuinely CPU-bound paths use a separate bounded pool and are explicitly marked; new edge concerns land in the gateway filter chain and new business capabilities in business services, the two not being interchangeable. Both are review-only — and the CPU-bound classification is the most silently-failing rule here: misclassifying raises no error, it starves carrier threads under load, and the judgement rests entirely on the author.

**Honest note** The gateway design details are structural arguments with no measured per-request saving. The tracing-context cost was written down at decision time rather than discovered later, and subsequently became a shared library.
