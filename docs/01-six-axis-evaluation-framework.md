# 0048 · 六軸評估框架

> **這則展示**：把反覆爭論的問題（服務如何拆分）固定成一條判準。
> 選集索引見 [README](../README.md)。

---

## Context and Problem Statement

更早的一則（`0031`）用**一條**軸回答了「這個服務值不值得拆」：擴展經濟性——舊 monolith 本來就能水平擴展，拆分真正買到的是「不必為一個熱點功能付整個應用的成本」。

問題是同類問題後來反覆出現，每次都得重吵一輪：沒有共同的比較標準，討論退化成誰的直覺比較大聲，而且吵完不收斂，下次照樣再吵。

## Decision Drivers

- **判準要能收斂**——同一個判準下次還要能用，否則只是把同一場爭論重開一次
- **框架不得預設任何方向**——一組只證成得了「該拆」的判準沒有用，它必須同樣能得出「不拆」
- **必須看得到單軸看不到的成本與風險**——`0031` 只問擴展經濟性，而一個本地交易碎裂成跨服務分散式交易的成本、以及一刀落下後資料曝險面是收窄還是擴大，在單軸下完全不會出現

## Considered Options

| 選項                    | 判定                                   |
| --------------------- | ------------------------------------ |
| 沿用 `0031` 的單一軸（擴展經濟性） | 否決——它回答過一次就答不動了，上面第三條列的成本與風險在這條軸上不存在 |
| 每次個案重新討論，不固定判準        | 否決——這正是要解決的現象本身：沒有共同標準，討論不收斂，下次照樣再吵  |
| 固定成一組軸，任何邊界提案逐軸攤開     | **採用**——見下節                          |

> 這一節是依原紀錄的 Context 回溯整理成欄位的：原始紀錄沒有並列的選項比較，被比較的其實是「維持現狀的兩種做法」與「固定成框架」。

## Decision Outcome

固定成六條軸。任何有爭議的邊界提案——要拆或要合都一樣——都得逐軸攤開兩個選項差在哪，才能進到決策：

| # | 軸 | 問什麼 | 性質 |
|---|---|---|---|
| 1 | 效能與獨立擴展性 | 各部分能不能依自己的負載形狀獨立擴？對低流量元件，這項效益趨近於零 | 效益 |
| 2 | 失效域隔離 | 一邊出事會不會拖垮另一邊？拆開有沒有縮小爆炸半徑 | 效益 |
| 3 | 交付獨立性（發版耦合） | 各自能不能按自己的節奏發版，不必等對方、不必協調 | 效益 |
| 4 | 實作複雜度／開發速度 | 建立與變更這條邊界要付什麼——明確包含一個本地交易碎裂成跨服務分散式交易的成本 | 成本 |
| 5 | 維運人力／維護負擔 | 團隊吃不吃得下多出來的部署單位、監控面、事故量 | 成本 |
| 6 | 資料安全隔離 | 這一刀是收窄了元件碰得到的資料，還是擴大了曝險面 | 風險 |

**三效益、兩成本、一風險**是刻意的對稱。它不是「效益條數比較多所以該拆」的計分規則——存在的目的是讓框架**不預設任何一個方向**。

**首次應用的結論是不拆。** 案例是管理端功能該不該按領域拆成多個獨立部署的服務，結論是維持單一服務——方向與一般微服務的預設傾向相反。這比任何自我宣稱都更能說明那組效益／成本對稱是真的在運作，而不是替一個已經發生的拆分補一份事後正當化。

它與 [`0010` 六條邊界原則](02-boundary-principles.md) 是兩題：`0048` 問這條邊界值不值得畫，`0010` 問刀落在哪。軸 4、軸 6 是 `0010` 六條原則在上一層的對應物，兩份一起用。

## Consequences

- **六軸沒有相對權重。** 不同軸指向相反方向時，框架本身給不出答案，仍然要人做取捨判斷

## Confirmation

有爭議的邊界提案未逐軸列出差異者，不進入決策。**沒有工具能檢查這件事**，這項機制單純是團隊共識。

## 誠實補充

- **首次應用是回溯重建，不是當場紀錄。** 原始討論沒被留下來，六軸裡有三條（交付獨立性、實作複雜度、資料安全隔離）在紀錄裡是人工回溯補上的，該紀錄自己標了這點。
- **這份紀錄本身也是回溯寫入的。** 框架在實務上先被用了一段時間，後來才進入決策紀錄集。

---

## English summary

**Context** An earlier record answered "is this split worth it" on a single axis — scaling economics. The same class of question kept recurring with no shared standard, so it never converged.

**Drivers** The criterion had to converge — the same class of question kept recurring with no shared standard, so discussion degraded into whose intuition was loudest. It had to presuppose no direction, since a criterion that can only justify splitting is useless. And it had to surface what a single axis cannot: a local transaction fragmenting into a distributed one, and whether a cut narrows or widens data exposure.

**Options** Keeping `0031`'s single scaling-economics axis, or re-arguing each case from scratch — both being the status quo that produced the problem — against fixing a set of axes that every boundary proposal lays out. *This comparison is a retrospective reorganisation of the original prose; the record carried no side-by-side option list.*

**Outcome** Six axes every boundary proposal — split or merge — must lay out before it can proceed: independent scalability, fault-domain isolation, delivery independence (benefits); implementation complexity, explicitly including one local transaction fragmenting into a distributed one, and operational headcount (costs); data-security isolation (risk). Three benefits, two costs, one risk — the symmetry exists so the framework presupposes no direction, not as a scoring rule. **Its first recorded application concluded not to split**, against the usual microservices-leaning default. It answers whether a boundary is worth drawing; [`0010`](02-boundary-principles.md) answers where it falls, and the two are used together.

**Consequences** The axes carry no relative weighting, so opposing axes still require a human trade-off.

**Confirmation** A disputed boundary proposal that has not laid out an axis-by-axis comparison does not proceed to a decision. No tool checks this — the mechanism is purely team consensus.

**Honest note** The first application is a retrospective reconstruction — three of the six axes (delivery independence, implementation complexity, data-security isolation) were filled in after the fact, and the record notes this itself. The record was itself written up after the framework had already been in use — present in practice, absent from the record.
