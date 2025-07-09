# ASP.NET Core MVC 教學 (.NET 8)：瞭解資料模型、DTO 與 POCO

在物件導向的軟體開發中，我們不斷地在處理「物件」與「資料」。然而，不是所有的物件都生而平等。根據其職責與結構，它們可以被歸類為不同的模式。在 ASP.NET Core MVC 的世界裡，清晰地辨別 POCO、DTO 和資料模型 (Data Model) 的差異與用途，是建立一個結構清晰、可維護、可擴展應用程式的基礎。

## 1. POCO (Plain Old CLR Object)

POCO 是最基本、最單純的物件概念。它的全名是 "Plain Old CLR Object" (或有時稱為 Plain Old C# Object)，意指一個不依賴任何特定框架、沒有繼承任何框架基礎類別的簡單 C# 物件。

**POCO 的特徵：**
*   只包含資料 (屬性) 和可能的簡單業務邏輯 (方法)。
*   不包含與持久化 (如 Entity Framework)、序列化 (如 `[Serializable]`) 或任何特定框架相關的屬性 (Attribute) 或邏輯。
*   它是一個獨立的、可移植的業務物件。

**範例：**
一個代表核心業務領域的 `Book` 物件。

```csharp
// 這是一個純粹的 POCO
public class Book
{
    public Guid Id { get; private set; }
    public string Title { get; private set; }
    public string Author { get; private set; }
    private int _publicationYear;

    public Book(Guid id, string title, string author, int publicationYear)
    {
        // 在建構函式中可以有驗證邏輯
        if (string.IsNullOrWhiteSpace(title))
            throw new ArgumentException("Title cannot be empty.", nameof(title));
        
        Id = id;
        Title = title;
        Author = author;
        _publicationYear = publicationYear;
    }

    // 簡單的業務方法
    public bool IsPublishedInLeapYear()
    {
        return DateTime.IsLeapYear(_publicationYear);
    }
}
```
POCO 是你應用程式中**領域模型 (Domain Model)** 的理想候選者，它代表了最核心的業務概念。

## 2. 資料模型 (Data Model / Entity Model)

當一個 POCO 被用來**直接對應到資料庫的資料表結構**時，它就扮演了「資料模型」或「實體 (Entity)」的角色。在 ASP.NET Core 中，這通常是透過 Entity Framework Core (EF Core) 來實現的。

**資料模型的特徵：**
*   其結構 (屬性、型別) 通常與資料庫中的一個資料表完全對應。
*   它可能會被加上一些與資料庫持久化相關的屬性 (Data Annotations)，例如 `[Table]`, `[Key]`, `[ForeignKey]`。
*   它代表了資料的「形狀」，是應用程式與資料庫溝通的橋樑。

**範例：**
使用 EF Core 的 `Book` 實體。

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

// 這個 POCO 現在扮演了「資料模型」的角色
[Table("Books")] // 對應到資料庫的 "Books" 資料表
public class BookEntity
{
    [Key] // 標示為主鍵
    public int Id { get; set; }

    [Required] // 標示為必要欄位
    [MaxLength(200)]
    public required string Title { get; set; }

    [MaxLength(100)]
    public required string Author { get; set; }

    public int PublicationYear { get; set; }

    // 導覽屬性 (Navigation Property)，代表資料表之間的關聯
    public virtual ICollection<ReviewEntity> Reviews { get; set; } = new List<ReviewEntity>();
}
```
**核心原則**：**絕對不要**將你的資料模型 (Entity) 直接暴露給前端或外部 API。這會導致嚴重的問題，例如：
*   **過度暴露 (Over-posting)**：惡意使用者可能傳入他們不該修改的欄位 (如 `IsAdmin`)。
*   **耦合性**：任何資料庫結構的變更都會直接衝擊到 API 或 UI。
*   **洩漏內部結構**：將資料庫的設計細節暴露給外界。

## 3. DTO (Data Transfer Object)

為了解決上述問題，DTO 應運而生。DTO 是一個專門為了在不同層級或系統之間**傳輸資料**而設計的簡單物件。它的存在就是為了當作一個資料的「容器」或「信封」。

**DTO 的特徵：**
*   通常只包含屬性，沒有或極少有方法。
*   其結構是為了滿足某個特定通訊需求 (如一個 API 端點或一個 View) 而量身打造的。
*   它負責將資料從內部模型 (如 Entity) 安全地、有效地傳遞到外部。

**範例：**
假設我們有一個 API 端點需要回傳書的列表，但只需要顯示書名和作者。

```csharp
// 一個專門用於書本列表的 DTO
public class BookSummaryDto
{
    public required string Title { get; set; }
    public required string Author { get; set; }
}

// 一個用於建立新書的 DTO
public class CreateBookDto
{
    [Required(ErrorMessage = "書名為必填項")]
    [StringLength(200)]
    public required string Title { get; set; }

    [Required(ErrorMessage = "作者為必填項")]
    [StringLength(100)]
    public required string Author { get; set; }

    [Range(1000, 2100)]
    public int PublicationYear { get; set; }
}
```

**在控制器中使用 DTO：**
```csharp
[ApiController]
[Route("api/books")]
public class BooksController(IBookRepository bookRepo) : ControllerBase
{
    // GET /api/books
    [HttpGet]
    public async Task<ActionResult<IEnumerable<BookSummaryDto>>> GetBooks()
    {
        var bookEntities = await bookRepo.GetAllAsync();

        // 將 Entity 轉換為 DTO
        var bookDtos = bookEntities.Select(b => new BookSummaryDto
        {
            Title = b.Title,
            Author = b.Author
        });

        return Ok(bookDtos);
    }

    // POST /api/books
    [HttpPost]
    public async Task<IActionResult> CreateBook([FromBody] CreateBookDto createDto)
    {
        // 將 DTO 轉換為 Entity
        var newBook = new BookEntity
        {
            Title = createDto.Title,
            Author = createDto.Author,
            PublicationYear = createDto.PublicationYear
        };

        await bookRepo.AddAsync(newBook);
        return CreatedAtAction(nameof(GetBookById), new { id = newBook.Id }, newBook);
    }
}
```
> **提示**：手動進行 Entity 和 DTO 之間的轉換很繁瑣且容易出錯。在真實專案中，強烈建議使用如 **AutoMapper** 這類的對應工具庫來自動化這個過程。

## 結論

清晰地劃分這三種物件的角色，是構建良好軟體架構的關鍵。

| 類別 | 職責 | 主要位置 | 特徵 |
| :--- | :--- | :--- | :--- |
| **POCO** | 代表核心業務概念 | 領域層 (Domain Layer) | 獨立、無框架依賴、可包含業務邏輯 |
| **資料模型 (Entity)** | 對應資料庫結構 | 基礎設施層/資料存取層 | 與 DB 表格一致，可包含持久化屬性 |
| **DTO** | 在層與層之間傳輸資料 | 應用層/展現層 (API/MVC) | 為特定通訊需求設計，通常只有屬性 |

**黃金法則**：
*   用 **Entity** 與資料庫溝通。
*   用 **DTO** 與外部世界 (瀏覽器、手機 App、其他服務) 溝通。
*   用 **POCO** 代表你系統中不變的核心業務邏輯。
