# ASP.NET Core MVC 教學 (.NET 8)：使用非同步控制器的注意事項

在現代 Web 開發中，非同步程式設計 (Asynchronous Programming) 已不是一個選項，而是一項基本要求。ASP.NET Core 從底層設計上就完全擁抱了非同步，以 `async` 和 `await` 為核心，旨在提高應用程式的**吞吐量 (Throughput)** 和**延展性 (Scalability)**。

在 MVC 中，這體現在「非同步控制器 (Asynchronous Controllers)」的使用上。一個非同步的 Action 方法其返回型別為 `Task<IActionResult>`，並在其內部使用 `await` 來呼叫 I/O 密集型操作。

```csharp
// 同步 Action
public IActionResult GetProduct(int id)
{
    var product = _db.Products.Find(id); // 阻塞執行緒
    return View(product);
}

// 非同步 Action
public async Task<IActionResult> GetProductAsync(int id)
{
    var product = await _db.Products.FindAsync(id); // 非阻塞
    return View(product);
}
```

雖然使用非同步能帶來巨大好處，但如果使用不當，也可能引入難以察覺的錯誤。本文將探討在使用非同步控制器時需要注意的關鍵事項。

## 1. 非同步的真正目的：釋放執行緒，提高吞吐量

一個常見的誤解是：`async/await` 會讓單個請求變得更快。**通常不會**。事實上，由於狀態機的開銷，單個非同步呼叫的總耗時甚至可能比同步版本略長一點點。

非同步的真正威力在於**伺服器端資源的利用**。

*   在**同步**模型中，當一個請求正在等待資料庫查詢或外部 API 呼叫完成時，它所佔用的伺服器執行緒會被**阻塞 (Blocked)**，無法處理任何其他工作。
*   在**非同步**模型中，當 `await` 一個 I/O 操作時，執行緒會被**釋放 (Released)** 回執行緒池。在這個等待期間，該執行緒可以去處理其他傳入的請求。當 I/O 操作完成後，框架會從執行緒池中再取一個可用的執行緒來繼續執行 `await` 後面的程式碼。

這意味著，在面對高併發請求時，非同步應用程式可以用**更少的執行緒**來服務**更多的使用者**，從而極大地提高了伺服器的吞吐量和延展性。

**原則：只要你的 Action 中有 I/O 操作（資料庫、檔案讀寫、網路呼叫），就應該使用非同步。**

## 2. "Async All the Way" - 非同步一路到底

一旦你決定在某個方法中使用 `async`，那麼整個呼叫鏈 (Call Stack) 都應該是非同步的。絕對要避免使用 `.Result` 或 `.Wait()` 來同步地等待一個 `Task` 完成。

**錯誤的作法 (同步阻塞)**：
```csharp
public IActionResult GetUserData()
{
    // 這會阻塞當前執行緒，直到 Task 完成，完全抵銷了非同步的優勢
    // 更糟糕的是，在某些情況下（特別是舊的 ASP.NET），這可能導致死結 (Deadlock)
    var user = _userService.GetUserAsync("test").Result; 
    return View(user);
}
```

**正確的作法 (非同步傳播)**：
```csharp
public async Task<IActionResult> GetUserDataAsync()
{
    var user = await _userService.GetUserAsync("test");
    return View(user);
}
```
這個原則被稱為 "Async All the Way"。從你的 Action 方法，到你的服務層，再到你的倉儲層，都應該是 `async Task`。

## 3. 注意 `HttpContext` 的使用

在 `await` 之後，繼續執行的程式碼**可能**會在一個不同的執行緒上。雖然 ASP.NET Core 的同步上下文 (`SynchronizationContext`) 通常會確保 `HttpContext` 仍然可用，但在某些複雜或自訂的執行緒場景下，直接跨 `await` 存取 `HttpContext` 可能會產生非預期的行為。

一個最佳實踐是：**在 `await` 之前，從 `HttpContext` 中讀取所有你需要的資料，並存到區域變數中。**

**不太好的作法：**
```csharp
public async Task<IActionResult> UpdateProfile()
{
    // ...
    await _userService.UpdateAsync(model);

    // 在 await 之後才存取 HttpContext
    var userName = HttpContext.User.Identity.Name; 
    _logger.LogInformation("User {UserName} updated profile.", userName);
    
    return RedirectToAction("Index");
}
```

**推薦的作法：**
```csharp
public async Task<IActionResult> UpdateProfile()
{
    // 在 await 之前讀取所需資料
    var userName = HttpContext.User.Identity.Name; 
    // ...

    await _userService.UpdateAsync(model);

    // 使用區域變數
    _logger.LogInformation("User {UserName} updated profile.", userName);

    return RedirectToAction("Index");
}
```
這讓程式碼的意圖更清晰，也避免了潛在的執行緒安全問題。

## 4. 小心非同步的 `void` 方法

`async void` 應該**僅僅**用於事件處理常式 (Event Handlers)，例如 WinForms 或 WPF 的按鈕點擊事件。在 ASP.NET Core 中，**絕對要避免使用 `async void`**。

`async void` 方法有幾個嚴重的問題：
*   **無法被 `await`**: 呼叫者無法知道這個非同步操作何時完成。
*   **錯誤處理困難**: `async void` 方法中拋出的例外無法被標準的 `try-catch` 捕捉，通常會直接導致應用程式進程崩潰。
*   **測試困難**: 單元測試框架無法追蹤 `async void` 方法的完成狀態。

如果一個方法需要非同步執行但不需要回傳值，請使用 `async Task`。

```csharp
// 錯誤
public async void LogUserActivity()
{
    await _logService.WriteAsync("User logged in");
}

// 正確
public async Task LogUserActivityAsync()
{
    await _logService.WriteAsync("User logged in");
}
```

## 5. CancellationToken 的使用

對於可能長時間執行的非同步操作，提供一個取消機制是非常重要的。如果使用者在一個耗時的資料庫查詢完成前關閉了瀏覽器或取消了請求，伺服器應該停止這個不必要的工作，以釋放資源。

ASP.NET Core 會為每個 HTTP 請求提供一個 `CancellationToken`。你可以將它傳遞到你的非同步方法中。

```csharp
public async Task<IActionResult> GenerateReport(CancellationToken cancellationToken)
{
    try
    {
        // 將 cancellationToken 傳遞到所有支援它的非同步方法中
        var reportData = await _reportService.GetDataAsync(cancellationToken);
        var reportFile = await _fileGenerator.CreateAsync(reportData, cancellationToken);

        return File(reportFile, "application/pdf");
    }
    catch (OperationCanceledException)
    {
        // 當請求被取消時，EF Core 或 HttpClient 會拋出這個例外
        _logger.LogInformation("Report generation was canceled.");
        return new EmptyResult(); // 或其他適當的回應
    }
}
```
Entity Framework Core, `HttpClient` 以及許多現代的 .NET 函式庫都支援 `CancellationToken`。養成傳遞它的習慣，可以讓你的應用程式反應更靈敏，資源利用更高效。

## 結論

非同步控制器是構建高效能、高延展性 ASP.NET Core 應用程式的基石。

1.  **理解目的**：非同步是為了提高**吞吐量**，而不是加快單一請求。
2.  **Async All the Way**：避免使用 `.Result` 或 `.Wait()` 來阻塞非同步操作。
3.  **謹慎處理 `HttpContext`**：在 `await` 前讀取所需資料。
4.  **禁用 `async void`**：在 Web 開發中，永遠使用 `async Task` 或 `async Task<T>`。
5.  **傳遞 `CancellationToken`**：為你的應用程式加入優雅的取消能力。

遵循這些原則，你就能夠充分利用 `async/await` 的威力，同時避免其潛在的陷阱。
