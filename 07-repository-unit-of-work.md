# ASP.NET Core MVC 教學 (.NET 8)：理解 Repository 與 Unit of Work 設計樣式

當應用程式的規模逐漸擴大，如何有效地組織資料存取邏輯，並將其與業務邏輯分離，成為一個至關重要的議題。Repository (倉儲) 和 Unit of Work (工作單元) 是兩種經典的設計模式，它們共同協作，為應用程式提供一個抽象、可測試、可維護的資料存取層。

本文將探討這兩種模式的概念，以及如何在 ASP.NET Core 專案中結合 Entity Framework Core 來實現它們。

## 1. Repository (倉儲) 模式

Repository 模式的核心思想是**將資料存取邏輯從業務邏輯中分離出來**。它扮演著一個中介者的角色，介於領域模型 (Domain Model) 和資料對應層 (Data Mapping Layers) 之間，模擬一個記憶體中的物件集合。

對於業務邏輯層來說，它不再需要關心資料是來自 SQL Server、Cosmos DB 還是其他任何儲存體。它只需要和 Repository 溝通，告訴它「給我某個 ID 的產品」或「儲存這個新的訂單」。Repository 內部則封裝了所有與資料庫互動的細節 (例如使用 EF Core 的 `DbContext`)。

### Repository 的職責：
*   提供一個類似集合的介面來存取領域物件。
*   封裝查詢資料庫的邏輯。
*   將業務邏輯與資料庫技術解耦。

### 實作範例

首先，我們定義一個通用的 Repository 介面，這有助於保持一致性。

**`IGenericRepository.cs`**
```csharp
// T 必須是一個 class (代表實體)
public interface IGenericRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity); // Update 通常是同步的，因為 EF Core 的狀態追蹤機制
    void Delete(T entity);
}
```

接著，是這個通用介面的具體實現。

**`GenericRepository.cs`**
```csharp
using Microsoft.EntityFrameworkCore;

// 這個 Repository 依賴於 DbContext
public class GenericRepository<T>(ApplicationDbContext context) : IGenericRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context = context;
    internal DbSet<T> _dbSet = context.Set<T>();

    public async Task AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
    }

    public void Delete(T entity)
    {
        _dbSet.Remove(entity);
    }

    public async Task<IEnumerable<T>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }

    public async Task<T?> GetByIdAsync(int id)
    {
        return await _dbSet.FindAsync(id);
    }

    public void Update(T entity)
    {
        _dbSet.Update(entity);
    }
}
```

我們也可以為特定的實體建立專屬的 Repository，以包含更複雜的查詢邏輯。

**`IProductRepository.cs`**
```csharp
public interface IProductRepository : IGenericRepository<Product>
{
    Task<IEnumerable<Product>> GetTopSellingProductsAsync(int count);
}
```

**`ProductRepository.cs`**
```csharp
public class ProductRepository(ApplicationDbContext context) 
    : GenericRepository<Product>(context), IProductRepository
{
    public async Task<IEnumerable<Product>> GetTopSellingProductsAsync(int count)
    {
        // 假設有相關的銷售數據可以查詢
        return await _dbSet
            .OrderByDescending(p => p.TotalSales)
            .Take(count)
            .ToListAsync();
    }
}
```

## 2. Unit of Work (工作單元) 模式

Unit of Work 模式負責**維護一個受業務異動影響的物件列表，並協調這些異動的寫入以及併發問題的處理**。簡單來說，它確保了一系列操作能夠在一個單一的資料庫交易 (Transaction) 中完成，要麼全部成功，要麼全部失敗。

當你使用 Entity Framework Core 時，`DbContext` 類別本身就已經實作了 Unit of Work 模式。當你新增、修改、刪除實體時，`DbContext` 會在記憶體中追蹤這些變更。直到你呼叫 `SaveChanges()` 或 `SaveChangesAsync()` 方法時，它才會將所有被追蹤的變更生成對應的 SQL 指令，並在一個交易中一次性地提交到資料庫。

### 為什麼還需要封裝 Unit of Work？

既然 `DbContext` 已經是 Unit of Work，為什麼我們還需要再建立一個 `UnitOfWork` 類別來封裝它呢？

1.  **集中管理 Repositories**：Unit of Work 類別可以作為一個「工廠」，統一管理和提供所有 Repository 的實例。這樣可以確保所有 Repository 共享同一個 `DbContext` 實例，從而保證它們在同一個交易範圍內運作。
2.  **單一的儲存點**：將 `SaveChanges()` 的呼叫集中到 Unit of Work 類別中。業務邏輯層只需要呼叫 `unitOfWork.CompleteAsync()`，而不需要直接依賴 `DbContext`。這進一步加強了業務邏輯與 EF Core 的解耦。
3.  **易於擴展**：未來如果需要加入分散式交易或其他邏輯，只需要修改 Unit of Work 的實現，而不需要動到上層的業務程式碼。

### 實作範例

**`IUnitOfWork.cs`**
```csharp
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    ICategoryRepository Categories { get; }
    // ... 其他 Repositories

    Task<int> CompleteAsync();
}
```

**`UnitOfWork.cs`**
```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    public IProductRepository Products { get; private set; }
    public ICategoryRepository Categories { get; private set; }

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
        Products = new ProductRepository(_context);
        Categories = new CategoryRepository(_context);
    }

    public async Task<int> CompleteAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

### 在 `Program.cs` 中註冊

```csharp
// 註冊 DbContext
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// 註冊 Unit of Work
// 使用 Scoped 生命週期，確保在同一個 HTTP 請求中，所有服務都拿到同一個 UnitOfWork 實例
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

### 在 Controller 中使用

```csharp
public class OrdersController(IUnitOfWork unitOfWork, ILogger<OrdersController> logger) : Controller
{
    public async Task<IActionResult> CreateOrder(OrderViewModel model)
    {
        try
        {
            // 1. 從 ViewModel 建立訂單實體
            var order = new Order { /* ... */ };
            await unitOfWork.Orders.AddAsync(order);

            // 2. 更新產品庫存
            var product = await unitOfWork.Products.GetByIdAsync(model.ProductId);
            product.Stock -= model.Quantity;
            unitOfWork.Products.Update(product);

            // 3. 一次性提交所有變更
            await unitOfWork.CompleteAsync();

            return Ok();
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to create order.");
            return BadRequest();
        }
    }
}
```

## 結論

| 模式 | 核心職責 | 優點 |
| :--- | :--- | :--- |
| **Repository** | 封裝特定實體的資料查詢邏輯。 | 解耦業務與資料存取、集中查詢邏輯、可測試性。 |
| **Unit of Work** | 維護一個交易內的所有變更，並統一提交。 | 保證資料一致性、集中管理 Repositories、單一儲存點。 |

雖然 EF Core 的 `DbContext` 已經內建了這兩種模式的基礎功能，但透過明確地建立 Repository 和 Unit of Work 的抽象層，我們可以獲得一個更清晰、更有組織、更易於維護和測試的應用程式架構。這對於需要長期發展和維護的專案來說，是一項非常有價值的投資。
