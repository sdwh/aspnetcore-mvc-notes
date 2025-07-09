# ASP.NET Core MVC 教學 (.NET 8)：強型別表單開發技巧

在 Web 開發中，處理 HTML 表單是與使用者互動最核心的一環。一個設計良好的表單不僅關乎使用者體驗，更直接影響到資料的正確性與安全性。ASP.NET Core MVC 提供了基於「強型別模型 (Strongly-typed Model)」的表單開發方式，它利用 C# 的型別系統，將後端模型與前端表單緊密地、安全地連結在一起。

這種方式相比於傳統手動讀取 `Request.Form["fieldName"]` 的作法，具有壓倒性的優勢，包括編譯時檢查、IntelliSense 支援、以及與模型驗證框架的無縫整合。

## 1. 強型別的核心：ViewModel

強型別表單的基礎是一個專門為該表單（或 View）設計的 **ViewModel**。這個 ViewModel 精確地定義了表單需要收集的所有欄位、它們的資料型別，以及相關的驗證規則。

**範例：一個使用者註冊的 ViewModel**
```csharp
using System.ComponentModel.DataAnnotations;

public class RegisterViewModel
{
    [Required(ErrorMessage = "電子郵件為必填項")]
    [EmailAddress(ErrorMessage = "請輸入有效的電子郵件格式")]
    [Display(Name = "電子郵件地址")]
    public required string Email { get; set; }

    [Required]
    [StringLength(100, ErrorMessage = "{0} 的長度必須介於 {2} 到 {1} 個字元之間。", MinimumLength = 8)]
    [DataType(DataType.Password)]
    [Display(Name = "密碼")]
    public required string Password { get; set; }

    [DataType(DataType.Password)]
    [Display(Name = "確認密碼")]
    [Compare("Password", ErrorMessage = "密碼與確認密碼不相符。")]
    public required string ConfirmPassword { get; set; }

    [Range(typeof(bool), "true", "true", ErrorMessage = "您必須同意服務條款")]
    [Display(Name = "我同意服務條款")]
    public bool AgreeToTerms { get; set; }
}
```
這個 ViewModel 不僅定義了資料的「形狀」，更透過 Data Annotations 定義了資料的「規則」。

## 2. View 中的強型別輔助方法

在 Razor View 中，我們使用一系列以 `asp-` 為前綴的 **Tag Helper** 來將 HTML 元素與 ViewModel 的屬性進行繫結。

**`Register.cshtml`**
```html
@model RegisterViewModel

<h2>建立新帳戶</h2>

<form asp-action="Register" method="post">
    @* 用於顯示非欄位相關的錯誤 *@
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>

    <div class="form-group mb-3">
        @* asp-for 會自動讀取 Display(Name=...) *@
        <label asp-for="Email"></label>
        <input asp-for="Email" class="form-control" />
        @* asp-validation-for 會顯示該欄位的錯誤訊息 *@
        <span asp-validation-for="Email" class="text-danger"></span>
    </div>

    <div class="form-group mb-3">
        <label asp-for="Password"></label>
        <input asp-for="Password" class="form-control" />
        <span asp-validation-for="Password" class="text-danger"></span>
    </div>

    <div class="form-group mb-3">
        <label asp-for="ConfirmPassword"></label>
        <input asp-for="ConfirmPassword" class="form-control" />
        <span asp-validation-for="ConfirmPassword" class="text-danger"></span>
    </div>

    <div class="form-check mb-3">
        <input asp-for="AgreeToTerms" class="form-check-input" />
        <label asp-for="AgreeToTerms" class="form-check-label"></label>
        <br/>
        <span asp-validation-for="AgreeToTerms" class="text-danger"></span>
    </div>

    <button type="submit" class="btn btn-primary">註冊</button>
</form>

@section Scripts {
    @{
        // 引入非侵入式驗證腳本，啟用客戶端驗證
        await Html.RenderPartialAsync("_ValidationScriptsPartial");
    }
}
```

### Tag Helper 的魔力

*   **`@model RegisterViewModel`**: 宣告這個 View 是強型別的，其模型為 `RegisterViewModel`。
*   **`asp-for="PropertyName"`**:
    *   自動產生正確的 `id` 和 `name` 屬性 (e.g., `id="Email" name="Email"`)，這對於模型繫結至關重要。
    *   自動填入模型屬性的**目前值**到 `value` 屬性中。這在編輯表單或表單提交失敗後需要重新顯示時非常有用。
    *   根據屬性的資料型別選擇合適的 `type`。例如，`[DataType(DataType.Password)]` 會讓 `<input>` 產生 `type="password"`。對於 `bool` 型別，會產生 `type="checkbox"`。
*   **`label asp-for`**: 自動從 `[Display(Name = "...")]` 屬性讀取文字。
*   **`span asp-validation-for`**: 產生用於顯示驗證錯誤的 `<span>`，並帶有 `data-valmsg-for` 屬性，以便客戶端驗證腳本能找到它。

## 3. Controller 中的處理流程

在 Controller 中，Action 方法的參數直接就是我們的 ViewModel，ASP.NET Core 的模型繫結機制會自動將表單送過來的資料填充到 ViewModel 實例中。

**`AccountController.cs`**
```csharp
public class AccountController : Controller
{
    public IActionResult Register()
    {
        return View();
    }

    [HttpPost]
    [ValidateAntiForgeryToken] // 防止 CSRF 攻擊
    public async Task<IActionResult> Register(RegisterViewModel model)
    {
        // 檢查所有 Data Annotations 和 IValidatableObject 的規則是否都通過
        if (ModelState.IsValid)
        {
            // 驗證通過，執行註冊邏輯...
            // 例如，將 ViewModel 對應到 User Entity，然後存到資料庫
            var user = new AppUser { Email = model.Email, ... };
            var result = await _userManager.CreateAsync(user, model.Password);

            if (result.Succeeded)
            {
                // 成功後重導向
                return RedirectToAction("Index", "Home");
            }
            
            // 如果 UserManager 建立失敗，將錯誤加到 ModelState
            foreach (var error in result.Errors)
            {
                ModelState.AddModelError(string.Empty, error.Description);
            }
        }

        // 如果 ModelState.IsValid 為 false，或後續邏輯失敗
        // 將使用者傳入的 model 再次傳回 View
        // 這樣 Tag Helper 就能自動重新填入使用者已輸入的資料
        return View(model);
    }
}
```

## 4. 強型別開發的優勢總結

1.  **編譯時安全 (Compile-time Safety)**：如果你在 `asp-for` 中打錯了屬性名稱，專案將無法編譯通過，從而在開發早期就發現錯誤。
2.  **重構友好 (Refactoring-friendly)**：當你在 ViewModel 中重新命名一個屬性時，Visual Studio 的重構工具會自動更新所有在 View 中使用到該屬性的地方。
3.  **IntelliSense 支援**：在 View 中編寫 `asp-for` 時，可以獲得完整的程式碼提示。
4.  **自動化 (Automation)**：自動產生 `id`, `name`, `value` 和驗證屬性，減少了大量的手動編碼和出錯的機會。
5.  **程式碼乾淨**：View 的程式碼更接近純 HTML，更具可讀性。
6.  **無縫整合驗證**：後端 ViewModel 的驗證規則可以無縫地、自動地應用到前端表單，實現伺服器端和客戶端雙重驗證。

強型別表單開發是 ASP.NET Core MVC 的核心實踐之一。它將 C# 的型別優勢延伸到了前端開發中，構建了一套高效、安全、易於維護的表單處理工作流程。
