# ASP.NET Core MVC 教學 (.NET 8)：動作過濾器應用實務 (Action Filters)

在開發 MVC 應用程式時，我們經常需要在多個 Action 方法執行前後，執行一些共通的邏輯，例如：記錄日誌、驗證權限、處理例外或快取。如果將這些重複的程式碼散佈在每個 Action 中，將會違反 DRY (Don't Repeat Yourself) 原則，導致維護上的困難。

ASP.NET Core MVC 的「過濾器 (Filters)」正是為了解決這個問題而生。它允許我們將這些橫切關注點 (Cross-Cutting Concerns) 從業務邏輯中抽離，以一種宣告式 (Attribute) 的方式應用到 Action、控制器，甚至是全域。本文將聚焦於最常用的「動作過濾器 (Action Filter)」。

## 1. 過濾器管線 (Filter Pipeline)

ASP.NET Core 的請求處理管線中，過濾器在 MVC 的環節扮演重要角色。不同類型的過濾器會在不同階段執行，其順序如下：

1.  **授權過濾器 (Authorization Filters)**: 最先執行，用於判斷使用者是否有權限存取。
2.  **資源過濾器 (Resource Filters)**: 在授權之後、模型繫結之前執行。適合用來做快取，若快取命中則可直接中斷後續流程。
3.  **動作過濾器 (Action Filters)**: 在模型繫結完成後，**緊鄰著 Action 方法執行前後**。這是本文的重點。
4.  **例外過濾器 (Exception Filters)**: 當前面環節發生未處理的例外時，用來捕捉並處理例外。
5.  **結果過濾器 (Result Filters)**: 在 Action 方法成功執行並產生 `IActionResult` 後，於該 Result **執行前後**被呼叫。

## 2. 建立自訂動作過濾器

動作過濾器讓我們可以在 Action 執行前 (`OnActionExecuting`) 和執行後 (`OnActionExecuted`) 插入自訂邏輯。

### 同步過濾器

讓我們來建立一個記錄 Action 執行時間的同步過濾器。

**`LogExecutionTimeFilter.cs`**
```csharp
using Microsoft.AspNetCore.Mvc.Filters;
using System.Diagnostics;

// 繼承自 ActionFilterAttribute
public class LogExecutionTimeAttribute : ActionFilterAttribute
{
    private readonly ILogger<LogExecutionTimeAttribute> _logger;
    private Stopwatch? _stopwatch;

    // 透過建構函式注入 Logger
    public LogExecutionTimeAttribute(ILogger<LogExecutionTimeAttribute> logger)
    {
        _logger = logger;
    }

    // 在 Action 執行前
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        _stopwatch = Stopwatch.StartNew();
        _logger.LogInformation(
            "Action [{ActionName}] on controller [{ControllerName}] starting.",
            context.ActionDescriptor.DisplayName,
            context.Controller.GetType().Name);
        
        base.OnActionExecuting(context);
    }

    // 在 Action 執行後
    public override void OnActionExecuted(ActionExecutedContext context)
    {
        _stopwatch?.Stop();
        _logger.LogInformation(
            "Action [{ActionName}] executed in {ElapsedMilliseconds}ms.",
            context.ActionDescriptor.DisplayName,
            _stopwatch?.ElapsedMilliseconds);

        // 檢查執行過程中是否發生未處理的例外
        if (context.Exception != null)
        {
            _logger.LogError(context.Exception, "An unhandled exception occurred during action execution.");
        }
        
        base.OnActionExecuted(context);
    }
}
```

### 非同步過濾器

在現今大量使用 `async/await` 的非同步世界中，建立非同步版本的過濾器是更推薦的做法。非同步過濾器需要實作 `IAsyncActionFilter`。

**`AsyncLogExecutionTimeFilter.cs`**
```csharp
using Microsoft.AspNetCore.Mvc.Filters;
using System.Diagnostics;

public class AsyncLogExecutionTimeAttribute : IAsyncActionFilter
{
    private readonly ILogger<AsyncLogExecutionTimeAttribute> _logger;

    public AsyncLogExecutionTimeAttribute(ILogger<AsyncLogExecutionTimeAttribute> logger)
    {
        _logger = logger;
    }

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var stopwatch = Stopwatch.StartNew();
        _logger.LogInformation("Async Action [{ActionName}] starting.", context.ActionDescriptor.DisplayName);

        // ********************************************************************
        // ** await next(); 是關鍵，它會執行後續的過濾器以及 Action 方法本身 **
        // ********************************************************************
        var resultContext = await next();

        stopwatch.Stop();
        _logger.LogInformation(
            "Async Action [{ActionName}] executed in {ElapsedMilliseconds}ms.",
            resultContext.ActionDescriptor.DisplayName,
            stopwatch.ElapsedMilliseconds);

        if (resultContext.Exception != null)
        {
            _logger.LogError(resultContext.Exception, "An unhandled exception occurred.");
        }
    }
}
```

## 3. 套用過濾器

定義好過濾器後，我們有三種層級可以套用它：

### a. Action 層級

只對單一 Action 方法生效。

```csharp
public class HomeController : Controller
{
    [HttpGet]
    [TypeFilter(typeof(AsyncLogExecutionTimeAttribute))] // 使用 TypeFilter 來啟用依賴注入
    public IActionResult Index()
    {
        Thread.Sleep(100); // 模擬耗時操作
        return View();
    }
}
```
> **注意**: 因為我們的過濾器需要透過建構函式注入 `ILogger`，所以不能直接用 `[AsyncLogExecutionTime]`，而必須使用 `[TypeFilter(typeof(AsyncLogExecutionTimeAttribute))]` 或 `[ServiceFilter(typeof(AsyncLogExecutionTimeAttribute))]` 來讓 DI 容器建立它。

### b. Controller 層級

對該控制器下的所有 Action 方法生效。

```csharp
[TypeFilter(typeof(AsyncLogExecutionTimeAttribute))]
public class ProductsController : Controller
{
    public IActionResult Index() { /* ... */ }
    public IActionResult Details(int id) { /* ... */ }
}
```

### c. 全域層級

對應用程式中所有的 MVC Action 方法生效。這是設定全域日誌、全域例外處理的最佳位置。

**`Program.cs`**
```csharp
var builder = WebApplication.CreateBuilder(args);

// 將自訂的 Filter 註冊到 DI 容器
builder.Services.AddScoped<AsyncLogExecutionTimeAttribute>();

builder.Services.AddControllersWithViews(options =>
{
    // 將 Filter 加入到全域過濾器集合中
    options.Filters.Add<AsyncLogExecutionTimeAttribute>();
});

// ...
```

## 4. .NET 8 的新選擇：`IEndpointFilter`

值得一提的是，從 .NET 7 開始為 Minimal API 設計的 `IEndpointFilter`，在 .NET 8 中功能更為強大。它提供了與 Action Filter 類似的功能，但更輕量、效能更好，並且是針對路由端點 (Endpoint) 而非 MVC 的 Action。如果你的應用程式大量使用 Minimal API，`IEndpointFilter` 會是比 Action Filter 更現代的選擇。

## 結論

動作過濾器是 ASP.NET Core MVC 中實踐 AOP (Aspect-Oriented Programming) 的強大工具。

1.  **保持控制器簡潔**：將日誌、快取、權限檢查等橫切關注點移至過濾器中。
2.  **優先使用非同步版本**：實作 `IAsyncActionFilter` 以符合現代非同步程式設計的需求。
3.  **善用依賴注入**：透過 `[TypeFilter]` 或 `[ServiceFilter]` 在過濾器中注入所需服務。
4.  **選擇正確的套用層級**：根據需求，在 Action、Controller 或全域層級套用過濾器，以達到最大程度的程式碼複用。
