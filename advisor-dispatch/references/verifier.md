# 高風險工單的獨立 verifier

適用：動到安全／權限／金流、併發、資料遷移、跨多檔核心邏輯的工單。在 advisor 自己 review 過關後再派。

## 模型選擇（動態判斷，不寫死模型名）

1. **外部模型 CLI 真的跑得動**（可選）→ verifier 走該 CLI。跨模型家族的獨立性最強（同家族共享訓練盲點）。
   `which <cli>` 只證明執行檔存在，不證明已登入、額度還在——要**實際跑一次最小指令**確認會回應，
   失敗就走第 2 條。此為可選模組，沒有就直接走第 2 條。
2. 外部 CLI 不可用 → verifier = **advisor 同級**的 fresh subagent
3. 硬下限：無論走哪條，verifier 階級 **≥ implementer**

```
Agent({
  subagent_type: "general-purpose",
  name: "verify-ticket-<n>",
  model: "<依上述順序選定>",
  prompt: <verifier prompt>
})
```

無論哪顆模型當 verifier，下方合約一字不變——換的是眼睛，不是規則。

派之前 advisor 先把 diff 落成檔案（verifier 的**起點**，不是唯一能看的東西）：
`git -C <worktree> diff <base>..HEAD > <scratchpad>/ticket-<n>-diff.txt`

## verifier 合約

- **隔絕的是敘事，不是資訊**：**不給**實作過程、implementer 自述、advisor 的 review 結論；
  但可**唯讀翻整個 repo**（既有呼叫端、未修改但受影響的 consumer、相關測試、專案慣例）。
  給它：工單驗收標準＋diff 檔路徑＋repo 路徑（唯讀）。
- **對抗式立場**：任務是「找出這個 diff 為什麼可能是錯的」，不是「確認它對」。預設懷疑，找不到問題才放行。
- **只判定不動手**：回傳 `CONFIRMED`（驗收達成、找不到反例）或 `REFUTED`（附具體反例：什麼輸入／路徑會出錯）。
  絕不自己改 code。
- REFUTED → 當作 review 未過，帶反例進修正迴圈。
- 外部 CLI 類 verifier 的 prompt 明寫「跑完必須立刻回報」，巡視時直接檢查它落的結果檔。

## 收斂規則（advisor 自己決定，不上桌問輪數）

verifier 永遠找得到東西，沒有收斂規則會無限退回。判準是**嚴重度**：

1. **輪數上限只管 advisor 主動再派幾次**：第一輪找產品缺陷；修完後第二輪限縮為
   「第一輪 findings＋宣稱的修法＋連帶回歸」。第二輪之後不再主動派。
2. **嚴重缺陷不分輪次一律修**（壞資料、漏權限、算錯錢、併發重複寫入），修完再驗修法。
   品質項在第二輪之後由 advisor 判斷；不修的寫進工單「已知弱點」，不再退回。
3. **第三輪仍抓到嚴重缺陷 ＝ 設計錯了**：停止 patch，向用戶說明「這張單要重做或重拆」與理由。

對用戶回報分開列「修了什麼」與「決定不修什麼、為什麼」。

## 外部 CLI 的行程清理（僅走外部 CLI 時）

某些 CLI 在 worktree 內執行後會留下脫離終端的常駐背景行程；worktree 刪除後它仍活著並累積。
清完 worktree 後檢查有無 cwd 已不存在的孤兒行程再回收（做法依該 CLI 而定，未通用驗證）。有 job 進行中一律不動。
