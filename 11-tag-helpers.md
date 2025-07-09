# ASP.NET Core MVC 教學 (.NET 8)：使用 Tag Helper 開發檢視頁面

在傳統的 ASP.NET MVC 中，我們經常使用 `@Html.ActionLink()`, `@Html.BeginForm()` 這類的 HTML 輔助方法 (HTML Helpers) 來產生 HTML。雖然功能強大，但它們的語法是 C# 方法呼叫，與原生 HTML 的樣貌相去甚遠，對於前端開發者來說可能不太直觀。

為了解決這個問題，ASP.NET Core 引入了 **Tag Helper**。Tag Helper 是一種伺服器端元件，它能以屬性 (Attribute) 的形式附加到標準的 HTML 標籤上，並在伺服器端處理這些標籤，最終產生新的 HTML。這種方式讓 Razor 檔案看起來更像純 HTML，提高了可讀性和可維護性。

## 1. 啟用 Tag Helper

要使用 Tag Helper，你必須在專案中啟用它們。這通常是透過在 `_ViewImports.cshtml` 檔案中加入 `@addTagHelper` 指令來完成的。預設的專案範本已經幫我們做好了。

**`_ViewImports.cshtml`**
```csharp
// "*": 載入指定組件中的所有 Tag Helper
// "Microsoft.AspNetCore.Mvc.TagHelpers": 包含內建 Tag Helper 的組件名稱
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers 
```

## 2. 常用的內建 Tag Helper

ASP.NET Core 提供了許多非常有用的內建 Tag Helper，涵蓋了表單、連結、圖片、環境等常用功能。

### a. 表單相關 (Form Tag Helpers)

這是最常用的一組 Tag Helper，它們極大地簡化了表單的建立和與模型的繫結。

*   **`<form>` (Form Tag Helper)**:
    *   `asp-controller`: 指定目標 Controller。
    *   `asp-action`: 指定目標 Action。
    *   `asp-route-{value}`: 添加額外的路由參數。

*   **`<label>` (Label Tag Helper)**:
    *   `asp-for`: 繫結到模型的某個屬性，會自動從模型的 `[Display]` 屬性產生標籤文字。

*   **`<input>`, `<textarea>`, `<select>` (Input and Select Tag Helpers)**:
    *   `asp-for`: 繫結到模型的屬性，會自動設定 `id`, `name` 和 `value`。對於 `<select>`，它還能處理 `<option>` 的選中狀態。
    *   `asp-items`: 用於 `<select>` 標籤，指定下拉選單的選項來源 (必須是 `IEnumerable<SelectListItem>`)。

*   **`<span>` (Validation Message Tag Helper)**:
    *   `asp-validation-for`: 顯示指定模型屬性的驗證錯誤訊息。

*   **`<div>` (Validation Summary Tag Helper)**:
    *   `asp-validation-summary`: 顯示所有或僅模型層級的驗證錯誤摘要。

**綜合範例：**
```html
@model LoginViewModel

<div asp-validation-summary="ModelOnly" class="text-danger"></div>

<form asp-controller="Account" asp-action="Login" asp-route-returnurl="@Model.ReturnUrl" method="post">
    <div class="form-group">
        <label asp-for="Email"></label>
        <input asp-for="Email" class="form-control" />
        <span asp-validation-for="Email" class="text-danger"></span>
    </div>
    <div class="form-group">
        <label asp-for="Password"></label>
        <input asp-for="Password" class="form-control" />
        <span asp-validation-for="Password" class="text-danger"></span>
    </div>
    <button type="submit" class="btn btn-primary">登入</button>
</form>
```
這段程式碼會被渲染成包含正確 `action`, `id`, `name`, `value` 和 `data-val` 屬性的標準 HTML 表單，實現了強型別繫結和客戶端驗證。

### b. 連結與圖片 (Anchor and Image Tag Helpers)

*   **`<a>` (Anchor Tag Helper)**:
    *   `asp-controller`, `asp-action`, `asp-route-{value}`: 用來產生指向 MVC Action 的 URL。這比手動拼接 URL 更安全，因為它會遵循你的路由表。
    *   `asp-fragment`: 指定 URL 中的片段部分 (hash tag)，例如 `#details`。

*   **`<script>`, `<link>`, `<img>` (Cache Busting Tag Helpers)**:
    *   `asp-append-version="true"`: 這是一個非常有用的功能！它會為靜態檔案 (JS, CSS, 圖片) 的 URL 自動附加一個版本號查詢字串 (例如 `?v=...`)，這個版本號是根據檔案內容的雜湊值產生的。當你更新檔案內容時，版本號會改變，強制瀏覽器下載新檔案，而不是使用舊的快取版本。

**範例：**
```html
<!-- 產生連結 -->
<a asp-controller="Products" asp-action="Details" asp-route-id="123">查看產品 123</a>

<!-- 啟用快取破壞 -->
<link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
<script src="~/js/site.js" asp-append-version="true"></script>
```

### c. 環境 (Environment Tag Helper)

*   **`<environment>`**: 這個 Tag Helper 允許你根據當前的執行環境 (Development, Staging, Production) 來決定是否渲染其中的內容。這對於在開發時載入未壓縮的 CSS/JS，而在生產環境中載入壓縮後的版本非常有用。

**範例：**
```html
<environment include="Development">
    <!-- 這段內容只在開發環境中呈現 -->
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.css" />
</environment>
<environment exclude="Development">
    <!-- 這段內容在非開發環境 (Staging, Production) 中呈現 -->
    <link rel="stylesheet" 
          href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css"
          integrity="..."
          crossorigin="anonymous" />
</environment>
```

### d. 部分檢視 (Partial Tag Helper)

*   **`<partial>`**: 用來渲染一個部分檢視 (Partial View)。它比傳統的 `@Html.Partial()` 或 `@Html.RenderPartial()` 語法更直觀。
    *   `name`: 指定 Partial View 的名稱。
    *   `for`: 傳遞一個模型運算式給 Partial View。
    *   `model`: 傳遞一個物件實例給 Partial View。

**範例：**
```html
<!-- 渲染 _UserInfo.cshtml，並將 Model.CurrentUser 作為其模型 -->
<partial name="_UserInfo" for="CurrentUser" />
```

## 3. Tag Helper 的優勢

1.  **HTML-Friendly 語法**：Razor 檔案看起來更像原生 HTML，前端開發者更容易理解和編輯。
2.  **強型別與 IntelliSense**：由於 `asp-for` 等屬性是強型別的，Visual Studio/VS Code 可以提供更好的程式碼提示和編譯時檢查。
3.  **關注點分離**：將產生 HTML 的複雜邏輯從 View 中抽離到 C# 類別中，讓 View 更乾淨。
4.  **可測試性**：Tag Helper 本身是 C# 類別，可以對其進行單元測試。

## 結論

Tag Helper 是 ASP.NET Core MVC 中一個革命性的改進，它讓 View 的開發體驗更加流暢和直觀。

*   **優先使用 Tag Helper**：對於表單、連結等常見任務，應優先使用內建的 Tag Helper，而不是舊的 HTML Helper。
*   **善用 `asp-append-version`**：務必為你的 CSS 和 JS 檔案啟用此功能，以避免客戶端快取問題。
*   **利用環境 Tag Helper**：根據不同環境載入不同的資源，以優化開發和部署流程。

從 C# 方法呼叫轉變為 HTML 屬性，Tag Helper 成功地在伺服器端程式碼的強大功能與前端開發的熟悉體驗之間架起了一座橋樑。
