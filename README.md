# ralph-enhanced

> 融合五個開源工具最佳實踐的 Ralph autonomous agent loop 強化版 skills。  
> 讓 Claude Code 自動把你的想法實作完，失敗時不盲目重試，token 不浪費。

基於 [snarktank/ralph](https://github.com/snarktank/ralph)，融合了 [Wiggum](https://wiggum.app)、[ralphex](https://github.com/umputun/ralphex)、[ralph-loop-agent](https://github.com/vercel-labs/ralph-loop-agent)、[multi-agent-ralph-loop](https://github.com/alfredolopez80/multi-agent-ralph-loop) 的核心優點。

---

## 這是什麼

Ralph 是一種 autonomous coding 模式：你寫規格，AI 自己把程式碼做完。

原版 ralph 的問題是：一個 story 失敗就從頭重跑，token 一直燒，也不知道為什麼卡住。

**ralph-enhanced** 解決這三個痛點：

- **失敗不從頭來** — 階段隔離，只重跑失敗的那個階段
- **不盲目重試** — 記錄每次失敗原因，下次換不同方法
- **token 有上限** — 設定預算，快用完時自動暫停

---

## 功能

### `/prd` — 規格生成

把你的想法轉成結構化的 PRD markdown 文件。

- 用 A/B/C/D 問答快速釐清需求
- 產出標準格式的 User Stories + Acceptance Criteria
- 自動存到 `tasks/prd-[feature-name].md`
- 不會開始實作，只產規格

### `/ralph` — PRD 轉 JSON

把 `/prd` 產出的 markdown 轉成 `ralph.sh` 可執行的 `prd.json`。

- 自動推斷 `dependsOn`（DB schema 先於 API，API 先於 UI）
- 所有 story 預設 `passes: false`
- Description 確保自我完整（新 instance 不需要上下文就能執行）
- 存到 `scripts/ralph/prd.json`

### 階段隔離執行

每個 story 走五個獨立階段，失敗只重跑該階段：

```
Plan → Implement → Test → Verify → Done
```

- Plan：列出要改的檔案，不會亂改其他地方
- Implement：按計畫實作
- Test：跑 typecheck + 自動測試
- Verify：逐條確認 acceptance criteria

### `needs_human` 機制

超過 `maxRetriesPerStory` 次失敗，自動標記跳過，繼續跑其他 story，最後統一報告哪些需要人工介入。不讓一個卡住的 story 阻塞整個 loop。

### Solution Tracking

每次失敗都記錄嘗試過的方法和失敗原因，下一輪不走同一條死路。

### Token 預算保護

在 `prd.json` 設定 `tokenBudget`，快用完時完成當前 story 後暫停，等你確認再繼續。

---

## 安裝

### 安裝到 Claude Code（全域）

```bash
# 下載
git clone https://github.com/你的帳號/ralph-enhanced.git
cd ralph-enhanced

# 安裝 skills
cp -r skills/prd ~/.claude/skills/
cp -r skills/ralph ~/.claude/skills/
```

### 安裝到 Amp（全域）

```bash
cp -r skills/prd ~/.config/amp/skills/
cp -r skills/ralph ~/.config/amp/skills/
```

### Claude Code Marketplace（最快）

```
/plugin marketplace add 你的帳號/ralph-enhanced
/plugin install ralph-enhanced-skills@ralph-marketplace
```

---

## 使用方式

### 完整流程

```
Step 1   /prd 描述你想做什麼
Step 2   回答 A/B/C/D 問題
Step 3   /ralph 把 PRD 轉成 JSON
Step 4   跑 ralph.sh
Step 5   等它做完
```

### Step 1：在專案裡設定 ralph.sh

```bash
mkdir ~/my-project && cd ~/my-project
git init
mkdir -p scripts/ralph tasks

curl -o scripts/ralph/ralph.sh \
  https://raw.githubusercontent.com/snarktank/ralph/main/ralph.sh
curl -o scripts/ralph/CLAUDE.md \
  https://raw.githubusercontent.com/snarktank/ralph/main/CLAUDE.md
chmod +x scripts/ralph/ralph.sh
```

### Step 2：用 `/prd` 產生規格

在 Claude Code 裡：

```
/prd 幫我做一個 URL 縮短服務，需要短連結建立、跳轉、點擊統計
```

回答問題後產出 `tasks/prd-url-shortener.md`。

### Step 3：用 `/ralph` 轉換成 JSON

```
/ralph 把 tasks/prd-url-shortener.md 轉成 prd.json
```

產出 `scripts/ralph/prd.json`。

### Step 4：執行

```bash
export ANTHROPIC_API_KEY=sk-ant-你的key

./scripts/ralph/ralph.sh --tool claude 10
```

---

## prd.json 格式

```json
{
  "branchName": "ralph/feature-name",
  "tokenBudget": 50000,
  "maxRetriesPerStory": 3,
  "userStories": [
    {
      "id": "US-001",
      "title": "Story 標題",
      "description": "自我完整的描述，包含輸入、輸出、檔案位置、驗證方式。",
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

與原版 ralph 的差異：

| 欄位 | 原版 | 強化版 |
|---|---|---|
| 進度追蹤 | `status: "pending/done"` | `passes: false/true/"needs_human"` |
| 失敗記錄 | 無 | `failureLog: []` |
| 重試次數 | 無上限 | `maxRetriesPerStory` |
| Token 控制 | 無 | `tokenBudget` |
| 嘗試次數 | 無 | `attempts` |

---

## 進度監控

```bash
# 查看所有 story 狀態
cat scripts/ralph/prd.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
for s in d['userStories']:
    print(f'[{s[\"passes\"]}] {s[\"id\"]} (嘗試:{s.get(\"attempts\",0)}x) {s[\"title\"]}')
"

# 查看哪些需要人工介入
cat scripts/ralph/prd.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
blocked = [s for s in d['userStories'] if s['passes'] == 'needs_human']
for s in blocked:
    print(f'🔴 {s[\"id\"]}: {s[\"title\"]}')
    for log in s.get('failureLog', []):
        print(f'   └─ {log}')
"

# 查看 log
tail -30 scripts/ralph/progress.txt
```

---

## 遇到 `needs_human` 時

```bash
# 1. 看卡關原因
tail -50 scripts/ralph/progress.txt

# 2. 修改 prd.json 裡的 description（寫更具體）
#    或把那個 story 拆成更小的兩個

# 3. 重置狀態
#    把 passes 改回 false，attempts 改回 0，failureLog 清空

# 4. 繼續跑
./scripts/ralph/ralph.sh --tool claude 10
```

---

## Auto-handoff（大專案推薦）

Context 用到 90% 時自動建新 instance 繼續，不會因為 context 爆掉而中斷。

在 `~/.config/amp/settings.json` 加入：

```json
{
  "amp.experimental.autoHandoff": {
    "context": 90
  }
}
```

---

## 致謝

這個專案融合了以下開源工具的設計：

- [snarktank/ralph](https://github.com/snarktank/ralph) — 原始 PRD 驅動 loop 架構
- [Wiggum CLI](https://wiggum.app) — 階段隔離執行模型
- [vercel-labs/ralph-loop-agent](https://github.com/vercel-labs/ralph-loop-agent) — Token 預算與 cost 控制
- [umputun/ralphex](https://github.com/umputun/ralphex) — 失敗模式分類與 retry 邏輯
- [alfredolopez80/multi-agent-ralph-loop](https://github.com/alfredolopez80/multi-agent-ralph-loop) — Solution tracking 與記憶層設計

原始 Ralph 模式由 [Geoffrey Huntley](https://ghuntley.com/ralph/) 提出。

---

## License

MIT
