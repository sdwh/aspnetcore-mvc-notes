# ASP.NET Core MVC 教學 (.NET 8)：在 ASP.NET Core 實現背景任務

在許多 Web 應用程式中，有些任務的執行時間很長，或者不需要立即完成，不應該在處理使用者 HTTP 請求的過程中同步執行。如果在一個 Controller Action 中執行一個耗時 5 分鐘的操作，那麼使用者的瀏覽器將會一直處於等待狀態，直到操作完成，這會導致極差的使用者體驗和請求逾時。

這些「**發起後不理 (Fire-and-Forget)**」或**排程性**的任務，就應該在「背景 (Background)」中執行。常見的背景任務包括：
*   發送電子郵件或簡訊通知。
*   產生報表或處理上傳的檔案。
*   與外部 API 進行長時間的資料同步。
*   定時清理過期資料或執行資料庫維護。

ASP.NET Core 提供了內建的機制來實現輕量級的背景任務，同時也有更強大、更可靠的第三方函式庫可供選擇。

## 1. 內建的輕量級解決方案：`IHostedService`

ASP.NET Core 提供了 `IHostedService` 介面，它允許你註冊一個與 Web 應用程式生命週期一起啟動和停止的長時間執行服務。這是實現背景任務最基礎、最直接的方式。

一個更方便的起點是繼承 `BackgroundService` 抽象類別，它實作了 `IHostedService`，並提供了一個 `ExecuteAsync` 方法讓我們覆寫。

### 範例：一個簡單的佇列式郵件發送服務

我們來設計一個場景：Controller Action 只需要將「要發送的郵件」這個工作項目放入一個記憶體中的佇列，然後立即回傳給使用者。一個在背景執行的 `BackgroundService` 會不斷地從這個佇列中取出項目並真正地發送郵件。

**步驟 1：建立一個單例的佇列服務**
```csharp
using System.Threading.Channels;

// 這個服務將作為單例 (Singleton) 存在，用於在 Controller 和背景服務之間傳遞工作
public interface IBackgroundTaskQueue
{
    ValueTask QueueBackgroundWorkItemAsync(Func<CancellationToken, ValueTask> workItem);
    ValueTask<Func<CancellationToken, ValueTask>> DequeueAsync(CancellationToken cancellationToken);
}

public class BackgroundTaskQueue : IBackgroundTaskQueue
{
    private readonly Channel<Func<CancellationToken, ValueTask>> _queue;

    public BackgroundTaskQueue(int capacity)
    {
        var options = new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait
        };
        _queue = Channel.CreateBounded<Func<CancellationToken, ValueTask>>(options);
    }

    public async ValueTask QueueBackgroundWorkItemAsync(Func<CancellationToken, ValueTask> workItem)
    {
        if (workItem == null) throw new ArgumentNullException(nameof(workItem));
        await _queue.Writer.WriteAsync(workItem);
    }

    public async ValueTask<Func<CancellationToken, ValueTask>> DequeueAsync(CancellationToken cancellationToken)
    {
        var workItem = await _queue.Reader.ReadAsync(cancellationToken);
        return workItem;
    }
}
```
> 我們使用了 `System.Threading.Channels`，這是一個高效能的、專為生產者/消費者模式設計的執行緒安全佇列。

**步驟 2：建立背景服務 `QueuedHostedService`**
```csharp
public class QueuedHostedService(
    IBackgroundTaskQueue taskQueue,
    ILogger<QueuedHostedService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Queued Hosted Service is running.");

        while (!stoppingToken.IsCancellationRequested)
        {
            // 從佇列中等待並取出工作項目
            var workItem = await taskQueue.DequeueAsync(stoppingToken);

            try
            {
                // 執行工作
                await workItem(stoppingToken);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Error occurred executing {WorkItem}.", nameof(workItem));
            }
        }
        
        logger.LogInformation("Queued Hosted Service is stopping.");
    }
}
```

**步驟 3：在 `Program.cs` 中註冊**
```csharp
// 註冊佇列服務為單例
builder.Services.AddSingleton<IBackgroundTaskQueue>(ctx => new BackgroundTaskQueue(100));

// 註冊背景服務
builder.Services.AddHostedService<QueuedHostedService>();
```

**步驟 4：在 Controller 中使用**
```csharp
public class EmailController(IBackgroundTaskQueue queue, ILogger<EmailController> logger) : Controller
{
    [HttpPost]
    public IActionResult SendEmail(string to, string subject)
    {
        // 將發送郵件的工作加入佇列
        queue.QueueBackgroundWorkItemAsync(async token =>
        {
            // 這裡模擬一個耗時的郵件發送過程
            await Task.Delay(TimeSpan.FromSeconds(5), token);
            logger.LogInformation("Email to {To} with subject '{Subject}' sent!", to, subject);
        });

        // 立即回傳，不等待郵件發送完成
        return Ok("郵件已加入發送佇列。");
    }
}
```

### `IHostedService` 的優缺點
*   **優點**：內建、輕量、無需額外依賴，適合簡單的背景任務。
*   **缺點**：
    *   **不具持久性**：如果應用程式在任務執行完畢前崩潰或重啟，記憶體佇列中的所有任務都會遺失。
    *   **無內建 UI**：沒有儀表板可以監控任務狀態。
    *   **單點故障**：只在執行它的那個 Web 伺服器實例上運作。在負載平衡環境中，它不是一個可靠的分散式解決方案。

## 2. 更強大可靠的選擇：Hangfire

對於任何嚴肅的、需要可靠性的背景任務處理，強烈建議使用像 **Hangfire** 這樣的專業函式庫。

Hangfire 是一個功能極其豐富的開源背景任務處理框架，它解決了 `IHostedService` 的所有缺點。

**Hangfire 的核心功能：**
1.  **持久化儲存**：它使用一個持久化的儲存體（如 SQL Server, Redis）來存放任務。即使應用程式重啟，任務也不會遺失，會在重啟後繼續執行。
2.  **自動重試**：如果一個任務因為暫時性問題（如網路抖動）而失敗，Hangfire 會自動進行重試。
3.  **儀表板 (Dashboard)**：提供一個美觀的 Web UI，可以讓你即時監控所有任務的狀態（排隊中、執行中、成功、失敗），並可以手動觸發或刪除任務。
4.  **多種任務類型**：
    *   **Fire-and-forget**: 發起後不理。
    *   **Delayed**: 延遲一段時間後執行。
    *   **Recurring**: 定時重複執行（例如，每晚午夜執行）。
    *   **Continuations**: 在某個任務成功後接續執行另一個任務。
5.  **分散式友好**：天生為負載平衡環境設計。

### 使用 Hangfire
整合 Hangfire 相對簡單，主要包含安裝 NuGet 套件、在 `Program.cs` 中進行設定（指定儲存體、啟用儀表板），然後就可以在程式碼中注入 `IBackgroundJobClient` 來建立任務。

**建立一個背景任務：**
```csharp
public class MyController(IBackgroundJobClient backgroundJobClient) : Controller
{
    public IActionResult GenerateReport()
    {
        // 將 ReportService.Generate() 這個方法的呼叫排入背景執行
        backgroundJobClient.Enqueue<IReportService>(service => service.Generate());
        
        return Ok("報表正在背景產生中...");
    }
}
```

## 結論

在 ASP.NET Core 中實現背景任務有多種選擇，你需要根據任務的重要性和複雜性來決定使用哪種工具。

1.  **`IHostedService` / `BackgroundService`**：
    *   **適用場景**：輕量級的、允許遺失的、與應用程式生命週期相關的任務。例如，應用程式啟動時預熱快取，或一個簡單的記憶體佇列處理器。
    *   **核心考量**：它不提供持久性保證。

2.  **Hangfire (或類似的 Quartz.NET, Coravel)**：
    *   **適用場景**：任何需要**可靠性**、**持久性**、**監控**和**排程**的背景任務。這是絕大多數業務場景下的推薦選擇。
    *   **核心考量**：它是一個功能完整的解決方案，能處理從簡單到複雜的各種背景處理需求。

正確地將耗時操作轉移到背景執行，是打造反應靈敏、使用者體驗流暢的 Web 應用程式的關鍵一步。
