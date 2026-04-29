1. windows manifest 是只有 slui 可以使用的參數還是類似環境的參數
2. autoElevate=true 是甚麼意思、功能
3. registry、file handler 的解說
4. UACME 是開源的嗎 ?
5. HKCU 為何
6. tatic 是 privilege escalation (補)
7. HKCU 背後的設計理念是甚麼、為何至今都沒有修掉
8. Akagi64.exe RegSetValue 的 Type: REG_SZ 是甚麼意思、Data: C:\Windows\System32\cmd.exe 是覆寫還是null 被寫入
9. Akagi64.exe QueryDirectory 是在做什麼
10. 一般user可以呼叫 slui ?
11. slui 0x03 之後應該出現讓使用者輸入金鑰的畫面，為甚麼沒有出現而是直接跳出cmd
12. 右邊的流程圖改詳細一點
13. 應該是先改HKCU之後再啟動slui.exe，讓slui.exe讀到錯誤的HKCU
14. 取出UAC 白名單