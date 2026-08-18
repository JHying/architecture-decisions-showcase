# 0017 · Blue-Green 與人工閘門

> **這則展示**：全自動流程裡哪一步值得人工監控，以及這個"人工步驟"真正的代價。
> 選集索引見 [README](../README.md)。

---

## Context and Problem Statement

面對真實使用者、有一定併發量的環境，一次出錯的部署必須同時滿足兩件事：**在影響到所有人之前就能被發現**，以及**能立刻回退**。

## Decision Drivers

- **壞版本要在影響到所有人之前就被發現**——需要一個真實流量看不到的觀察窗
- **要能立刻回退**——不能是「重新部署舊版本」這種以分鐘計的動作
- **安全機制裝不裝，取決於這個環境失敗會波及誰**，不是「我們是不是應該一律嚴格」（見下方範圍段）

## Considered Options

| 選項                | 判定                                                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Rolling update    | 否決——最簡單、額外資源最少的模型，但一個「細微地壞掉」的版本仍然會在滾動過程中逐步吃到真實流量，**沒有一個隔離的觀察窗**                                                             |
| Blue-Green + 自動更版 | 否決——它確實在不放慢交付的前提下拿到了即時回退，但基本健康檢查（process 起來了嗎、有回應嗎、程式有問題嗎）**攔不住每一類退化**；有些故障只在真實或接近真實的流量形狀下才會顯現                              |
| Blue-Green + 人工更版 | **採用**（高風險環境）——新 pod 起在 preview 路由路徑上，對真實流量不可見，舊版本繼續服務全部流量；操作者對著 preview pod 觀察 dashboard 與健康訊號後明確晉升，mesh（Istio）的流量規則才翻到新版本 |

注意第二個選項的判定：**自動更版不是「比較差的選項」，它是在一個特定維度上不夠**——健康檢查能證明「它活著」，不能證明「它沒有把某類請求算錯」。這裡並不真的平手，但差異很明確：**多等的那幾分鐘是可回收的損失，晉升一個算錯結果的版本不是。**

## Decision Outcome

高風險環境採用 **Blue-Green + 人工更版閘門**：新 pod 起在 preview 路由路徑上、對真實流量不可見，舊版本繼續服務全部流量；操作者對著 preview pod 觀察 dashboard 與健康訊號後明確晉升，mesh（Istio）的流量規則才翻到新版本。

選擇的方向是**偏向失敗代價比較小的那一邊**：多等的那幾分鐘是可回收的損失，晉升一個算錯結果的版本不是。

## 範圍：這道閘門並沒有被無差別套用

低風險的開發者環境仍然用 rolling update，因為它們本來就在各自隔離的、以分支為單位的 namespace 裡，爆炸半徑已被限制住。

**人工安全機制是否需要，判準是這個環境失敗會波及誰，不是「我們是不是應該一律嚴格」。**

## Consequences

兩條都不好看，紀錄照實寫：

- **部署期間執行中的 pod 數大約翻倍**（舊版本與新版本並存），這是為了換到安全性而接受的資源成本
- **人工閘門為使用它的環境增加了部署延遲**——這是對「完全自動、無人值守的交付」的一次明確取捨。它的實際意義比字面更重：**這類重大交付必須要有人可以看 dashboard 時進行**；無人值守時，就不能執行。

## Confirmation

**自動回退觸發條件**（錯誤率、延遲、訊息佇列積壓、健康檢查失敗越過設定閾值）**可以在不等待人工動作的情況下撤銷更版。**

所以人工閘門是**更版前的保險，不是「已經晉升壞版本」的防線**。這兩件事須分開：

> 人負責「要不要往前」，自動化負責「已經往前了但情況不對」

同一套流量控制也支撐 canary（切一小部分流量、觀察、再加大）與流量鏡像，兩者都是同一個晉升機制上的步驟變化，不需要額外基礎設施。

**但流量鏡像有一條限制必須一起讀**（見 [`0047`](03-mesh-control-plane.md)）：鏡像是 fire-and-forget，代理層複製的是完整請求（含 POST/PUT body），**被鏡像的那一側會真的把寫入執行下去**。回應被丟棄不代表副作用被丟棄。它只在唯讀請求、帶去重鍵的冪等寫入、或目標接完全獨立的影子資料層時安全——不能把它完全當成「零風險的免費驗證手段」。

## 誠實補充

- **人工閘門的成本沒有量化。** 紀錄寫的是「為使用它的環境增加部署延遲」，沒有寫增加多少、多少次部署因為等不到人而延到隔天。「依賴人的可用性」是這個設計的真實成本。
- **「Blue-Green 讓壞版本更早被發現」沒有前後量測。** 它是結構性論證（觀察窗的有無），沒有攔截率統計。

---

## English summary

**Context** In an environment serving real users at meaningful concurrency, a bad deploy must satisfy two things at once: it must be detectable before it reaches everyone, and it must be instantly reversible.

**Drivers** A bad version must be detectable before it reaches everyone, which needs an observation window invisible to real traffic; rollback must be immediate rather than a redeploy measured in minutes; and whether the safety mechanism is installed at all is decided by who a failure reaches.

**Options** Rolling updates were rejected: simplest and cheapest in resources, but a subtly broken version still takes on real traffic progressively, with **no isolated observation window**. Blue-Green with automatic promotion was rejected — and **not as the inferior option**. It genuinely delivers instant rollback without slowing delivery; it falls short on one specific axis: basic health checks prove the process is up and answering, not that it computes some class of request correctly, and some regressions only surface under real or near-real traffic shape. Blue-Green with a **human promotion gate** was adopted for high-stakes environments: new pods start on a preview routing path, invisible to real traffic, while the old version continues serving everything; an operator watches dashboards and health signals against the preview pods and promotes explicitly, at which point the mesh (Istio) traffic rules flip.

**Outcome** Blue-Green with a human promotion gate, for high-stakes environments. The shape of that reasoning: the extra minutes of waiting are a recoverable loss, promoting a version that computes wrong answers is not.

**Scope** The gate is deliberately not applied uniformly. Low-stakes developer environments stay on rolling updates, because per-branch isolated namespaces already cap the blast radius. **Whether a safety mechanism is installed is decided by who a failure reaches, not by whether the team should be rigorous in general.**

**Consequences** Both are stated plainly. Running pods roughly double during a deploy, as the resource cost of the safety. And the gate adds deployment latency for the environments using it — which means more than the words suggest: **delivery speed is now bound to someone being free to look at a dashboard.** Late at night, on a holiday, or while the person is in a meeting, the path simply stops there. That is the real bill for keeping a human in the loop, and it cannot be written up as "a few extra minutes".

**Confirmation** Automated rollback triggers — error rate, latency, message-queue lag, health-check failures crossing configured thresholds — can revert a promotion **without waiting for a human**. So the gate insures *forward* promotion only; it is not the sole defence against a bad version already promoted. The split is deliberate: the human decides whether to go forward, automation handles having gone forward and being wrong.

The same traffic control also supports canary (shift a small slice of traffic, observe, then increase) and mirroring, both step variations on the same promotion mechanism, needing no additional infrastructure.

But mirroring carries one constraint that must be read with it (see [`0047`](03-mesh-control-plane.md)): it is fire-and-forget over a full copied request including bodies, so the mirrored side really executes the write — discarding the response does not discard the side effect. It is safe only for read-only requests, idempotent writes with a dedupe key, or a target on a fully independent shadow data layer — it cannot be treated as a fully risk-free, free validation method.

**Honest note** The gate's cost is described but never measured — no figure for the added latency, and no count of deploys that slipped to the next day waiting for a person. The earlier-detection claim is structural, with no interception statistics behind it.
