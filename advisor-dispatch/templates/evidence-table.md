# 驗收證據表範本

ticket-<N>#a<K>

| 驗收項 | 驗證方式 | 可重現證據 | implementer 判定 | advisor 複驗 |
|---|---|---|---|---|
| <驗收標準 1 原文> | `<完整指令>` | exit 0；<關鍵輸出，能看出測到哪個行為>；log: <路徑> | PASS | PENDING |
| <驗收標準 2 原文（手動）> | 環境：<…>；步驟：<…>；預期：<…> | 實際觀察：<…> | PASS | PENDING |
| <驗收標準 3 原文> | <…> | <缺相依／服務跑不起來等原因> | UNVERIFIED | PENDING |
| 已知弱點 | — | <弱點與原因；沒有寫「無」> | — | — |

advisor 複驗欄由 advisor 填；任一列為 FAIL／UNVERIFIED／PENDING 不得 merge（結構相依例外見 references/evidence-table.md）。
