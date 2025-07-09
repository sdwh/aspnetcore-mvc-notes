# ASP.NET Core MVC 教學 (.NET 8)：瞭解主版頁面與實務開發技巧

在建構網站時，幾乎所有的頁面都會共享相同的佈局，例如頁首 (Header)、頁尾 (Footer)、導覽列 (Navigation Bar) 和側邊欄 (Sidebar)。如果我們在每個獨立的 View 檔案中都複製貼上這些共通的 HTML 結構，那將會是一場維護的災難。

為了解決這個問題，ASP.NET Core MVC 提供了「主版頁面」(Layout Page)，在 Razor 中通常稱為 **Layout**。它允許我們定義一個共通的 HTML 範本，然後讓其他 View 將其自身的內容「注入」到這個範本的指定位置。

## 1. Layout 的基本結構

在一個標準的 ASP.NET Core MVC 專案中，預設的 Layout 檔案通常位於 `Views/Shared/_Layout.cshtml`。`Shared` 資料夾是一個特殊的資料夾，其中的 View 和 Partial View 可以被任何 Controller 共用。

一個典型的 `_Layout.cshtml` 檔案結構如下：

**`_Layout.cshtml`**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - MyWebApp</title>
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
</head>
<body>
    <header>
        <nav class="navbar navbar-expand-sm navbar-light bg-white border-bottom box-shadow mb-3">
            <!-- 導覽列內容 -->
        </nav>
    </header>

    <div class="container">
        <main role="main" class="pb-3">
            @RenderBody() <!-- 核心：子頁面內容將會被渲染到這裡 -->
        </main>
    </div>

    <footer class="border-top footer text-muted">
        <div class="container">
            &copy; 2024 - MyWebApp
        </div>
    </footer>

    <script src="~/lib/jquery/dist/jquery.min.js"></script>
    <script src="~/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>

    @await RenderSectionAsync("Scripts", required: false) <!-- 可選區塊：用於載入特定頁面的腳本 -->
</body>
</html>
```

### 關鍵方法：
*   **`@RenderBody()`**: 這是 Layout 的核心。每個使用此 Layout 的子 View 的內容，都會被渲染在這個方法所在的位置。一個 Layout 檔案中**只能有一個** `@RenderBody()`。
*   **`@RenderSectionAsync(name, required)`**: 這個方法定義了一個「區塊 (Section)」。子 View 可以選擇性地提供這個區塊的內容。
    *   `name`: 區塊的名稱，例如 "Scripts", "SideBar"。
    *   `required`: 一個布林值。如果設為 `true`，則所有使用此 Layout 的 View 都**必須**定義這個區塊，否則會拋出錯誤。如果設為 `false`，則是可選的。

## 2. 如何在 View 中使用 Layout

### a. 指定 Layout

在一個子 View (例如 `Views/Home/Index.cshtml`) 的頂部，可以透過設定 `Layout` 屬性來指定它要使用哪個主版頁面。

```csharp
@{
    ViewData["Title"] = "首頁";
    Layout = "~/Views/Shared/_Layout.cshtml"; // 明確指定 Layout 路徑
}
```

### b. `_ViewStart.cshtml`

在每個 View 中都寫一次 `Layout = ...` 仍然很繁瑣。ASP.NET Core 提供了一個特殊的檔案 `_ViewStart.cshtml`，它位於 `Views` 資料夾的根目錄。在這個檔案中設定的程式碼，會在該資料夾及其子資料夾下的所有 View 執行**之前**被執行。

因此，我們通常會在這裡設定全站預設的 Layout。

**`_ViewStart.cshtml`**
```csharp
@{
    Layout = "_Layout"; // Razor 會自動在 /Views/Shared/ 或當前目錄尋找 _Layout.cshtml
}
```
有了這個檔案，除非你需要為某個特定的 View 指定一個不同的 Layout，否則就不需要在 View 檔案中再寫 `Layout = ...` 了。

### c. 定義 Section

如果 Layout 中定義了 Section，子 View 可以使用 `@section` 語法來提供其內容。

**`Index.cshtml`**
```html
@{
    ViewData["Title"] = "產品列表";
}

<h1>產品列表</h1>
<p>這裡是產品列表的內容...</p>

@section Scripts {
    <script>
        // 這段 script 只會在這個頁面被載入
        console.log("歡迎來到產品列表頁面！");
    </script>
}
```
這段 `<script>` 將會被渲染到 `_Layout.cshtml` 中 `@await RenderSectionAsync("Scripts", ...)` 的位置。這對於只在特定頁面才需要的 JavaScript 檔案或程式碼來說，是最佳的載入方式。

## 3. 實務開發技巧

### a. 巢狀 Layout (Nested Layouts)

有時候你可能需要多層次的 Layout。例如，一個網站的「後台管理」區塊，所有頁面都有相同的側邊欄，但它們仍然共享全站的頁首和頁尾。

這時，你可以建立一個「後台專用」的 Layout，而這個 Layout 本身又去使用「全站共用」的 Layout。

**`_AdminLayout.cshtml`**
```csharp
@{
    Layout = "_Layout"; // 這個後台 Layout 使用了全站的 _Layout
}

<h2>後台管理</h2>

<div class="row">
    <div class="col-md-3">
        <!-- 後台專用的側邊欄 -->
        <ul class="nav flex-column">
            <li class="nav-item"><a class="nav-link" href="#">儀表板</a></li>
            <li class="nav-item"><a class="nav-link" href="#">使用者管理</a></li>
        </ul>
    </div>
    <div class="col-md-9">
        @RenderBody() <!-- 注入後台各個子頁面的內容 -->
    </div>
</div>

@section Scripts {
    @* 如果後台頁面也定義了 Scripts 區塊，將其內容渲染出來 *@
    @if (IsSectionDefined("Scripts"))
    {
        @RenderSection("Scripts", required: false)
    }
}
```
**`IsSectionDefined(name)`** 方法可以用來檢查子 View 是否定義了某個 Section。

### b. 根據條件選擇 Layout

在 `_ViewStart.cshtml` 或 Controller 的 Action 方法中，你可以根據邏輯來動態決定要使用哪個 Layout。

**`_ViewStart.cshtml`**
```csharp
@{
    // 假設請求的路徑包含 /Admin/
    if (Context.Request.Path.StartsWithSegments("/Admin"))
    {
        Layout = "_AdminLayout";
    }
    else
    {
        Layout = "_Layout";
    }
}
```

## 結論

Layout 是 Razor 中實現程式碼複用和保持網站外觀一致性的關鍵工具。

1.  **`_Layout.cshtml`**: 定義網站的通用 HTML 骨架。
2.  **`@RenderBody()`**: 標示子 View 內容的插入點，每個 Layout 只能有一個。
3.  **`@RenderSectionAsync()`**: 定義可選或必要的內容區塊，常用於載入 CSS 或 JavaScript。
4.  **`_ViewStart.cshtml`**: 設定預設的 Layout，避免在每個 View 中重複指定。
5.  **巢狀 Layout**: 透過讓一個 Layout 使用另一個 Layout，可以實現更複雜的佈局結構。

善用 Layout，可以讓你的 View 檔案保持乾淨，只專注於該頁面獨有的內容，從而大幅提升開發效率和網站的可維護性。
