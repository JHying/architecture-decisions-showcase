# 0047 · 骨幹該不該兼任 mesh 控制平面

> **這則展示**：**推翻自己原本寫下的理由**——每個選項都滿足的需求，不能拿來決定選哪一個。
> 選集索引見 [README](../README.md)。

---

## Context and Problem Statement

既有的設定骨幹（Consul）已經扛著兩個職責：服務註冊與協調、以及設定投遞。第三個問題是**服務間（east-west）流量治理要不要也交給它**。

需求本身很硬：多個版本同時跑在同一叢集，測試請求必須在**每一跳**都留在同一個版本上（`A(v2) → B(v2) → C(v2)`）——只要 v2 的 A 打進了 v1 的 B，測試結果就不作數。ingress／gateway 層對此無能為力：它只看得到跨進叢集邊界的那一刀，請求進來之後在服務之間怎麼跳，它沒有辦法控制。

## Decision Drivers

這一則最值得看的是**它把自己原本寫錯的東西改掉的那一步**：

>原本上面那條需求被標成 **Primary** 決策驅動因素，結論就掛在它上面。
>
> 但改版後它被降級，理由很明顯：**這本來就是既有骨幹具備的路由能力**——router 依路徑前綴與 HTTP header 比對、splitter 依權重分流到各 subset、resolver 定義 subset——**同樣能在每一跳做決定**。兩個候選都完整滿足它。
>
> **一個每個選項都能滿足的目標，不能是決策的理由。**

真正決定的反而是另外三條，而三條指向同一個方向：

| 差異化因素            | 內容                                                                                                   |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| **失效域耦合**（最強的一條） | 既有骨幹已經扛著發現與設定中心。若再讓它當 mesh 的控制平面，它的**一次**事故會同時中斷兩件互相獨立的事：服務拿不到設定，**而且**流量治理同時失明                      |
| **變更節奏與擁有權邊界**   | 設定變更是應用團隊高頻做的日常交付動作；流量治理變更是平台／SRE 側，不同節奏、不同風險容忍度、不同審查流程。合成一個元件，等於讓一邊的例行變更有機會變成另一邊的事故                 |
| **流量鏡像缺口**       | Consul 的支援功能集裡沒有一級的流量鏡像。要拿到它得掉進非受管的逃生口，在骨幹自己的抽象之外手工維護原生 proxy 設定——這正好抵消掉「重用既有元件」本來要買的維運簡單性，還多帶一份升級風險 |

## Considered Options

| 選項 | 判定 |
| --- | --- |
| 不引入 mesh，east-west 治理靠 ingress／gateway | 否決——ingress 只看得到跨進叢集邊界的那一刀，請求進來之後怎麼在服務之間跳它控制不了，每一跳的版本釘選做不到 |
| 既有骨幹（Consul）兼任 mesh 控制平面 | 否決——上一節那三條差異化因素：失效域耦合、變更節奏與擁有權邊界、流量鏡像缺口 |
| 獨立的 mesh（Istio） | **採用**——見下節 |

被降級的那條需求（每一跳版本釘選）在這張表上仍然有作用：**它排除的是第一個選項，不是第二個**——這正是「必要條件不等於決定性條件」的具體長相。

## Decision Outcome

採用**獨立的 mesh（Istio）**。

## Consequences

- 既有骨幹繼續為設定而存在，mesh 在旁邊當第二個控制平面——**兩個都要升級、要輪替憑證、要有人維護**
- 每個 pod 多一個 sidecar 的資源開銷、每次呼叫多一跳的除錯鏈，是「引入 mesh」相對「不引入」的代價，**這部分兩個 mesh 選項的代價一樣**——所以它不能拿來當「不要獨立 mesh」的理由。重用既有元件並不會把這筆開銷折扣掉
- 獨立 mesh 才有的流量鏡像能力，不是只有好處：**鏡像是 fire-and-forget，代理層複製的是完整請求（含 POST/PUT body），被鏡像的那一側會真的把寫入執行下去。** 回應被丟棄不等於副作用被丟棄。所以它只在三種情況下安全——唯讀請求、帶去重鍵的冪等寫入、或目標接的是完全獨立的影子資料層。這條直接約束了[晉升前驗證](08-human-promotion-gate.md)能怎麼用它

## Confirmation

原始紀錄沒有留下確認機制，這裡照實記下缺口：

- **沒有自動化檢查在防止職責回流。** 設定投遞與流量治理的分離，靠的是兩個系統各自的設定面；若有人把流量治理規則寫回設定骨幹，沒有工具會攔截
- **「每一跳都停在同一版本」本身是可觀測的**（一個測試請求實際走過哪些 pod 是查得到的），但紀錄裡沒有把它寫成一條進版前的檢查

## Revisit 條件

**如果哪天流量鏡像的需求消失、且設定中心與流量治理的擁有權併進同一個團隊，三條差異化因素會同時失效**——那時這個決策就該重審，因為維持兩個控制平面分離的理由已經不成立。

## 誠實補充

- **對既有骨幹路由能力的評估來自其官方支援功能集，不是實測。** 「兩個選項在每一跳的版本釘選上等價」是文件層級的比較，沒有在同一個叢集裡各搭一次做對照。
- **這是這批紀錄裡唯一一則改掉了自己原本的決策理由的。** 可以有兩種讀法：一是格式把一個站不住的驅動因素逼了出來；二是原本那一版本來就寫得不夠嚴謹。兩種都成立。

---

## English summary

**Context** The existing configuration backbone (Consul) already carried discovery and configuration delivery; the third question was whether east-west traffic governance should join them. The requirement is hard — several versions run in one cluster and a test request must stay pinned to its version at *every* hop, which an ingress layer cannot do because it only sees the cluster boundary.

**Drivers, corrected in revision** That requirement had originally been recorded as the **Primary** driver, and the revision demotes it, because the existing backbone's own routing — header and path-prefix matching, weighted splitting across subsets, subset resolution — satisfies it just as completely as a dedicated mesh's sidecars. **A requirement both candidates satisfy cannot be what decides between them**; its only remaining job is to rule out having no mesh at all.

**Differentiators** An incident in one component must not blind configuration delivery *and* traffic governance at once; configuration changes (frequent, application-team-owned) and traffic-governance changes (platform-owned) should not share a change surface; first-class traffic mirroring is absent from Consul's supported feature set.

**Options** No mesh at all, with east-west governance left to the ingress layer — rejected, because ingress sees only the cluster boundary and cannot keep a request pinned to its version at every hop. The existing backbone extended into a mesh control plane — rejected on the three differentiators above. A dedicated mesh — adopted. Note where the demoted requirement still does work: it rules out the first option, not the second.

**Outcome** A dedicated mesh (Istio).

**Consequences** A second control plane to upgrade and rotate certificates for. Per-pod sidecar overhead and the extra debugging hop are the price of having *any* mesh, paid identically by both options, so they are not an argument against the dedicated one. Mirroring is fire-and-forget over a full copied request including bodies, so the mirrored side really executes writes — discarding the response does not discard the side effect, and it is safe only for read-only requests, idempotent writes with a dedupe key, or a fully independent shadow data layer.

**Confirmation** The original record leaves none, and the gap is recorded as such. No automated check prevents the responsibilities merging back: the separation rests on each system's own configuration surface, and traffic-governance rules written back into the configuration backbone would be caught by nothing. Per-hop version pinning is itself observable — which pods a test request actually traverses is retrievable — but was never written down as a pre-promotion check.

**Revisit** If mirroring is no longer needed and both ownerships merge into one team, all three differentiators lapse at once.

**Honest note** The routing-capability comparison is documentation-level, not measured side by side. This is the only record here whose stated reason changed while its outcome did not — readable either as the format forcing out an untenable driver, or as the original version having been written loosely.
