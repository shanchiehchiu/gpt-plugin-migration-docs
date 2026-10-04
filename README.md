# GPT 外掛程式遷移說明

本文件說明：為什麼要把現有的「自訂 GPT」改成「外掛程式」，以及實際製作上的步驟與注意事項。

## 背景：為什麼要做這件事

ChatGPT 的「我的 GPT」清單裡出現通知：**12 月 11 日前**沒有遷移的 GPT 會失效，需要遷移成「外掛程式」。要理解這件事，得先搞懂新舊兩套機制的差異：

- **舊版 Custom GPT**：靠「自然語言 instructions」+ 選配的「Actions」（貼一份 OpenAPI schema，GPT 就能呼叫自訂的 REST API）運作。這套機制是 OpenAI 自家、封閉的格式，只有 ChatGPT 看得懂。
- **新版「外掛程式」**：底層走的是 **MCP（Model Context Protocol)**——Anthropic 提出、目前已變成業界共通標準的協定。ChatGPT、Claude、Gemini、Grok 現在都支援用同一套 MCP 伺服器當作「外掛／連接器」。

OpenAI 強制遷移的原因，從它的角度看有兩個：

1. **技術收斂**：與其維護自家專屬的 Actions/OpenAPI 格式，不如直接支援業界共通的 MCP，省去維護成本，也讓第三方開發者只要做一套就能接所有家 AI。
2. **功能擴充**：新架構（Apps SDK）讓外掛不只是「呼叫 API 拿資料」，還能回傳互動式 UI 元件、處理身分驗證（OAuth）等，Actions 做不到這些。

實際影響是：**任何用 Actions 包出來給 GPT 用的功能，都要重新包成一個 `/mcp` 端點**，這個端點不是「換個 API 格式」而已，而是要能回應 MCP 協定定義的握手流程（`initialize` → `tools/list` → `tools/call`）。

MCP 的核心設計是「工具探索」而非「固定 API 文件」：客戶端（ChatGPT/Claude）連上 `/mcp` 端點後，先問「你有哪些工具」（`tools/list`），伺服器動態回傳工具清單與參數 schema，再呼叫（`tools/call`）。這跟 Actions 要先貼好一份靜態 OpenAPI schema 的做法完全不同——好處是改工具不用重新上傳 schema，壞處是參數 schema 的寫法有些陷阱（optional 參數容易寫錯）。

## 兩個套件的分工

遷移的實作可以用下面兩個現成的 Claude Code skill（皆為 MIT 授權）：

- https://github.com/shanchiehchiu/laravel-mcp-adapter
- https://github.com/shanchiehchiu/laravel-mcp-oauth-adapter

| | `laravel-mcp-adapter` | `laravel-mcp-oauth-adapter` |
|---|---|---|
| 做什麼 | 把 Laravel 應用包成 MCP Streamable HTTP 伺服器，開一個 `/mcp` 端點 | 在前者之上疊加 OAuth 2.1 授權碼 + PKCE 登入流程 |
| 驗證方式 | 靜態 API 金鑰 | 使用者登入授權（ChatGPT 走 CIMD、Claude/Gemini/Grok 走 DCR） |
| 什麼時候需要它 | 只有自己要用，或對方連接器介面可以貼金鑰 | 對方連接器介面**只有「Login with OAuth」按鈕**、沒有貼金鑰欄位時（ChatGPT 的新版 Connector UI 常是這樣） |
| 前提 | 一個全新或既有的 Laravel 專案 | 必須先有一個能動的 `/mcp` 端點（通常就是前者裝好的結果） |

**關鍵判斷點**：如果要掛的是 ChatGPT 的新版外掛介面，它多半只給「Login with OAuth」這個選項，沒有貼金鑰的欄位——這種情況一定要兩個套件疊著用，不能只裝 `mcp-adapter`。

## 製作步驟

1. **裝 `mcp-adapter` skill 到目標 Laravel 專案**

   ```bash
   git clone https://github.com/shanchiehchiu/laravel-mcp-adapter .claude/skills/laravel-mcp-adapter
   ```

   在該專案打開 Claude Code，請它「讀 SKILL.md 幫我建立 MCP adapter」。它會照著 `templates/`（config、middleware、controller、範例工具）複製進專案，接好路由。

2. **把 Actions 裡的功能改寫成 MCP 工具**

   原本 Actions 的每一支 API，對應改寫成一個 MCP tool（定義好 name、參數 schema、執行邏輯）。這一步需要逐一盤點目前 GPT 用到的 Actions endpoint。

3. **本地驗證協定握手**

   跑 `initialize` → `tools/list` → `tools/call` 確認工具真的能被發現、呼叫、回傳正確資料。

4. **視對方連接器介面決定要不要加 OAuth**

   如果 ChatGPT（或其他 client）的新增連接器畫面沒有貼金鑰欄位，再疊加：

   ```bash
   git clone https://github.com/shanchiehchiu/laravel-mcp-oauth-adapter .claude/skills/laravel-mcp-oauth-adapter
   ```

   同樣請 Claude Code 讀 `SKILL.md` 執行——它會加 migration、OAuth controllers、登入/同意頁。預設假設用 Laravel 內建 `App\Models\User` + `web` guard + 帳密登入；如果專案的登入方式不同，只需要調整 `AuthorizeController::login()` 這一處。

5. **部署，並在 ChatGPT 後台建立新的 Connector**

   把 `/mcp` 的網址（正式環境網域，不能是 localhost）貼進 ChatGPT「新增連接器」的設定，走完 OAuth 登入或貼金鑰，確認工具清單出現、能正確呼叫。

6. **把舊 GPT 的 instructions 搬過來**

   新外掛的「個性／用途說明」還是要靠 instructions 文字設定，這塊照搬舊 GPT 的描述即可，技術遷移的重點只在「工具怎麼被呼叫」這件事。

## 要注意的坑

- `mcp-oauth-adapter` 的 `references/spec-gotchas.md` 記錄了 7 個容易寫錯的 OAuth/PKCE/JWT 細節，包含一次真實的 **OpenAI Codex 連線失敗案例**，根因追到 RFC 8252 loopback redirect_uri 的處理方式——若要接 ChatGPT/Codex，這份文件值得先看過。
- MCP 工具的參數 schema 寫法對 **optional 參數**有陷阱（`mcp-adapter` 的 `references/` 有特別說明），寫錯會導致 client 端呼叫失敗或參數傳不進來。
- 兩個套件都已驗證過：`mcp-adapter` 從零建一個全新 Laravel 10 專案跑通；`mcp-oauth-adapter` 更進一步跑完真實的 DCR 註冊 → 登入 → 同意 → token 交換 → 呼叫 `/mcp` → refresh token 輪替，並在正式環境接過 ChatGPT、OpenAI Codex、Claude。
