# ASP.NET Core MVC 教學 (.NET 8)：瞭解 ViewModel 的用途與使用技巧

在 ASP.NET Core MVC 的架構中，"V" 代表 View (檢視)，"M" 代表 Model (模型)。然而，直接將資料庫的實體模型 (Entity Model) 傳遞給 View 是一種非常不好的做法，這會導致 UI 與資料庫結構的緊密耦合。為了解決這個問題，我們引入了一個重要的設計模式——**ViewModel**。

ViewModel，即「檢視模型」，是一個專門為了服務某個特定 View 而設計的類別。它的存在是為了**塑模 (Shape)** View 所需的資料和行為，充當 Controller 和 View 之間的中介者。

## 1. 為什麼需要 ViewModel？

使用 ViewModel 能帶來許多好處：

1.  **解耦 (Decoupling)**：將 View 與底層的資料模型 (Entity) 隔離開來。資料庫的欄位變更（例如改名、增刪）不會直接破壞 View，只需要調整 ViewModel 與 Entity 之間的對應即可。
2.  **塑模特定資料 (Data Shaping)**：一個 View 可能需要顯示來自多個不同資料模型的資料。ViewModel 可以將這些零散的資料彙整成一個單一、強型別的物件，方便 View 使用。
3.  **安全性 (Security)**：避免了「過度提交 (Over-posting)」攻擊。View 只會繫結 ViewModel 中明確定義的屬性，惡意使用者無法提交表單來修改他們不該碰到的後端欄位（例如 `IsAdmin`, `Salary` 等）。
4.  **包含展示邏輯 (Presentation Logic)**：ViewModel 不僅可以包含資料，還可以包含與顯示相關的邏輯，例如下拉式選單的選項、格式化後的字串等。
5.  **簡化 View**：有了 ViewModel，View 的 Razor 語法可以變得非常乾淨和簡單，只需專注於呈現資料，而不需要進行複雜的邏輯判斷或資料轉換。

## 2. ViewModel 的實際應用

讓我們透過一個部落格文章的例子來看看 ViewModel 如何運作。

### 場景

我們需要建立一個「建立新文章」的頁面。這個頁面需要：
*   一個標題輸入框。
*   一個內容編輯區。
*   一個下拉式選單，讓使用者可以選擇文章的分類。
*   一個標籤 (Tags) 輸入框，使用者可以用逗號分隔輸入多個標籤。

### 傳統做法 (不使用 ViewModel)

如果不使用 ViewModel，Controller 可能會這樣寫：

```csharp
public class PostsController : Controller
{
    private readonly IPostRepository _postRepo;
    private readonly ICategoryRepository _categoryRepo;

    // ...

    public async Task<IActionResult> Create()
    {
        // 需要從多個地方獲取資料，並用 ViewBag/ViewData 傳遞
        ViewBag.Categories = new SelectList(await _categoryRepo.GetAllAsync(), "Id", "Name");
        return View();
    }

    [HttpPost]
    public async Task<IActionResult> Create(PostEntity post, string tags)
    {
        // ... 手動處理 tags，儲存 post ...
    }
}
```
這種做法的缺點：
*   `ViewBag` 是動態型別，沒有編譯時檢查，容易打錯字而出錯。
*   View 需要知道 `PostEntity` 的結構，並且還需要處理 `tags` 這個額外的參數。
*   Controller 的 `Create` POST 方法直接接收 `PostEntity`，有 Over-posting 風險。

### 使用 ViewModel 的優雅做法

現在，我們為這個「建立新文章」的 View 量身打造一個 ViewModel。

**`CreatePostViewModel.cs`**
```csharp
using Microsoft.AspNetCore.Mvc.Rendering;
using System.ComponentModel.DataAnnotations;

public class CreatePostViewModel
{
    [Required(ErrorMessage = "標題是必填的")]
    [StringLength(100)]
    public required string Title { get; set; }

    [Required(ErrorMessage = "內容是必填的")]
    public required string Content { get; set; }

    [Display(Name = "分類")]
    [Required(ErrorMessage = "請選擇一個分類")]
    public int SelectedCategoryId { get; set; }

    [Display(Name = "標籤 (以逗號分隔)")]
    public string? Tags { get; set; }

    // 這個屬性用於承載下拉選單的選項，它不參與模型繫結
    public IEnumerable<SelectListItem>? Categories { get; set; }
}
```

**`PostsController.cs` (使用 ViewModel)**
```csharp
public class PostsController(IPostRepository postRepo, ICategoryRepository categoryRepo) : Controller
{
    public async Task<IActionResult> Create()
    {
        // 1. 建立 ViewModel 實例
        var viewModel = new CreatePostViewModel
        {
            // 2. 填充 View 所需的輔助資料 (下拉選單)
            Categories = new SelectList(await _categoryRepo.GetAllAsync(), "Id", "Name")
        };
        return View(viewModel);
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(CreatePostViewModel viewModel)
    {
        // 3. 檢查模型驗證結果
        if (!ModelState.IsValid)
        {
            // 如果驗證失敗，需要重新填充下拉選單並返回原 View
            viewModel.Categories = new SelectList(await _categoryRepo.GetAllAsync(), "Id", "Name", viewModel.SelectedCategoryId);
            return View(viewModel);
        }

        // 4. 將 ViewModel 對應回 Entity Model
        var postEntity = new PostEntity
        {
            Title = viewModel.Title,
            Content = viewModel.Content,
            CategoryId = viewModel.SelectedCategoryId,
            CreatedAt = DateTime.UtcNow
        };

        // 處理標籤...
        var tagList = viewModel.Tags?.Split(',', StringSplitOptions.RemoveEmptyEntries).ToList() ?? [];

        await _postRepo.CreateAsync(postEntity, tagList);

        return RedirectToAction("Index");
    }
}
```

**`Create.cshtml` (強型別 View)**
```html
@model CreatePostViewModel

<h2>建立新文章</h2>

<form asp-action="Create" method="post">
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>
    
    <div class="form-group">
        <label asp-for="Title"></label>
        <input asp-for="Title" class="form-control" />
        <span asp-validation-for="Title" class="text-danger"></span>
    </div>

    <div class="form-group">
        <label asp-for="SelectedCategoryId"></label>
        <select asp-for="SelectedCategoryId" asp-items="@Model.Categories" class="form-control">
            <option value="">-- 請選擇 --</option>
        </select>
        <span asp-validation-for="SelectedCategoryId" class="text-danger"></span>
    </div>

    @* 其他欄位 ... *@

    <button type="submit" class="btn btn-primary">建立</button>
</form>
```

## 3. ViewModel vs. DTO

ViewModel 和 DTO (Data Transfer Object) 的概念非常相似，都是為了傳遞資料而存在的簡單物件。它們的主要區別在於**意圖**和**使用場景**：

*   **ViewModel**: **專為 MVC 的 View 設計**。它可能包含 `SelectListItem`、格式化邏輯或與 UI 互動相關的屬性。它的生命週期通常只在一次 HTTP 請求-回應之間。
*   **DTO**: **專為跨應用程式邊界 (如 Web API) 傳輸資料而設計**。它應該是純粹的資料容器，不包含任何展示邏輯，以便任何客戶端（Web、行動 App、其他服務）都能使用。

簡單來說，你可以把 ViewModel 看作是一種**特定於 MVC 展示層的 DTO**。

## 結論

ViewModel 是 MVC 開發中不可或缺的一個模式。

1.  **為每個 View 量身打造**：不要試圖建立一個通用的、適用於多個 View 的巨大 ViewModel。保持其專一性。
2.  **保持扁平化**：ViewModel 的結構應該是扁平的，直接對應 View 上的欄位，避免複雜的巢狀物件。
3.  **利用資料驗證**：在 ViewModel 的屬性上使用 Data Annotations (`[Required]`, `[StringLength]` 等) 來進行伺服器端驗證。
4.  **使用 AutoMapper**：對於複雜的應用，手動在 ViewModel 和 Entity 之間進行對應會變得很繁瑣。學習並使用 AutoMapper 這類工具可以極大地提升開發效率。
