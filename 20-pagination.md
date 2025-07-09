# ASP.NET Core MVC 教學 (.NET 8)：大量資料分頁的開發技巧

當應用程式需要顯示大量資料時（例如，產品列表、訂單歷史、使用者紀錄），一次性將所有資料從資料庫載入到記憶體並呈現在單一頁面上，是絕對不可行的。這不僅會消耗大量的伺服器記憶體和網路頻寬，更會導致瀏覽器因需要渲染過多的 HTML 元素而變得極度緩慢甚至崩潰。

解決這個問題的標準方案就是**分頁 (Pagination)**。分頁的核心思想是只從資料庫中選取使用者當前需要檢視的一小部分資料（一「頁」），從而實現高效的資料展示。

## 1. 分頁的兩種主要方式

### a. 偏移量分頁 (Offset-based Pagination)

這是最傳統、最常見的分頁方式。它依賴於兩個參數：
*   **頁碼 (Page Number)**：使用者想看的頁數。
*   **頁面大小 (Page Size)**：每頁顯示的項目數量。

在後端，我們使用這兩個參數來計算要從資料庫中「跳過 (Skip)」多少筆記錄，然後再「選取 (Take)」多少筆記錄。

**LINQ 查詢範例：**
```csharp
public async Task<List<Product>> GetProductsAsync(int pageNumber = 1, int pageSize = 10)
{
    // 確保頁碼和頁面大小在合理範圍內
    pageNumber = Math.Max(1, pageNumber);
    pageSize = Math.Clamp(pageSize, 5, 50);

    var products = await _context.Products
                                 .OrderBy(p => p.Name) // **分頁前必須先排序！**
                                 .Skip((pageNumber - 1) * pageSize)
                                 .Take(pageSize)
                                 .ToListAsync();
    return products;
}
```
**關鍵點**：在使用 `Skip` 和 `Take` 進行分頁之前，**必須**先對資料進行排序 (`OrderBy`)。如果沒有明確的排序，資料庫回傳記錄的順序是不確定的，這會導致分頁結果在不同請求之間出現混亂或重複。

**優點**：
*   實現簡單直觀。
*   允許使用者直接跳轉到任意頁面。

**缺點**：
*   **效能問題**：當 `Skip` 的數量非常大時（例如，跳到第一萬頁），許多資料庫的效能會急劇下降，因為它們仍然需要掃描並捨棄前面所有被跳過的記錄。
*   **資料不一致**：如果在使用者翻頁的過程中，有新的資料被插入或舊的資料被刪除，可能會導致某些項目被重複看到或被遺漏。

### b. 指標分頁 (Cursor-based Pagination / Keyset Pagination)

指標分頁是一種更現代、效能更好的分頁方式，特別適用於「無限滾動 (Infinite Scrolling)」的場景。它不使用頁碼，而是依賴於一個「指標 (Cursor)」，這個指標通常是上一頁**最後一筆記錄的唯一且有序的值**（例如 ID 或建立時間）。

查詢下一頁資料的邏輯變成：「給我 N 筆記錄，條件是這些記錄的排序值要大於（或小於）上次的指標值」。

**LINQ 查詢範例：**
假設我們按 `Id` 遞增排序。
```csharp
public async Task<List<Product>> GetProductsAfterAsync(int? lastSeenId = null, int pageSize = 10)
{
    var query = _context.Products.OrderBy(p => p.Id);

    if (lastSeenId.HasValue)
    {
        // 使用 WHERE 條件來取代 Skip
        query = query.Where(p => p.Id > lastSeenId.Value);
    }

    var products = await query.Take(pageSize).ToListAsync();
    return products;
}
```

**優點**：
*   **高效能**：`WHERE Id > @lastSeenId` 這樣的查詢可以高效地利用資料庫索引，即使在處理極大資料表時也能保持穩定的高效能。它避免了 `Skip` 帶來的效能問題。
*   **資料一致性**：由於查詢是基於一個固定的值，即使在翻頁過程中資料發生變動，也不會出現重複或遺漏項目的問題。

**缺點**：
*   **實現稍複雜**：需要傳遞和管理指標。
*   **無法直接跳頁**：使用者只能一頁一頁地向後（或向前）導覽，無法直接跳到第 50 頁。

## 2. 在 Controller 和 View 中實現

一個完整的分頁方案不僅需要後端的查詢邏輯，還需要將分頁資訊（如總頁數、目前頁碼）傳遞給 View，以便渲染分頁控制項。

### 建立一個分頁模型

我們可以建立一個通用的 `PagedList<T>` 或 `PaginationViewModel<T>` 來封裝分頁結果和相關資訊。

```csharp
public class PagedResult<T>
{
    public List<T> Items { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;
}
```

### Controller Action
```csharp
public async Task<IActionResult> Index(int pageNumber = 1)
{
    const int pageSize = 10;

    // 1. 獲取總項目數
    var totalCount = await _context.Products.CountAsync();

    // 2. 查詢當前頁的資料
    var items = await _context.Products
                              .OrderBy(p => p.Name)
                              .Skip((pageNumber - 1) * pageSize)
                              .Take(pageSize)
                              .ToListAsync();

    // 3. 建立分頁結果模型
    var pagedResult = new PagedResult<Product>
    {
        Items = items,
        PageNumber = pageNumber,
        PageSize = pageSize,
        TotalCount = totalCount
    };

    return View(pagedResult);
}
```
**效能提示**：注意，我們執行了兩次資料庫查詢（一次 `CountAsync`，一次 `ToListAsync`）。對於非常大的資料表，這可能是必要的。對於中小型資料表，有些開發者可能會選擇先載入所有 ID 到記憶體再分頁，但這通常不是最佳實踐。

### View 的實現

在 View 中，我們接收 `PagedResult<Product>` 作為模型，然後渲染資料列表和分頁控制項。

**`Index.cshtml`**
```html
@model PagedResult<Product>

<h1>產品列表</h1>

<table class="table">
    @foreach (var product in Model.Items)
    {
        <tr><td>@product.Name</td></tr>
    }
</table>

@* 分頁控制項 *@
<div>
    @if (Model.HasPreviousPage)
    {
        <a asp-action="Index" asp-route-pageNumber="@(Model.PageNumber - 1)" class="btn btn-secondary">
            上一頁
        </a>
    }
    @if (Model.HasNextPage)
    {
        <a asp-action="Index" asp-route-pageNumber="@(Model.PageNumber + 1)" class="btn btn-secondary">
            下一頁
        </a>
    }
    <span class="ms-3">
        頁碼 @Model.PageNumber / @Model.TotalPages
    </span>
</div>
```

## 3. 選擇哪種分頁方式？

*   對於傳統的、需要顯示頁碼並允許使用者隨意跳轉的管理後台或列表頁面，**偏移量分頁** 仍然是一個可接受且易於實現的選擇。你需要意識到它在極大資料量下的潛在效能問題。
*   對於面向使用者的、採用「無限滾動」或「載入更多」按鈕的資訊流（如社交媒體、新聞列表），**指標分頁** 是效能和體驗上的最佳選擇。

## 結論

處理大量資料分頁是後端開發的基本功。

1.  **永遠在資料庫層面分頁**：絕對不要將所有資料載入到應用程式記憶體中再進行分頁。使用 `Skip`/`Take` 或 `WHERE` 條件讓資料庫完成這項繁重的工作。
2.  **分頁前必須排序**：這是保證分頁結果穩定性和正確性的前提。
3.  **了解兩種模式的權衡**：根據你的具體場景（需要跳頁 vs. 無限滾動）來選擇使用偏移量分頁還是指標分頁。
4.  **封裝分頁邏輯**：建立一個可重用的分頁模型 (如 `PagedResult<T>`) 來封裝分頁資料和元數據，讓 Controller 和 View 的程式碼更乾淨。
