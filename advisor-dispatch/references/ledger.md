# 進度 Ledger

放 scratchpad 或 repo 內 git-ignored 路徑，每個事件追加一行：

```
ticket-1#a1 dispatching
ticket-1#a1 dispatched (model=<model>, worktree=<path>, base=<完整 SHA>)
ticket-1 question q1: <一句摘要>
ticket-1 q1 answered
ticket-1#a1 nudge #1
ticket-1 review round 1: FAIL (<原因>)
ticket-1 review round 2: PASS
ticket-1 merged (<完整 SHA>); worktree removed
ticket-2#a1 superseded by ticket-2#a2 (<model>): <原因>; a1 agent 已終止, worktree 交給 a2 沿用
ticket-3 canceled: <原因>; worktree removed, branch deleted
decision: <議題> → <選擇>（<日期>，<誰決定>）
deploy: verified on <env> at <endpoint>
```

過程事件只用這些詞：`dispatching`、`dispatched`、`nudge #k`、`unverifiable`、`stale report ignored`、
`question qM: <摘要>`、`qM answered`、`decision: <議題> → <選擇>`、`review round N: PASS|FAIL`、
`blocked (on <具名對象>)`。

**工單只有三種終態，每一種都必須帶原因或去向，並寫明 agent 與 worktree 怎麼處置，沒帶就不算收尾：**

- `merged (<完整 SHA>)`
- `superseded by <新工單或新編號>: <原因>`（退回重派、換 model 重派、被拆單取代）。
  舊 agent 必須已終止；worktree 寫明是交給新的那次沿用，還是已移除
- `canceled: <原因>`（advisor 自行取消時在原因寫明並當場告知用戶）。worktree 與 branch 一併清掉；
  有要保留的成果就寫明保留在哪

**收工前檢查**：`git worktree list` 的每一棵都要對應到 ledger 裡還在進行、或終態寫明「沿用／保留」的工單，
對不上的就是殭屍；還沒記 `answered` 的 question 必須是零。

Compaction 後掃 ledger，任何沒有終態行的工單都要查明現況，不得當作已完成或已放棄。
