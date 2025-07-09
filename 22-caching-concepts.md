# ASP.NET Core MVC 教學 (.NET 8)：實作快取的正確觀念

在高效能 Web 應用程式的開發中，快取 (Caching) 是一種至關重要的技術。其核心思想是將那些不經常變動但需要耗費大量資源（CPU、I/O）來產生的資料，暫時儲存到一個存取速度更快的儲存體中（例如，記憶體）。當下一次有相同的請求時，應用程式可以直接從快取中回傳資料，而無需重新計算或查詢資料庫，從而極大地提升回應速度、降低伺服器負載。

ASP.NET Core 提供了多層次、功能豐富的快取機制。理解它們的適用場景和正確的使用方式，是進行效能優化的關鍵。

## 1. 快取的類型

ASP.NET Core 主要提供兩種類型的快取：

### a. 記憶體快取 (In-Memory Caching)
*   **原理**：將快取資料直接儲存在應用程式伺服器的記憶體中。
*   **優點**：存取速度極快，因為它就在應用程式的進程內。實現簡單。
*   **缺點**：
    *   **與伺服器綁定**：快取內容只存在於單一伺服器實例上。在負載平衡的多伺服器環境（Web Farm / Web Garden）中，不同伺服器之間的快取是**不共享**的，可能導致資料不一致（這被稱為「黏性會話 (Sticky Sessions)」問題）。
    *   **生命週期短**：當應用程式重新啟動或崩潰時，所有快取資料都會遺失。
    *   **消耗伺服器記憶體**：儲存大量快取會增加伺服器的記憶體壓力。
*   **適用場景**：單一伺服器部署，或者快取的資料允許在不同伺服器間存在短暫不一致。適合快取少量、頻繁存取的資料。

### b. 分散式快取 (Distributed Caching)
*   **原理**：將快取資料儲存在一個獨立於應用程式伺服器的外部共享儲存體中。
*   **常見的儲存體**：Redis, SQL Server, NCache。**Redis** 是目前最流行、效能最好的選擇。
*   **優點**：
    *   **多伺服器共享**：在負載平衡環境中，所有伺服器實例共享同一個快取，保證了資料的一致性。
    *   **生命週期長**：即使應用程式伺服器全部重新啟動，快取資料依然存在。
    *   **不消耗應用程式記憶體**：快取儲存的壓力轉移到了專門的快取伺服器上。
*   **缺點**：
    *   **存取速度較慢**：因為需要透過網路進行存取，其延遲比記憶體快取要高。
    *   **部署和維護更複雜**：需要額外設定和維護一個快取伺服器（如 Redis）。
*   **適用場景**：負載平衡的多伺服器環境、微服務架構、需要持久化或跨應用程式共享的快取資料。

## 2. 實作記憶體快取 (`IMemoryCache`)

ASP.NET Core 透過 `IMemoryCache` 介面提供記憶體快取功能。

**步驟 1：在 `Program.cs` 中註冊服務**
```csharp
builder.Services.AddMemoryCache();
```

**步驟 2：在服務或控制器中注入並使用**
```csharp
public class CategoryService(ICategoryRepository repo, IMemoryCache memoryCache)
{
    private const string AllCategoriesCacheKey = "AllCategories";

    public async Task<List<Category>> GetAllCategoriesAsync()
    {
        // 嘗試從快取中獲取資料
        if (memoryCache.TryGetValue(AllCategoriesCacheKey, out List<Category> categories))
        {
            // 快取命中 (Cache Hit)
            return categories;
        }

        // 快取未命中 (Cache Miss)
        // 1. 從資料庫讀取資料
        categories = await repo.GetAllAsync();

        // 2. 設定快取選項
        var cacheEntryOptions = new MemoryCacheEntryOptions()
            // 設定絕對到期時間為 10 分鐘後
            .SetAbsoluteExpiration(TimeSpan.FromMinutes(10))
            // 設定滑動到期時間為 2 分鐘（2 分鐘內未被存取則過期）
            .SetSlidingExpiration(TimeSpan.FromMinutes(2));

        // 3. 將資料存入快取
        memoryCache.Set(AllCategoriesCacheKey, categories, cacheEntryOptions);

        return categories;
    }
}
```
這個「**先檢查快取，若無則產生並存入快取**」的模式被稱為 "Cache-Aside Pattern"，是使用快取最常見的模式。

## 3. 實作分散式快取 (`IDistributedCache`)

ASP.NET Core 透過 `IDistributedCache` 介面提供分散式快取。它支援多種後端實現。

### 以 Redis 為例

**步驟 1：安裝 NuGet 套件**
```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

**步驟 2：在 `Program.cs` 中註冊服務**
```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyWebApp_"; // 可選的前綴，用於區分不同應用的快取
});
```

**步驟 3：在服務中注入並使用**
`IDistributedCache` 的 API 與 `IMemoryCache` 不同，它只能儲存 `byte[]`。因此，我們通常需要手動進行物件的序列化（例如，轉成 JSON 字串）和反序列化。

```csharp
public class ProductService(IProductRepository repo, IDistributedCache distributedCache)
{
    public async Task<Product?> GetProductByIdAsync(int id)
    {
        string cacheKey = $"Product_{id}";
        
        // 1. 嘗試從 Redis 獲取資料 (byte[])
        byte[] cachedData = await distributedCache.GetAsync(cacheKey);

        if (cachedData != null)
        {
            // 快取命中，反序列化
            var cachedProductJson = Encoding.UTF8.GetString(cachedData);
            return JsonSerializer.Deserialize<Product>(cachedProductJson);
        }

        // 快取未命中
        var product = await repo.GetByIdAsync(id);
        if (product != null)
        {
            // 序列化成 JSON 字串，再轉成 byte[]
            var productJson = JsonSerializer.Serialize(product);
            var dataToCache = Encoding.UTF8.GetBytes(productJson);

            var options = new DistributedCacheEntryOptions()
                .SetAbsoluteExpiration(TimeSpan.FromMinutes(30));

            // 存入 Redis
            await distributedCache.SetAsync(cacheKey, dataToCache, options);
        }

        return product;
    }
}
```

## 4. 快取失效 (Cache Invalidation) 的重要性

快取最大的挑戰之一就是**快取失效**。當原始資料（資料庫中的資料）發生變更時（新增、修改、刪除），必須有一種機制來清除或更新對應的快取，否則使用者將會看到過期的、不正確的資料。

**常見的失效策略：**
1.  **時間到期 (Time-based Expiration)**：為快取設定一個固定的生命週期（絕對到期或滑動到期）。這是最簡單但可能不及時的策略。
2.  **主動清除 (Explicit Invalidation)**：在執行更新資料庫的程式碼後，立即手動呼叫 `cache.Remove(cacheKey)` 來清除對應的快取。這是最精確的策略。

**範例：**
```csharp
public async Task UpdateProductAsync(Product product)
{
    // 1. 更新資料庫
    await _repo.UpdateAsync(product);

    // 2. 清除該產品的快取
    string cacheKey = $"Product_{product.Id}";
    await _distributedCache.RemoveAsync(cacheKey);
}
```

## 結論

快取是提升應用程式效能的雙面刃，用得好能上天堂，用不好則會帶來資料不一致的地獄。

1.  **明確快取目標**：只快取那些「讀取頻繁、變動不頻繁」且「產生代價高昂」的資料。
2.  **選擇合適的類型**：根據你的部署環境（單機 vs. 負載平衡）來選擇使用記憶體快取還是分散式快取。
3.  **制定失效策略**：這是快取設計中最重要的一環。必須確保在資料變更時，快取能夠被及時地清除或更新。
4.  **不要快取過大的物件**：快取巨大的物件會快速耗盡記憶體或增加網路傳輸負擔。只快取你真正需要的資料（例如，使用 DTO）。
5.  **處理快取穿透/雪崩**：在設計高併發系統時，還需考慮快取穿透（查詢一個不存在的 key）、雪崩（大量 key 同時失效）等進階問題，並設定相應的保護機制。
