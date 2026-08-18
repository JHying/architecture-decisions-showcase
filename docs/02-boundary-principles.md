# 0010 · 六條邊界原則

> **這則展示**：六條原則**彼此獨立**——任一條成立就足以支撐一條邊界，也不會被只談另一條的論證削弱。
> 選集索引見 [README](../README.md)。

---

## Context and Problem Statement

[`0048`](01-six-axis-evaluation-framework.md) 回答「這條邊界值不值得畫」，但沒說**刀落在哪**。

實際上有好幾個服務被拆開（或刻意留在一起）的理由跟 throughput 完全無關：協定轉譯、安全敏感度、變更頻率、正確性隔離——每一個都自己就能撐起一條邊界。

## Decision Drivers

- **邊界的理由不只有 throughput**——協定轉譯、安全敏感度、變更頻率、正確性隔離，每一個都自己就能撐起一條邊界
- **每條理由必須單獨成立**——不需要湊齊多條才算數，一條成立就足夠
- **一條理由不能被只談另一條的論證推翻**——否則邊界爭論會退化成用單一維度否決一個多維度的決定

## Considered Options

| 選項 | 判定 |
| --- | --- |
| 直接沿用 [`0048`](01-six-axis-evaluation-framework.md) 六軸當落刀依據 | 否決——`0048` 回答的是「這條邊界值不值得畫」，它不回答**刀落在哪** |
| 一條通則 | 否決——實際存在的邊界理由彼此無關（協定轉譯、安全敏感度、變更頻率、正確性隔離），收不進同一條規則 |
| 六條彼此獨立的原則清單 | **採用**——見下節 |

> 這一節是依原紀錄的 Context 回溯整理成欄位的，原始紀錄沒有並列的選項比較。

## Decision Outcome

不是一條通則，是六條原則的檢查清單：

| #   | 原則                                                              |
| --- | --------------------------------------------------------------- |
| 1   | 把連線生命週期的關注點與業務領域的關注點拆開——高連線數/低單連線 CPU 與低連線數/高單請求運算，是兩種不相容的擴展形狀  |
| 2   | 把外部協定轉譯切成獨立服務，不是同一服務裡疊一層轉譯模組——協定的詞彙與怪癖只讓這個服務知道，下游服務完全不必知道外部協定存在 |
| 3   | 把極少變動、以廣播而非逐請求查詢的全域設定，從請求時邏輯中分離                                 |
| 4   | 把身分與個人檔案視為兩個領域——憑證/token 簽發的變更節奏與風險輪廓，跟一般帳戶資料不同，即使兩者都「屬於同一個使用者」 |
| 5   | 把正確性關鍵的批次工作隔離成自己的失效域——不與日常即時流量共用同一個 broker、同一個 cache            |
| 6   | 把管理端流量視為與使用者面流量不同的失效域——管理端的尖峰不得劣化使用者面延遲；變更以非同步方式傳播，不走熱路徑上的同步呼叫  |

最吃重的性質是**彼此獨立**，它帶來一個很實用的推論：

> 一條為了原則 5（失效域隔離）而畫的邊界，**不會被一個只談原則 1（擴展）的論證削弱。**

這句話直接改變 review 的品質。沒有它，邊界爭論會退化成「這兩個服務流量差不多，合併吧」——用單一維度推翻一個多維度的決定。

## Consequences

**六條獨立原則意味著六份獨立的正當性要維護**，邊界 review 得逐條檢查，不能套一條萬用規則。

## Confirmation

任何提出的邊界（新拆分或提議的合併）都必須指名是六條中的哪一條在支撐它；「感覺對」本身不構成理由。

執行方式**只有 review，沒有任何自動化攔截**——沒有工具能在 CI 上檢查「這條邊界有沒有原則支撐」，也沒有工具能在邊界被侵蝕時告警。

## 誠實補充

- **「彼此獨立」這個性質沒有被實證。** 它是從六條原則的內容推得的，用途是避免單維度論證推翻多維度邊界；紀錄裡沒有統計過實際 review 中它被援引了幾次、擋下過什麼。

---

## English summary

**Context** [`0048`](01-six-axis-evaluation-framework.md) established whether a boundary is worth drawing, not where the cut falls. Several services were split, or deliberately kept together, for reasons unrelated to throughput: protocol translation, security sensitivity, change frequency, correctness isolation.

**Drivers** Boundary reasons are not all throughput-shaped; each must stand on its own rather than needing to be combined with others; and none may be overturned by an argument that addresses only a different one.

**Options** Reusing [`0048`](01-six-axis-evaluation-framework.md)'s six axes as the cutting criterion was rejected — it answers whether a boundary is worth drawing, not where it falls. A single general rule was rejected because the real reasons are mutually unrelated and do not fit under one. *Also a retrospective reorganisation; the original carried no option list.*

**Outcome** A checklist of six principles: connection lifecycle vs business domain; external protocol translation isolated into its own service (not merely an in-process anti-corruption layer); static configuration out of request-time logic; identity separate from profile; correctness-critical batch work in its own failure domain; administrative traffic in its own failure domain. The load-bearing property is that they are **mutually independent** — a boundary drawn for failure-domain isolation is not weakened by an argument that only addresses scaling. Without that, boundary debates degrade into "these two have similar traffic, merge them": a single dimension overturning a multi-dimensional decision.

**Consequences** Six independent justifications to maintain; boundary review is item by item, with no universal rule to apply.

**Confirmation** Every proposed boundary must name which principle supports it; "it feels right" is not a reason. Enforcement is review only — no tool can check that a boundary has principled support, and none alerts when one erodes.

**Honest note** The independence property was never empirically tested; it is inferred from the content of the six principles, with no count of how often it was actually invoked in review.
