[English](#english) | [繁體中文](#繁體中文)

---

## 繁體中文

# Architecture Decisions in Practice

> 八則代表作，記錄我在一個高併發平台上**怎麼把問題框起來、比較了什麼、為什麼否決其他選項、以及付出了什麼代價**，不只是最後做了哪個決策。

一份只寫「我們用 Kafka」的紀錄沒有價值：讀的人無從了解背後的原因，並且在未來知道該去避免什麼問題。所以這裡每一則都必須回答四件事——**當時的約束是什麼、評估過哪些選項、為什麼否決它們、這個選擇的代價是什麼**——外加一條大多數決策紀錄會漏掉的：**怎麼確認這個決策真的被遵守**。

### 從哪裡開始

| 你有       | 看這個                                                                             |
| -------- | ------------------------------------------------------------------------------- |
| **30 秒** | 下方的 [八則代表作摘要](#八則代表作) 與 [我怎麼做決策](#我怎麼做決策)                                       |
| **5 分鐘** | [`0048` 六軸評估框架](docs/01-six-axis-evaluation-framework.md) — 最能看出一條判準怎麼從單軸長成六軸框架 |

---

### 八則代表作

挑選標準不是「最重要的架構」，而是**每一則各展示一種不同的推理方式**：

| #                                                  | 決策                | 這則展示什麼                                      |
| -------------------------------------------------- | ----------------- | ------------------------------------------- |
| [`0048`](docs/01-six-axis-evaluation-framework.md) | 六軸評估框架            | 把反覆爭論的問題（服務如何拆分）固定成一條判準                     |
| [`0010`](docs/02-boundary-principles.md)           | 六條邊界原則            | 六條原則**彼此獨立**：任一條成立就足以支撐一條邊界，也不會被只談另一條的論證削弱  |
| [`0047`](docs/03-mesh-control-plane.md)            | 骨幹該不該兼任 mesh 控制平面 | **推翻自己原本寫下的理由**：每個選項都滿足的需求，不能拿來決定選哪一個      |
| [`0013`](docs/04-secrets-transit-pattern.md)       | 密鑰 transit 模式     | 兩個被否決的選項壞在**三個不同的失效面**：供應鏈曝光、執行期記憶體曝光、稽核粒度不足 |
| [`0046`](docs/05-cross-region-propagation.md)      | 跨區狀態傳播            | 決定選項的是**失敗的代價有多大**，不是「保證不會出錯」               |
| [`0002`](docs/06-embedded-container.md)            | 內嵌容器選型            | 區分**「代價太高」與「結構上做不到」**；也是唯一附有客觀通過門檻的一則       |
| [`0019` / `0024`](docs/07-concurrency-model.md)    | 併發模型              | 同一條判準套在兩種 workload 上，**得出相反的答案**            |
| [`0017`](docs/08-human-promotion-gate.md)          | Blue-Green 與人工閘門  | 全自動流程裡哪一步值得人工監控，以及這個"人工步驟"真正的代價             |

---

### 我怎麼做決策

這八則底下是同一套習慣。格式用 [MADR](https://adr.github.io/madr/)，而中間那欄是這張表的重點：**每條習慣都由格式裡一個固定欄位撐住**，不是靠自律記得去做。

| 做法                  | 靠哪個 MADR 欄位撐住                   | 這個欄位要求什麼                                                 |
| ------------------- | ------------------------------- | -------------------------------------------------------- |
| **先找出真正的決策關鍵**      | `Context and Problem Statement` | 界定正在被決定的是哪一個問題，以及當時生效的約束。問題本身沒定義清楚之前，選項比較沒有共同基準          |
| **必要條件不等於決定性條件**    | `Decision Drivers`              | 列出用來分辨選項的因素。每個選項都滿足的條件是門檻，不是理由——它只能排除選項，不能在剩下的選項之間做決定    |
| **區分「代價太高」與「做不到」**  | `Decision Drivers`              | 驅動因素要標明性質：成本型的可以用資源談判，也會被反覆重談；結構型的是硬上限，花多少錢都不會移動         |
| **被否決的選項要寫出否決理由**   | `Considered Options`            | 並列所有被認真考慮過的選項，每一個都附判定與理由。只留下最終方案，讀者無從判斷這是想過的結果還是碰巧的結果    |
| **一定要有 Bad**        | `Consequences`                  | 記下決策成立之後長期要承受的東西，好壞都寫。Bad 欄位空白代表代價還沒想清楚                  |
| **問「反悔的話，要改的東西落在哪一層」** | `Consequences` | 寫下推翻這個決策要付的代價落在哪裡：要動到每個服務的程式碼，就等於接近不可逆，做決定前的確信度門檻要拉高；只要改一份設定檔，就可以用比較低的確信度先走，日後再換 |
| **怎麼確認它真的被遵守**      | `Confirmation`                  | 寫下這個決策靠什麼維持：自動化檢查、CI gate，或「只有 review、沒有工具」——沒有機制時就照實寫沒有 |
| **平手時偏向較安全的失敗模式**   | `Decision Outcome`              | 記下最後選了哪一個，以及選擇的方向。選項互有優劣時，就是兩權相害取其輕                      |

---

### 誠實評估：這份紀錄的限制

- **這些是去識別化後的通用版本。** 原始紀錄含專案內容不公開；這裡保留的是推理脈絡、選項比較與量化證據，移除了業務語境，因此有些決策讀起來比較抽象。
- **量化證據很薄，而且薄在兩個方向。** 有數字的那些是 directional（原文即如此標註）——跑在開發機等級硬體、有明確併發上限，支撐「A 比 B 好」而非生產容量保證；而好幾項宣稱的改善根本沒有前後量測（輪詢改推送的負載改善、可觀測性縮短的根因時間、三層冪等各層的實際攔截比例），它們是結構性論證，不是實驗數據。
- **其中一則是純架構推理，完全沒有量測。** `0046` 跨區狀態傳播尚未經 production 負載驗證；紀錄裡列出了驗證它需要哪些數字（複寫延遲的穩態與 P99、跨區 RTT、分區後的追平時間），但那些數字目前不存在。
- **有幾條已知會靜默出錯的規則，防線只有人。** `0010` 的邊界正當性、`0013` 的「設定裡沒有密鑰」、`0019` 的 CPU-bound 分類——都沒有自動化攔截。八則裡只有 `0002` 有客觀的通過門檻。ADR 自己標註了這點，標註的意義在於"需要重點 review 的關注點"。
- **部分欄位是回溯補齊的。** 八則現在都具備同一組欄位，但其中幾則的 `Decision Drivers` 與 `Considered Options` 是依原紀錄內文回溯整理成欄位的，不是當時就以該欄位寫下的（該節內有標註）；`0047` 與 `0046` 的 `Confirmation` 則是照實記下「原紀錄沒有確認機制」與「確認條件尚未執行」，而不是補上一個並不存在的檢查。
- **這裡只放了八則。** 同一批紀錄還涵蓋通訊協定分層、訊息語意與冪等、契約演進、可觀測性選型、schema 分發、儲存庫策略等主題，沒有收進來——這是選樣展示，不是全集。

---

### 相關作品

| repo                                                                                       | 關係                                                                        |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| [`distributed-tracing-reference`](https://github.com/JHying/distributed-tracing-reference) | 自動儀器化觸及不到的三個邊界怎麼補——`0019` 那條「虛擬執行緒不繼承 tracing context 需應用層補」的代價後來變成這個共用套件 |
| [`db-as-code`](https://github.com/JHying/db-as-code)                                       | schema 分發策略的可運行參考實作（該則不在本選集內）                                             |
| [`engineering-hub-showcase`](https://github.com/JHying/engineering-hub-showcase)           | 這些決策紀錄產出時所用的 spec-driven 工作流                                              |

---

## English

# Architecture Decisions in Practice

> Eight selected records of how I framed a problem, what I compared, why the other options were rejected, and what the choice cost. **What tool was picked is the least interesting part.**

A record that says "we chose Kafka" is worth nothing: the reader cannot tell a reasoned outcome from a lucky one. So each entry answers four things — **what the constraints were, which options were evaluated, why the others were rejected, and what this choice cost** — plus one that most decision records omit: **how compliance is actually confirmed.**

### Where to start

| You have | Read this |
| --- | --- |
| **30 seconds** | [The eight records](#the-eight-records) and [how I decide](#how-i-decide) below |
| **2 minutes, and only care what the format is worth** | [One record, before and after a revision](#what-the-format-does-one-record-before-and-after) — the outcome did not change; the reason did |
| **5 minutes** | [`0048`, the six-axis framework](docs/01-six-axis-evaluation-framework.md) — the clearest view of how one criterion grew from a single axis into a six-axis framework whose first application concluded not to split |

---

### The eight records

The selection criterion is not "the most important architecture" — it is that **each one demonstrates a different reasoning move**:

| # | Decision | What it demonstrates |
| --- | --- | --- |
| [`0048`](docs/01-six-axis-evaluation-framework.md) | Six-axis evaluation framework | Turning a recurring argument into a criterion — whose first application concluded **not** to split |
| [`0010`](docs/02-boundary-principles.md) | Six boundary principles | **Mutually independent**: any one is sufficient to justify a boundary, and none is weakened by an argument addressing only another |
| [`0047`](docs/03-mesh-control-plane.md) | Should the backbone also be the mesh control plane | **Correcting its own recorded reason**: a requirement both options satisfy cannot decide between them |
| [`0013`](docs/04-secrets-transit-pattern.md) | Secrets via the transit pattern | Two rejected options landing on **three different failure surfaces**: supply chain, runtime memory, audit granularity |
| [`0046`](docs/05-cross-region-propagation.md) | Cross-region state propagation | The decisive argument is the **shape of the failure**, not its absence |
| [`0002`](docs/06-embedded-container.md) | Embedded container | Separating **"too expensive" from "structurally impossible"**; the only objective pass condition in the set |
| [`0019` / `0024`](docs/07-concurrency-model.md) | Concurrency model | One criterion, two workloads, **opposite answers** |
| [`0017`](docs/08-human-promotion-gate.md) | Blue-Green with a human gate | Which slot in an automated pipeline is worth a human, and what that human really costs |

---

### How I decide

The same habits run under all eight. The format is [MADR](https://adr.github.io/madr/), and the middle column is the point of this table: **each habit is held up by a specific field**, not by remembering to be disciplined.

| Practice | Held up by | What the field requires |
| --- | --- | --- |
| **Establish which question is being decided** | `Context and Problem Statement` | States which question is on the table and the constraints in force. Until the problem is defined, comparing options has no common basis |
| **A necessary condition is not a deciding one** | `Decision Drivers` | Lists the factors that tell the options apart. A condition every option satisfies is a threshold, not a reason — it can rule options out, it cannot choose among the ones left |
| **Separate "too expensive" from "impossible"** | `Decision Drivers` | Each driver states its kind: a cost can be negotiated with resources and will be re-litigated; a structural ceiling does not move however much is spent |
| **Every rejected option carries its reason** | `Considered Options` | Lists every option seriously considered, each with its verdict and the reason behind it. With only the winner recorded, a reader cannot distinguish a reasoned outcome from a lucky one |
| **There must be a Bad** | `Consequences` | Records what the decision commits to over its lifetime, good and bad alike. An empty Bad column means the cost has not been worked out |
| **Ask where the cost of reversing it would land** | `Consequences` | States what undoing this decision would take: if it means editing every service's code the decision is near-irreversible, so the bar for certainty up front is much higher; if it lands in a single configuration file, you can proceed on less certainty and change your mind later |
| **How adherence is verified** | `Confirmation` | States what keeps the decision true — an automated check, a CI gate, or "review only, no tooling" — and says so plainly where no mechanism exists |
| **On a tie, prefer the safer failure mode** | `Decision Outcome` | Records which option was chosen and the direction of the choice. Where options trade off, what is compared is which failure is recoverable |

---

### What the format does: one record, before and after

The table above is a claim; this is the evidence. `0047` — whether east-west traffic governance should be handed to the existing configuration backbone — was revised recently, and **the outcome did not change by a single word. What changed was the reason.**

| | Before | After |
| --- | --- | --- |
| `Decision Drivers` | "a test request must stay pinned to one version at every hop" — marked **Primary** | The same requirement, demoted to "necessary, but does not distinguish the options"; the weight moves to failure-domain coupling, change cadence / ownership, and a traffic-mirroring gap |
| `Considered Options` | ingress-only / sidecar mesh / application-level version routing | extend the existing backbone into a mesh / a dedicated mesh / no mesh |
| `Decision Outcome` | adopt a mesh | adopt a mesh (**unchanged**) |

The reason for the demotion is short, and it is in the record: **the existing backbone's own routing can make a decision at every hop too, so both candidates satisfy that requirement — and a requirement both candidates satisfy cannot be what decides between them.**

This belongs in the README because it is the only way a format can prove its own worth. Written as prose — "we adopted a mesh because we need version pinning across the whole call chain" — the sentence reads perfectly well and the error sits inside it with no visible seam. Once drivers are one field and options are another, **"can this driver actually tell these options apart?"** becomes a question you can see, and ask.

The same revision flags a second thing that is easy to mis-book: per-pod sidecar overhead and the extra debugging hop are the price of having *any* mesh, paid identically by both mesh options, so they are not an argument against the dedicated one. That kind of distinction — a real cost that does not fall *between* the options being compared — only surfaces once each option's pros and cons are laid out separately.

Full context in [`0047`](docs/03-mesh-control-plane.md).

---

### Honest assessment: limits of this record

- **These are de-identified, generalised versions.** The originals contain project content and are not public. The reasoning, option comparison and quantitative evidence are preserved; the business context is removed, which makes some entries read more abstractly than they were.
- **The quantitative evidence is thin.** Where figures exist they are explicitly directional — developer-grade hardware, stated concurrency caps, supporting "A beats B" rather than capacity planning.
- **One entry is pure architectural reasoning with no measurement at all.** `0046`, cross-region state propagation, has not been validated under production load. The record states which numbers would validate it — steady-state and P99 replication lag, inter-region round-trip latency, catch-up time after a partition — and those numbers do not currently exist.
- **An ADR records a proposal and its reasoning, not an outcome.** How an architectural question is finally disposed of involves factors outside the technical argument — where decision authority sits, organisational familiarity, appetite for risk — none of which is in scope here.
- **Several known silent-failure modes are defended only by people.** `0010`'s boundary justification, `0013`'s "no secrets in the config source", and `0019`'s CPU-bound classification have no automated interception; only `0002` has an objective pass condition.
- **Some fields were filled in retrospectively.** All eight now carry the same set of fields, but in several of them `Decision Drivers` and `Considered Options` were reorganised into fields from the surrounding prose rather than written as those fields at the time (each such section says so). `0047`'s and `0046`'s `Confirmation` record, respectively, that the original had no confirmation mechanism and that the confirmation conditions have not yet been executed — neither invents a check that does not exist.
- **Only eight records are included here.** The same body also covers protocol layering, message semantics and idempotency, contract evolution, observability selection, schema distribution and repository strategy, among others. This is a selection, not the complete set.

---

### Related work

| repo | Relationship |
| --- | --- |
| [`distributed-tracing-reference`](https://github.com/JHying/distributed-tracing-reference) | The boundaries auto-instrumentation cannot reach — `0019`'s recorded cost, that virtual threads do not inherit tracing context, became this library |
| [`db-as-code`](https://github.com/JHying/db-as-code) | A runnable reference for schema distribution (that record is not in this selection) |
| [`engineering-hub-showcase`](https://github.com/JHying/engineering-hub-showcase) | The spec-driven workflow these records were produced under |
