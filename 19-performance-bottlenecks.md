# ASP.NET Core MVC 教學 (.NET 8)：如何快速找出效能瓶頸

應用程式的效能是使用者體驗的決定性因素。一個反應遲緩的網站會導致使用者流失和業務損失。找出並解決效能瓶頸 (Performance Bottlenecks) 是應用程式生命週期中持續進行的重要工作。幸運的是，.NET 和 ASP.NET Core 生態系統提供了多種強大的工具，可以幫助我們快速定位問題所在。

本文將介紹幾種從簡單到進階的方法，來診斷和分析 ASP.NET Core 應用程式的效能問題。

## 1. 低垂的果實：日誌與計時

在引入複雜的工具之前，最簡單的效能分析方法就是**策略性地使用日誌和計時器**。

### a. 手動計時

使用 `System.Diagnostics.Stopwatch` 來測量特定程式碼區塊的執行時間。

```csharp
public async Task<IActionResult> ImportData()
{
    var stopwatch = Stopwatch.StartNew();
    
    // 步驟 1: 讀取檔案
    var data = await _fileReader.ReadAsync();
    stopwatch.Stop();
    _logger.LogInformation("讀取檔案耗時: {ElapsedMilliseconds} ms", stopwatch.ElapsedMilliseconds);

    // 步驟 2: 處理資料
    stopwatch.Restart();
    var processedData = _dataProcessor.Process(data);
    stopwatch.Stop();
    _logger.LogInformation("處理資料耗時: {ElapsedMilliseconds} ms", stopwatch.ElapsedMilliseconds);

    // 步驟 3: 存入資料庫
    stopwatch.Restart();
    await _dbRepository.SaveAsync(processedData);
    stopwatch.Stop();
    _logger.LogInformation("存入資料庫耗時: {ElapsedMilliseconds} ms", stopwatch.ElapsedMilliseconds);

    return Ok();
}
```
這種方法雖然原始，但對於快速判斷一個複雜流程中哪個步驟是主要耗時點非常有效。

### b. 動作過濾器 (Action Filter)

在之前的文章中，我們建立過一個記錄 Action 執行時間的過濾器。將這樣的過濾器應用到全域，可以幫助你發現哪些頁面或 API 端點的回應時間最長，從而縮小排查範圍。

## 2. 資料庫效能瓶頸

根據經驗，**絕大多數的 Web 應用程式效能瓶頸都與資料庫互動有關**。

### a. EF Core 的日誌記錄

Entity Framework Core 可以記錄它產生的所有 SQL 查詢。在開發環境中，將其日誌級別設定為 `Information`，就可以在主控台或輸出視窗中看到每一條執行的 SQL 和其耗時。

**`appsettings.Development.json`**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      // 顯示 EF Core 的日誌，包括執行的 SQL
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```
透過檢查日誌，你可以發現：
*   **執行緩慢的查詢 (Slow Queries)**：某條 SQL 本身就執行了很長時間。
*   **N+1 查詢問題**: 這是最常見的效能殺手之一（後續文章會詳細介紹）。
*   **產生了非預期的複雜查詢**: EF Core 產生的 SQL 可能比你想像的要複雜得多。

### b. 資料庫分析工具

使用你的資料庫本身提供的效能分析工具，例如 SQL Server 的 **Query Store** 或 **Execution Plan Analyzer**，可以直接分析慢查詢的執行計畫，找出缺少索引、全資料表掃描等問題。

## 3. 使用專業的分析工具 (Profilers)

當簡單的計時和日誌無法定位問題時，就需要使用專業的效能分析工具 (Profiler)。Profiler 會深入你的應用程式執行過程，收集關於 CPU 使用率、記憶體分配、方法呼叫次數和時間等詳細資料。

### a. Visual Studio 內建的效能分析工具

Visual Studio (Enterprise/Professional 版本) 提供了非常強大的內建效能分析工具。

**使用方法：**
1.  在 Visual Studio 中，選擇 **偵錯 (Debug) -> 效能分析工具 (Performance Profiler)** (或按 `Alt + F2`)。
2.  選擇你感興趣的分析工具。對於效能瓶頸，最常用的有：
    *   **CPU 使用率 (CPU Usage)**: 分析哪個方法的 CPU 佔用最高。
    *   **.NET 物件配置追蹤 (.NET Object Allocation Tracking)**: 分析哪個方法產生了最多的記憶體垃圾，導致 GC (垃圾回收) 壓力。
    *   **資料庫 (Database)**: 專門用來分析 ADO.NET 和 EF Core 的查詢。
3.  開始執行分析。操作你的應用程式，重現效能緩慢的場景。
4.  停止收集，Visual Studio 會產生一份詳細的報告。

**分析報告：**
報告會以「火焰圖 (Flame Graph)」或「函式樹 (Call Tree)」的形式展示結果。你可以清楚地看到哪個方法的總執行時間最長（包括它呼叫的其他方法），以及哪個方法的「自我時間 (Self Time)」最長（即方法本身邏輯的耗時）。這能讓你精確地定位到具體的耗時函式。

### b. MiniProfiler

MiniProfiler 是一個輕量級的、專為 ASP.NET 設計的效能分析函式庫。它會在你的網頁右下角顯示一個小小的計時結果，點開後可以看到從後端處理到資料庫查詢的每一個步驟的詳細耗時。

它對於在**開發和測試環境**中即時發現 N+1 查詢或慢查詢非常有用，因為它將效能資料直接呈現在 UI 上，非常直觀。

### c. Application Performance Management (APM) 工具

對於**生產環境**的持續監控，你需要使用專業的 APM 工具，例如：
*   **Azure Application Insights**
*   **Datadog**
*   **New Relic**

這些工具會在背景持續收集應用程式的效能指標、錯誤、依賴項呼叫（資料庫、外部 API）等資料。它們能夠：
*   自動發現效能異常的端點。
*   提供分散式追蹤 (Distributed Tracing)，追蹤一個請求在多個微服務之間的完整路徑和耗時。
*   提供詳細的儀表板和告警功能。

對於任何正式上線的專案，整合一套 APM 工具是保障服務品質的必要投資。

## 結論

找出效能瓶頸是一個由粗到細的過程：

1.  **宏觀觀察**：使用日誌、計時器或 APM 工具，找出哪些頁面或功能區塊是緩慢的。
2.  **聚焦資料庫**：絕大多數問題來源於此。啟用 EF Core 日誌，分析 SQL 查詢，檢查是否存在 N+1 問題或缺少索引。
3.  **微觀分析**：使用 Visual Studio 的效能分析工具，對確認有問題的程式碼區塊進行 CPU 和記憶體分析，定位到具體的耗時方法。
4.  **持續監控**：在生產環境中部署 APM 工具，建立效能基線，並在發生效能衰退時及時獲得通知。

記住，效能優化不是一次性的工作，而是一個需要持續測量、分析和改進的循環過程。
