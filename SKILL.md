---
name: coding-dispatch
description: >
  分解多步驟編程任務，透過 subagent 依序執行，支援 cron 斷點續傳。
  主動用於多步驟實作任務（例如「實作功能 X 需要 A、B、C」、「重構模組 Y 並更新呼叫者」）。
  支援一般 subagent 或 Pi Coding Agent 執行編程任務。
---

# Coding Dispatch

任務分派代理。將複雜編程任務分解為子任務，顯示計畫後依序執行，支援斷點續傳。

---

## Phase 1: 任務分析與分解

1. **理解範圍** — 讀取相關文件（如 `git/CLAUDE.md`）了解專案背景。

2. **分解為 3–7 個子任務** — 每個子任務應：
   - 獨立可完成（基於先前子任務結果）
   - 有明確完成條件
   - 影響 1–3 個檔案
   - 自足到讓新 subagent 可直接執行

3. **立即顯示計畫**（使用此格式）：

```
已將任務切分為 N 個子任務：

Task1: [名稱] — [說明、影響檔案]
Task2: [名稱] — [說明、影響檔案]
Task3: ...

確認後開始執行。
```

4. **等待用戶確認**後繼續。

---

## Phase 2: 建立狀態檔案與 Cron 恢復

### Step 2a — 寫入 `dispatch_state.json`

寫入**專案工作目錄**（被修改程式碼所在的目錄），使用絕對路徑：

```json
{
  "cron_task_id": "dispatch-YYYYMMDD-HHMMSS",
  "working_directory": "/absolute/path/to/project",
  "original_task": "原始用戶請求",
  "accumulated_context": "",
  "tasks": [
    {
      "id": 1,
      "name": "子任務名稱",
      "description": "完整說明",
      "files_to_read": ["path/to/file1"],
      "definition_of_done": ["條件 1", "條件 2"],
      "status": "pending",
      "summary": null
    }
  ]
}
```

Status: `pending` → `in_progress` → `completed` | `failed`

### Step 2b — 註冊 Cron 恢復任務

使用 `mcp__scheduled-tasks__create_scheduled_task`：
- `taskId`: 與 state 檔案相同的 `cron_task_id`
- `cronExpression`: `"*/10 * * * *"`（每 10 分鐘）
- `description`: `"Recovery monitor for dispatch task"`
- `prompt`: 下方恢復提示（用實際 `STATE_FILE_PATH` 替換）

**恢復提示模板**（替換 `STATE_FILE_PATH`）：

```
你是編程分派恢復代理。讀取：STATE_FILE_PATH

1. 解析 JSON 狀態檔案。
2. 若所有任務 `completed`：停用 cron job，結束。
3. 找到第一個非 `completed` 的任務：
   - 若為 `in_progress` 視為 `pending`（重置）
   - 更新狀態為 `in_progress`
4. 構建 subagent 提示（見下方模板）
5. 啟動 subagent（不 run_in_background）
6. 完成後更新狀態、summary、accumulated_context
7. 若全部完成：停用 cron job

## Subagent 選擇

**一般編程任務**：使用 `subagent_type: "general-purpose"`

**Pi Coding Agent 任務**（更適合編程）：
```bash
# 互動模式
bash pty:true workdir:~/project command:"pi 'Your task'"

# 非互動模式
bash pty:true command:"pi -p 'Summarize src/'"

# 指定 provider/model
bash pty:true command:"pi --provider openai --model gpt-4o-mini -p 'Your task'"
```
Pi 支援 Anthropic prompt caching（PR #584, 2026-01），編程任務更高效。
```

### Step 2c — 建立 TodoWrite 追蹤

每子任務一項，初始皆為 `pending`。

---

## Phase 3: 依序執行

依序執行子任務（不使用 `run_in_background`）：

1. **更新 todo** — 標記為 `in_progress`
2. **更新 state 檔案** — status → `"in_progress"`
3. **構建 subagent 提示**（下方模板）
4. **啟動 subagent** — 等待完成
5. **成功**：todo → `completed`，更新 state（status、summary、accumulated_context）
6. **失敗**：重試一次；二次失敗則標記 `failed` 並停止

### Subagent 提示模板

```
## Context
Working directory: {working_directory}
Original task: {original_task}

## Prior Work
{accumulated_context — 每完成子任務一行}

## Your Task: {task name}
{task description}

## Files to Read First
{files_to_read}

## Definition of Done
{checklist}

## Constraints
- 遵循 CLAUDE.md 編程規範（若存在）
- 保持程式碼庫可運作
- 子任務僅用 general-purpose 或 Pi

完成後回覆：
1. 建立/修改的檔案
2. 實作內容
3. 決策
4. 問題或疑慮
```

---

## Phase 4: 完成與清理

1. **停用 cron job** — `mcp__scheduled-tasks__update_scheduled_task` + `enabled: false`

2. **回報用戶**：

```
## Task Complete

所有 N 個子任務已完成。已停用進度追蹤排程。

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
- **State 檔案是唯一真相** — 每次狀態變更都寫入 `dispatch_state.json`
- **自足提示** — 子代理無記憶，所有上下文透過 `accumulated_context` 嵌入
- **僅用 general-purpose 或 Pi** — 不使用其他代理類型
- **依序不並行** — 不使用 `run_in_background`
- **增量進度** — 每子任務保持程式碼庫可運作
- **Cron 清理必要** — 完成後務必停用 scheduled task
