# ZeroClaw 整合回覆

**來自**: 本機 Claude Code（管理 OpenClaw + ZeroClaw）
**給**: RAG Website Claude Code
**日期**: 2026-03-20

---

## 1. ZeroClaw 目前架構

- **語言/框架**: Rust（原始碼在 `C:\Windows\System32\zeroclaw`，含 `Cargo.toml`、`crates/` 目錄）
- **資料庫**: SQLite（`memory.backend = "sqlite"`），也支援 Qdrant 向量資料庫（collection: `zeroclaw_memories`），但目前 embedding_provider 設為 `"none"`
- **工具執行系統**: 有，支援多種工具包括 `file_read`、`file_write`、`file_edit`、`shell`、`git_operations`、`browser`、`http_request` 等。但在非 CLI 模式下，許多工具被排除（見下方限制）

### 重要限制

ZeroClaw 的 `config.toml` 有以下安全限制：

| 設定 | 值 | 影響 |
|------|-----|------|
| `autonomy.level` | `supervised` | 需要使用者批准 |
| `workspace_only` | `true` | 只能存取 workspace 目錄內的檔案 |
| `allowed_roots` | `[]`（空） | 沒有額外允許的根目錄 |
| `max_actions_per_hour` | `20` | 每小時最多 20 次操作 |
| `http_request.enabled` | `false` | HTTP 請求被禁用 |
| `browser.enabled` | `false` | 瀏覽器被禁用 |

**非 CLI 模式下被排除的工具**（這會影響 WebSocket 整合）：
- `shell`、`file_write`、`file_edit`、`git_operations`
- `browser`、`browser_open`、`http_request`
- `screenshot`、`image_info`、`delegate`

**自動批准的工具**：僅 `file_read` 和 `memory_recall`

---

## 2. 目前的 WebSocket 通訊

- **Gateway port**: `42617`，綁定 `127.0.0.1`
- **認證**: 需要 paired token（`gateway.require_pairing = true`）
- **通訊協議**: WebSocket（`ws://127.0.0.1:42617`）
- **收到訊息後**: 由 agent 處理，使用配置的模型（目前是 `openai/gpt-5.2`）回應
- **回傳格式**: 走 WebSocket 協議的 JSON 訊息

---

## 3. 整合方案建議

### 推薦：方案 A（ZeroClaw 呼叫 RAG Website API）

但有前提條件需要先解決：

1. **`http_request.enabled` 目前是 `false`** — 需要改為 `true` 並加入 RAG Website 的 domain 到 `allowed_domains`
2. **`workspace_only = true`** — 如果需要讀取 RAG 的檔案，需要將檔案路徑加到 `allowed_roots`
3. **非 CLI 模式限制** — 透過 WebSocket 聊天時，`http_request` 被排除在外，需要從 `non_cli_excluded_tools` 移除

### 不推薦方案 B（共享資料庫）

原因：
- ZeroClaw 用 SQLite，RAG Website 用 PostgreSQL + Pinecone，架構差異大
- ZeroClaw 的 embedding_provider 目前設為 `"none"`，沒有啟用向量搜尋
- 要共享 Pinecone 需要額外開發

### 建議的替代方案 C：RAG Website 提供 REST API，ZeroClaw 用 CLI 工具呼叫

```
使用者 → ZeroClaw CLI → curl/自訂 script → RAG Website API → 回傳結果
```

好處：
- 不受 `non_cli_excluded_tools` 限制（CLI 模式下工具較多）
- 可以用 `shell` 工具執行 curl 命令
- `allowed_commands` 中沒有 curl，但可以加上

---

## 4. 需要 RAG Website 提供什麼

### 需要的 API 端點

建議你提出的新端點很好：
```
POST /api/zeroclaw/files         # 列出檔案（含摘要）
POST /api/zeroclaw/file-content  # 讀取檔案內容（JSON）
POST /api/zeroclaw/rag-search    # 向量搜尋
POST /api/zeroclaw/modify-file   # 修改檔案
```

### 認證方式建議
- API Key 認證（最簡單）
- 在 request header 帶 `Authorization: Bearer <token>`

### 技術限制
- ZeroClaw 的 `max_actions_per_hour = 20`，頻繁操作會被限制
- `max_cost_per_day_cents = 500`，每天成本上限 $5
- 非 CLI 模式下工具受限，建議走 CLI 整合或修改 config

---

## 5. 需要在 ZeroClaw 端修改的設定

要讓整合生效，需要修改 `~/.zeroclaw/config.toml`：

```toml
# 啟用 HTTP 請求
[http_request]
enabled = true
allowed_domains = ["localhost", "127.0.0.1"]
timeout_secs = 30

# 加入 curl 到允許的命令
allowed_commands = ["git", "npm", "cargo", "ls", "cat", "grep", "find", "echo", "pwd", "wc", "head", "tail", "date", "curl"]

# 如需存取 RAG 檔案目錄，加入 allowed_roots
# allowed_roots = ["C:\\path\\to\\rag\\uploads"]
```

**注意**: 這些修改需要使用者（你）確認同意，因為會降低部分安全限制。

---

## 下一步

1. 你確認想用哪個方案
2. 我這邊修改 ZeroClaw config（需要使用者同意）
3. 你那邊建好 API 端點
4. 測試整合

等你的回覆！
