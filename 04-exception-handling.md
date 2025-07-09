# ASP.NET Core MVC 教學 (.NET 8)：如何正確的處理例外 (Exception Handling)

在任何應用程式中，例外 (Exception) 都無可避免。一個強健的應用程式不僅要能完成預期功能，更要在發生非預期錯誤時，能夠優雅地處理，避免服務崩潰、保護敏感資訊不外洩，並提供有用的回饋給使用者和開發者。

本文將深入探討在 ASP.NET Core MVC (.NET 8) 中處理例外的各種策略與最佳實務。

## 1. 開發環境 vs. 生產環境

ASP.NET Core 鼓勵我們針對不同的執行環境採取不同的例外處理策略。

### 開發環境：開發者例外頁面 (Developer Exception Page)

在開發階段，我們需要盡可能詳細的錯誤資訊來快速定位問題。ASP.NET Core 預設的專案範本已經為我們配置好了「開發者例外頁面」。

**`Program.cs` 設定：**
```csharp
var app = builder.Build();

// 判斷是否為開發環境
if (app.Environment.IsDevelopment())
{
    // 啟用開發者例外頁面，它會顯示詳細的堆疊追蹤、請求資訊等
    app.UseDeveloperExceptionPage(); 
}
else
{
    // 生產環境下的設定
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}
```
這個頁面是開發者的好朋友，但**絕對不能**在生產環境中使用，否則會將伺服器路徑、原始碼片段等敏感資訊暴露給外部使用者。

## 2. 全域例外處理策略 (Global Exception Handling)

為所有未被 `try-catch` 捕捉的例外提供一個統一的處理機制，是保護應用程式的基礎防線。

### a. 例外處理中介軟體 (Exception Handler Middleware)

這是傳統且最常見的全域例外處理方式。透過 `app.UseExceptionHandler()` 中介軟體，我們可以指定一個錯誤處理路徑。當後續管線中發生任何未處理的例外時，請求會被重新導向到這個路徑。

**`Program.cs` 設定：**
```csharp
app.UseExceptionHandler("/Home/Error");
```

**`HomeController.cs` 中的處理動作：**
```csharp
using Microsoft.AspNetCore.Diagnostics;
using System.Diagnostics;

// ...
[AllowAnonymous]
[ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;

    public HomeController(ILogger<HomeController> logger)
    {
        _logger = logger;
    }

    public IActionResult Error()
    {
        // 從 HttpContext 的 Features 中取得例外資訊
        var exceptionHandlerPathFeature = HttpContext.Features.Get<IExceptionHandlerPathFeature>();
        
        if (exceptionHandlerPathFeature?.Error != null)
        {
            // **重要**：將完整的例外資訊記錄下來，以便追蹤
            _logger.LogError(
                exceptionHandlerPathFeature.Error,
                "Unhandled exception at path: {Path}",
                exceptionHandlerPathFeature.Path);
        }

        // 顯示一個通用的錯誤頁面給使用者
        return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier });
    }
}
```
**`Error.cshtml` 檢視：**
這個檢視應該只顯示對使用者有幫助的通用訊息，例如「系統發生未預期的錯誤，請稍後再試」以及一個請求 ID 以便追蹤。

### b. .NET 8 的新利器：`IExceptionHandler`

.NET 8 引入了 `IExceptionHandler` 介面，提供了一個更現代、更強大、可測試性更高的全域例外處理方案。它將例外處理邏輯完全從 MVC 控制器中解耦，成為一個獨立的服務。

**建立自訂處理器 `GlobalExceptionHandler.cs`：**
```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

internal sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        // 記錄詳細的例外日誌
        logger.LogError(exception, "An unhandled exception has occurred: {Message}", exception.Message);

        // 建立一個標準化的錯誤回應 (ProblemDetails)
        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Server Error",
            Detail = "An internal server error has occurred. Please contact support."
        };

        httpContext.Response.StatusCode = problemDetails.Status.Value;

        // 以 JSON 格式回傳錯誤資訊，這對 Web API 特別有用
        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        // 回傳 true 代表例外已被處理
        return true;
    }
}
```

**在 `Program.cs` 中註冊：**
```csharp
// 1. 註冊自訂的 Exception Handler
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
// 2. 註冊 ProblemDetails 服務，讓 API 能回傳標準化錯誤
builder.Services.AddProblemDetails();

// ...
var app = builder.Build();

// 3. 啟用 Exception Handler 中介軟體
app.UseExceptionHandler(_ => { }); // 空的 lambda 表示使用 DI 容器中註冊的 IExceptionHandler
```
**最佳實務**：對於新的 .NET 8 專案，**強烈建議使用 `IExceptionHandler`** 作為主要的全域例外處理策略。

## 3. 特定情境的例外處理

### a. `try-catch` 區塊

全域處理器適用於「未預期」的例外。但對於某些「可預期」的例外，例如資料庫的並行衝突 (`DbUpdateConcurrencyException`) 或呼叫外部 API 的網路問題 (`HttpRequestException`)，使用傳統的 `try-catch` 區塊仍然是最佳選擇。它能讓你針對特定錯誤執行重試、回滾或回傳特定錯誤訊息的邏輯。

```csharp
public async Task<IActionResult> UpdateProfile(UserProfile model)
{
    try
    {
        await _userService.UpdateAsync(model);
        return RedirectToAction("Index");
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // 記錄錯誤
        _logger.LogWarning(ex, "Concurrency conflict for user {UserId}", model.Id);
        // 給使用者明確的提示
        ModelState.AddModelError(string.Empty, "此資料已被他人修改，請重新整理後再試。");
        return View(model);
    }
}
```

### b. 例外過濾器 (Exception Filters)

如果你需要為某個特定的控制器或 Action 套用與全域不同的例外處理邏輯，可以使用例外過濾器。它只對 MVC 管線內的錯誤有效。

## 結論

一個成熟的例外處理策略應該是多層次的：

1.  **開發時**：使用「開發者例外頁面」獲取最詳細的錯誤資訊。
2.  **生產時**：
    *   **首選**：使用 .NET 8 的 `IExceptionHandler` 建立一個強健、解耦的全域處理器。
    *   **傳統方式**：使用 `UseExceptionHandler()` 中介軟體將錯誤導向一個通用的錯誤頁面。
3.  **特定邏輯**：在業務邏輯程式碼中，使用 `try-catch` 來處理可預期的、需要特別處理的例外。
4.  **日誌記錄**：無論使用哪種策略，**務必將完整的例外資訊 (含堆疊追蹤) 記錄到日誌系統中**，這是事後排查問題的唯一線索。
5.  **使用者體驗**：永遠不要將原始的例外訊息顯示給終端使用者。
