# CLAUDE.md — SocialPost（社群貼文產生器）

單檔前端工具：輸入主題／方向／內容 → 依 Facebook／Instagram／X／Threads 各平台的字數上限與語氣慣例產生優化貼文文案 → Facebook 可直接登入帳號自動發布（文字／圖片／影片），其餘三個平台複製文字後手動貼上發布。無建置步驟、無框架、無 package.json，直接開啟 `index.html`（`file://`）或以靜態伺服器託管即可。

`manual.html` 是操作手冊，含 Facebook App 串接前置條件說明、AI 金鑰設定步驟、隱私說明、使用警語與創作者資料。創作者經歷內容與 `Prompt/manual.html`、`sbir-generator/manual.html`、`icap-generator/manual.html`、`phoenix-loan-generator/manual.html` 為同一份，更新其中一邊時同步其餘各邊。

## 範圍界定（重要，勿隨意擴大）

- **只有 Facebook 做真正的自動發布**（透過官方 Facebook JS SDK 登入＋Graph API）。Instagram／X／Threads 只做 AI 文案優化＋媒體預覽＋複製文字，**不做這三個平台的 API 串接**——IG 需透過 FB Graph API 的 Instagram Business 帳號、X／Threads 是完全不同的 API 與審核流程，複雜度留待未來視需求擴充，不要在沒有明確需求下主動加上。
- Facebook 串接假設使用者**已經有**自己的 Facebook 開發者帳號、App、與粉專管理權限——`manual.html` 只說明「怎麼把已有的 App ID 接上本工具」，不寫「如何申請開發者帳號」的完整教學。

## 架構

單一 `index.html`：內嵌 `<style>` 與 `<script>`，IIFE 包裝，無外部資源除了選用的 AI API fetch 與選用的 Facebook JS SDK（`https://connect.facebook.net/zh_TW/sdk.js`，只有使用者按下「登入 Facebook」時才動態載入）。

- **`PLATFORM_SPECS`**：四平台的字數上限（`charLimit`）、建議字數（`sweet`）、hashtag 建議數量與說明、語氣要求，同時驅動 AI prompt 組合（`buildAiPrompt()`）與規則式 fallback（`ruleBasedGenerate()`）。新增/調整平台規則只需改這個表。
- **貼文素材**：主題 textarea、語氣下拉（`TONE_TEMPLATES`：親切口語／專業正式／促購優惠／正式公告）、平台複選（`.platform-check`）、圖片多選上傳（`FileReader.readAsDataURL` 產生縮圖，`mediaState.images` 陣列）、影片單一上傳（`URL.createObjectURL` 餵 `<video>` 預覽，`mediaState.video`）。素材與草稿（主題/語氣/角色/目的/平台勾選）存 `localStorage`（key: `socialpostDraft`），重整頁面可恢復。
- **貼文角色／人設＋貼文目的（2026-08-18 新增，皆為「預設下拉 + 自訂」模式）**：`roleSelect`（品牌官方小編／老闆本人／網紅／專業顧問／吉祥物角色／✏️自訂角色）與 `purposeSelect`（促購導購／品牌曝光／互動增粉／活動宣傳／公告通知／教育科普／建立信任／✏️自訂目的），選到 `custom` 時對應的 `roleCustomInput`／`purposeCustomInput` 文字框才會顯示（`syncCustomInputVisibility()`）。`ROLE_DESCRIPTIONS`／`PURPOSE_DESCRIPTIONS` 是餵給 AI prompt 用的完整描述句（比 `ROLE_LABELS`／`PURPOSE_LABELS` 這兩個純 UI 顯示用的短標籤更詳細），`getRoleText()`/`getPurposeText()` 統一處理「custom 選項讀自訂輸入框、否則讀對應描述」的邏輯，`buildAiPrompt()` 只有在文字非空時才插入對應 prompt 段落（自訂但留空＝不特別指定，不是擋下產生）。**只有 AI 模式會套用**，規則式 fallback（`ruleBasedGenerate()`）不讀這兩個欄位，`genReport` 未填金鑰時的訊息會明講這點。`EXAMPLES` 五組範例都補上對應的 `role`/`purpose` 鍵，`initExamples()` 套用範例時一併設定並同步顯示/隱藏自訂輸入框。
- **上傳圖片會被 AI 分析並融入文案（2026-08-18 新增）**：這是本工具第一次讓 `callLLM()` 支援視覺輸入，做法直接比照 `product-title-generator/index.html` 已驗證過的 `imageDataUrl` 參數模式（`splitDataUrl()` 拆 `data:mime;base64,data`），差異是這裡**改成陣列**——`callLLM(provider, model, apiKey, prompt, onRetry, imagesDataUrls)`，Claude 走 `content:[{type:'image',...}, {type:'image',...}, {type:'text',text}]`（多張圖片區塊在前、文字在最後）、OpenAI/OpenRouter 走 `content:[{type:'text',text}, {type:'image_url',...}, ...]`、Gemini 走 `parts:[{text}, {inline_data,...}, ...]`。`generateBtn` click handler 用 `MAX_IMAGES_FOR_AI = 3` 只取 `mediaState.images` 最先上傳的前 3 張（避免 payload/費用暴增），`buildAiPrompt()` 多兩個參數 `roleText`/`purposeText` 之後再加 `hasImages`（布林），為真時插入一段要求模型「先觀察圖片內容、讓文字呼應圖片」的指示。規則式引擎沒有視覺能力，未填 API 金鑰時圖片只作為發布素材、不影響文案生成（`imageAiHint` 提示文字會在有上傳圖片時顯示，說明這個限制）。**已用攔截 `window.fetch` 的方式驗證過**：組出的 Claude payload 的 `content` 陣列確實含 `image`/`text` 兩種 block type、prompt 文字裡角色/目的/圖片提示段落皆正確插入，測完已還原 `window.fetch`。
- **內建範例（`EXAMPLES`，2026-08-15 新增）**：5 組**完全虛構**的中小企業情境（咖啡店週末優惠／瑜伽教室招生／手作甜點新品／寵物店颱風公休公告／選物店年中特賣），涵蓋四種語氣與不同平台組合，比照 `Prompt`／`sbir-generator` 既有範例模式。主題輸入框下方一排藥丸狀按鈕（`initExamples()`），點下去直接帶入 `topic`／`tone`／平台勾選並存草稿；若輸入框已有內容會先 `confirm()` 詢問是否覆蓋（跟「套用範例」慣例一致）。
- **AI 優化引擎（選用，比照 `Prompt/index.html`／`sbir-generator` 既有模式，修改時互相參照）**：`AI_PROVIDERS`（Claude/OpenAI/Gemini/OpenRouter）、`callLLM()`（含 Claude 的 `anthropic-dangerous-direct-browser-access` header、429/500/503/529 重試、180 秒逾時）、`extractJsonObject()` 皆為同一套實作。差異點：`buildAiPrompt()` 依 `PLATFORM_SPECS` 與使用者勾選的平台组一份「請針對這幾個平台各自產生完整貼文全文（含 hashtag）」的 prompt，要求回傳 `{platformKey: "完整貼文文字", ...}` 的 JSON，只含勾選的平台。設定存 `localStorage`（key: `socialpostApiConfig`）。
- **規則式 fallback（`ruleBasedGenerate()`）**：沒有 API 金鑰時使用，不連網。用 `TONE_TEMPLATES` 加上開頭/結尾語＋原文，`extractHashtags()` 用停用詞表從主題文字挑關鍵詞組 hashtag，超過平台字數上限就截斷。AI 呼叫失敗或某平台未取得有效結果時，也會**單獨對該平台**退回這個 fallback（見 `generateBtn` click handler 裡的 `missing` 邏輯），不會整批失敗。
- **結果卡片（`renderResults()`）**：每個勾選平台一張 `.result-card`（左側邊框色對應該平台品牌色），文字框可編輯（`hashtag` 直接併入同一段可編輯文字，不是分開欄位，方便發布/複製時就是完整貼文全文），即時字數計量對照該平台上限（超過會標紅）。非 Facebook 卡片有「複製文字」按鈕（`navigator.clipboard.writeText`，不支援時退回 `document.execCommand('copy')`）。
- **一鍵複製全部（`#copyAllBtn`，2026-08-15 新增）**：卡片 2 標題列右側按鈕，點下去把 `#resultsGrid` 內目前**所有**已產生平台的卡片（讀 `.result-title` 當標籤、`.result-text` 當內容，即時反映使用者手動編輯過的文字，不是原始產生結果的快照）以「【平台名稱】\n內容」格式串接、平台間空一行，一次呼叫既有的 `copyText()` 複製到剪貼簿。沒有任何結果時顯示 toast 提示、不呼叫剪貼簿 API。
- **Facebook 發布**（`fbPanelHtml()` / `initFbPanel()` / `publishToFacebook()`）：
  - App ID 輸入框存 `localStorage`（key: `socialpostFbConfig`，含 `appId` 與上次選擇的 `pageId`）。
  - `loadFbSdk()` 動態插入 SDK `<script>`，`window.fbAsyncInit` 內 `FB.init({appId, version:'v21.0'})`——**刻意延後到使用者填好 App ID 並按下「登入 Facebook」才載入**，因為 App ID 是使用者輸入值、頁面載入當下未知。
  - 登入走 `FB.login(cb, {scope:'pages_show_list,pages_manage_posts,pages_read_engagement'})`；成功後 `FB.api('/me/accounts', cb)` 取得使用者管理的粉專清單（含各粉專自己的 `access_token`），下拉選單選定後存 `pageId`。
  - **實際發布呼叫改用 `fetch` + `FormData` 直接打 `graph.facebook.com`（`graphFetch()`），不是 `FB.api()`**——這是刻意的選擇：Graph API 的寫入端點對純 `fetch`/`FormData` 上傳（`access_token` 隨表單欄位送出）有良好支援，比起讓 SDK 的 `FB.api()` 處理瀏覽器 `File` 物件的多媒體上傳更可預期。四種發布路徑：純文字 `/feed`、單圖 `/photos`、多圖（先各自 `published=false` 上傳取得 photo id，再用 `attached_media[i]` 組 `/feed`）、影片 `/videos`。
  - **發布前一定 `confirm()` 二次確認**（會影響真實外部系統的不可逆動作）。
  - **已知風險 / 尚待實測**：本專案建置時沒有真實 Facebook App／粉專可供端對端測試，四種 Graph API 發布路徑是依官方文件實作、未經實機驗證。第一次接上真實 App ID 時，建議依序測試「純文字」→「單張圖片」→「多張圖片」→「影片」，若 CORS 或參數格式有出入，優先檢查 `graphFetch()` 的錯誤訊息（`data.error.message`）與 Graph API 版本號（目前寫死 `v21.0`，官方棄用舊版後需要更新）。

## 頂部跑馬燈（2026-08-16 新增）

`#marqueeBar` 顯示跟工作區其他工具共用同一份 Google Sheet 維護的公告內容，同一個授權伺服器 Apps Script 網址（`AKfycbwKX0.../exec`，與 `ai-video-studio`／`food-finder`／`ai-prompt-generator` 等系列共用同一顆端點）。本工具**沒有序號登入機制**，做法比照這些同樣沒有登入機制的工具：頁面載入時直接 POST 一個空序號給該網址，`doPost` 不論序號是否有效都會附上 `marquee` 陣列，前端只取這個欄位、忽略 `valid`/`reason`。`localStorage` key 為 `socialpostMarquee`。先讀快取立即顯示、再背景 fetch，每 20 分鐘重抓一次；抓取失敗靜默忽略。

版面整合：`.topbar` 原本是 `position:sticky;top:0`，跑馬燈用 `position:fixed` 疊在最上面＋`body.has-marquee{padding-top:30px}` 把內容往下推，並且把 `.topbar` 的 `top` 也一併調整成 `body.has-marquee .topbar{top:30px}`（不是 `margin-top`，理由見 `shared-widget-rollout` skill：`margin-top` 只影響文件流裡的初始位置，捲動到吸頂那一刻還是會貼齊原本的 `top:0` 被蓋住）。跑馬燈文字用 `--violet` 主色（`#7c6cf5`），與本工具既有配色一致。獨立 `<script>` IIFE，跟主程式邏輯、PWA 安裝腳本互不相依。

**已驗證**：Node `fetch()` 直接打共用端點確認能拿到正確的 `marquee` 陣列（**curl 直接 `-L -X POST` 這個網址會因為 302 轉址被降級成 GET 而報 411，是 curl 本身的行為，不代表端點壞掉**，測這個端點要用 `fetch()` 或瀏覽器）；Playwright 對本機靜態伺服器實際驗證跑馬燈正確顯示、`body.has-marquee`／`.topbar top:30px` 皆正確套用、無版面重疊（截圖確認）。

**2026-08-20 更新（`Code.gs` 未改動、不需重新部署）**：`render()` 新增 `lastKey`（`JSON.stringify(items)`）比對，內容沒變就不重繪，CSS animation 不再被重置歸零重跑；新增 `appendParsedText()`／`buildTrackContent()` 支援 `[文字](https://...)` 連結語法（`createTextNode` 組 DOM，避免 XSS），資料格式仍是純字串陣列，向下相容。已 commit＋push（GitHub Pages 自動重新部署）。

## 隱私與警語

無自建後端、無資料上傳到本工具以外的伺服器。AI 金鑰、Facebook App ID／登入權杖、貼文草稿都只存在使用者瀏覽器的 `localStorage`。首頁與手冊皆明列使用警語：Facebook 發布是公開且不可復原的動作、請勿輸入真實個資或機密資料、僅供教學與個人使用禁止商業化。修改功能時這些警語需一併檢視是否仍準確。

## 部署（2026-08-15）

已推公開 GitHub repo：<https://github.com/M255525/SocialPost>，已開 GitHub Pages（`.github/workflows/deploy-pages.yml`，Actions 部署模式，比照 [[workspace-git-repos]] 記載的「不要用 legacy branch-source」慣例）：<https://m255525.github.io/SocialPost/>。頁尾加了訪客次數計數器（`visitor-badge.laobi.icu` 的 SVG badge，`<img>` 直接嵌入 `.footer-meta`，`page_id=m255525.socialpost`，免金鑰免後端）。`README.md` 比照 `ai-image-prompt-studio` 的既有格式撰寫。

## 加入主畫面（PWA，2026-08-16 新增）

比照工作區其餘已上線 GitHub Pages 工具（`ai-image-prompt-studio`／`ai-prompt-generator`／`food-finder` 等，見 [[pwa-install-rollout]]）的既有做法：`manifest.json`＋`icons/`（主色 `#7c6cf5` 背景、白色「貼」字圖示，PIL＋`msjhbd.ttc` 產生，192／512／maskable-512／apple-touch-icon 四種尺寸）＋`service-worker.js`（network-first＋同源快取備援，跨網域請求略過不進快取——本工具的 AI/Facebook API 呼叫本來就是瀏覽器直接打外部網域，SW 不會攔到）。安裝按鈕（`#installBtn`，📲 加入主畫面）放在 `.footer-meta`（新增 `.footer-link-btn` 樣式），因為本工具**有 `showToast()`**（既有的 toast 提示機制），安裝失敗時直接沿用，不需要像沒有 toast 的專案那樣改用 `alert()`。`<head>` 補上 `manifest`／`theme-color`／`apple-touch-icon`／`apple-mobile-web-app-*`／`mobile-web-app-capable` 標籤；安裝腳本是獨立 `<script>` IIFE（含 iOS/iPadOS/macOS Safari 判斷，`beforeinstallprompt` 不會在這些瀏覽器觸發、要改用文字指引），跟上方主程式邏輯互不相依。

**2026-08-14 那次工作區全站 PWA 推廣時本工具被排除**（見 [[pwa-install-rollout]]：「經檢查後確認沒有 GitHub Pages 部署」），因為當時本工具尚未上線；2026-08-15 才推公開 repo＋開 GitHub Pages，補上是這之後的事。

**安裝按鈕「有出現但沒反應」的 bug（2026-08-16，使用者實測回報）**：原本安裝腳本用 `if (typeof showToast === 'function') showToast(fallbackMessage());` 當 `deferredPrompt` 是 `null` 時的回饋——**但 `showToast()` 是宣告在主程式那個獨立 IIFE 裡，PWA 安裝腳本是另一個完全獨立的 `<script>` 區塊／IIFE，函式作用域不會跨區塊共享**，所以 `typeof showToast` 在安裝腳本裡永遠是 `'undefined'`，點按鈕在 `deferredPrompt` 為 `null` 的情況下（例如瀏覽器沒觸發 `beforeinstallprompt`，或已安裝過）**完全沒有任何回饋、主控台也不會報錯**，跟使用者的回報「按鈕鍵但沒有對應功能」完全吻合。修法：安裝腳本改成自己實作 `notify(msg)`（直接操作 `#toast` 這個 DOM 元素，不依賴外部函式是否存在），`deferredPrompt.prompt()` 補上 try/catch。排查時發現這是同一套安裝腳本在工作區另外兩個姊妹專案（`ai-image-prompt-studio`／`ai-prompt-generator`／`ai-music-prompt-studio`，共三個）裡也存在的系統性 bug，已一併修正，詳見 [[pwa-install-rollout]]。

## 指令

無建置/測試指令。修改 `index.html` 或 `manual.html` 後用瀏覽器開啟驗證，或 `python -m http.server 8778 --directory SocialPost` 暫起伺服器測完關閉。用 Preview MCP 驗證時，`preview_eval` 讀 DOM／觸發事件比截圖可靠（與工作區其他單檔工具已知的截圖偶發逾時問題相同）。

### 桌面版 exe（尚未打包）

比照 `phoenix-loan-generator/launcher.py` 的模式，`launcher.py` 已就緒（固定 **8778 埠**——工作區埠號分配：8765 ai-course-hub、8766 video-editor、8767 fruit-ninja-cam、8770 phoenix-loan exe、8771 icap exe、8772 sbir exe、8773 ai-video-studio、8774 ai-video-studio 桌面版 exe、8775 IPA_Kano dashboard exe、8776 Dashboard、8777 Prompt exe、**8778 本專案**、8779 sbir-gen-s、8780 icap_s、8781 aivideo-studio-s）。需要打包時，比照以下指令（PowerShell、絕對路徑）：

```powershell
$proj = "C:\Users\mark_\AI Test\行銷內容工具\SocialPost"
cd $proj
python -m PyInstaller --onefile --console --name SocialPostGenerator `
  --distpath "$proj\socialpost" --workpath "$env:TEMP\pyi-build-socialpost" --specpath "$env:TEMP" `
  --add-data "$proj\index.html;." --add-data "$proj\manual.html;." `
  launcher.py
```

exe 未簽章，首次執行會遇 SmartScreen 警告；新建置的二進位檔可能被 Smart App Control 暫時封鎖數小時（雲端信譽尚未建立），詳見全域記憶 `windows-smart-app-control-dll-blocks`，不要建議使用者關閉 Smart App Control。**若之後要打包，App ID 對應的 Facebook App 需把「應用程式網域」與「有效的 OAuth 重新導向 URI」設定為 `http://127.0.0.1:8778`（固定埠號的原因之一，也是為了讓 Facebook 登入的網域白名單不用每次調整）。**
