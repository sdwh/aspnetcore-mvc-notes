# ASP.NET Core MVC 教學 (.NET 8)：使用 HttpClient 的正確觀念

在現代的微服務架構和分散式系統中，讓一個應用程式去呼叫另一個服務的 API 是一個極其常見的需求。在 .NET 世界裡，執行 HTTP 請求的標準工具就是 `System.Net.Http.HttpClient`。

然而，`HttpClient` 的使用方式如果不得當，很容易會引發一些嚴重的效能和資源問題，例如「通訊端耗盡 (Socket Exhaustion)」。因此，了解如何正確地管理和使用 `HttpClient` 的生命週期，是每一位 .NET 開發者的必備知識。

## 1. 傳統的錯誤用法：`using` 新的 `HttpClient`

初學者最容易犯的錯誤，就是在每次需要發起請求時，都 `new` 一個新的 `HttpClient` 實例，並用 `using` 語句將其包起來。

**絕對錯誤的作法：**
```csharp
public async Task<string> GetUserDataAsync()
{
    // 每一次呼叫這個方法，都會建立一個新的 HttpClient
    using (var client = new HttpClient()) 
    {
        var result = await client.GetStringAsync("https://api.example.com/users/1");
        return result;
    }
}
```
**為什麼這是錯的？**
`HttpClient` 雖然實作了 `IDisposable`，但它的設計初衷是**可重用的 (reusable)**。當你 `Dispose` 一個 `HttpClient` 實例時，它底層所佔用的 TCP 通訊端 (Socket) 並不會立即被釋放，而是會進入一個 `TIME_WAIT` 狀態，以確保網路上所有延遲的封包都已經處理完畢。這個狀態會持續一小段時間（通常是 240 秒）。

如果在高併發的情況下，你的程式碼不斷地 `new` 和 `Dispose` `HttpClient`，將會快速地產生大量處於 `TIME_WAIT` 狀態的通訊端。作業系統可用的通訊端數量是有限的（通常是幾萬個），當所有可用的通訊端都被耗盡時，你的應用程式將無法再建立任何新的網路連線，從而導致「通訊端耗盡」錯誤，整個應用程式會因此癱瘓。

## 2. 稍微好一點但仍有問題的用法：靜態單例

為了解決上述問題，下一個直覺的想法是：「既然 `HttpClient` 是可重用的，那我就建立一個靜態的 (static) 單例，讓整個應用程式共享這一個實例。」

**有潛在問題的作法：**
```csharp
private static readonly HttpClient _staticClient = new HttpClient();

public async Task<string> GetUserDataAsync()
{
    var result = await _staticClient.GetStringAsync("https://api.example.com/users/1");
    return result;
}
```
這種方式確實解決了通訊端耗盡的問題，因為 `HttpClient` 實例永遠不會被 `Dispose`。但在某些情況下，它會帶來一個新的問題：**無法即時反應 DNS 變更**。

當 `HttpClient` 建立一個連線時，它會進行一次 DNS 查詢來解析主機名稱的 IP 位址。單例的 `HttpClient` 會長時間地保持這些連線，並且不會主動地去重新解析 DNS。如果 `api.example.com` 的 IP 位址因為負載平衡、故障轉移等原因發生了變更，你的靜態 `HttpClient` 可能仍然會嘗試連線到舊的、已經失效的 IP 位址，導致請求失敗。

## 3. .NET Core 的最佳實踐：`IHttpClientFactory`

為了一勞永逸地解決上述所有問題，ASP.NET Core 2.1 引入了 `IHttpClientFactory`。它是一個專門用來集中管理和建立 `HttpClient` 實例的工廠服務，被公認為是在 .NET Core 中使用 `HttpClient` 的**最佳實踐**。

`IHttpClientFactory` 的優點：
1.  **管理 `HttpClientMessageHandler` 的生命週期**：它在底層維護了一個 `HttpClientMessageHandler` 的快取池。這解決了通訊端耗盡問題，因為它重用了處理常式。
2.  **自動處理 DNS 變更**：快取池中的處理常式會被定期回收和替換，確保了 DNS 查詢能夠及時更新。
3.  **提供集中的設定點**：可以集中設定 `HttpClient` 的基礎位址、標頭、逾時時間等。
4.  **與 Polly 整合**：可以輕鬆地整合 Polly 函式庫，來實現重試 (Retry)、斷路器 (Circuit Breaker)、逾時 (Timeout) 等彈性策略 (Resilience Policies)。

### 如何使用 `IHttpClientFactory`

**步驟 1：在 `Program.cs` 中註冊**

```csharp
builder.Services.AddHttpClient();
```
這會註冊最基本的 `IHttpClientFactory` 服務。

**步驟 2：在服務中注入並使用**

```csharp
public class MyApiService(IHttpClientFactory httpClientFactory)
{
    public async Task<string> GetUserDataAsync()
    {
        // 從工廠建立一個 HttpClient
        var client = httpClientFactory.CreateClient();
        
        var result = await client.GetStringAsync("https://api.example.com/users/1");
        return result;
    }
}
```

### 進階用法：具名用戶端 (Named Clients)

你可以為不同的外部 API 設定不同的具名 `HttpClient`。

**`Program.cs`**
```csharp
builder.Services.AddHttpClient("GitHub", client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json");
    client.DefaultRequestHeaders.Add("User-Agent", "MyWebApp");
});
```

**在服務中使用**
```csharp
public class GitHubService(IHttpClientFactory httpClientFactory)
{
    public async Task<string> GetReposAsync(string userName)
    {
        // 建立一個具名的 client
        var client = httpClientFactory.CreateClient("GitHub");
        
        var result = await client.GetStringAsync($"users/{userName}/repos");
        return result;
    }
}
```

### 更進階的用法：型別用戶端 (Typed Clients)

這是最推薦、最結構化的方式。它將 `HttpClient` 的邏輯封裝到一個強型別的服務類別中。

**步驟 1：建立一個服務類別**
```csharp
public class GitHubService
{
    private readonly HttpClient _httpClient;

    // HttpClient 是透過建構函式注入的
    public GitHubService(HttpClient httpClient)
    {
        _httpClient = httpClient;
        // 可以在這裡設定 client，但更推薦在 AddHttpClient 中設定
    }

    public async Task<string> GetReposAsync(string userName)
    {
        var result = await _httpClient.GetStringAsync($"users/{userName}/repos");
        return result;
    }
}
```

**步驟 2：在 `Program.cs` 中註冊型別用戶端**
```csharp
builder.Services.AddHttpClient<GitHubService>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.Add("Accept", "application/vnd.github.v3+json");
    client.DefaultRequestHeaders.Add("User-Agent", "MyWebApp");
});
```

**步驟 3：在 Controller 中直接注入 `GitHubService`**
```csharp
public class HomeController(GitHubService gitHubService) : Controller
{
    public async Task<IActionResult> Index()
    {
        var reposJson = await gitHubService.GetReposAsync("dotnet");
        // ...
        return View();
    }
}
```
這種方式提供了最好的封裝和可測試性。

## 結論

`HttpClient` 的正確使用是構建可靠、高效能分散式應用的關鍵。

1.  **絕對禁止**在每次請求時 `new HttpClient()`。
2.  **避免使用**靜態單例的 `HttpClient`，以防 DNS 問題。
3.  **永遠使用 `IHttpClientFactory`**：這是 .NET Core 提供的標準且最佳的解決方案。
4.  **優先考慮型別用戶端 (Typed Clients)**：它提供了最好的程式碼結構、封裝和可測試性。
5.  **結合 Polly**：利用 `AddPolicyHandler` 來為你的 HTTP 呼叫增加重試、斷路器等彈性機制，打造更強健的應用程式。
