# Provenance Graph（溯源圖）— 概念篇

## 1. 一句話定義

**把系統 log 重新組織成一張有向圖，記錄「誰（subject）對誰（object）做了什麼（action），何時做的」**。
這張圖捕捉的是**因果關係**（causality），而不是時間順序。我們不再問「第 1234 行 log 寫了什麼」，而是問「這個檔案是怎麼變成現在這樣的？」「這個進程的存在源自哪裡？」

學術上有時叫 **audit graph / causal graph / data provenance graph**，最早可追溯到 King & Chen (SOSP 2003) 的 BackTracker，後續有 SPADE、CamFlow、HOLMES、UNICORN、ATLAS 等一系列研究。

---

## 2. 為什麼需要它？— 對比 log 的限制

原始 log 是**時間軸上的一串獨立事件**。它有三個本質缺陷：

| Log 本身的問題 | Provenance Graph 怎麼解 |
|---|---|
| 事件之間沒有顯式關聯，要靠人腦串 | 因果關係變成圖上的邊，可程式化遍歷 |
| 用 PID 串會被 fork / PID 重用打斷 | 用穩定的全域識別子（process GUID、inode、registry path）當節點 ID |
| 關鍵字搜尋只能找到「字面相符」的 log | 圖遍歷可以找到「因果相關但字面不同」的 log |

關鍵心智模型轉換：
> Log 回答 "What happened at time T?"
> Provenance graph 回答 "What caused X?" 和 "What did X affect?"

---

## 3. 圖的基本結構

### 3.1 節點（Nodes）— 系統中的實體

通常分成幾類，每類有自己的穩定 ID：

| 節點類型 | 穩定 ID（不會被重用的識別子） |
|---|---|
| Process | ProcessGuid（不是 PID） |
| File | 完整路徑 + inode/FileId |
| Registry Key/Value | 完整 key path |
| Network endpoint | (5-tuple) 或 (host, port, protocol) |
| User / Session | SID + LogonId |
| Module / DLL | 路徑 + hash |

> 「為什麼節點 ID 一定要穩定」是這套方法的核心。PID 會被回收，IP 會 DHCP 換，路徑可能被搬移；如果 ID 不穩定，圖就會在實體還活著的時候被「切斷」，前後變成兩個假節點。

### 3.2 邊（Edges）— 實體之間的操作

每條邊代表一次操作，邊上至少要帶三個屬性：

- **動作類型**（create / read / write / delete / load / connect / spawn …）
- **時間戳**（後面用來做時序篩選）
- **來源 log 的指標**（讓你能回溯到原始事件）

邊有方向，方向通常代表**資訊或控制流**：
- `Process A ──spawn──▶ Process B`：A 創造了 B
- `Process A ──write──▶ File F`：A 影響了 F
- `File F ──read──▶ Process A`：F 影響了 A（注意方向是反過來的，因為資訊是從 F 流入 A）

> 邊的方向是「資訊/影響從誰流向誰」，不是「誰主動操作誰」。這個慣例讓 backward / forward 追蹤的語意一致。

### 3.3 一個小例子

某個攻擊片段的事件序列：
```
T1: pwsh.exe 寫入  C:\temp\evil.dll
T2: pwsh.exe 修改  HKCU\Software\Classes\...\Command = "evil.dll"
T3: legit.exe 啟動 (auto-elevated)
T4: legit.exe 載入 C:\temp\evil.dll
T5: legit.exe 開了 cmd.exe
```

對應的 provenance graph：
```
            ┌──write──▶ evil.dll ──load──▶ legit.exe ──spawn──▶ cmd.exe
pwsh.exe ──┤                                  ▲
            └──write──▶ HKCU\...\Command ─────┘
```

注意兩件事：
1. **`pwsh.exe` 和 `legit.exe` 之間沒有父子關係**，可是圖把它們連起來了 — 透過共享的中間客體（檔案、機碼）。這就是「為什麼 PID 追蹤一定不夠」的具體展示。
2. 從 `cmd.exe` 反向追，可以一路追回 `pwsh.exe`。從 `pwsh.exe` 正向追，可以列出所有受影響的東西。

---

## 4. 怎麼建這張圖（Construction）

不管 log 來源是什麼，建圖都是同一個三步驟：

### Step 1: Normalization（標準化）

把每筆異質 log 轉成統一的三元組（或四元組）：
```
(timestamp, subject, action, object, [attrs])
```
這一步是工程活，不同 log source 欄位不同，但目標是抹平差異。例如：
- 一個 process create 事件 → `(t, parent_proc, spawn, child_proc)`
- 一個 file write 事件 → `(t, proc, write, file)`
- 一個 registry set 事件 → `(t, proc, regset, regkey)`

### Step 2: Entity Resolution（實體解析）

決定**「兩筆 log 提到的是不是同一個實體」**。這是建圖最容易出錯的地方：

- 同一個 process 在不同事件裡可能只給 PID、有時給完整 image path、有時給 hash → 要全部對到同一個 ProcessGuid
- 同一個檔案可能在不同事件裡寫成 `C:\foo\bar.exe` 和 `\Device\HarddiskVolume3\foo\bar.exe` → 要正規化路徑
- rename / move 之後是同一個檔案還是不同檔案？（通常 inode 不變代表同一個）

### Step 3: Edge Insertion（連邊）

對每筆標準化後的事件，在 subject 和 object 兩個節點間加一條帶屬性的邊。
如果節點不存在就建，已存在就重用。

---

## 5. 怎麼用這張圖（Querying）

建好圖之後，「找可疑 log」就變成幾個固定的圖演算法：

### 5.1 Backward Tracing（反向追因 / Root Cause Analysis）

從一個已知可疑節點 `v` 出發，**沿著入邊反向 BFS/DFS**，得到所有「可能導致 `v` 出現/被改變」的祖先節點。

語意：「`v` 為什麼會變成這樣？源頭在哪？」

### 5.2 Forward Tracing（正向追蹤 / Impact Analysis）

從 `v` 出發**沿出邊正向遍歷**，列出所有可能受 `v` 影響的後代節點。

語意：「`v` 已經汙染了哪些東西？我清理時要動哪些資產？」

### 5.3 Cross-host / Cross-source Stitching

如果 log 來自多台主機或多個工具（EDR、防火牆、應用 log），把每邊圖的「邊界節點」（網路 endpoint、共享檔案）對齊就能黏成更大的圖，看出橫向移動。

### 5.4 三個常見的時序限制

純拓撲遍歷會抓到太多無關東西。實務上一定要加：

1. **時間單調性**：追因時，邊的時間必須早於起點時間（你不能被未來的事件導致）
2. **時間窗**：只看起點前後 N 分鐘 / 小時，避免追到開機以來的所有事
3. **節點分裂（version graph）**：同一個檔案在不同時間點被改寫，學術上會把它拆成 file@v1, file@v2…避免「未來的寫入污染了過去的讀取」這種假因果

---

## 6. 這個方法的本質優勢與限制

### 優勢

| 優勢 | 為什麼 |
|---|---|
| 對 PID 重用、fork、進程鏈中斷免疫 | 用穩定 ID + 跨實體因果邊 |
| 對關鍵字混淆免疫 | 不靠字面比對，靠結構 |
| 可解釋性強 | 結果是一張可視化的攻擊鏈，不是一個黑箱分數 |
| 衝擊評估「免費」 | Forward trace 直接告訴你受影響範圍 |

### 限制（必須誠實面對）

| 限制 | 後果 |
|---|---|
| **依賴審計來源完整性** — log 漏掉的事件 = 圖上漏掉的邊 | 攻擊者刪 log 或繞過 sensor 就會在圖上斷鏈 |
| **Dependency Explosion** — 一個長壽進程（瀏覽器、shell）會跟超多東西有邊 | Backward trace 會炸出幾千個祖先，需要剪枝 |
| **隱式因果不會自動出現** — 例如進程 A 看了一個 race condition 的記憶體變數 | 系統沒有 audit 的東西就不在圖上 |
| **儲存與計算成本** | 大型主機一天可以幾百萬條事件，要做圖壓縮、節點合併、增量索引 |

> 「Dependency explosion」是這個領域最主要的研究方向之一。常見對策：
> - 以**讀寫劃分時間區段**（unit partitioning，BEEP/ProTracer/MPI 等論文）把長壽進程切成多個邏輯單元
> - 以**來源信任度 / 規則打分**剪掉低分支線
> - 把 graph 化簡成 **summary graph**，只留高訊息含量的節點

---

## 7. 跟前面提到的方法的關係

| 方法 | 在 provenance graph 框架下是什麼 |
|---|---|
| 用 PID 追 | 把節點 ID 設成 PID 的特例 — 因為 ID 不穩定，圖會斷 |
| 字串比對 | 完全不建圖，只在邊的「動作描述」屬性上做 substring match |
| 規則打分 / Sigma | 在圖上找符合特定子圖樣式（subgraph pattern）的位置 |
| 異常偵測 / ML | 把圖當輸入特徵，學「正常子圖長什麼樣」 |

換句話說，**provenance graph 是一個底層資料結構**，前面那些方法都可以建在它之上、彼此組合，而不是互相取代。這也是為什麼老師說「想出一套方法」時，先有圖、再決定用什麼演算法去查它，是比較乾淨的分層。
