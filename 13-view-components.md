# ASP.NET Core MVC 教學 (.NET 8)：自訂 View Component 檢視元件

在開發 Web 應用程式時，有些 UI 片段不僅僅是靜態的 HTML，它們還需要執行自己的業務邏輯來獲取資料。例如，一個顯示當前購物車摘要的側邊欄、一個顯示最新文章列表的區塊，或是一個動態的分類導覽選單。

對於這些「微型 MVC」場景，ASP.NET Core 提供了一個完美的解決方案：**View Component**。View Component 類似於一個迷你的 Controller，它有自己的邏輯和自己的 View，並且可以被嵌入到任何父 View 中。

## 1. View Component vs. Partial View

初學者可能會對 View Component 和 Partial View (部分檢視) 感到困惑。它們的核心區別在於**是否包含後端邏輯**。

*   **Partial View**: 是一個沒有對應 Controller Action 的 `.cshtml` 檔案。它純粹是為了重用 **HTML 標記**。它的資料完全由呼叫它的父 View 提供。
*   **View Component**: 是一個包含了**後端 C# 邏輯**和一個**對應 View** 的組合。它能夠自己執行業務邏輯（例如，呼叫服務、查詢資料庫）來獲取所需的資料，而不需要父 View 的干預。

**選擇原則：**
*   如果 UI 片段只需要呈現傳入的資料，沒有自己的邏輯 -> 使用 **Partial View**。
*   如果 UI 片段需要自己去獲取資料或執行業務邏輯 -> 使用 **View Component**。

## 2. 建立 View Component

建立一個 View Component 包含兩個主要步驟：

1.  **建立 View Component 類別**：一個繼承自 `ViewComponent` 的 C# 類別。
2.  **建立對應的 View**：一個 `.cshtml` 檔案，用於呈現該元件的 HTML。

### 範例：一個顯示最新公告的 View Component

我們來建立一個 `LatestAnnouncements` 元件，它會顯示最新的 3 條公告。

**步驟 1：建立 View Component 類別**

按照慣例，View Component 類別通常放在專案根目錄下的 `ViewComponents` 資料夾中。

**`ViewComponents/LatestAnnouncementsViewComponent.cs`**
```csharp
using Microsoft.AspNetCore.Mvc;

// 假設有一個 IAnnouncementRepository 用於獲取公告資料
public class LatestAnnouncementsViewComponent(IAnnouncementRepository announcementRepo) : ViewComponent
{
    // View Component 的核心方法是 InvokeAsync 或 Invoke
    // 它可以接受參數
    public async Task<IViewComponentResult> InvokeAsync(int count)
    {
        // 1. 執行自己的業務邏輯，獲取資料
        var latestAnnouncements = await announcementRepo.GetLatestAsync(count);

        // 2. 將資料傳遞給對應的 View，並回傳
        // "Default" 是預設的 View 名稱
        return View("Default", latestAnnouncements); 
    }
}
```
**關鍵點：**
*   類別名稱必須以 `ViewComponent` 結尾，或者使用 `[ViewComponent]` 屬性來標記。
*   核心方法必須是 `InvokeAsync` 或 `Invoke`，並且必須回傳 `IViewComponentResult`。
*   它可以像 Controller 一樣使用**建構函式注入**來獲取所需的服務。
*   `View()` 方法會尋找對應的 Razor View 來渲染。

**步驟 2：建立 View Component 的 View**

View Component 的 View 檔案存放位置有嚴格的慣例：
*   `/Views/{ControllerName}/Components/{ComponentName}/{ViewName}.cshtml`
*   `/Views/Shared/Components/{ComponentName}/{ViewName}.cshtml` (最常用)
*   `/Pages/Shared/Components/{ComponentName}/{ViewName}.cshtml` (用於 Razor Pages)

對於我們的例子，我們將建立在第二個位置。

**`Views/Shared/Components/LatestAnnouncements/Default.cshtml`**
```html
@* 這個 View 的模型是從 ViewComponent 傳過來的 IEnumerable<Announcement> *@
@model IEnumerable<Announcement>

<div class="card">
    <div class="card-header">
        最新公告
    </div>
    <ul class="list-group list-group-flush">
        @if (Model.Any())
        {
            @foreach (var announcement in Model)
            {
                <li class="list-group-item">
                    <strong>@announcement.Title</strong>
                    <small class="d-block text-muted">@announcement.PostedDate.ToShortDateString()</small>
                </li>
            }
        }
        else
        {
            <li class="list-group-item">目前沒有公告。</li>
        }
    </ul>
</div>
```

## 3. 如何在 View 中呼叫 View Component

有兩種方式可以在父 View (例如 `_Layout.cshtml` 或 `Index.cshtml`) 中呼叫 View Component。

### a. 使用 `@await Component.InvokeAsync()`

這是比較傳統的方式，直接呼叫元件的 `InvokeAsync` 方法。

```html
<div class="col-md-4">
    @* 
      第一個參數 "LatestAnnouncements" 是元件的名稱 (去掉了 "ViewComponent" 後綴)
      第二個參數是一個匿名物件，用於傳遞參數給 InvokeAsync 方法
    *@
    @await Component.InvokeAsync("LatestAnnouncements", new { count = 3 })
</div>
```

### b. 使用 Tag Helper (推薦)

從 ASP.NET Core 2.0 開始，提供了一種更簡潔、更像 HTML 的 Tag Helper 語法來呼叫 View Component。

首先，確保在 `_ViewImports.cshtml` 中註冊了你的專案組件。
`@addTagHelper *, MyWebApp`

然後，你就可以使用 `<vc:component-name>` 語法。

```html
<div class="col-md-4">
    @* 
      標籤名稱是 <vc:[component-name]>，kebab-case 格式
      參數直接寫成標籤的屬性
    *@
    <vc:latest-announcements count="3"></vc:latest-announcements>
</div>
```
**Tag Helper 語法是目前推薦的方式**，因為它更具可讀性，並且與其他 Tag Helper 的風格保持一致。

## 4. View Component 的優勢

1.  **關注點分離 (SoC)**：將一個複雜頁面分解成多個獨立、可自給自足的元件，每個元件只關心自己的邏輯和呈現。
2.  **可測試性**：View Component 是一個普通的 C# 類別，可以對其進行隔離的單元測試，驗證其業務邏輯是否正確。
3.  **避免父 Controller 膨脹**：如果沒有 View Component，父 View 所需的所有資料都必須由對應的 Controller Action 來準備，這會導致 Action 變得非常臃腫和複雜。
4.  **效能**：`InvokeAsync` 的設計讓 View Component 可以非同步地執行其資料庫查詢等 I/O 密集型操作，不會阻塞主執行緒。

## 結論

View Component 是 ASP.NET Core MVC 中一個用於建立可重用、帶有後端邏輯的 UI 元件的強大工具。

*   當你需要一個能**自己獲取資料**的 UI 片段時，選擇 View Component。
*   遵循**命名和資料夾結構的慣例**，框架才能正確找到你的元件和它的 View。
*   優先使用**Tag Helper 語法 (`<vc:...>`)** 來呼叫 View Component，讓你的 View 更乾淨。
*   善用**建構函式注入**，讓你的 View Component 能夠存取應用程式中的各種服務。

掌握 View Component，你就能夠以一種更有組織、更模組化的方式來建構複雜的使用者介面。
