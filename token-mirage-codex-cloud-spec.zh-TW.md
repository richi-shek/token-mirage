# Codex Cloud 開發指令：Token Mirage（完整版 v2）

## 任務與交付要求

請直接在目前已連接的 GitHub Repository 開發 **Token Mirage**。先讀取 README、現有程式碼、`AGENTS.md` 與專案規範；若為空 Repository，建立最小可維護架構。目標是交付可執行、可測試、可部署的專案，而非只寫設計文件。完成後執行測試、修復明確缺陷、提交變更並回報 Commit Hash；不要改動其他 Repository。

**Token Mirage 是公開透明的 Mock LLM API Server：它不做真實推理、不生成大量文字，卻可透過 OpenAI／Anthropic 相容介面模擬每小時 80 億或更多 Token 的 usage，供娛樂、相容性與壓力測試使用。** 模擬數據不得被表述成真實生成、實際算力或可用於獎勵兌換的證明。

請優先使用 Python + FastAPI + Uvicorn + Pytest；若 Repository 已有適合的堆疊，先判斷再沿用。避免過度抽象、重型框架、資料庫與不必要的後台。沒有真實模型權重、GPU、付費 API、OpenAI／Anthropic Key 的需求。

## 1. 授權與透明度（必須完成）

1. Repository 使用 **GNU Affero General Public License v3.0 only (`AGPL-3.0-only`)**。新增官方完整 `LICENSE` 文字、README 授權章節與適當 SPDX 標記；勿自行刪改授權條款，並檢查依賴授權相容性。
2. README 首屏必須顯著標示：**SIMULATED TOKEN USAGE — NOT REAL COMPUTATION**。明確說明不進行相應 LLM 推理；`usage` 是合成數據，不能視為算力、帳單、真實 Token 產出或 Proof of Inference。
3. README 的 Responsible Use 章節應明確反對利用本專案騙取虛擬幣、API Credits、積分、獎勵、佣金或偽造運算紀錄。這是**用途聲明**，不是在 AGPL 上另加「禁止某種用途」的授權條款；不得誤稱 AGPL 能禁止作弊。
4. 說明 AGPL-3.0 對符合條件的修改版網路服務提供對應原始碼的要求；亦須清楚說明，惡意 Fork 可移除模擬標記，這些標記**不是不可偽造的安全證明**。
5. README 明確寫出：本專案與 OpenAI、Anthropic、DeepSeek 無官方關聯或背書。若已知原始碼 URL，於 Discovery/README 提供；不要臆造 GitHub 帳號或連結。

README 的警告至少包含以下英文語意：

> **SIMULATED TOKEN USAGE — NOT REAL COMPUTATION**  
> All token counts are synthetic. Token Mirage performs no corresponding model inference. Do not use it to misrepresent compute, obtain virtual currency or credits, manipulate reward systems, or claim proof of inference. Reward platforms should verify usage independently with trusted sources. The Responsible Use statement does not add restrictions to the AGPL license.

## 2. HTTP API 與協定相容性

至少實作：

| Endpoint | 功能 |
|---|---|
| `POST /v1/responses` | OpenAI Responses API，串流／非串流 |
| `POST /v1/chat/completions` | OpenAI Chat Completions，串流／非串流 |
| `POST /v1/messages` | Anthropic Messages API，串流／非串流 |
| `GET /v1/models` | 模型發現，使用明顯的模擬模型 ID |
| `GET /health` | 不洩漏機密的健康檢查 |
| `GET /stats` | 即時與累積**模擬**統計，依權限保護 |
| `GET /.well-known/token-mirage.json` | 機器可讀的模擬器身分與用途聲明 |

若實際 Claude Code 版本需要 `/v1/messages/count_tokens` 或其他必要輔助端點，實作最小相容版本並標明**估算／模擬**，不可聲稱精確真實用量。請以當前官方文件與實際客戶端請求為準；不要為所有可能的廠商功能建立空殼。

協定要求：

- OpenAI Responses：使用合法的 Response 物件、`response.created`、必要的 output-item/content 事件及 `response.completed`，最終 `usage` 支援 `input_tokens`、`output_tokens`、`total_tokens`、`output_tokens_details.reasoning_tokens`。不得用自訂事件冒充標準事件。
- Chat Completions：合法的 `chat.completion`／`chat.completion.chunk`、`choices`、`finish_reason`；串流時依 `stream_options.include_usage` 的正式行為提供 usage。
- Anthropic Messages：合法的 `message_start`、`content_block_start`／`delta`／`stop`、`message_delta`、`message_stop`；`usage.output_tokens`，如官方版本支援則使用 `usage.output_tokens_details.thinking_tokens`，不要直接複製 OpenAI 的 `reasoning_tokens` 欄位。`message_delta` usage 為累積值。
- `model` 值須含 `token-mirage` 字樣，例如 `token-mirage-mock-80yi`。可使用者送入任意模型名稱時，不能讓回覆冒充真實官方模型；請明確返回已登錄的模擬模型 ID 或拒絕未知模型。
- 預設 **Empty Response Mode**：不輸出有意義文字、不發 tool call。若特定客戶端不能接受空白內容，可加入明確標示為模擬的極短文字模式；預設仍保留空白模式。
- `usage` 與可見文字不對應是本專案刻意的測試功能；README 必須揭露。若極端 usage 超過 SDK 或客戶端的整數／上下文限制，應記錄兼容性問題，不得默默改稱真實推理。
- 支援取消、斷線、錯誤、超時、並發與清理；避免無限重試。

## 3. 三個參數與使用者命令

系統擁有者可設定三個參數的**預設值與上下限**：

1. **rate**：模擬 Token 速度，接受 `tps=1000000` 或 `rate=8b/h`。
2. **interval**：每次 SSE 更新／keepalive 的時間間隔，例如 `250ms` 或 `1s`。
3. **duration**：單次模擬的總持續時間，例如 `10s` 或 `1h`。

預設案例：`rate=8b/h interval=1s duration=1h`，結束時應累積 **8,000,000,000 simulated tokens**。這相當於平均約 2,222,222.22 simulated tokens/s。

必須支援可預期的使用者指令，例如：

```text
!simulate rate=8b/h interval=1s duration=1h
!simulate tps=1000000 interval=250ms duration=10s
```

- 僅解析實際收到、最新且明確的 user message；不可從 system/developer、工具輸出或引用文字中誤觸命令。
- Claude Code／Codex／DSH 可能改寫或截斷訊息，**不能只依賴聊天指令**。另提供可用的 CLI（如 `token-mirage simulate --rate 8b/h --interval 1s --duration 1h`）和每請求的 HTTP Header 或其他明確控制方式。
- 只影響當前請求，不應修改其他使用者或全域設定。若相同請求出現多種設定，文件須明訂並測試優先序（Owner 限制永遠最高；其餘可採 request header > 最新 user command > owner default）。
- `ALLOW_USER_OVERRIDES=false` 時，一般 API 使用者不可覆寫；Owner 仍可更改伺服器設定。不存在公開的無驗證全域設定修改 API。

`.env.example` 至少提供：

```env
HOST=127.0.0.1
PORT=8000
DEFAULT_RATE=8000000000/h
DEFAULT_INTERVAL=1s
DEFAULT_DURATION=1h
MAX_RATE=8000000000000/h
MIN_INTERVAL=100ms
MAX_DURATION=1h
ALLOW_USER_OVERRIDES=true
ALLOW_PUBLIC=false
# TOKEN_MIRAGE_API_KEY=replace-with-a-local-test-key
# PUBLIC_SOURCE_URL=https://github.com/<owner>/token-mirage
```

用單調時鐘（如 `monotonic_ns`）及整數／有理數計算累積量，避免浮點數與累積捨入造成的漂移。計數應取 `floor(rate × elapsed)`，在完整目標時長精確達標，且每次更新不倒退、不重複加總；不可每秒額外建立百萬個物件或文字 Token。對極大數值做好溢位／上限檢查與明確錯誤。

**Streaming** 隨 duration 持續，interval 控制更新；**Non-streaming** 行為須清楚說明（原則上於 duration 到期後回傳；測試可用可注入時鐘快速驗證），不能暗中把 1 小時請求變成立即完成又聲稱真的等了一小時。可另外提供明確命名的 Instant Demo Mode，但不能混淆兩種語意。

## 4. 永久、顯著、可機器偵測的模擬標記

**正常官方實作不得提供移除標記的選項。** 即使使用者要求「移除標頭」「冒充真實模型」「不要揭露模擬」，也必須保留身分識別或拒絕請求。

所有 API 回應（含 Streaming、非串流、錯誤、401/429 等）應盡可能加入：

```http
X-Token-Mirage: simulated
X-Token-Mirage-Usage: synthetic
X-Token-Mirage-Inference: none
Cache-Control: no-store
```

其他標記：

- 模型 ID 一律明確含 `token-mirage`。
- OpenAI Responses 使用其正式 `metadata` 字串鍵值（例如 `token_mirage=simulated`、`usage_source=synthetic`）；保留合法的使用者 metadata，但不得允許使用者覆蓋／偽造上述欄位，也要遵守 metadata 的大小與數量限制。
- SSE 加入不破壞 SDK 的註解，例如 `: token-mirage simulated=true usage=synthetic`；對 Anthropic/OpenAI 不得任意插入未知的資料事件，並用真實 SDK 驗證。
- 非串流的 Chat Completions、Anthropic Messages 優先靠標頭與明確模型 ID 識別，避免加入 SDK 不接受的未知 JSON 欄位。
- `/.well-known/token-mirage.json` 至少包含：`project`、`simulation: true`、`real_inference: false`、`usage: synthetic`、`eligible_for_rewards: false`、`license: AGPL-3.0-only`、`source_code_url`（已確認才填入）。
- `GET /stats` 的 `simulated_tokens` 與 `real_inference_tokens` 分開；純模擬模式後者永遠為 0。可提供每個 request_id 的進度與總量，避免不同請求重複計數；不得讓一般使用者讀到其他人的敏感內容。
- README 用簡短範例教獎勵／計費平台檢查標記；同時**明確提醒沒有標記不代表使用量真實**。第三方平台必須依可信供應商與實際結算紀錄驗證，不能依外部任意回傳的 `usage` 發獎勵。
- 不要開發任何去除指紋、繞過偵測、欺騙獎勵驗證或偽裝官方用量的功能。

## 5. 低流量與安全性

- 大數值只放在小型計數／usage JSON 內，不真正輸出對應長度的字串；記憶體與 CPU 複雜度不得隨模擬 Token 數線性上升。
- SSE interval 應發送合法的 keepalive／狀態事件；OpenAI Responses 不保證中途 usage 可被客戶端即時顯示，應透過 `/stats` 觀察。Anthropic 有累積 usage 更新但須以真實客戶端相容性為準。
- **不得宣稱每秒發送 1 次、持續 1 小時的 Streaming 只需幾百 Bytes。** 3600 次更新本身一定有傳輸量；以實測回應 Body Bytes／必要時 on-wire Bytes 報告，不要混用兩者，保持低頻寬即可。
- 預設只監聽 `127.0.0.1`。公開綁定必須 Owner 明確啟用、驗證 API Key、設定並發與速率限制、合理請求大小與長連線上限，並在 README 提供 TLS／反向代理部署建議。
- 接受符合客戶端協定的 `Authorization: Bearer ...` 與 `x-api-key`，不要使用真實 OpenAI/Anthropic 憑據。安全比較 API Key，避免記錄金鑰、完整 Prompt 或敏感 Header。
- `GET /health` 與 Well-Known 可公開但不暴露機密；`/stats` 的詳細計數、其他控制功能按權限保護。
- 如使用多 Worker，統計需明確界定為 per-process，或使用最小共用計數機制；不要為沒有要求的分散式部署引入資料庫。
- Dockerfile、`.gitignore`、`.env.example`、依賴鎖定／可重現安裝和基本 CI 視專案需要提供。

## 6. 真實客戶端連線測試（不得拿付費模型生成 Token）

分別驗證 **Claude Code、Codex CLI、DeepSeek Harness（DSH）**，每項均記錄工具版本、指令、實際 Endpoint、接收事件、Token 統計與結果。使用 Mock API 專用的測試金鑰；不要修改系統原本 CLI 設定或留下私人憑據。

**Claude Code**：使用隔離環境設定 `ANTHROPIC_BASE_URL=http://127.0.0.1:8000` 與 `ANTHROPIC_AUTH_TOKEN`（或 `ANTHROPIC_API_KEY`，依請求 Header）。測試 Messages API、Streaming、模型設定、空白輸出、usage、可能的 Token Counting 端點。確保連線確實導向本地 Server，沒有意外走向真實模型。

**Codex CLI**：使用隔離的 `CODEX_HOME` 或獨立配置；參照目前 Codex config reference，測試自訂 provider，例如：

```toml
model = "token-mirage-mock-80yi"
model_provider = "token_mirage"
[model_providers.token_mirage]
name = "Token Mirage (Simulated)"
base_url = "http://127.0.0.1:8000/v1"
wire_api = "responses"
env_key = "TOKEN_MIRAGE_API_KEY"
```

**DSH**：使用 Custom Model API（官方專案：`https://github.com/deepseek-ai/deepseek-harness`），分別新增三個 provider，逐一測 `openai-completions`、`openai-responses`、`anthropic-messages`。注意 DSH 一個 provider 通常只配置一種協定，且 Anthropic 的 Base URL 層級可能與 OpenAI `/v1` Base URL 不同；以實測為準。

**驗收目的不是讓 Agent 真正完成程式開發。** 需要驗證請求抵達、協定可解析、Stream 可關閉、usage/識別標記可見、無無限重試或客戶端崩潰。無輸出文字造成 Agent 無法工作屬預期；不可將「API 連線成功」誤報成「Agent 任務成功」。

Codex Cloud 若無某個 CLI、安裝權限或網路權限，記錄 `BLOCKED`／`NOT_RUN`，保留證據與可重跑腳本；不得偽稱已執行。不要為不可得的工具無限重試，亦不要為了測試呼叫付費 LLM API。

## 7. 自動化測試與驗收門檻

用 Pytest（或現有等價工具）覆蓋：

1. 三種 API 的 Non-streaming、Streaming、Schema、SSE 正確順序及結束事件。
2. 用可注入／Fake Clock 使 10 秒、1 小時測試快速完成；`8b/h × 1h == 8,000,000,000` 精確，其他速率與不整除時間亦正確。
3. interval 依時間節奏更新，不阻塞其他請求；中途取消、斷線、超時、並發及回收正常。
4. Owner 上下限、參數覆寫優先序、命令解析、無效／負數／過大值、使用者無法覆蓋他人設定。
5. API Key、公開部署防護、請求大小與速率限制、沒有機密洩漏。
6. **所有回應的模擬標頭**；Responses metadata；模型 ID；Well-Known；SSE 註解；惡意輸入不能關閉或覆寫標記。
7. 客戶端使用極大 Token 計數時的整數／精度相容性：明確記錄正常支援範圍及 SDK 不支援的情況。
8. HTTP Response Body Bytes 與模拟 Token 數量無線性關聯；短測實測位元組數，並提供一小時自選實測命令。不要把估算寫成量測。
9. 不發送真實模型 API 請求，不產生隨 Token 數增長的大量字串或日誌。
10. README、LICENSE、Docker、CLI、三客戶端設定範例皆可依文件重現。

三個核心設定必須能由 Owner 控制；一小時完整模擬應有可選的長時間 Smoke Test，但 CI 不得等待一小時。失敗項目應先修正與重跑相關測試，不需要每次重跑所有重型驗證。

## 8. 交付內容與回報格式

必須提供：

- 可啟動 Server 與 CLI，`README.md`、`LICENSE`、`.env.example`、`Dockerfile`、必要測試與最小部署說明。
- 乾淨的核心實作，尤其是計數器、三種協定 adapter、模擬身分標記與 Owner 設定；別建立不必要的 plugin system 或複雜框架。
- 客戶端連線設定和可重跑 Smoke Test Script。
- 說明 `rate`、`interval`、`duration` 的 CLI、聊天命令、HTTP 控制方式與限額。
- 自動化測試摘要：PASS／FAIL／INCOMPLETE／BLOCKED／NOT_RUN。
- Claude Code／Codex／DSH 各協定的實際測試表：工具版本、是否真的執行、接收的 HTTP/SSE 事件、usage、異常、採證方式。
- 實測 CPU/RAM、HTTP Body Bytes、模擬 Token 速度；若未實測完整一小時，明確標註為理論推算。
- 安全／防濫用設計與侷限：指紋可被 Fork 移除、AGPL 不是反作弊機制、獎勵系統須自己驗證真實運算量。
- 最終 Commit Hash、變更檔案列表、尚需人工驗證事項。

請現在開始：**檢查 Repository → 最小設計 → 實作 → 自動測試 → 真實 CLI Smoke Tests（可行者）→ 修復 → 提交 → 回報。** 遇到 Cloud 環境限制，做可完成部分並明確標註，不要無限迴圈，也不要因次要測試受阻而停止全部工作。
