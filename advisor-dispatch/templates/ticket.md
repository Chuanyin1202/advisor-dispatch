# 工單範本（貼進 Agent 的 prompt；< > 內是要填的）

派工編號：ticket-<N>#a<K>（回報時原樣帶回）
預期 base：<完整 SHA>（開工前 `git rev-parse HEAD` 核對，不一致就停下回報）

## 位置
這張工單是「<專案／功能>」的第 <N> 張，負責 <一句話>。

## 目標
<一句話：做什麼>

## 驗收標準（每條都要可驗證）
1. <例：POST /x 傳重複 position 回 409；測試 test_xxx 通過>
2. <例：從 A 頁點返回回到 B 頁並保留篩選條件（手動，環境：…）>

## 檔案所有權（只准動這些）
- <path/or/dir>

## 測試要求
- 跑：<完整指令>
- 新增：<要新增的測試與覆蓋的行為>

## 不做什麼
- <明確排除的範圍外項目>

## 工作規則
- 你在獨立 worktree 工作，只准動所有權清單內的檔案；需要動清單外檔案就停下回報。
- 依賴自己在 worktree 內安裝，不准 symlink 主 repo 的 node_modules／.venv；回報前跑
  `find . -maxdepth 2 -type l \( -name node_modules -o -name .venv \)`，必須無輸出。
- 完成後 commit 到你的 branch（<專案 commit 慣例>），不要 push、不要 merge。

## 回報格式
status: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
ticket: ticket-<N>#a<K>
worktree: <絕對路徑>
branch: <branch 名>
commits: <hashes>
驗收證據表: <依 evidence-table 格式，一條驗收標準一列，含「已知弱點」列>
疑慮: <有就寫>
