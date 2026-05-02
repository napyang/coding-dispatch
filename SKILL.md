---
name: coding-dispatch
description: >
  分解多步驟編程任務，透過 subagent 依序執行，支援 cron 斷點續傳。
  主動用於多步驟實作任務（例如「實作功能 X 需要 A、B、C」、「重構模組 Y 並更新呼叫者」）。
  適配 64k context 限制，最大化 subagent 委派以節省主 agent token。
  狀態檔案統一存放於 ~/.hermes/dispatches/，支援 crash 後自動恢復。
---

# Coding Dispatch

任務分派代理。將複雜編程任務分解為子任務，顯示計畫後依序執行，支援斷點續傳。

---

## 狀態檔案架構（兩層設計）

```
~/.hermes/
├── active_dispatches.json     ← 主索引：記錄所有進行中的 dispatch
└── dispatches/
    ├── dispatch-20260502-173400.json   ← 任務狀態檔案
    └── ...
```

### 主索引檔 `~/.hermes/active_dispatches.json`

```json
{
  "dispatches": [
    {
      "task_id": "dispatch-20260502-173400",
      "display_name": "實作 Wishing Feature",
      "project_dir": "/home/nap/git/2026_Wishing_Tree",
      "state_file": "~/.hermes/dispatches/dispatch-20260502-173400.json",
      "current_task": 3,
      "total_tasks": 5,
      "created_at": "2026-05-02T17:34:00",
      "updated_at": "2026-05-02T18:12:00",
      "cron_job_id": null
    }
  ]
}
```

- 任務全部完成後，從 `dispatches` 陣列中移除該筆記錄
- 新 session 啟動時讀取此檔案，自動發現未完成的 dispatch

### 狀態檔案 `~/.hermes/dispatches/{task_id}.json`

```json
{
  "task_id": "dispatch-20260502-173400",
  "display_name": "實作 Wishing Feature",
  "project_dir": "/home/nap/git/2026_Wishing_Tree",
  "original_task": "原始用戶請求",
  "accumulated_context": "",
  "current_task": 3,
  "tasks": [
    {
      "id": 1,
      "name": "子任務名稱",
      "description": "完整說明",
      "files_to_read": ["path/to/file1"],
      "definition_of_done": ["條件 1", "條件 2"],
      "delegate": true,
      "status": "completed",
      "summary": "建立了 core/wishing.py，實作 Wild counter 邏輯",
      "error": null,
      "retry_count": 0
    }
  ]
}
```

Status 狀態機: `pending` → `in_progress` → `completed` | `failed`

---

## 64k Context 節省策略

主 agent context 僅 64k，必須嚴格控制 token 消耗：

- **能丟給 subagent 的就丟** — 檔案讀寫、程式碼實作、script 執行、測試驗證都交給 leaf agent
- **subagent 回傳只留摘要** — 要求 subagent 回覆簡潔，不要大段程式碼貼回主 agent
- **accumulated_context 保持精簡** — 每個已完成的子任務只用 1-2 行摘要，不保留細節
- **主 agent 只做** — 任務拆解、決策判斷、狀態管理、最終整合
- **狀態檔案是斷點** — 每次子任務完成都寫入狀態檔案，crash 後可恢復

---

## Session 啟動檢查

每次新 session 啟動時，執行：

```python
# 讀取主索引檔
read_file("~/.hermes/active_dispatches.json")

# 若有進行中的 dispatch：
#   → 讀取對應狀態檔案
#   → 向用戶報告進度並詢問是否接續
#   → 用戶確認後從中斷點繼續
```

---

## Phase 1: 任務分析與分解

1. **理解範圍** — 讀取相關文件（如 `CLAUDE.md`、`README.md`、專案結構）了解專案背景。

2. **分解為 3–7 個子任務** — 每個子任務應：
   - 獨立可完成（基於先前子任務結果）
   - 有明確完成條件
   - 影響 1–3 個檔案
   - 自足到讓新 subagent 可直接執行

3. **標記 subagent 委派** — 每個子任務標記是否適合交給 subagent：
   - `delegate: true` — 檔案操作、程式碼實作、測試 → 丟給 leaf agent
   - `delegate: false` — 需要與使用者互動、需要主 agent 上下文 → 主 agent 處理

4. **立即顯示計畫**（使用此格式）：

```
已將任務切分為 N 個子任務：

Task1: [名稱] — [說明、影響檔案] (委派)
Task2: [名稱] — [說明、影響檔案] (主 agent)
Task3: ...

確認後開始執行。
```

5. **等待用戶確認**後繼續。

---

## Phase 2: 建立狀態檔案與 Cron 恢復

### Step 2a — 產生 task_id 並寫入狀態檔案

```
task_id = "dispatch-YYYYMMDD-HHMMSS"  # 用當前時間戳
state_file = "~/.hermes/dispatches/{task_id}.json"
```

確保 `~/.hermes/dispatches/` 目錄存在（不存在則建立）。

寫入狀態檔案（所有任務 status = `pending`）。

### Step 2b — 註冊到主索引檔

讀取 `~/.hermes/active_dispatches.json`，新增一筆記錄到 `dispatches` 陣列：

```json
{
  "task_id": "{task_id}",
  "display_name": "[任務簡稱]",
  "project_dir": "/absolute/path/to/project",
  "state_file": "~/.hermes/dispatches/{task_id}.json",
  "current_task": 1,
  "total_tasks": N,
  "created_at": "ISO 時間戳",
  "updated_at": "ISO 時間戳",
  "cron_job_id": null
}
```

若索引檔不存在，建立之。

### Step 2c — 註冊 Cron 恢復任務

使用 `cronjob` 工具建立恢復任務：

```
cronjob(
  action="create",
  name="dispatch-recovery-{task_id}",
  schedule="*/10 * * * *",
  prompt="你是編程分派恢復代理。

  1. 讀取主索引檔 ~/.hermes/active_dispatches.json
  2. 找到 task_id='{task_id}' 的記錄
  3. 若找不到（表示已完成）：用 cronjob(action='remove') 停用此 cron，結束
  4. 讀取狀態檔案 {state_file_path}
  5. 若所有任務 completed：
     - 從主索引檔移除該筆記錄並寫回
     - 用 cronjob(action='remove') 停用此 cron
     - 結束
  6. 找到第一個非 completed 的任務：
     - 若為 in_progress 視為 pending（重置）
     - 更新狀態為 in_progress 並寫回狀態檔案
     - 更新主索引檔的 current_task 和 updated_at
  7. 若該任務 delegate=true：
     - delegate_task(goal=description, context=accumulated_context, toolsets=['terminal','file'])
     - subagent 完成後更新狀態、summary、accumulated_context 並寫回
  8. 若該任務 delegate=false：
     - 標記為 completed，summary='由主 agent 執行（恢復模式跳過）'
     - 繼續下一個任務
  9. 更新 current_task 指向下個 pending 任務
  10. 若全部完成：從主索引檔移除記錄，停用 cron"
)
```

將 cron 回傳的 `job_id` 寫入主索引檔的 `cron_job_id` 欄位。

### Step 2d — 建立 Todo 追蹤

使用 `todo` 工具，每子任務一項，初始皆為 `pending`。

---

## Phase 3: 依序執行

依序執行子任務：

1. **更新 todo** — `todo` 標記為 `in_progress`
2. **更新狀態檔案** — status → `"in_progress"`，`retry_count` + 1，`current_task` 更新
3. **同步更新主索引檔** — `current_task` 和 `updated_at`
4. **根據 delegate 標記選擇執行方式**：

   **若 `delegate: true`** — 啟動 leaf subagent：
   ```
   delegate_task(
     goal=task.description,
     context=build_subagent_context(project_dir, original_task, accumulated_context, task),
     toolsets=['terminal', 'file']
   )
   ```

   **若 `delegate: false`** — 主 agent 直接執行

5. **成功**：
   - todo → `completed`
   - 狀態檔案：status → `completed`，寫入 summary（1-2 行精簡摘要）
   - accumulated_context 追加一行摘要
   - 主索引檔：`current_task` + 1，`updated_at` 更新

6. **失敗**：
   - 重試 1 次（`retry_count < 2`）
   - 二次失敗：標記 `failed`，記錄 error 訊息，跳過該任務繼續後續任務

### Subagent 提示模板（build_subagent_context）

```
## Context
Working directory: {project_dir}
Original task: {original_task}

## Prior Work (completed tasks)
{accumulated_context — 每完成子任務一行摘要}

## Your Task: {task.name}
{task.description}

## Files to Read First
{task.files_to_read}

## Definition of Done
{task.definition_of_done}

## Constraints
- 遵循專案編程規範（若存在 CLAUDE.md 或類似檔案）
- 保持程式碼庫可運作
- 完成後回覆：(1) 建立/修改的檔案 (2) 實作內容摘要 (3) 決策說明 (4) 問題或疑慮
- 回覆請精簡，不要大段貼程式碼
```

---

## Phase 4: 完成與清理

1. **停用 cron job** — `cronjob(action='remove', job_id=索引檔中的 cron_job_id)`

2. **從主索引檔移除記錄** — 寫回 `active_dispatches.json`

3. **更新 todo** — 全部標記 `completed`

4. **保留狀態檔案** — `~/.hermes/dispatches/{task_id}.json` 保留作為完成記錄（不清除）

5. **回報用戶**：

```
## Task Complete

所有 N 個子任務已完成。已清理進度追蹤。

### 變更摘要
1. **Task1** — [檔案、內容]
2. **Task2** — [檔案、內容]

### 修改的檔案
- `path/to/file` — [變更說明]

### 備註
- [實作決策]
- [後續事項]
```

---

## Key Principles

- **先顯示計畫** — 用戶確認前不寫程式碼
- **狀態檔案是唯一真相** — 每次狀態變更都寫入 `~/.hermes/dispatches/{task_id}.json`
- **主索引檔是恢復入口** — 新 session 讀 `~/.hermes/active_dispatches.json` 發現未完成任务
- **自足提示** — subagent 無記憶，所有上下文透過 `accumulated_context` 嵌入
- **依序不並行** — 子任務依序執行，不並行
- **增量進度** — 每子任務保持程式碼庫可運作
- **64k context 警覺** — 主 agent 只做協調，重活丟給 subagent
- **accumulated_context 精簡** — 每任務 1-2 行，不累積冗長內容
- **Cron 清理必要** — 完成後務必移除 cron job + 從主索引檔刪除記錄
- **失敗不阻塞** — 子任務失敗（重試後）標記 failed 並跳過，繼續後續任務
