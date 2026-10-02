---
name: advisor-dispatch
description: 派工／監工開發模式：主 session 只規劃、拆單、review、merge，實作交給各自 git worktree 裡的 subagent。用於「派工、開工單、平行開發、worktree、advisor 模式、delegate to subagents」；單線小改動不適用。Use when splitting work into tickets for isolated subagents with evidence-gated review.
---

# Advisor Dispatch — 派工監工開發模式

主 session 是 **advisor**：只做規劃、拆單、review、merge、盯部署，**不寫實作碼**。
實作全部由 subagent 在各自的 git worktree 完成。

為什麼這樣分工：advisor 的 context 要留給跨工單的協調與品質判斷；advisor 自己下場改 code
會被實作細節污染，也會和 worktree 裡的 agent 搶檔案。

## 前置條件與備援（先看這張表）

核心流程只需要：git、Agent tool（可指定 `isolation: "worktree"` 與 `model`）、Bash。
其餘都是「有就用，沒有就走備援」，概念不刪：

| 能力 | 有的話 | 沒有的話（備援） |
|---|---|---|
| `SendMessage`（繼續同一個 subagent） | 修正迴圈、催促都送給原 agent | 原 agent 已結束才能補派新 agent 進**同一棵 worktree**，prompt 附上上一輪 findings 與現況；原 agent 若還活著，禁止補派 |
| `ListAgents`（看 agent 是否還活著） | 巡視、重派前確認 | 用 worktree 的 commit／未提交修改有無變化＋Agent 完成通知判斷；無法確認終止就當它還活著 |
| 排程／背景喚醒（背景 sleep、cron 類工具） | 每 ~10 分鐘喚醒巡視 | 每次收到任何訊息就順便巡一輪，並告訴用戶「目前沒有自動喚醒，我只會在你互動時巡」 |
| 外部模型 CLI 當 verifier（例如另一家族的 coding agent） | 獨立性最強，見 `references/verifier.md` | verifier 改用 advisor 同級的 fresh subagent |
| 內容閘門（機械判斷「證據是否支持驗收項」） | 一次請求逐列判定 | advisor 逐列自己讀：證據是否對那條驗收項說話 |
| 專案既有計畫／spec | 直接拆單 | 先產一份計畫再拆 |

「未驗證」聲明：以上備援在本機以外的 Claude Code 版本是否都可行，作者未全部驗證；
工具缺失時以實際可用的為準。

## Model 選擇（參數化，不寫死）

- **Advisor** = 當前主 session 的 model，不需設定。
- **Implementer** = Agent tool 的 `model` 參數。預設用中階模型（如 `"sonnet"`）；用戶指定
  「這單給 opus」就改該張（或全部）。每次派工都**明確指定** model，不要省略
  （省略會繼承主 session model，通常是最貴的那顆）。
- **Verifier**（高風險工單才出動）：獨立於 implementer，階級 ≥ implementer。選擇順序見
  `references/verifier.md`。

## 流程總覽

```
0. 多步驟流程 → 先產流程契約（Step 1.5），接縫沒 owner 不准開工
1. 拆單（含檔案所有權切割）
2. 記下 base → 派工（每單一個 subagent + 獨立 worktree + 獨立 branch）
3. 逐單 review（證據閘門 → advisor 親自看 diff）
4. 未過 → 退回同一個 agent 修 → 重 review
5. 全部通過 → advisor 依序 merge
6. 盯部署 → 遠端驗證通過才算收工（驗證期間凍結）
```

## Step 1: 拆單

沒有計畫先產計畫。有計畫則拆成工單，每張必含（範本見 `templates/ticket.md`）：
**目標**（一句話）、**驗收標準**（可驗證，不是「做好」）、**檔案所有權清單**、
**測試要求**、**不做什麼**。

### 檔案所有權切割（平行的前提）

worktree 只解決「working tree 互相踩」，**不解決 merge conflict**。
- 同一個檔案在同一批平行工單中**只能屬於一張**
- 必須動同一檔案（共用 route 註冊、schema）→ 這兩張串行，或抽成前置工單先做完
- 拆完逐張比對所有權清單，無交集才准平行派出

### 平行 vs 串行：只是排程，不是兩套流程

工單相依、擠在同一批檔案、需求邊做邊定 → 改**串行**：一次一張，DONE → review → 通過才派下一張。
每張照樣走完整合約（worktree、回報格式、證據閘門、review）。
**禁止**「不能平行 → advisor 自己下場做 → 跳過 review」。不用本 skill 的唯一理由是任務太小
（一張工單都嫌多），不是「無法平行」。

## Step 1.5: 流程契約（多步驟流程必做）

觸發：工單涉及兩個以上步驟、或用戶要的是「走完一條流程」而非「改好一個東西」。
拆單**之前**先產三欄表：

| 步驟 | 進入方式（誰、從哪來、帶什麼狀態／參數） | 離開方式（每個出口去向，含返回／取消／失敗） |

- **接縫必須逐條指派給具名工單**（跳轉目標、返回路徑、context 參數、狀態延續、錯誤退回哪）。
  沒有 owner 的接縫不准開工。典型失敗：每張單各自驗收都過，合起來流程是斷的。
- 工單**禁止**用「不在本工單範圍」把接縫推掉，除非契約表已指名接手的是哪張。
- 契約表與 spec／設計稿對照一次，缺口當場列出。

## Step 1.6: 結構障礙必須上桌

實作中發現要動既有共用層／已上線功能／寫死的單一情境假設／別人的模組時，
**停下來把取捨擺給用戶決定**：繞過（相容層／過渡層／切窄工單）vs 正面拆開，附成本、風險、建議。
用戶拍板後在 ledger 記 `decision: <議題> → <選擇>（日期）`，受影響工單改寫後才繼續派。
**禁止**自己選擇繞過然後繼續派工——怕動到上線功能是用戶的風險決策，不是 advisor 可以吞下的。

## Step 2: 派工

### 派工前置快照

呼叫 Agent 之前，先記下目標整合分支的**完整** base：`git -C <repo> rev-parse HEAD`。

- ledger 逐張記 `repo + 目標 branch + 完整 SHA`（短 SHA 只用於顯示）
- 串行工單在**前一張 merge 並通過整合測試後重新取得** base
- 預期 base 寫進工單 prompt，要求 implementer 開工前核對，不一致就停下回報
- Step 3 一律用 ledger 的完整 base 算 diff，不准延後補記
- **每次派工有自己的編號** `ticket-N#aK`（K 遞增；重派、換 model、recovery 都算新的一次），
  寫進工單 prompt，要求回報原樣帶回
- 呼叫前先在 ledger 追加 `ticket-N#aK dispatching`，成功再追加 `dispatched`。呼叫報錯或逾時
  **不准直接重派**：先確認這次到底有沒有開成（`git worktree list`，有 `ListAgents` 就一起看），
  開成了就沿用，確定沒開成才以下一個編號重派。直接重派會讓同一張單同時有兩個 agent

```
Agent({
  subagent_type: "general-purpose",
  name: "ticket-1-<簡短功能名>",
  model: "<implementer model>",
  isolation: "worktree",
  prompt: <工單 prompt>
})
```

同一批平行工單在**同一個 message 一起派**。

工單 prompt 必含（implementer 的合約）：
1. 一句話說明這張工單在整個專案中的位置
2. 工單全文（目標、驗收標準、檔案所有權、測試要求、不做什麼）
3. 「你在獨立 worktree 工作，只准動所有權清單內的檔案；需要動清單外檔案就停下回報」
4. 「完成後 commit 到你的 branch，**不要 push、不要 merge**」（commit 格式依專案慣例）
5. 回報格式：`status`（DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED）＋派工編號
   ＋ worktree 絕對路徑 ＋ branch 名 ＋ commit hashes ＋ 驗收證據表 ＋ 疑慮
6. 「不准把主 repo 的依賴目錄（`node_modules`、`.venv` 等）symlink 進 worktree，要在 worktree
   內自己安裝（symlink 會讓型別檢查與測試讀到主樹程式碼，exit 0 卻沒驗到你的改動）。回報前跑
   `find . -maxdepth 2 -type l \( -name node_modules -o -name .venv \)`，必須無輸出」。
   advisor review 時重跑同一條，有輸出就退回，該工單所有綠燈證據作廢。

不要把整個 session 歷史貼進 prompt，fresh agent 只需要工單、會碰到的介面、全域約束。

### 驗收證據表

一條驗收標準一列，不得合併或省略；格式與判定規則見 `references/evidence-table.md`（含結構相依例外）。
核心規則：任一列是 FAIL／UNVERIFIED／PENDING，整張工單不得判通過、不得 merge。

### 處理回報狀態

- **先驗派工編號**：回報的 `ticket-N#aK` 必須等於 ledger 裡目前生效的那一次。對不上或沒帶
  → 不收、不進 review，ledger 追加 `ticket-N#aK stale report ignored`，並確認舊 agent 已終止。
- **DONE / DONE_WITH_CONCERNS** → 進 review（concerns 先讀，涉及正確性先處理）
- **NEEDS_CONTEXT** → 補資訊，送給同一個 agent 繼續
- **BLOCKED** → 缺 context 就補；任務太難就換更強 model 重派；工單太大就拆；計畫有問題就問用戶。
  **絕不**原樣重派同一個 prompt 期待不同結果。
- **沒有回報／null**（agent 被中止或中途死掉）→ **不得**視為 DONE，有 commit 也不得直接 review。
  ① 先確認 agent 是否真的終止，還活著就等或停掉——**絕不**讓第二個 agent 進同一棵 worktree
  （未 commit 的成果會被沖掉，不可恢復）
  ② 用 `git worktree list`、branch 名、ledger 的 base 找回現場：`git status --short`、
  `git log <base>..HEAD`、`git diff <base>..HEAD`、未提交修改、有無動到所有權清單外檔案
  ③ 有可用成果 → 確認原 agent 已終止後派 recovery agent 補完並重交完整回報；沒有 → 從原 base 重派。
  commit 只代表有可恢復成果，不代表完成，不得跳過 DONE → 證據閘門 → review。

### 主動巡視（防放生）

不要假設 agent 一定會回報。advisor 停在原地等，用戶看到的就是「放生」。
- **每 ~10 分鐘巡一輪**；沒有訊息可等時排一個喚醒（見前置條件表）。
- **用事實判定，不等自述**。每輪對每張單取
  `git -C <worktree> log --oneline <base>..HEAD | wc -l` 與 `git -C <worktree> status --short`，跟上輪比：
  有變化 → working，不打擾；無變化且 agent 已 idle → stalled；
  派出超過 3 分鐘仍無 commit 也無未提交修改 → unclaimed（先確認還活著，再送只指向工單的催促）。
- **idle 但沒有最終回報**：① 先看 worktree 與結果檔，有成果直接 review；② 沒成果就催，
  訊息第一行寫「你閒置 N 分鐘、我缺什麼」，**不重貼工單全文**，ledger 記 `nudge #k`；
  ③ 催兩次沒動靜 → ledger 記 `unverifiable` 並告知用戶。沒回應 ≠ 已終止，無法確認終止就繼續巡視，
  不得派第二個 agent 進同一棵 worktree。
- **回報被截斷** → 當下要求補送缺的段落。
- **裁決點**：agent 提問當下就回；問題進來記 `ticket-N question qM: <摘要>`，回覆後記 `qM answered`。
- 外部 verifier 類 agent 的 prompt 明寫「跑完必須立刻回報」，巡視時直接檢查它落的結果檔。

## Step 3: Review（advisor 親自，逐單）

```bash
git -C <worktree-path> log --oneline <base>..HEAD
git -C <worktree-path> diff <base>..HEAD
```
（base = ledger 的完整 SHA，不要用 `HEAD~1`——多 commit 工單會被截斷）

三個 verdict，缺一不可：
1. **Spec 符合度**：驗收標準逐條對，少做、多做（範圍外加料）都是 fail
2. **品質**：正確性、邊界條件、測試是否真的在驗東西、是否違反專案慣例。自問：
   改動依賴的前提在所有執行路徑下都成立嗎？讀取端全找了嗎？兄弟狀態的 producer/consumer 都補了嗎？
3. **端到端親走**（有流程契約的工單專用，是宣告「可以測試了」的前置條件）：
   advisor **本人**把契約表路徑從頭跑到尾（UI 就點過每個出口與返回；CLI／API 就依序執行整條鏈），
   走過的路徑寫進 ledger。不得以 implementer 的截圖或自述代替。走不完就是未過。
   跨工單 E2E 因結構相依須等合併後才走得完 → 適用證據表的具名例外，merge 後**立刻**補跑。

**證據閘門**：沒有證據表、缺列、PASS 列沒有可重現證據、或任一列還是 FAIL／UNVERIFIED／PENDING
→ 直接退回，不開始看 code，也不得 merge。

**內容閘門**：證據閘門只查形式；最常見的假通過是「證據貼了，但那段證據不支持那條驗收項」
（`1 passed` 貼在測別的行為的驗收項底下、stdout 裡其實有 `1 failed` 卻判 PASS）。
advisor 逐列把「驗收項原文＋implementer 判定」與該列證據配對，判 `supports / contradicts /
says_nothing`：`contradicts` 與 `says_nothing` 都退回；拿不準就親自看那一列。
有可用的機械判斷工具時可一次請求問完所有列，但：`supports` 不代表通過、它不判斷證據真偽、
呼叫失敗就跳過這段（其餘閘門不放寬）。

**UI 工單（有設計稿時）**：偏離設計稿的每一條都要有裁決來源（誰、哪天）。
「有記錄」不等於「被授權」，寫得整齊的偏離表更要查來源。

**advisor 複驗**（不是讀 implementer 貼的輸出）：高風險驗收項全部重跑
（安全／權限／金流、併發、資料遷移、跨多檔核心邏輯，逐項判定），其餘至少挑一項**有實質風險**的重跑
——挑最便宜的做形式合規不算數。沒有指令就親走一條手動路徑。跑不出同樣結果 = 未過；
環境跑不起來 = 該項維持 UNVERIFIED 並擋住 merge，不得以自述代替。

### 高風險工單：加一道獨立 verifier

碰到安全／權限／金流、併發、資料遷移、跨多檔核心邏輯的工單，advisor 過關後**再派 fresh-context
verifier** 補一刀（advisor 剛拆完單，容易跟著 implementer 的邏輯走）。
合約、模型選擇、收斂規則見 `references/verifier.md`。低風險工單不需要，多派只是燒錢。

## Step 4: 修正迴圈

Review 未過 → 把 findings 送回**同一個 agent**（保有完整 context，比重派便宜準確；
沒有 SendMessage 時見前置條件表）：
- Findings 要具體：檔案、行為、期望 vs 實際
- 修正後**更新原證據表**，重跑受影響的驗收項
- 修完 advisor **重新 review**，不可因為「它說修好了」就放行
- 同一個 finding 退回 2 次還修不好 → 換更強 model 重派，或停下問用戶

Advisor 全程不動手改 code。手癢想「順手修一下」就是違規。

## Step 5: Merge

- 有相依關係的先 merge 被依賴的；用專案慣例的 merge 方式
- conflict（代表所有權切割漏了）→ 退回該工單 agent 在它的 worktree rebase，advisor 不自己解
- **每 merge 一張跑一次全量測試／typecheck** 再 merge 下一張——單張各自綠不代表合起來綠
- 全部完成後清理：`git worktree remove <path>` + `git branch -d <ticket-branch>`，
  `git worktree list` 確認清完。superseded／canceled 工單的 worktree 同樣要處置
- 若 verifier 走外部 CLI 並在 worktree 內留下常駐背景行程，worktree 刪除後一併回收（見 `references/verifier.md`）
- 禁 `git reset --hard`、禁 force push；push 前確認 remote／branch
- **push 必須有用戶明確同意**：merge 完成 ≠ 可以 push；用戶說過「不要 push 等我通知」就是鐵律

## Step 6: 盯部署

Merge 完成不是結束。依專案部署流程執行並盯到底：
- 部署前確認目標環境（哪個 host／pipeline），不確定就查不要猜
- CI/CD 專案盯到 pipeline 綠
- 部署後**遠端驗證**——打實際環境 endpoint／看實際頁面／查 log；本地測試通過不算數
- 驗證失敗 → **先留現場**（完整 SHA、環境、log、回應、畫面）再查根因；不要先 re-deploy 沖掉現場

**驗證期間凍結**（部署一啟動就生效到驗證收尾）：完整 SHA＋目標環境寫進 ledger 且不准變；
期間不得 merge 別的工單、不得 push、不得重新部署。用戶在驗這次部署時同樣適用。
回報用語：只能說「已部署，遠端驗證了 X、Y」，觀察到什麼說什麼。

## 進度 Ledger（防 compaction 失憶）

長 session 會被壓縮，光靠記憶會重派已完成的工單（最貴的失敗模式）。
開工時建立 ledger（放 scratchpad 或 repo 內 git-ignored 路徑），事件追加一行。格式、詞彙、
三種終態與收工前檢查見 `references/ledger.md`。Compaction 後先讀 ledger + `git log` 再決定下一步，
相信 ledger，不相信記憶。

## 紅線

- 兩張碰同一檔案的工單平行派出
- Advisor 自己寫實作碼、自己解 merge conflict
- 派出去就只等回報，超過 10 分鐘沒巡視（「放生」）
- 沒有完整證據表就開始 review、review 未過就 merge
- 證據表有列是 FAIL／UNVERIFIED／PENDING 卻放行（除非符合結構相依的具名例外）
- 派工不指定 model
- 未經用戶同意就 push
- 部署後沒做遠端驗證就宣布完成；驗證進行中就 merge／push／重新部署
- 工單 agent 動了所有權清單外的檔案而 review 沒抓到
- 多步驟流程沒產流程契約，或契約裡有接縫沒指派 owner；用「不在本工單範圍」推掉接縫
- 碰到結構障礙自己選擇繞過而沒交給用戶決定
- 沒親自走完端到端路徑就跟用戶說「可以測了」
- 同一棵 worktree 同時跑兩個 agent（未 commit 的成果會被沖掉，不可恢復）
- 收下派工編號不符的回報，或 Agent 呼叫結果不明就直接重派
- 把「沒回應」當成「已終止」就派 recovery agent 進同一棵 worktree
- 高風險工單只靠 advisor 自審、跳過獨立 verifier；或讓 verifier 順手改 code
