# Proto-Sync Bot

> 跨多倉庫 Protobuf / OpenAPI 定義自動同步機器人

Proto-Sync Bot 是一個自動化工具，用於解決微服務架構中多倉庫共用 API 定義時的同步難題。當中心的 Protobuf 或 OpenAPI 定義倉庫發生變更時，工具會自動分析影響範圍，為所有依賴服務產生同步 PR，並自動修復因 API 變更引發的編譯錯誤。

## 解決的核心痛點

在微服務架構中，多個服務共用同一份 API 定義（如 `.proto` 或 `openapi.yaml`）是常見模式。然而，每次修改定義檔後，開發者需要手動更新每個依賴服務的依賴版本、重新產生客戶端程式碼、修復因 API 變更導致的程式碼編譯錯誤，以及同步更新各服務的 API 文件。這套流程耗時且易出錯。我們的團隊維護著 12 個微服務倉庫，過去三個月因同步遺漏導致了 7 次線上故障，平均每次修復耗時約 40 分鐘。Proto-Sync Bot 將這個流程完全自動化。

## 技術架構

- **執行框架**：[Aider](https://github.com/paul-gauthier/aider) + [OpenClaw](https://github.com/openclaw/claw)
- **多代理協作**：[CrewAI](https://github.com/crewAIInc/crewAI)
- **可觀測性**：Langfuse（全鏈路追蹤）
- **編程語言**：Python 3.9+

## 核心工作流程

**階段一：變更監聽與解析**  
透過 GitHub webhook 監聽 API 定義倉庫的 main 分支變更：拉取 `git diff` 取得變更檔案清單，解析 diff 結構，識別變更類型（新增／修改／刪除的 `message`、`field`、`service`、`method`），並自動過濾無關變更（如註解變更）。

**階段二：影響範圍分析**  
遍歷所有微服務倉庫，讀取各服務的依賴文件（`go.mod`、`package.json`），識別依賴當前 API 定義倉庫的所有服務。對每個受影響服務進行靜態程式碼分析：掃描 import 路徑，識別引用的 proto 型別，將引用型別與本次變更型別做比對，評估影響等級（低／中／高）。

**階段三：自動修復與 PR 生成**  
對每個受影響服務並行執行：建立新分支、更新依賴版本號、執行 `protoc` 或 `openapi-generator` 重新產生客戶端程式碼。若產生編譯錯誤，則解析錯誤日誌，定位問題檔案和行號，自動修復程式碼（重命名字段、調整函式簽名等），重新編譯（最多迭代 3 次）。接著從 proto 註解中提取 API 說明，更新服務的 README，最後發起 Pull Request 並自動填入變更摘要。

**階段四：CI 失敗閉環修復**  
PR 合入後若 CI 失敗，Agent 自動接收失敗通知，拉取完整日誌，在本地環境重現錯誤，定位問題（測試用例、設定檔等），修復後強制推送新 commit，重新觸發 CI。

## 實際成效

工具已在內部環境試運作 6 週：處理 23 次 API 定義變更，自動發起 87 個 PR，PR 合入率 83.2%。單次變更人工耗時約 1.5 小時，自動化後僅需約 12 分鐘。6 週內因同步遺漏導致的線上故障為 0 次。

## 快速開始

環境需求：Python 3.9+、Git 2.30+、protoc（Protocol Buffers 編譯器）、openapi-generator（選用）。首先 clone 專案並進入目錄：`git clone https://github.com/t0nyJing/proto-sync-bot.git && cd proto-sync-bot`，然後安裝依賴：`pip install -r requirements.txt`，複製設定檔模板：`cp config.example.yaml config.yaml`。接著編輯 `config.yaml`，填入你的 GitHub Personal Access Token、API 定義倉庫位址（如 `組織/central-api-repo`）、受影響的服務清單（`services` 列表）以及 webhook 密鑰。完成後執行 `python main.py --monitor` 即可啟動監聽。

## 專案結構

proto-sync-bot/
├── agents/ # 任務代理模組
│ ├── dependency_updater.py
│ ├── code_generator.py
│ ├── compile_fixer.py
│ └── pr_creator.py
├── analyzer/ # 靜態程式碼分析模組
├── orchestrator/ # 中央編排器
├── webhook_handler.py # GitHub webhook 處理
├── main.py # 主進入點
├── config.example.yaml # 設定檔模板
├── requirements.txt
└── README.md


## 貢獻

歡迎提交 Issue 和 Pull Request。

## 授權條款

MIT License
