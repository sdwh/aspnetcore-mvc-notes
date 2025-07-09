# ASP.NET Core MVC 教學 (.NET 8)：深入理解網址路由與 SEO 實作技巧

路由 (Routing) 是 ASP.NET Core 的核心功能，它負責解析傳入的 HTTP 請求 URL，並將其分派到對應的處理程式 (在 MVC 中通常是控制器的動作方法)。一個設計良好的路由系統不僅能讓應用程式的結構更清晰，更是實現搜尋引擎最佳化 (SEO) 的基石。

本文將引導你深入了解 ASP.NET Core MVC 的路由機制，並分享結合 .NET 8 特性的 SEO 實作技巧。

## 1. 路由的兩種主要風格

ASP.NET Core MVC 主要支援兩種設定路由的方式：

### a. 約定式路由 (Convention-based Routing)

這是在 `Program.cs` (或舊版的 `Startup.cs`) 中，透過 `MapControllerRoute` 方法定義全域路由規則的傳統方式。請求的 URL 會根據預先設定的範本 (pattern) 來對應到 Controller 和 Action。

**`Program.cs` 設定範例：**

```csharp
var builder = WebApplication.CreateBuilder(args);

// 添加 MVC 服務
builder.Services.AddControllersWithViews();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

app.UseAuthorization();

// 定義一個名為 'default' 的約定式路由
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

- **`{controller=Home}`**: URL 的第一段對應到控制器名稱，如果省略，則預設為 `Home`。
- **`{action=Index}`**: 第二段對應到動作方法名稱，如果省略，則預設為 `Index`。
- **`{id?}`**: 第三段是可選的 (`?` 的作用) 參數 `id`。

### b. 屬性式路由 (Attribute-based Routing)

這是將路由規則直接定義在控制器或動作方法上的現代作法。它提供了更精確的控制力，讓 URL 結構與程式碼邏輯緊密結合，特別適合開發 Web API 或需要高度自訂 URL 的情境。

**控制器範例：**

```csharp
[Route("products")] // 在控制器層級設定路由前綴
public class ProductsController : Controller
{
    [HttpGet] // GET /products
    public IActionResult Index() { /* ... */ }

    [HttpGet("{id:int}")] // GET /products/123 (含約束)
    public IActionResult Details(int id) { /* ... */ }

    [HttpGet("special-offers")] // GET /products/special-offers
    public IActionResult SpecialOffers() { /* ... */ }
}
```

**最佳實務建議**：對於大多數現代應用程式，**優先考慮使用屬性式路由**。它提供了更好的可讀性和可維護性，因為 URL 結構直接寫在對應的 Action 上方，一目了然。約定式路由則適用於非常傳統、遵循固定 `Controller/Action/Id` 模式的網站。

## 2. 路由約束 (Route Constraints)

路由約束能限制路由參數必須符合的格式，確保只有合法的 URL 才能匹配到 Action，從而避免錯誤或非預期的行為。

**範例：**

```csharp
// 只有當 id 是整數時才會匹配
[HttpGet("user/{id:int}")]
public IActionResult GetUserById(int id) { /* ... */ }

// 只有當 name 至少有 3 個字元時才會匹配
[HttpGet("user/{name:minlength(3)}")]
public IActionResult GetUserByName(string name) { /* ... */ }

// 使用正則表達式約束，sku 必須是 "SKU" 開頭後面接 4 位數字
[HttpGet("items/{sku:regex(^SKU\d{{4}}$)}")]
public IActionResult GetItemBySku(string sku) { /* ... */ }
```

使用路由約束可以有效過濾無效請求，並回傳 404 Not Found，這比在 Action 內部用 `if` 判斷更有效率。

## 3. SEO 友善的 URL 設計技巧

一個對 SEO 友善的 URL 應該是簡短、有意義且易於人類閱讀的。這有助於搜尋引擎理解頁面內容，也方便使用者分享。

### a. 使用 "Slug" 取代 ID

與其使用像 `/posts/123` 這樣無意義的 ID，不如使用從標題產生的 "slug"，例如 `/posts/aspnet-core-routing-guide`。

**實作 Slug：**

1.  在你的資料模型中增加一個 `Slug` 欄位。
2.  當建立或更新內容時，根據標題自動產生一個 URL 友善的 slug (小寫、用連字號 `-` 取代空白和特殊字元)。
3.  在路由中使用 slug 作為識別碼。

**控制器範例：**

```csharp
public class BlogController(IBlogRepository blogRepo) : Controller
{
    // 路由使用 slug，並約束它不能只是數字
    [HttpGet("post/{slug:alpha:minlength(5)}")]
    public async Task<IActionResult> Post(string slug)
    {
        var post = await blogRepo.GetBySlugAsync(slug);
        if (post == null)
        {
            return NotFound();
        }
        return View(post);
    }
}
```

### b. 小寫 URL

搜尋引擎將大小寫不同的 URL 視為不同的頁面，這可能導致重複內容問題。我們可以強制所有 URL 都轉為小寫。

**`Program.cs` 設定：**

```csharp
// ... 其他設定 ...

// 添加路由服務，並設定 URL 為小寫
builder.Services.AddRouting(options =>
{
    options.LowercaseUrls = true;
    options.LowercaseQueryStrings = true; // 查詢字串也轉為小寫
});

var app = builder.Build();

// ...
```

### c. 移除結尾斜線 (Trailing Slash)

`/products/` 和 `/products` 可能被視為兩個不同的 URL。統一它們的格式很重要。

**`Program.cs` 設定 (使用 URL Rewrite 中介軟體)：**

```csharp
using Microsoft.AspNetCore.Rewrite;

// ...
var app = builder.Build();

// 設定重寫規則，將結尾有斜線的 URL 301 永久重導向到無斜線的版本
var rewriteOptions = new RewriteOptions()
    .AddRedirect("(.*)/$", "$1", StatusCodes.Status301MovedPermanently);
app.UseRewriter(rewriteOptions);

// ...
```

## 4. .NET 8 Minimal API 與 MVC 路由整合

雖然本文主要討論 MVC，但值得一提的是，.NET 8 的 Minimal API 提供了更輕量、高效能的端點定義方式。在一個專案中，你可以混合使用 MVC 控制器和 Minimal API 端點，它們共享同一個路由系統。

**`Program.cs` 混合範例：**

```csharp
// ...
var app = builder.Build();

// ... (MVC 路由設定)
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

// 定義一個 Minimal API 端點
app.MapGet("/api/health", () => Results.Ok(new { Status = "Healthy" }))
   .WithTags("System"); // 使用 OpenAPI 標籤分組

app.Run();
```

## 結論

路由是 ASP.NET Core MVC 應用程式的骨架。透過策略性地使用屬性式路由、路由約束，並遵循 SEO 最佳實務，你可以打造出結構清晰、效能優異且搜尋引擎友善的網站。

1.  **優先使用屬性式路由**：讓 URL 定義與程式碼邏輯保持同步。
2.  **善用路由約束**：在路由層級就過濾掉無效請求。
3.  **設計人類可讀的 URL**：使用 slug、小寫 URL 並統一結尾斜線格式。
4.  **保持一致性**：為你的網站選擇一種主要的 URL 結構風格並堅持下去。
