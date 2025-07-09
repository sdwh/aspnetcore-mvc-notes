# ASP.NET Core MVC 教學 (.NET 8)：避免 N+1 查詢問題

在所有使用 ORM (Object-Relational Mapper) 框架（如 Entity Framework Core）的應用程式中，「N+1 查詢問題」可以說是最常見、最隱蔽，也最具破壞性的效能殺手。如果不加以注意，一個看似簡單的頁面可能會在背後悄悄地對資料庫發起成百上千次的查詢，導致頁面載入極度緩慢，並給資料庫帶來巨大壓力。

理解 N+1 問題的成因並學會如何解決它，是每一位後端開發者的必修課。

## 1. 什麼是 N+1 查詢問題？

N+1 查詢問題發生在當你需要載入一個主實體列表（例如，文章列表），並且對於列表中的**每一個**主實體，你又需要去載入其關聯的子實體（例如，每篇文章的作者資訊）。

這個過程會分解成：
*   **第 1 次查詢**：獲取所有主實體（N 篇文章）。
*   **接下來的 N 次查詢**：對於每一篇文章，單獨發起一次查詢來獲取其作者的資訊。

總共執行了 `1 + N` 次資料庫查詢，因此得名「N+1」。

### 範例：一個典型的 N+1 場景

假設我們有 `Blog` 和 `Post` 兩個實體，一個部落格可以有多篇文章。

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Post> Posts { get; set; } = new();
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int BlogId { get; set; }
    public Blog Blog { get; set; } // 導覽屬性
}
```

現在，我們想在一個頁面上顯示所有文章的標題以及它們所屬的部落格名稱。

**一個會導致 N+1 的錯誤寫法：**
```csharp
// Controller Action
public async Task<IActionResult> Index()
{
    // 第 1 次查詢：獲取所有文章
    var posts = await _context.Posts.ToListAsync();
    return View(posts);
}
```

**`Index.cshtml` View:**
```html
@model List<Post>

@foreach (var post in Model)
{
    <tr>
        <td>@post.Title</td>
        <td>
            @* 
              這裡觸發了延遲載入 (Lazy Loading)。
              對於每一篇文章，EF Core 都會發起一次新的查詢
              來獲取 post.Blog 的資料。
              這就是 N 次查詢的來源！
            *@
            @post.Blog.Name 
        </td>
    </tr>
}
```
如果 `posts` 列表有 100 篇文章，這段程式碼將會對資料庫執行 `1 + 100 = 101` 次查詢！

## 2. 解決方案：預先載入 (Eager Loading)

解決 N+1 問題的核心思想是：**在第一次查詢時，就告訴 Entity Framework Core，我不僅需要主實體，還需要它關聯的哪些子實體，請一次性地將它們全部載入。** 這個過程被稱為「預先載入 (Eager Loading)」。

EF Core 提供了多種方法來實現預先載入。

### a. `Include()` 方法

`Include()` 是最直接、最常用的預先載入方法。它告訴 EF Core 在產生 SQL 時，使用 `JOIN` 來將關聯的資料表一併查詢出來。

**修正後的 Controller Action:**
```csharp
public async Task<IActionResult> Index()
{
    // 使用 Include() 來預先載入 Blog 導覽屬性
    var posts = await _context.Posts
                              .Include(p => p.Blog) // <--- 關鍵！
                              .ToListAsync();
    return View(posts);
}
```
現在，EF Core 只會產生**單一一次** SQL 查詢，該查詢使用 `LEFT JOIN` 將 `Posts` 和 `Blogs` 兩個資料表關聯起來，並一次性獲取所有需要的資料。資料庫的查詢次數從 `N+1` 變成了 `1`。

### b. `ThenInclude()` 方法 (用於多層級載入)

如果你需要載入多層級的關聯資料（例如：部落格 -> 文章 -> 評論），你可以使用 `ThenInclude()`。

```csharp
var blogs = await _context.Blogs
                          .Include(b => b.Posts)       // 載入文章
                          .ThenInclude(p => p.Comments) // 載入文章的評論
                          .ToListAsync();
```

### c. 投影 (Projection) 與 `Select()`

另一種非常高效的方法是使用 LINQ 的 `Select()` 方法進行「投影」。投影的意思是，我們不查詢完整的實體物件，而是只查詢我們在 View 中真正需要的資料，並將其塑造成一個 DTO (Data Transfer Object) 或匿名型別。

EF Core 足夠聰明，它會分析你的 `Select()` 運算式，並產生一個只包含所需欄位的最佳化 SQL 查詢。

**使用 DTO 進行投影：**
```csharp
// 一個專門用於此 View 的 ViewModel/DTO
public class PostSummaryViewModel
{
    public string PostTitle { get; set; }
    public string BlogName { get; set; }
}

// Controller Action
public async Task<IActionResult> Index()
{
    var postSummaries = await _context.Posts
                                      .Select(p => new PostSummaryViewModel
                                      {
                                          PostTitle = p.Title,
                                          BlogName = p.Blog.Name // EF Core 會自動處理 JOIN
                                      })
                                      .ToListAsync();
    return View(postSummaries);
}
```
**投影的優點：**
*   **極致效能**：只從資料庫傳輸絕對必要的欄位，減少了網路流量和記憶體佔用。
*   **自動處理 JOIN**：你不需要手動寫 `Include()`，EF Core 會自動分析並產生 JOIN。
*   **安全性**：避免了將完整的實體模型傳遞給 View，降低了過度提交 (Over-posting) 的風險。

**最佳實務**：對於所有唯讀 (Read-only) 的列表頁面，**強烈建議優先使用投影 (`Select`) 的方式**。它通常是效能最好的選擇。

## 3. 如何發現 N+1 問題？

1.  **監控 EF Core 日誌**：在開發環境中，啟用 EF Core 的命令日誌記錄。如果你在頁面載入時，看到控制台中短時間內連續輸出了大量相似的 SQL 查詢，那幾乎可以肯定是 N+1 問題。
2.  **使用效能分析工具**：像 Visual Studio 內建的資料庫分析工具或 MiniProfiler 這樣的工具，可以非常直觀地展示出一個請求中發生的所有資料庫查詢和它們的耗時，讓 N+1 問題無所遁形。

## 結論

N+1 查詢是 Entity Framework Core (以及其他 ORM) 使用者最需要警惕的效能陷阱。

1.  **理解成因**：問題源於在迴圈中觸發對關聯屬性的延遲載入 (Lazy Loading)。
2.  **使用 `Include()`**：對於需要載入完整實體及其關聯實體的場景（例如，編輯頁面），使用 `Include()` 和 `ThenInclude()` 進行預先載入。
3.  **優先使用投影 (`Select`)**：對於唯讀的列表頁面，使用 `Select()` 將資料投影到 DTO 或 ViewModel 中，這是最高效、最安全的方式。
4.  **保持警覺**：在開發過程中，持續監控日誌和使用分析工具，及早發現並修復 N+1 問題。

養成預先載入和投影的習慣，是確保你的 ASP.NET Core 應用程式資料庫效能的關鍵。
