# 社群貼文產生器 SocialPost

輸入一個主題／方向／內容，依 Facebook／Instagram／X（Twitter）／Threads 各平台的字數上限與語氣慣例，產生優化版貼文文案；Facebook 可直接登入你的粉專帳號一鍵發布，其餘平台複製文字後手動貼上即可。

🔗 **線上使用**：<https://m255525.github.io/SocialPost/>

## 這是什麼

- **一份主題，四種平台優化文案**：只需填一次主題、語氣、勾選發布平台，各平台各自的字數上限、hashtag 數量建議、語氣要求都內建在 `PLATFORM_SPECS`，不用自己記各平台的慣例。
- **內建 5 組虛構情境範例**：咖啡店週末優惠、瑜伽教室招生、手作甜點新品、寵物店颱風公休公告、選物店年中特賣，涵蓋親切口語／專業正式／促購優惠／正式公告四種語氣，一鍵套用快速上手。
- **只有 Facebook 做真正的自動發布**：透過官方 Facebook JS SDK 登入你自己的帳號，呼叫 Graph API 發布純文字／單圖／多圖／影片。Instagram／X／Threads 只做 AI 文案優化＋媒體預覽，**不做這三個平台的 API 串接**——複製文字後自行手動貼上發布。
- **BYOK 或不連網雙模式**：填自己的 Claude／OpenAI／Gemini／OpenRouter API 金鑰可用 AI 優化文案；不填金鑰則退回本機規則式引擎（依語氣範本＋關鍵詞 hashtag 產生基本版本，不連網、不需金鑰）。

## 怎麼用

1. 開啟 <https://m255525.github.io/SocialPost/>
2. 輸入主題／方向／內容，或點一個內建範例快速帶入
3. 選語氣、勾選要發布的平台，視需要上傳圖片／影片素材
4. 展開「AI 優化設定」貼上你自己的 API 金鑰（選用），按「產生優化貼文」
5. 檢查各平台的結果卡片，Facebook 卡片可登入粉專直接發布；其餘平台按「複製文字」自行貼上

詳細操作說明見 [manual.html](https://m255525.github.io/SocialPost/manual.html)。

### API 金鑰申請網址

| 服務商 | 申請網址 |
|---|---|
| Claude（Anthropic） | <https://console.anthropic.com/> |
| OpenAI | <https://platform.openai.com/api-keys> |
| Gemini（Google AI Studio） | <https://aistudio.google.com/apikey> |
| OpenRouter | <https://openrouter.ai/keys> |

### Facebook 發布功能前置條件

需要你自己已經有 Facebook 開發者帳號、一個 App，以及要發布的粉專之管理權限；把 App ID 貼進頁面上的欄位即可。申請開發者帳號與 App 的完整流程請參考 Facebook 官方文件，不在本工具的操作手冊涵蓋範圍內。

## 技術架構

純前端單檔工具，**沒有任何建置流程、框架、npm 依賴**：

| 項目 | 做法 |
|---|---|
| 貼文文案優化 | 瀏覽器直接 `fetch` 你選擇的 LLM 服務商官方 API（無後端代理）；未填金鑰則退回本機規則式引擎 |
| Facebook 發布 | 官方 Facebook JS SDK 登入 + `fetch`/`FormData` 直接呼叫 Graph API（純文字 `/feed`、單圖與多圖 `/photos`+`attached_media`、影片 `/videos`） |
| 金鑰／權杖／草稿儲存 | `localStorage`，只在使用者自己的瀏覽器裡 |
| 圖片／影片預覽 | `FileReader`／`URL.createObjectURL`，發布前純本機預覽 |

## 本機開發

不需要任何建置工具或安裝依賴，純靜態檔案：

```bash
git clone https://github.com/M255525/SocialPost.git
cd SocialPost
python -m http.server 8000
```

開啟 `http://localhost:8000`。

## 檔案結構

```
index.html       主程式（貼文素材 + 內建範例 + AI優化/規則式引擎 + 結果卡片 + Facebook發布）
manual.html       操作手冊
launcher.py       可攜式桌面版啟動器（PyInstaller 打包用，尚未打包成 exe）
CLAUDE.md         開發筆記／架構決策紀錄
```

## 隱私與資料

本 repo 公開的只有程式碼。你填寫的主題、勾選設定、AI 服務商設定、Facebook App ID 與登入權杖，只存在自己瀏覽器的 `localStorage`，不會上傳到本工具以外的任何伺服器。按下「產生優化貼文」且已填 API 金鑰時，主題內容會直接連線送到你選擇的 AI 服務商；按下「發布到 Facebook」時，貼文內容與媒體檔案會直接送到 Facebook Graph API——這兩個動作都不經過本工具作者或任何第三方伺服器。頁尾的訪客次數計數器僅記錄不重複頁面造訪次數，不追蹤個別使用者或收集任何個資。

## 授權/用途

僅供教學與個人使用，禁止未經授權公開發布、販售或商業化使用。
