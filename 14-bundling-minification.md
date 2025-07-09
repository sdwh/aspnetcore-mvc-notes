# ASP.NET Core MVC 教學 (.NET 8)：正確的使用 JS/CSS 合併與最小化

在現代 Web 開發中，網站的載入速度是影響使用者體驗和 SEO 排名的關鍵因素。一個主要的優化手段就是減少 HTTP 請求的數量和傳輸的資料大小。對於 JavaScript (JS) 和 CSS 檔案來說，這可以透過「**合併 (Bundling)**」和「**最小化 (Minification)**」來實現。

*   **合併 (Bundling)**：將多個 JS 或 CSS 檔案合併成一個單一的檔案。這樣瀏覽器就只需要發起一次 HTTP 請求，而不是多次，從而減少了網路延遲。
*   **最小化 (Minification)**：從 JS 和 CSS 檔案中移除所有不必要的字元（如空白、換行、註解），並縮短變數名稱，以減小檔案的體積，加快下載速度。

ASP.NET Core 提供了多種工具和策略來實現這個目標，並且強調在**開發環境**中使用原始檔案以便偵錯，在**生產環境**中使用合併和最小化後的檔案以獲得最佳效能。

## 1. 開發環境 vs. 生產環境

ASP.NET Core 的環境管理機制在這裡扮演了重要角色。我們通常會使用 `environment` Tag Helper 來根據當前環境載入不同的檔案。

**`_Layout.cshtml` 中的典型範例：**
```html
<environment include="Development">
    <!-- 開發時，直接引用原始的 CSS 檔案 -->
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.css" />
    <link rel="stylesheet" href="~/css/site.css" />
</environment>
<environment exclude="Development">
    <!-- 生產時，引用合併且最小化後的 site.bundle.min.css -->
    <link rel="stylesheet" href="~/css/site.bundle.min.css" asp-append-version="true" />
</environment>
```
這種模式確保了開發者在偵錯時看到的是可讀的原始碼，而使用者在生產環境中獲得的是最佳化的版本。

## 2. 如何實現合併與最小化？

在早期的 ASP.NET Core 版本中，曾有內建的執行時期 (runtime) 合併與最小化工具。但從 ASP.NET Core 3.0 開始，官方推薦將這個過程轉移到**建置時期 (build-time)** 處理。這意味著合併與最小化的操作應該在專案部署之前完成，而不是在伺服器收到請求時才動態處理。

這樣做的好處是：
*   **減少伺服器負擔**：伺服器不需要在每次應用程式啟動或請求時都消耗 CPU 資源來處理這些檔案。
*   **更穩定**：預先處理好的檔案更可預測，避免了執行時期可能發生的錯誤。
*   **與現代前端工作流程整合**：可以與 Gulp, Webpack, Vite 等專業的前端建置工具無縫整合。

目前，對於大多數 ASP.NET Core 專案，最推薦的工具是 **BuildBundlerMinifier**。

## 3. 使用 BuildBundlerMinifier

`BuildBundlerMinifier` 是一個 NuGet 套件，它提供了一種簡單的方式來設定和執行合併與最小化，並且能與 Visual Studio 的建置過程整合。

**步驟 1：安裝 NuGet 套件**

在你的專案中，安裝 `BuildBundlerMinifier` NuGet 套件。

```bash
dotnet add package BuildBundlerMinifier
```

**步驟 2：建立設定檔 `bundleconfig.json`**

在專案的根目錄下，新增一個名為 `bundleconfig.json` 的檔案。這個檔案用來定義哪些檔案要被合併，以及輸出的檔案名稱和位置。

**`bundleconfig.json` 範例：**
```json
[
  {
    "outputFileName": "wwwroot/css/site.bundle.css",
    "inputFiles": [
      "wwwroot/lib/bootstrap/dist/css/bootstrap.css",
      "wwwroot/css/site.css"
    ],
    // 可以為最小化單獨設定選項
    "minify": {
      "enabled": true,
      "renameLocals": true
    },
    // 產生 Source Map 有助於在生產環境偵錯
    "sourceMap": false
  },
  {
    "outputFileName": "wwwroot/js/site.bundle.js",
    "inputFiles": [
      "wwwroot/lib/jquery/dist/jquery.js",
      "wwwroot/lib/bootstrap/dist/js/bootstrap.bundle.js",
      "wwwroot/js/site.js"
    ],
    "minify": {
      "enabled": true,
      "renameLocals": true
    },
    "sourceMap": false
  }
]
```
*   `outputFileName`: 指定合併後輸出的檔案路徑和名稱。通常會輸出到 `wwwroot` 資料夾下。
*   `inputFiles`: 一個陣列，按照順序列出需要被合併的原始檔案。**順序非常重要**，因為像 jQuery 這種基礎函式庫必須在依賴它的腳本之前被載入。
*   `minify`: 控制最小化選項。

**步驟 3：執行合併與最小化**

安裝完套件並建立好設定檔後，有幾種方式可以觸發這個過程：

*   **在 Visual Studio 中**：
    *   右鍵點擊 `bundleconfig.json` 檔案，你會看到「Bundler & Minifier」的相關選單，可以手動更新、清理。
    *   更重要的是，它會自動掛鉤到「建置 (Build)」事件上。當你建置專案時，它會自動檢查輸入檔案是否有變更，並在需要時重新產生合併後的檔案。
*   **使用命令列**：
    你可以使用 `dotnet bundle` 命令來手動執行。
    ```bash
    dotnet bundle
    ```
    這對於設定 CI/CD (持續整合/持續部署) 管線非常有用。

## 4. 搭配 `asp-append-version`

在 `_Layout.cshtml` 中，我們看到 `<link>` 和 `<script>` 標籤上使用了 `asp-append-version="true"`。這個 Tag Helper 會為靜態檔案 URL 附加一個版本號查詢字串，該版本號是根據檔案內容的雜湊值計算的。

```html
<link rel="stylesheet" href="~/css/site.bundle.min.css" asp-append-version="true" />
<script src="~/js/site.bundle.min.js" asp-append-version="true"></script>
```
**渲染結果可能像這樣：**
`<link rel="stylesheet" href="/css/site.bundle.min.css?v=Kl_d-5_f_e_g-h_i-j_k" />`

這是一個至關重要的最佳實務！它解決了**瀏覽器快取**的問題。當你部署新版的 CSS/JS 檔案時，由於檔案內容改變，雜湊值也會改變，URL 跟著改變，這會強制使用者的瀏覽器下載最新的檔案，而不是使用存在快取中的舊版本。

## 結論

正確地管理前端資源是現代 Web 應用程式效能優化的基礎。

1.  **區分環境**：使用 `environment` Tag Helper 在開發和生產環境中載入不同的資源。
2.  **建置時期處理**：採用 `BuildBundlerMinifier` 這類工具，在專案建置時就完成合併與最小化，以減輕伺服器負擔。
3.  **注意順序**：在 `bundleconfig.json` 中，確保檔案（特別是 JS）的載入順序是正確的。
4.  **啟用快取破壞**：永遠為你的合併後檔案啟用 `asp-append-version="true"`，以確保使用者總能獲取到最新的版本。

遵循這些原則，你的 ASP.NET Core 應用程式將能以更快的速度、更少的請求，為使用者提供更流暢的體驗。
