# advisor-dispatch：派工／監工開發模式（Claude Code skill）

版本：v1.1.0

English: [README.md](README.md)

## 用途
讓主 session 當 **advisor**（只規劃、拆單、review、merge、盯部署），實作交給各自 git worktree 裡的
subagent。重點是品質管線，不是「多開幾個 agent」：

- 每張工單必含：目標、驗收標準、檔案所有權、測試要求、不做什麼
- 檔案所有權不重疊才平行；相依就串行
- implementer 交**驗收證據表**，advisor 先過證據閘門，再親自看 diff、重跑高風險項
- 高風險工單加一道獨立 verifier（對抗式、只判定不改 code）
- 定時巡視（防「派完就放生」）、ledger（防 compaction 失憶）、merge 後逐張跑全量測試、部署後遠端驗證

不適用：單線小改動（一張工單都嫌多）。

## 內容
```
advisor-dispatch/
├── SKILL.md                    核心流程（載入時讀這個）
├── references/
│   ├── evidence-table.md       證據表規則與「結構相依」例外
│   ├── verifier.md             獨立 verifier 合約、模型選擇、收斂規則
│   └── ledger.md               進度 ledger 格式與三種終態
└── templates/
    ├── ticket.md               工單範本
    └── evidence-table.md       驗收證據表範本
```
templates 與 references 在 SKILL.md 中被引用，用到時才讀。

## 安裝（兩種方式，建議用 A）

**A. 放進 skills 目錄（建議）**
```bash
# 個人全域
mkdir -p ~/.claude/skills && cp -R advisor-dispatch ~/.claude/skills/
# 或只給某個專案
mkdir -p <專案>/.claude/skills && cp -R advisor-dispatch <專案>/.claude/skills/
```
理由：skill 本身就是一個資料夾加 SKILL.md，複製即用，不需要 manifest、版本發佈或 marketplace；
要改成自己的慣例（commit 格式、測試指令）也直接改檔。

**B. 包成 plugin**
適合要發給多人並持續更新（版本號、一個指令更新）的情況：需要另外加 plugin manifest 與 marketplace 才能分發。
這份副本**沒有**附 manifest，也**未在此環境實測 plugin 安裝流程**；現階段分享給少數人，A 的成本較低。

安裝後重開 Claude Code session，輸入 `/` 應看得到 `advisor-dispatch`（或直接說「幫我派工做 X」讓它依 description 觸發）。

## 前置條件
- 必要：git、可用的 Agent tool（能指定 `isolation: "worktree"` 與 `model`）、Bash
- 有就用、沒有走備援（SKILL.md 開頭有對照表）：`SendMessage`、`ListAgents`、排程喚醒工具、外部模型 CLI 當 verifier
- 專案要有可跑的測試／typecheck 指令，否則證據表大多會落成 UNVERIFIED

## 使用範例
```
幫我用派工模式做：新增訂單匯出 API（CSV）與對應的前端按鈕。
先拆單、給我看檔案所有權切割，我同意再派。實作用 sonnet，安全相關那張加 verifier。
```
預期行為：advisor 先產計畫與工單表 → 你確認 → 記 base SHA → 同一個 message 派出各單 →
巡視 → 逐單收證據表、看 diff、複驗 → 全過後依序 merge 並跑全量測試 → 問你能不能 push。

## 限制
- SKILL.md、references、templates 為英文；本 README 為繁體中文版，英文版見 README.md
- 為求嚴謹，流程偏重；小任務請直接做
- 備援路徑（沒有 SendMessage／ListAgents／排程）在其他 Claude Code 版本是否都可行，**未驗證**
- 未經你明確同意不會 push；部署不受本 skill 管控，依專案自己的部署流程
- 無 hook／腳本強制，規則靠 advisor 自律；紅線清單是提醒不是機制

## 授權
MIT，見 `LICENSE`。
