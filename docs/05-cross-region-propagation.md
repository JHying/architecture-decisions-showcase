# 0046 · 跨區狀態傳播

> **這則展示**：決定選項的是**失敗的代價有多大**，不是「保證不會出錯」。
> 選集索引見 [README](../README.md)。

---

## Context and Problem Statement

平台部署在不只一個地理區域。在某一區發生的狀態變更，必須讓其他區看得到。

而兩區之間是 WAN：往返延遲比區內高一個數量級，**而且「不穩定」是正常營運狀態，不是事故**。

骨幹選型當時把「有成長為跨區事件分發的空間」列為驅動因素之一，但沒有說事件**怎麼**跨越區界。這一則補的就是那個空缺。

## Decision Drivers

- **不得有靜默遺失**——遺失本身可以談，「不知道漏掉了哪些」不行
- **區域自主性**——遠端區降級時本地區不能跟著降級，那正是多區部署存在要提供的東西
- **不為這份資料不需要的保證付費**——強跨區一致不是這份資料的需求，而且付了錢也常常拿不到
- **WAN 不穩定是常態，不是事故**——方案必須在鏈路反覆抖動下仍然可運作

## Considered Options

三個被否決的選項各自敗在**不同的驅動因素**上：

| 選項 | 否決理由 |
| --- | --- |
| 跨區複寫型資料庫叢集 | 為一個**這份資料並不需要的保證**付費——強一致引擎的 active-active 路徑帶有可觀授權成本、且仍把衝突解決推回應用層；而真正便宜好跑的託管多主方案是以 last-writer-wins 解衝突，**等於沒有交付當初付錢買的強一致** |
| 區間直接長連線 | 直接卡在靜默遺失那條底線：WAN 抖動會照常斷開長連線，斷開時兩端都**無法列舉漏掉了哪些**，也沒有 offset 可續傳，恢復得自己設計一套對帳協定 |
| 同步跨區 HTTP | 把 WAN 往返延遲直接放到請求路徑上，並**耦合了跨區可用性**——遠端區降級時呼叫端跟著降級，那正是多區部署存在要提供的區域自主性 |

## Decision Outcome

**每區一個 Kafka 叢集，之間以跨叢集鏡射複寫（cross-cluster mirroring）組成 active-active，複寫工具採用 MirrorMaker 2。**

每區把狀態變更事件發到自己的本地叢集，同時消費本地 topic 與從其他區鏡射進來的 topic。被複寫的 topic 名稱帶有來源叢集前綴，因此鏡射進來的 topic 不會被誤認成本地的、也不會被複寫回原點——**遠端 topic 命名因此是 consumer 契約的一部分，不是實作細節**。

複寫工具在 MirrorMaker 2 與 Confluent Cluster Linking 之間選了前者：Cluster Linking 是 Confluent Platform / Confluent Cloud 專屬功能，選它等於把跨區複寫機制綁進特定供應商的生態——未來若要換掉 broker 供應商，這個機制得重新設計；MirrorMaker 2 是 Apache Kafka 原生、開源免授權費，不因供應商而綁死。

## 決定性論證是失敗的代價

不是「不會掉訊息」，是**失敗長什麼樣子**：

> 跨叢集複寫**本身也是非同步的**：區域或區間鏈路失效時，還沒鏡射過去的事件會延遲，原叢集若一併失去也可能真的遺失。
>
> 它買到的不是「不會掉」，而是**把不可觀測的遺失換成可觀測的落後**——複寫 lag 是可監控、可告警的指標，鏈路恢復後 consumer 從已提交的 offset 續傳，停機期間累積的 backlog 是被補送而不是被跳過。

## Consequences

- **最終一致成為明確的應用層責任**：跨區不保留事件順序，consumer 必須冪等並容忍重複與亂序
- **複寫拓樸洩漏進應用程式碼**：consumer 要同時訂閱本地與帶前綴的遠端 topic，而不是被 broker 藏起來
- **N 個區就是 N 個叢集加上區間複寫管線**要營運、監控、升級
- **任何未來真正需要強跨區一致的需求，在這個設計裡沒有位置**，得另開一個決策，而不是延伸這一個

## Confirmation

這一則的確認條件**尚未執行**，但已經被寫成可量測的形式：各區配對之間的穩態與 P99 複寫 lag、實測的區間往返延遲對照這裡假設的值、刻意製造已知時長的鏈路分割後排空 backlog 的追趕時間、以及確認命名前綴策略下沒有任何 topic 被複寫回原叢集。

其中複寫 lag 同時是**運行期的確認機制**：這個決策買到的東西就是「落後可觀測」，所以 lag 指標本身可監控、可告警，就是這條性質還成立的證據。

在上述數字出現以前，這一節是**設計目標，不是已經通過的檢查**。

## 誠實補充

- **這一則尚未經 production 驗證。** 它是架構推理，不是量測結果。要驗證它至少需要：各區配對之間的穩態與 P99 複寫 lag、實測的區間往返延遲對照這裡假設的值、刻意製造已知時長的鏈路分割後排空 backlog 的追趕時間、以及確認命名策略下沒有任何 topic 被複寫回原叢集。**在這些數字出現以前，跨區路徑上任何容量或延遲數字都只是設計目標，必須照這樣描述。**
- **多叢集加複寫管線的維運成本沒有被估算。**

---

## English summary

**Context** The platform runs in more than one geographic region, and a state change in one must become visible in the others. Between them sits a WAN: an order of magnitude more round-trip latency than intra-region, and **instability is normal operation, not an incident**. An earlier backbone decision listed room to grow into cross-region distribution as a driver, but never said *how* events cross the boundary.

**Drivers** No silent loss — loss itself is negotiable, not knowing what was missed is not. Regional autonomy, so a degraded remote region does not drag the local one down with it. No paying for a guarantee this data does not need. And WAN instability treated as normal operation, so any option has to keep working across repeated jitter.

**Options** A cross-region replicated datastore was rejected for charging for a guarantee this data does not need — the strongly consistent engines carry substantial licensing on their active-active path and still push conflict resolution back to the application, while the cheap managed multi-master options resolve by last-writer-wins, so the strong consistency being paid for is not delivered. Direct inter-region long-lived connections fail the silent-loss bar: WAN jitter drops them, and neither end can enumerate what was missed or resume from an offset. Synchronous cross-region HTTP puts WAN latency on the request path and couples regional availability, defeating the autonomy multi-region deployment exists to provide.

**Outcome** One Kafka cluster per region, joined by cross-cluster mirroring in an active-active topology, using MirrorMaker 2. Each region produces state-change events locally and consumes both its local topics and those mirrored in from elsewhere. Replicated topic names carry a source-cluster prefix, so a mirrored topic is never mistaken for a local one and cannot be replicated back — which makes **remote topic naming part of the consumer contract, not an implementation detail**. MirrorMaker 2 was picked over Confluent Cluster Linking: Cluster Linking is a Confluent Platform / Confluent Cloud exclusive feature, and adopting it would tie the cross-region mechanism to one vendor's ecosystem — a future broker-vendor change would force a redesign of this mechanism. MirrorMaker 2 is native to open-source Apache Kafka and carries no licence cost or vendor lock-in.

**The decisive argument is the cost of the failure, not its absence.** Replication is itself asynchronous: when a region or the inter-region link fails, events not yet mirrored are delayed, and can genuinely be lost if the origin cluster is lost with them. What it buys is **observable lag instead of invisible loss** — lag is a monitorable, alertable metric; consumers resume from committed offsets when the link returns, so the outage produces a backlog that is delivered rather than a gap that is skipped.

**Consequences** Eventual consistency becomes an explicit application concern — ordering is not preserved across regions, so consumers must be idempotent and tolerate duplicates and reordering. The replication topology leaks into consumer subscriptions rather than being hidden by the broker. N regions means N clusters plus replication plumbing to run, monitor and upgrade. And anything genuinely requiring strong cross-region consistency has no home in this design; it needs its own decision rather than an extension of this one.

**Confirmation** Not yet executed, but stated in measurable form: steady-state and P99 replication lag per region pair, measured inter-region round-trip latency against the assumed value, backlog catch-up time after a deliberate partition of known duration, and confirmation that the prefix naming strategy lets no topic replicate back to its origin. Replication lag doubles as the runtime confirmation — what this decision buys is observable lag, so the metric being monitorable and alertable is itself the evidence that the property still holds. Until those numbers exist, this section is a design target rather than a passed check.

**Honest note** This record has not been validated under production load. It states which numbers would validate it — steady-state and P99 replication lag per region pair, measured inter-region round-trip latency against the assumed value, catch-up time after a deliberate partition of known duration, and confirmation that no topic replicates back to its origin — and those numbers do not currently exist. Until they do, every capacity or latency figure on this path is a design target and must be described as one.
