# Espalio Workflow — 使用指南（安裝版）

> 這份是給**下載安裝檔來用**的人。它說明安裝、連接編輯器、日常用法、排程、設定與常見問題。
>
> 本檔由主 repo 的 `packaging/publish-update.ps1` 於每次發布時自動同步到發布 repo，
> **請勿在發布 repo 直接編輯**——那裡的版本會在下次發布時被覆蓋。
> 要修改請改主 repo 的 `packaging/release-usage.md`。

---

## 0. 兩個產品，先分清楚

| 產品 | 做什麼 | 下載 | 必裝？ |
|---|---|---|---|
| **AI Memory Vault**（記憶核心） | 記憶的寫入、語意搜尋、索引、自我更新 | [AI-Memory-Vault-core-releases](https://github.com/gabrielchen0314/AI-Memory-Vault-core-releases/releases/latest) | 必裝，要先裝 |
| **Espalio Workflow**（本插件，原名 Vault Workflow） | 收工、晨報、日／週／月報與排程、Agent 任務、starter pack、橋接 Profile、同行審查閘門、多模型辯論 | [AI-Memory-Vault-workflow-releases](https://github.com/gabrielchen0314/AI-Memory-Vault-workflow-releases/releases/latest) | 選用 |

只要記憶功能的人只裝核心就好。要個人工作流才裝本插件，而且**先裝核心，再裝插件**。

下文的命令都用完整路徑。為了好讀，先設一個變數：

```powershell
$Wf = "C:\Program Files\Vault Workflow\vault-workflow.exe"   # 安裝時改過目錄就換成實際位置
& $Wf --version
```

資料目錄是 `%APPDATA%\AI-Memory-Vault`（與核心共用）；`<Vault>` 指你的知識庫資料夾。

## 1. 安裝

需求：Windows 10 以上、64 位元；記憶核心 AI Memory Vault **5.0 以上**已安裝並在執行。
插件的安裝檔會檢查核心版本，沒裝或低於 5.0 就拒絕安裝。

執行 `Vault-Workflow-Setup-v<版本>-<commit>.exe`，預設安裝到 `C:\Program Files\Vault Workflow`。
**保留「登入時自動啟動 Espalio Workflow daemon」的勾選**——插件的常駐服務（`127.0.0.1:8766`）與排程都跑在它裡面，
編輯器也經由它連線。

裝完之後：

| 項目 | 內容 |
|---|---|
| 開始功能表 | **Vault Workflow**（資料夾沿用原名）底下有「首次設定」「排程管理」「從舊版遷移（預覽）」「解除安裝」 |
| 常駐排程 | `VaultWorkflow-Daemon-AtLogon`（登入時啟動）、`VaultWorkflow-Daemon-Watchdog`（每 5 分鐘看護） |
| 環境變數 | `VAULT_WORKFLOW_HOME`（安裝目錄）、`VAULT_WORKFLOW_UPDATE_SOURCE`（更新來源） |

安裝程式不會把執行檔加進 PATH。

## 2. 首次設定

裝完之後分兩條路，**只走其中一條**。

### A. 全新使用者（沒用過 4.x）

開始功能表 → **Vault Workflow → 首次設定**，依序問：

1. **Vault 路徑**——知識庫要放哪
2. **回應語言**
3. **使用者名稱與信箱**
4. **所屬組織**——可以多個；名稱只能用小寫英數、`-`、`_`
5. **LLM 供應商**
6. **Starter pack**——預設全不勾，見[第 8 節](#8-starter-pack)

重跑也安全：每一題按 Enter 保留方括號內的現值，其他設定不會動。
之後要改單一段：`& $Wf --setup-section user`（可用 `vault`／`user`／`org`／`llm`／`language`／`packs`）。

### B. 從 AI Memory Vault 4.x 升級

先裝好核心 5.0 與本插件，然後**完全結束 Claude 桌面 App 與 Antigravity**，從外部開的 PowerShell 執行一鍵遷移：

```powershell
# 先預覽（什麼都不改）
powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Program Files\Vault Workflow\scripts\migrate-to-split.ps1" -DryRun
# 確認無誤後正式執行
powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Program Files\Vault Workflow\scripts\migrate-to-split.ps1"
```

它會遷移設定檔、把各編輯器的 MCP 設定與收工／晨報排程改接插件。每處改動都記錄在
`%APPDATA%\AI-Memory-Vault\migrate-backup-<時間>\`，並附還原腳本 `restore-migration.ps1`。

- **結束碼 2**：有設定檔因為 App 還開著而先跳過了。關掉那個 App 後重跑即可，已改好的不會重複改。
- 不需要再跑「首次設定」，遷移會保留你原本的設定。

## 3. 接上編輯器

編輯器的 MCP 設定指向插件的 gateway。它是每個編輯器 session 一個的輕量行程，記憶功能由它轉給核心：

```
C:\Program Files\Vault Workflow\gateway\vault-workflow-gateway.exe
```

**Claude Code**

```powershell
claude mcp add --scope user ai-memory-vault -- "C:\Program Files\Vault Workflow\gateway\vault-workflow-gateway.exe"
```

**VS Code**（`%APPDATA%\Code\User\mcp.json`）

```json
{
  "servers": {
    "ai-memory-vault": { "type": "stdio", "command": "C:/Program Files/Vault Workflow/gateway/vault-workflow-gateway.exe" }
  }
}
```

**Cursor**（`%USERPROFILE%\.cursor\mcp.json`）、**Claude 桌面 App**（`%APPDATA%\Claude\claude_desktop_config.json`）、
**Antigravity**（`%USERPROFILE%\.gemini\antigravity\mcp_config.json`）

```json
{
  "mcpServers": {
    "ai-memory-vault": { "command": "C:/Program Files/Vault Workflow/gateway/vault-workflow-gateway.exe" }
  }
}
```

**Codex**（`%USERPROFILE%\.codex\config.toml`）

```toml
[mcp_servers.ai-memory-vault]
command = 'C:\Program Files\Vault Workflow\gateway\vault-workflow-gateway.exe'
```

> 注意：Claude 桌面 App 與 Antigravity **開著時會把設定檔整份寫回**，改之前要先完全結束（系統匣 → 結束）。

接好後重開編輯器 session，工具清單會出現 `session_bootstrap`、`search_vault` 等插件工具。

Espalio 目前尚未公開發布，公開後會在這裡補上取得方式。

**機器由 Espalio 管理時**：編輯器改由 Espalio 的單一入口（MCP server 名稱 `espalio`）連線，不需要再加上面的 `ai-memory-vault`。
這時本插件的工具名會帶 `workflow_` 前綴（例如 `workflow_session_bootstrap`），核心的工具帶 `memory_` 前綴。
標為不可逆的工具（`task`、`note_delete`、`note_admin`）照樣列在清單，但每次呼叫都要在 Espalio 的確認視窗核准；
運維工具沒有向 Espalio 宣告，經這個入口永遠看不到。

## 4. 日常使用

下面的呼叫是在編輯器裡請 AI 做的事；你通常用自然語言說，AI 會呼叫對應的工具。

### 進場

每個 session 開始時呼叫一次，取得導航、Agent 定義、規則、高信心直覺、Skill 清單與守門狀態：

```
session_bootstrap(agent="@CodeReviewer", task_keywords="這次要做的事")
```

### 收工

收工把當天的對話登錄進 Vault、回顧直覺卡片、更新各專案的 `status.md` 與交接檔、寫每日回顧，最後提交。

**前提**：先安裝 `end-of-day` starter pack（見[第 8 節](#8-starter-pack)）。沒裝時收工會明確拒絕執行，不會偷偷跑一套你沒同意的流程。

| 方式 | 作法 |
|---|---|
| 在編輯器裡 | 跟 AI 說「收工」，它會呼叫 `end_of_day`。工作交給常駐服務在背景執行，立即回傳一個 run 編號 |
| 命令列 | `& $Wf --end-of-day` |
| 排程 | 見[第 7 節](#7-排程) |

查詢與補救：

```powershell
& $Wf --end-of-day --status          # 每個步驟的狀態、耗時、失敗原因
& $Wf --end-of-day --resume          # 續跑上一次沒跑完的
& $Wf --end-of-day --resolve <項目>   # 人處理完後，把待處理項目移出
& $Wf --end-of-day-mode status       # 目前的收工模式
& $Wf --end-of-day-mode legacy       # 退回以單一 AI session 跑完整份流程的舊模式
```

收工失敗時會跳通知，隔天的晨報最前段也會寫出原因與修法。最常見的是 AI CLI 的登入過期：執行 `claude` 重新登入後補跑即可。

### 晨報

晨報列出所有專案未完成的待辦，加上前一天收工的告警與更新提醒，寫到 `<Vault>\personal\reviews\daily\<日期>-brief.md`。
手動產生：`& $Wf --task morning-brief --headless`。

### 報告、待辦與專案

```
generate_report(kind="review", scope="personal", period="daily")     # 每日總回顧；period 可用 weekly、monthly
generate_report(kind="project_status", organization="myorg", project="myproj")
project(action="list")                                               # 列出組織與專案
project(action="create", organization="myorg", project="newproj")    # 建立新專案
todo(action="add", file_path="workspaces/myorg/projects/myproj/status.md", todo_text="…")
knowledge_pipeline(action="check_pending")                           # 哪些對話還沒寫進 Vault
```

寫入工具遇到不存在的專案會先擋下並回問，不會自動建立。

## 5. 工具一覽

預設 29 支。打錯 `action` 會回錯誤並列出合法值。

| 類別 | 工具 | 用途 |
|---|---|---|
| 進場 | `session_bootstrap` | 一次取得這個 session 需要的脈絡 |
| | `dispatch_agent`、`load_skill` | 載入 Agent 行為定義、讀 Skill 全文 |
| | `agent_admin`、`skill_admin` | Agent 與 Skill 的清單與草稿 |
| | `select_model`、`jarvis_status` | 模型層級建議、外部觀察系統狀態 |
| 記憶 | `search_vault`、`grep_vault`、`read_note` | 語意搜尋、精確搜尋、讀筆記 |
| | `write_note`、`edit_note`、`sync_vault` | 寫入、局部修改、同步索引 |
| | `search_instincts`、`instinct` | 搜尋直覺卡片、卡片的建立／確認／糾正／晉升 |
| | `memory_stack` | 分層記憶的內容與 token 使用量 |
| 筆記管理 | `note_list`、`note_write`、`note_delete` | 列出；搬移或批次寫入；刪除（不可逆） |
| | `note_admin` | 已棄用，下一個次版號移除，改用上面三支 |
| 工作流 | `log_ai_conversation` | 登錄對話 |
| | `project`、`todo` | 專案與待辦 |
| | `generate_report` | 日／週／月報、月度復盤、專案進度與狀態 |
| | `knowledge_pipeline` | 知識萃取：找候選、待登錄檢查、產生知識卡片 |
| | `scheduler`、`task` | 內建排程任務；動態任務與 Agent 任務 |
| | `end_of_day` | 收工 |
| | `create_pull_request` | 在 GitHub 開 PR（需要設定 `GITHUB_TOKEN`） |

另有 4 支運維工具（`index_admin`、`vault_doctor`、`git_admin`、`update_admin`），預設不顯示；
把 `config.json` 的 `mcp.expose_admin_tools` 設為 `true` 並重啟常駐服務後才會出現。

## 6. 命令列

| 命令 | 說明 |
|---|---|
| `& $Wf --setup` ／ `--setup-section <區段>` | 完整設定精靈；只設定一個區段 |
| `& $Wf --end-of-day` | 收工。另有 `--status`、`--resume`、`--resolve`、`--dry-run` |
| `& $Wf --end-of-day-mode dag\|legacy\|status` | 切換或查看收工模式 |
| `& $Wf --task <id> --headless` | 執行一個排程任務，如 `daily-summary`、`morning-brief` |
| `& $Wf --list-tasks` | 列出可執行的排程任務 |
| `& $Wf cli` | 互動式命令列；`--menu` 為選單模式 |
| `& $Wf --usage-report --days 7` | 近 7 天的 AI 用量統計（唯讀） |
| `& $Wf --check-update` ／ `--apply-update` | 檢查新版；下載並安裝 |
| `& $Wf --version` | 版本 |

## 7. 排程

排程有三種，各管各的。

**常駐服務內建的排程**：每日摘要（22:00）、近期脈絡（22:30）、每 4 小時的對話提取、每週一的週報與 AI 週報、
每月 1 日的月報、AI 月報、晉升掃描、架構檢查與月度復盤等。個別任務可在設定檔的
`extensions.workflow.scheduler.jobs.<任務 id>` 關掉（`"enabled": false`）或改時間（`"cron"`），改完要重啟常駐服務。

**Windows 排程**（收工、晨報、Vault 自動提交）不會在安裝時自動註冊，要用的話執行：

```powershell
& "C:\Program Files\Vault Workflow\scripts\register-vault-tasks.ps1"                   # 註冊
& "C:\Program Files\Vault Workflow\scripts\register-vault-tasks.ps1" -DailyAt "18:30"  # 改收工時間
& "C:\Program Files\Vault Workflow\scripts\register-vault-tasks.ps1" -Unregister       # 移除
```

| 排程 | 預設時間 | 做什麼 |
|---|---|---|
| `Vault-EndOfDay-Daily` | 每天 19:00 | 收工 |
| `Vault-MorningBrief-Daily` | 每天 08:30 | 晨報 |
| `Vault-AutoCommit-OnLock` | 鎖定工作站時，以及每 2 小時 | 提交、拉取並推送 Vault 的差異（Vault 是 git repo 時才有用） |

從 4.x 遷移的機器，這些排程由遷移腳本改接插件，不用重新註冊。
機器由 Espalio 管理時，收工、晨報、自動提交已由 Espalio 排程，不要再執行 `register-vault-tasks.ps1`，否則會重複執行。

**選用排程**（清理殘留的 AI CLI 行程、資源監看等，預設不裝）：

```powershell
& "C:\Program Files\Vault Workflow\scripts\register-optional-tasks.ps1" -List
```

開始功能表的「排程管理」可以查看排程、立即執行一次、看 log 與診斷；新增與刪除排程需要系統管理員。

### Agent 任務

Agent 任務讓排程在指定時間用 AI CLI 無人值守地做一件事，產出寫進 Vault。最簡單的做法是跟 AI 說要排什麼，
它會建立 `<Vault>\automation\tasks\<id>.md`。幾個規則：

- 任務以「產出檔存在且非空」判定成功；產出已存在就不重跑。
- 允許使用的工具要列白名單。
- cron 的最小間隔是 5 分鐘，星期用名稱（如 `mon-fri`）。
- 連續失敗 3 次會自動停用，並在晨報告警。
- 新增或修改任務後要**重啟常駐服務**才會重新排入。

## 8. Starter pack

全新安裝的 Vault 沒有任何預設的風格、角色或收工流程。這些都裝在 starter pack 裡，要你明確選擇才會進來。

| Pack | 內容 | 裝了之後 |
|---|---|---|
| `coding-style` | 命名與格式規範、各語言的 style skill | 寫程式時的風格守門開始作用 |
| `agent-roles` | 15 個 Agent 行為定義與任務分派流程（需要先裝 `coding-style`） | `dispatch_agent` 有完整名單 |
| `end-of-day` | 收工流程 | 收工與收工排程可用 |

```powershell
& $Wf --setup-section packs     # 互動式勾選安裝或移除
```

三個 pack 裝的都是作者本人的工作方式，作為可運作的範例：可以直接用、可以改，也可以完全不裝、自己寫一套。

## 9. 設定檔

設定檔是 `%APPDATA%\AI-Memory-Vault\config.json`，與核心共用。插件的設定全部放在 `extensions.workflow` 底下。
API 金鑰不放設定檔，設定檔只記環境變數的名稱。

常用的鍵（都在 `extensions.workflow` 底下）：

| 鍵 | 預設 | 說明 |
|---|---|---|
| `user.name`、`user.organizations` | 空 | 使用者名稱、所屬組織 |
| `llm.provider` | `ollama` | LLM 供應商 |
| `llm.locality_keywords` | `[]` | 隱私關鍵字。任務內容或路徑含這些字就強制用本機模型。組織名會自動併入 |
| `llm.cloud_ok_orgs` | `[]` | 允許把內容送到雲端模型的組織。不在這裡的組織工作區一律用本機模型 |
| `headless_cli.provider` | `claude` | 收工與 Agent 任務預設使用的 AI CLI |
| `headless_cli.claude_command` 等 | 同名 | 各 AI CLI 的執行檔名稱；不在 PATH 時填完整路徑 |
| `scheduler.jobs.<id>.enabled`、`.cron` | 內建值 | 內建排程的開關與時間 |
| `events.end_of_day_skill` | 空 | 收工要執行的 skill。空的時候收工拒絕執行 |
| `packs.installed` | `[]` | 已安裝的 starter pack |
| `inventory.project_roots` | `{}` | `"組織/專案"` → 本機專案資料夾。要登記到專案資料夾本身，不要登記上層目錄 |
| `inventory.directory_aliases` | `{}` | 本機路徑 → 組織與專案。資料夾名稱與 Vault 專案名對不上時用 |
| `agent_task.disabled_task_ids` | `[]` | 這台機器不自動排程的 Agent 任務 |

從舊版升上來的機器，另外檢查這三個鍵：`llm.locality_keywords`（補上專案代號等隱私關鍵字）、
`inventory.schedule_owners`（其他工具的排程名 → `null`）、`inventory.architecture_status_note`
（架構檢查要把待辦寫進哪個 `status.md`）。不填不會壞，只是對應的檢查會提示。

## 10. 同行審查與辯論

**計畫檔的同行審查閘門**是一支 Claude Code 的 Stop hook：在一個 session 裡寫過 `.plans\` 底下的檔案，
而檔尾沒有有效的審查簽章時，收尾會被擋下，要求先做同行審查。閘門只認簽章，不認是誰審的。
簽章綁定當下的內容，內容改過就要重新審查；只是打勾或補上完成證據不會讓審查失效。

擋下之後預設要執行的審查 skill 叫 `claude-peer-review`（由一個全新脈絡、換成審查角色的 AI 逐輪審查，通過才簽章）。
這個 skill **不隨插件出貨**：要用這道閘門，請先把自己的 `claude-peer-review.md` 放進
`<Vault>\workspaces\_global\skills\`，否則擋停訊息會指向一個不存在的 skill。

**多模型辯論**是一個獨立的 MCP server：多個 AI CLI 針對同一議題各自獨立提案、交叉反駁多輪，收斂後合成一份共識文件。
要用的話另外註冊一個 MCP server，例如 Claude Code：

```powershell
claude mcp add --scope user ai-debate -- "C:\Program Files\Vault Workflow\debate\vault-workflow-debate.exe"
```

之後直接用自然語言請 AI 開一場辯論，指定議題與結果要寫到哪個檔。至少要有 2 個已登入可用的 AI CLI。
預設的參與者是 `codex` 與 `claude`。

## 11. 自動更新

插件每 6 小時檢查一次發布 repo 的 `latest.json`，有新版時跳一次通知。
手動更新：`& $Wf --apply-update`（下載、驗證雜湊、啟動安裝檔）。插件與核心的更新各自獨立。

每一版改了什麼，看該版 [Release](../../releases) 頁面的說明。

## 12. 解除安裝

從「設定 → 應用程式」解除安裝 Espalio Workflow。常駐服務的兩支排程會移除；其他還指向插件目錄的排程
（收工、晨報、git 自動提交等）會被**停用**而不是刪除，清單寫在
`%APPDATA%\AI-Memory-Vault\workflow-uninstall-disabled-tasks.txt`，重裝後可以再啟用。
你的筆記與設定（`%APPDATA%\AI-Memory-Vault\`、Vault 資料夾）不會被刪。

## 13. 常見問題

**Q：編輯器連不上？**
依序確認兩個服務：`curl http://127.0.0.1:8766/version`（插件）、`curl http://127.0.0.1:8765/version`（核心）。
插件沒回應時，看護排程每 5 分鐘會自己把它拉回來；想立刻恢復就執行
`Start-ScheduledTask -TaskName VaultWorkflow-Daemon-AtLogon`。核心沒回應見核心發布 repo 的 USAGE.md。

**Q：工具回「核心未執行」？**
記憶核心沒在服務。把核心拉起來即可，插件不用重啟。

**Q：改了 Claude 桌面 App 的設定，重開又變回去？**
App 開著時會把記憶體裡的舊設定寫回。完全結束後再改，或重跑遷移腳本。

**Q：插件安裝檔說找不到記憶核心？**
先到 [AI-Memory-Vault-core-releases](https://github.com/gabrielchen0314/AI-Memory-Vault-core-releases/releases/latest) 安裝核心 5.0 以上。

**Q：收工一執行就結束，什麼都沒做？**
還沒安裝 `end-of-day` starter pack。執行 `& $Wf --setup-section packs`。

**Q：收工失敗了怎麼辦？**
看通知或隔天晨報寫的原因。修好後 `& $Wf --end-of-day --resume` 續跑；`--status` 可以看卡在哪一步。

**Q：新增的 Agent 任務沒有被排進去？**
新增、改時間或開關之後要重啟常駐服務。

**Q：命令以結束碼 75 結束？**
記憶核心沒在服務（沒啟動，或正在啟動、重啟）。先確認核心在執行再重試。

**Q：排程觸發時閃出視窗？**
執行 `powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Program Files\Vault Workflow\scripts\use-hidden-launcher.ps1" -WhatIf`
預覽，確認後拿掉 `-WhatIf`。改之前每支排程都會先匯出備份。

**Q：log 在哪裡？**
都在 `%APPDATA%\AI-Memory-Vault\`：`workflow-daemon-watchdog.log`（常駐服務看護）、`end-of-day.log` 與
`end-of-day-runs\`（收工）、`update-install.vault-workflow.log`（更新安裝）。

---

<sub>問題回報請到發布 repo 的 [Issues](../../issues)，附上兩個服務的 `/version` 回應與相關的 log。</sub>
