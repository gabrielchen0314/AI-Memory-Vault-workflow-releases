# Vault Workflow — AI Memory Vault 的個人工作流插件（安裝版發布）

在 [AI Memory Vault 記憶核心](https://github.com/gabrielchen0314/AI-Memory-Vault-core-releases) 之上，
加上一整套個人工作流：收工整理、晨報、日／週／月報與排程、Agent 任務、starter pack、橋接 Profile 切換、Ai-Debate。

> 本 repo 只放**安裝檔與更新資訊**（Release、`latest.json`），不放原始碼。
> 本檔由主 repo 的 `packaging/release-readme.md` 於每次發布時同步，請勿在這裡直接編輯。

---

## 同一組的發布 repo

| Repo | 內容 | 需要嗎 |
|---|---|---|
| [AI-Memory-Vault-core-releases](https://github.com/gabrielchen0314/AI-Memory-Vault-core-releases) | 記憶核心 5.0 起 | ✅ 必裝，**要先裝** |
| **AI-Memory-Vault-workflow-releases**（本 repo） | Vault Workflow 插件 | 選用 |
| [AI-Memory-Vault-releases](https://github.com/gabrielchen0314/AI-Memory-Vault-releases) | 4.x 舊版，**已凍結、不再更新** | ❌ |

## 系統需求

- Windows 10 以上、64 位元
- **記憶核心 AI Memory Vault 5.0 以上**已安裝並在執行（插件的安裝檔會檢查，沒有或太舊就拒絕安裝）

## 安裝

到 [Releases](../../releases/latest) 下載 `Vault-Workflow-Setup-v<版本>.exe` 並執行，預設裝到 `C:\Program Files\Vault Workflow`。
保留「登入時自動啟動 vault-workflow daemon」的勾選——插件的常駐服務（`127.0.0.1:8766`）與排程都跑在它裡面。

裝完之後分兩條路，**只走其中一條**：

### A. 全新使用者（沒用過 4.x）

1. 開始功能表 → **Vault Workflow → 首次設定**：Vault 路徑、使用者、組織、LLM、starter pack。
   每題按 Enter 保留方括號內的現值，不會動到核心的其他設定。
2. 編輯器改接插件的 gateway（見下方〈連接編輯器〉）。

### B. 從 AI Memory Vault 4.x 升級

先裝好核心 5.0 與本插件，然後**完全結束 Claude 桌面 App 與 Antigravity**，從外部開的 PowerShell 執行一鍵遷移：

```powershell
# 先預覽（什麼都不改）
powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Program Files\Vault Workflow\scripts\migrate-to-split.ps1" -DryRun
# 確認無誤後正式執行
powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Program Files\Vault Workflow\scripts\migrate-to-split.ps1"
```

它會遷移設定檔、把各編輯器的 MCP 設定與收工／晨報排程改接插件，每處改動都記錄並產生還原腳本
（`%APPDATA%\AI-Memory-Vault\migrate-backup-<時間>\restore-migration.ps1`）。

- **結束碼 2**：有設定檔因為 App 還開著而先跳過了——關掉那個 App 後重跑即可，已改好的不會重複改。
- 不需要再跑「首次設定」，遷移會保留你原本的設定。

## 連接編輯器

編輯器的 MCP 設定指向 gateway（每個編輯器 session 一個輕量行程；記憶功能由它轉給核心）：

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

> ⚠️ Claude 桌面 App 與 Antigravity **開著時會把設定檔整份寫回**，改之前要先完全結束（系統匣 → 結束）。
> 接好後重開編輯器 session，工具清單會出現 `session_bootstrap`、`search_vault` 等插件工具。

## 裝好之後

- **改單一段設定**：`& "C:\Program Files\Vault Workflow\vault-workflow.exe" --setup-section user`（`user`／`org`／`llm`／`packs`）
- **排程管理**：開始功能表 → Vault Workflow → 排程管理
- **選用排程**（清殘留 session、資源監看等，預設不裝）：

  ```powershell
  & "C:\Program Files\Vault Workflow\scripts\register-optional-tasks.ps1" -List
  ```

## 自動更新

插件每 6 小時檢查一次本 repo 的 `latest.json`，有新版時跳通知；也可以手動執行
`& "C:\Program Files\Vault Workflow\vault-workflow.exe" --apply-update`。插件與核心的更新各自獨立。

## 解除安裝

從「設定 → 應用程式」解除安裝 Vault Workflow。daemon 的兩支常駐排程會移除；其他還指向插件目錄的排程
（收工、晨報、git 自動提交…）會被**停用**而不是刪除，清單寫在
`%APPDATA%\AI-Memory-Vault\workflow-uninstall-disabled-tasks.txt`，重裝後可以再啟用。
你的筆記與設定（`%APPDATA%\AI-Memory-Vault\`、Vault 資料夾）不會被刪。

## 常見問題

**Q：編輯器連不上？**
依序確認兩個服務：`curl http://127.0.0.1:8766/version`（插件）、`curl http://127.0.0.1:8765/version`（核心）。
插件沒回應時，看護排程每 5 分鐘會自己把 daemon 拉回來；想立刻恢復就執行
`Start-ScheduledTask -TaskName VaultWorkflow-Daemon-AtLogon`。核心沒回應見核心 repo 的 USAGE.md。

**Q：改了 Claude 桌面 App 的設定，重開又變回去？**
App 開著時會把記憶體裡的舊設定寫回。完全結束後再改，或重跑遷移腳本。

**Q：插件安裝檔說找不到記憶核心？**
先到 [AI-Memory-Vault-core-releases](https://github.com/gabrielchen0314/AI-Memory-Vault-core-releases/releases/latest) 安裝核心 5.0 以上。

## 問題回報

請開 [Issue](../../issues)，附上兩個服務的 `/version` 回應與 `%APPDATA%\AI-Memory-Vault\` 底下相關的 log。
