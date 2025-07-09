# ASP.NET Core MVC 教學 (.NET 8)：深入理解模型繫結 (Model Binding)

模型繫結 (Model Binding) 是 ASP.NET Core MVC 框架的核心功能之一。它扮演著橋樑的角色，自動將傳入的 HTTP 請求資料 (例如表單欄位、路由參數、查詢字串) 轉換並對應到控制器動作 (Action) 方法的參數或其屬性上。精通模型繫結的運作方式，能讓你寫出更簡潔、更強健、更安全的控制器邏輯。

本文將使用 .NET 8 和最新的 C# 語法，帶你深入探索模型繫結的內部世界。

## 1. 模型繫結的來源 (Binding Sources)

ASP.NET Core MVC 會依照特定順序，從不同的請求來源尋找可用的資料來進行繫結。主要的來源包含：

| 屬性 (Attribute) | 來源說明 |
| :--- | :--- |
| `[FromRoute]` | 從路由範本中解析出的資料。例如 `/products/edit/123` 中的 `123`。 |
| `[FromQuery]` | 從查詢字串 (Query String) 中獲取的資料。例如 `/search?keyword=apple` 中的 `apple`。 |
| `[FromForm]` | 從 `Content-Type` 為 `application/x-www-form-urlencoded` 或 `multipart/form-data` 的請求主體 (Request Body) 中獲取的表單資料。 |
| `[FromBody]` | 從請求主體中獲取資料，通常用於 JSON 或 XML 格式的 Web API。一個 Action 中最多只能有一個 `[FromBody]` 參數。 |
| `[FromHeader]` | 從 HTTP 標頭 (Header) 中獲取資料。 |

如果沒有明確指定來源屬性，模型繫結器會依照預設的順序嘗試從各個來源尋找符合的資料，這個順序通常是：**Form -> Route -> Query**。然而，為了程式碼的清晰與可預測性，**強烈建議**總是明確地使用來源屬性。

## 2. 程式碼實務範例

讓我們透過一些範例來實際感受模型繫結的運作。

### 範例模型

首先，定義一個 ViewModel。這裡我們使用 C# 11 引入的 `required` 關鍵字，確保模型在建立時必須提供 `Name` 屬性，這有助於在編譯時期就發現潛在的 `null` 參考問題。

```csharp
public class ProductViewModel
{
    // 通常 Id 在建立時由後端生成，所以不設為 required
    public int Id { get; set; }

    // 品名與價格是必須的
    public required string Name { get; set; }
    public required decimal Price { get; set; }
}
```

### 控制器與動作方法

接著，我們建立一個 `ProductsController`。這裡採用 .NET 8 的**主建構函式 (Primary Constructor)** 來注入 `ILogger`，讓程式碼更簡潔。

```csharp
// 使用主建構函式注入依賴
public class ProductsController(ILogger<ProductsController> logger) : Controller
{
    private readonly ILogger<ProductsController> _logger = logger;

    // 範例 1: 從路由和查詢字串繫結簡單型別
    // 請求 URL: /Products/Details/101?showHistory=true
    [HttpGet("Products/Details/{id}")]
    public IActionResult Details([FromRoute] int id, [FromQuery] bool showHistory = false)
    {
        _logger.LogInformation("正在檢視產品 ID: {ProductId}，顯示歷史紀錄: {ShowHistory}", id, showHistory);
        // ... 查詢並回傳產品資料 ...
        return View();
    }

    // 範例 2: 從表單繫結複雜型別 (ViewModel)
    // 請求來自一個 <form method="post">
    [HttpPost]
    public IActionResult Create([FromForm] ProductViewModel model)
    {
        if (!ModelState.IsValid)
        {
            // 如果模型驗證失敗，回傳原始表單並顯示錯誤訊息
            return View(model);
        }

        _logger.LogInformation("成功建立產品: {ProductName}，價格: {ProductPrice}", model.Name, model.Price);
        // ... 儲存產品到資料庫 ...
        return RedirectToAction(nameof(Details), new { id = 102 }); // 假設新產品 ID 是 102
    }

    // 範例 3: 混合來源繫結
    // 請求 URL: /api/v2/products/search?keyword=laptop
    [HttpGet("/api/v{version:apiVersion}/products/search")]
    public IActionResult SearchApi([FromQuery] string keyword, [FromRoute] string version)
    {
        // 這是 API 情境，通常回傳 JSON
        return Json(new { ApiVersion = version, SearchTerm = keyword });
    }
    
    // 範例 4: 從請求主體繫結 (適用於 Web API)
    // 請求: PUT /api/products/101
    // Body: { "id": 101, "name": "進階無線滑鼠", "price": 1500 }
    [HttpPut("/api/products/{id}")]
    public IActionResult UpdateProduct([FromRoute] int id, [FromBody] ProductViewModel model)
    {
        if (id != model.Id)
        {
            return BadRequest("路由 ID 與模型 ID 不一致。");
        }
        
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }

        _logger.LogInformation("正在更新產品 ID: {ProductId}", id);
        // ... 更新資料庫 ...
        return Ok(model);
    }
}
```

## 3. 模型驗證 (Model Validation)

模型繫結與模型驗證密不可分。在繫結完成後，MVC 框架會自動觸發模型驗證。你可以透過 `ModelState.IsValid` 屬性來檢查繫結與驗證的結果是否成功。如果失敗 (例如 `required` 的欄位沒有提供值，或資料格式不符)，`ModelState` 會包含詳細的錯誤資訊，你可以將其回傳給使用者。

## 結論

模型繫結是 ASP.NET Core MVC 中一個強大且便利的機制。透過本文的介紹，你應該對其運作原理有了更深入的理解：

1.  **明確指定來源**：總是使用 `[FromRoute]`, `[FromQuery]`, `[FromForm]` 等屬性，增加程式碼的可讀性與穩定性。
2.  **善用新語法**：使用 .NET 8 的主建構函式和 C# 的 `required` 關鍵字，可以讓你的程式碼更現代、更安全。
3.  **結合模型驗證**：務必在處理繫結後的模型前，檢查 `ModelState.IsValid`，確保資料的正確性。

掌握模型繫結，是成為一位熟練的 ASP.NET Core MVC 開發者的關鍵一步。
