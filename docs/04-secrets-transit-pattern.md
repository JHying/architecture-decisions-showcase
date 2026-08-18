# 0013 · 密鑰 transit 模式

> **這則展示**：兩個被否決的選項壞在**三個不同的失效面**——供應鏈曝光、執行期記憶體曝光、稽核粒度不足。
> 選集索引見 [README](../README.md)。

---

## Context and Problem Statement

一個好改又集中的設定來源，天生就是密碼最容易被順手貼進去的地方。

處理方式不是寫一條規範然後靠 code review 攔，是**從架構上排除這個可能**：一般設定走設定骨幹（Consul），密碼與簽章金鑰一律走專責的機敏資料引擎（Vault），兩者是不同的系統、不同的存取路徑。

而簽章／加密金鑰一旦外洩，攻擊者就能偽造有效 token，所以問題不只是「金鑰存哪」，還包括**簽章這個動作在哪裡發生**。

## Decision Drivers

四條驅動因素直接對應下面比較表的四個欄位：

- **輪替是什麼等級的動作**——緊急輪替必須是操作動作，不能是「重建映像檔並重新部署每一個使用它的服務」這種部署動作
- **稽核記得到什麼粒度**——「金鑰被讀取過」與「金鑰被用在哪一次簽章上」是兩種不同的證據
- **金鑰材料在哪裡以明文存在過**——這決定了哪些系統被攻破就等於金鑰外洩
- **密碼學操作在哪裡執行**——程序內運算沒有網路成本；遠端引擎則是每一次簽章／加解密都要一次網路往返。這是唯一一條**被否決的選項勝出**的驅動因素

第四條在原紀錄裡沒有被列為驅動因素，只以代價的形式出現在 Consequences，這裡回溯補上。

還有一條前提性的：密碼與簽章金鑰**不得出現在設定來源**——這要從架構上排除，不是寫一條規範靠 code review 攔。

## Considered Options

| 選項 | 輪替是什麼 | 稽核記到什麼 | 洩漏面 | 密碼學呼叫成本 |
| --- | --- | --- | --- | --- |
| 磁碟上的 keystore 檔（烘進部署產物） | **部署動作**——要重建映像檔並重新部署每一個使用它的服務 | — | 原始金鑰躺在映像層裡，registry 一旦被攻破就直接曝光 | 程序內運算，無網路呼叫 |
| 金鑰放機敏資料儲存，啟動時載入應用記憶體 | 仍需 **pod 重啟**才能生效 | 只記得到「金鑰**被讀取過**」 | 金鑰會出現在 heap dump 裡 | 程序內運算，啟動時取一次金鑰 |
| transit engine 模式（引擎自己執行 sign/encrypt/decrypt） | **操作動作**——免停機、免重新部署 | **每一次密碼學呼叫各自被記錄** | 金鑰材料永不可匯出 | **每一次簽章／加解密都是一次遠端呼叫** |

這三條否決理由落在**三個不同的失效面**：供應鏈曝光、執行期記憶體曝光、稽核粒度不足。這是重點——不是「另外兩個比較差」，而是兩個被否決的選項各自壞在不同地方。

最後一欄的方向相反：那是採用方案唯一輸給另外兩個的地方，代價記在 Consequences。

第一個選項的問題可以寫成一句話：**它把一個本來該是「操作動作」的事，拿一個「部署動作」去做。** 這種錯配通常不會在功能上出錯，只會在你真正需要緊急輪替的那天出錯。

## Decision Outcome

採用 Vault 的 **transit engine 模式**。

順帶一提，這個決策**重用了已經為服務發現與設定在運作的基礎設施**，而不是引入一個新系統——骨幹選型讓這一題少了一個新的維運依賴。

## Consequences

**Vault 現在落在每一個需要簽章或解密的操作的關鍵路徑上**（例如登入）。它的可用性直接決定那條路徑能不能運作、且 I/O 變成必要項，效能是必須考慮的，所以必須以高可用配置運行、帶健康檢查的冗餘。

這是一個真實的取捨：**把金鑰外洩風險換成了可用性風險，而不是消除了風險。**

## Confirmation

「設定來源中永遠不存在密鑰」靠的不是掃描工具，是**兩個系統物理分離**這個架構事實。它不靠人守紀律，這比「有工具在掃」更接近結構性保證。

但「掃不到密鑰」與「證明它不存在」仍是兩件事，而後者在這份紀錄裡沒有對應的機械化檢查。

## 誠實補充

- **密鑰引擎成為登入路徑的硬依賴，但沒有記錄實際的可用性目標或演練結果。** ADR 要求它以高可用配置運作，沒有寫 SLO、也沒有寫演練頻率。這是這一則裡風險最集中的一處。
- **「沒有金鑰材料常駐應用記憶體」是陳述為一條性質，來源未指名任何檢測工具。**
- **遠端密碼學呼叫的延遲成本沒有量測。** 每次簽章／加解密多一次網路往返是結構性事實，但登入路徑的延遲預算、實測 P99、以及引擎降級時這條路徑的行為，紀錄裡都沒有數字。

---

## English summary

**Context** A conveniently editable, centralised configuration source is exactly where a password gets pasted. The possibility is designed out architecturally — ordinary configuration on the configuration backbone (Consul), passwords and signing keys on a dedicated secrets engine (Vault), different systems and different access paths — rather than caught in review. And because leaked signing keys mean forgeable tokens, the question includes *where signing happens*, not only where keys live.

**Drivers** Four, matching the four columns of the comparison. What level of action a rotation is — an emergency rotation must be an operational action, not a rebuild-and-redeploy of every consuming service. What granularity the audit trail records — that a key was *read* is different evidence from which signature it was used for. Where key material has ever existed in plaintext, which fixes what a compromise of any given system is worth. And where the cryptographic operation itself executes — in-process at no network cost, or in a remote engine at one round trip per call — the only driver on which a rejected option wins; the original record carried it as a consequence rather than a driver, and it is filled in here retrospectively. Plus one precondition: passwords and signing keys must not appear in the configuration source at all, designed out architecturally rather than caught in review.

**Options** A keystore file baked into the artifact makes rotation a *deployment* action instead of an *operational* one, and leaves the raw key sitting in an image layer. Loading keys into application memory leaves them visible in heap dumps, still requires a restart to rotate, and produces an audit trail recording that the key was *read* rather than what it was used for. Vault's transit engine keeps key material non-exportable, rotates as a single administrative action, and audits every individual cryptographic call — **adopted**.

The rejections land on three **different failure surfaces**: supply-chain exposure, runtime memory exposure, insufficient audit granularity. The first option's flaw in one sentence: it performs with a *deployment* action something that should be an *operational* one — a mismatch that never shows up functionally, only on the day an emergency rotation is needed. Notably, the decision reuses infrastructure already running for discovery and configuration rather than introducing a new system.

**Outcome** Vault's transit engine, reusing infrastructure already running for discovery and configuration rather than introducing a new system.

**Consequences** The engine is now a critical-path dependency for every signing or decryption operation, login included. Its availability decides whether that path works at all, and every cryptographic call becomes a mandatory network round trip, so performance is now a design concern too — it must run highly available with health-checked redundancy. Exposure risk was traded for availability risk, not removed.

**Confirmation** "No secrets in the configuration source" rests on the two systems being physically separate — closer to a structural guarantee than a scanner. But "no secret found" and "no secret exists" remain two different claims, and the second has no mechanical check here.

**Honest note** No availability target or drill result is recorded for the engine that now gates login, and the "no key material resident in memory" property names no detector. The latency cost is unmeasured too: an extra network round trip per signature or decryption is structural, but there is no budget, no measured P99 for the login path, and no recorded behaviour for that path while the engine is degraded.
