---
name: ralph
description: >
  使用 Ralph 自動化 agent loop 模式來實作功能、建構專案或執行多步驟開發任務。當使用者說「用 ralph 實作」、「幫我建一個 ralph loop」、「自動跑 claude code 完成任務」、「建立 PRD 讓 agent 自動做」、「我想用 ralph 模式」、「設定 ralph 跑這個專案」時，一定要使用此 skill。也適用於任何需要把大型任務拆成多個 user stories、讓 Claude Code 或 Amp 反覆迭代執行直到完成的場景。即使使用者只說「幫我自動實作這個功能」或「讓 AI 自己把這個做完」，也應該觸發此 skill。
user-invocable: true
---

# Ralph Skill（融合強化版）

Ralph 是一個 autonomous agent loop。本 skill 融合了 snarktank/ralph、Wiggum、ralphex、ralph-loop-agent 的核心優點。

---

## 核心設計原則

| 來源 | 採用的優點 |
|---|---|
| snarktank/ralph | PRD 驅動、AGENTS.md 跨迭代記憶、clean context |
| Wiggum | 階段隔離（Plan→Implement→Test→Verify）|
| ralph-loop-agent | token 上限保護、context summarization |
| ralphex | 失敗模式分類、rate limit retry |
| multi-agent-ralph-loop | solution tracking、壓縮記憶 |

---

## 完整工作流程

```
/prd  → tasks/prd-feature.md        ← 規格文件
/ralph → scripts/ralph/prd.json     ← 轉換成 JSON
ralph.sh → 執行 loop                ← 自動實作
```

每個 story 在 loop 內走這五個階段，每個階段獨立成功/失敗：

```
Plan → Implement → Test → Verify → Done
         ↑失敗只重跑這一階段，不從頭來↑
```

---

## 安裝

```bash
# 建立專案
mkdir ~/my-project && cd ~/my-project
git init
mkdir -p scripts/ralph tasks

# 下載 ralph.sh
curl -o scripts/ralph/ralph.sh \
  https://raw.githubusercontent.com/snarktank/ralph/main/ralph.sh
curl -o scripts/ralph/CLAUDE.md \
  https://raw.githubusercontent.com/snarktank/ralph/main/CLAUDE.md
chmod +x scripts/ralph/ralph.sh
```

---

## prd.json 結構（融合版）

```json
{
  "branchName": "ralph/feature-name",
  "tokenBudget": 50000,
  "maxRetriesPerStory": 3,
  "userStories": [
    {
      "id": "US-001",
      "title": "Story 標題",
      "description": "自我完整的描述。不能有『如前所述』。包含：輸入、輸出、檔案位置、驗證方式。",
      "acceptanceCriteria": [
        "具體可驗證的條件",
        "Typecheck passes",
        "Tests pass"
      ],
      "passes": false,
      "dependsOn": [],
      "attempts": 0,
      "failureLog": []
    }
  ]
}
```

**新增欄位說明：**
- `tokenBudget`：整個 session 的 token 上限，超過自動暫停
- `maxRetriesPerStory`：單一 story 最多重試幾次，超過標記為 `needs_human`
- `attempts`：已嘗試次數，ralph.sh 每次迭代自動 +1
- `failureLog`：記錄每次失敗的錯誤摘要，避免重複同樣路徑

---

## AGENTS.md 結構（融合版）

```markdown
# 專案名稱 — Agent Instructions

## 專案目標
一句話說明。

## 技術棧
- 語言、框架、資料庫版本

## 目錄結構
重要資料夾和檔案的用途。

## 規範
- 命名規則
- 特殊限制

## 階段 Checklist（每個 story 必須走完）
### Plan
- [ ] 讀完 prd.json 這個 story 的 description
- [ ] 列出要修改/建立的檔案清單
- [ ] 確認 dependsOn 的 story 已完成

### Implement
- [ ] 按 Plan 階段的檔案清單實作
- [ ] 不要修改 Plan 沒列到的檔案

### Test
- [ ] 寫或更新對應的測試
- [ ] 跑 typecheck：npm run typecheck 或 python -m mypy
- [ ] 跑測試：npm test 或 pytest

### Verify
- [ ] 重新讀 acceptanceCriteria，逐條確認
- [ ] 全部通過才把 passes 改為 true
- [ ] 把這次的學習寫進 AGENTS.md Learnings

## Learnings（自動累積，不要刪）
- 記錄踩過的坑和解法
- 格式：[US-xxx] 問題描述 → 解法

## Solution Tracking（避免重複失敗路徑）
- 記錄每個 story 嘗試過但失敗的方法
- 格式：[US-xxx] 嘗試過：方法A（失敗原因），方法B（失敗原因）
```

---

## 執行指令

```bash
export ANTHROPIC_API_KEY=sk-ant-你的key

# 標準執行（建議從小開始）
./scripts/ralph/ralph.sh --tool claude 5

# 加上 auto-handoff（context 90% 時自動換新 instance）
# 在 ~/.config/amp/settings.json 加：
# { "amp.experimental.autoHandoff": { "context": 90 } }
```

---

## 失敗處理流程

### Story 失敗時，ralph 應該做：

```
失敗次數 < maxRetriesPerStory
   → 把錯誤摘要寫進 failureLog
   → 把失敗方法寫進 AGENTS.md Solution Tracking
   → 換不同方法重試

失敗次數 == maxRetriesPerStory
   → 把 passes 改為 "needs_human"
   → 在 progress.txt 記錄卡關原因
   → 跳過此 story，繼續下一個
   → 結束後通知使用者哪些需要人工介入
```

### 使用者看到 `needs_human` 時：

```bash
# 1. 看卡關原因
cat scripts/ralph/progress.txt

# 2. 看嘗試過的方法
grep "needs_human\|failureLog" scripts/ralph/prd.json

# 3. 修改 description，或把 story 拆得更小
# 4. 把 passes 改回 false，attempts 歸零
# 5. 繼續跑
./scripts/ralph/ralph.sh --tool claude 5
```

---

## Token 保護機制

在 CLAUDE.md 開頭加入：

```markdown
## Token Budget
目前 session 的 token 預算來自 prd.json 的 tokenBudget 欄位。
每完成一個 story，在 progress.txt 記錄消耗的 token 估算。
若預算剩餘不足 10%，完成當前 story 後暫停，等待使用者確認繼續。
```

---

## Story 撰寫黃金規則

### 大小判斷
- 能在 2-3 句話描述清楚 → 適合一個 story
- 需要說「然後...然後...然後...」→ 要拆

### Description 必須包含
1. 輸入是什麼（檔案、資料格式）
2. 輸出是什麼（檔案位置、格式）
3. 用到哪些現有的函式/模組
4. 怎麼驗證完成（跑什麼指令）

### dependsOn 設計
```
資料庫 schema    → dependsOn: []
API endpoint    → dependsOn: ["US-001"]
UI component    → dependsOn: ["US-002"]
整合測試        → dependsOn: ["US-003"]

# 可以並行的 story 共享同一個依賴：
UI component A  → dependsOn: ["US-001"]
UI component B  → dependsOn: ["US-001"]  ← 和 A 並行
```

---

## 進度監控

```bash
# 查看所有 story 狀態
cat scripts/ralph/prd.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
for s in d['userStories']:
    status = s['passes']
    attempts = s.get('attempts', 0)
    print(f'[{status}] {s[\"id\"]} (嘗試:{attempts}x) {s[\"title\"]}')
"

# 查看最新 log
tail -30 scripts/ralph/progress.txt

# 查看哪些需要人工介入
cat scripts/ralph/prd.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
blocked = [s for s in d['userStories'] if s['passes'] == 'needs_human']
for s in blocked:
    print(f'🔴 {s[\"id\"]}: {s[\"title\"]}')
    for log in s.get('failureLog', []):
        print(f'   - {log}')
"
```
