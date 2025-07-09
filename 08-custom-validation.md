# ASP.NET Core MVC 教學 (.NET 8)：自訂輸入驗證與模型驗證

在 Web 應用程式中，確保使用者輸入的資料是有效且符合預期的，是保障系統穩定與安全的基礎。ASP.NET Core MVC 提供了一套強大且可擴展的模型驗證 (Model Validation) 框架。除了使用內建的驗證屬性 (Validation Attributes) 如 `[Required]`, `[StringLength]`, `[Range]` 之外，我們經常需要根據特定的業務規則來建立自訂的驗證邏輯。

本文將深入探討如何在 ASP.NET Core 中實作自訂的輸入驗證與模型驗證。

## 1. 自訂驗證屬性 (Custom Validation Attributes)

當內建的驗證屬性無法滿足你的需求時，最直接的方法就是建立一個自訂的驗證屬性。這需要你建立一個繼承自 `ValidationAttribute` 的類別，並覆寫 `IsValid` 方法。

### 範例：驗證出生日期不能晚於今天

假設我們在一個註冊表單中，需要驗證使用者填寫的出生日期 (`DateOfBirth`) 必須是過去的日期。

**`EnsureDateNotInFutureAttribute.cs`**
```csharp
using System.ComponentModel.DataAnnotations;

// 標示這個屬性只能用在屬性 (Property) 上
[AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
public class EnsureDateNotInFutureAttribute : ValidationAttribute
{
    public EnsureDateNotInFutureAttribute()
    {
        // 設定預設的錯誤訊息
        ErrorMessage = "日期不能晚於今天";
    }

    // 核心驗證邏輯
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        // 'value' 是使用者輸入的值
        if (value is DateTime dateValue)
        {
            if (dateValue.Date > DateTime.Now.Date)
            {
                // 驗證失敗，回傳一個包含錯誤訊息的 ValidationResult
                return new ValidationResult(ErrorMessage);
            }
        }
        
        // 如果 value 是 null 或驗證成功，回傳 Success
        // 注意：這個驗證器不處理 null 的情況，那是 [Required] 的職責
        return ValidationResult.Success;
    }
}
```

**在 ViewModel 中使用：**
```csharp
public class RegisterViewModel
{
    // ... 其他屬性

    [Required]
    [DataType(DataType.Date)]
    [Display(Name = "出生日期")]
    [EnsureDateNotInFuture] // 套用自訂的驗證屬性
    public DateTime DateOfBirth { get; set; }
}
```
現在，如果使用者選擇了一個未來的日期，`ModelState.IsValid` 將會是 `false`，並且會自動產生對應的錯誤訊息。

## 2. 跨屬性驗證 (Cross-Property Validation)

有時候，一個屬性的驗證規則需要依賴另一個屬性的值。例如，一個「確認密碼」欄位必須和「密碼」欄位的值完全相同。

### 方法一：在 ViewModel 中實作 `IValidatableObject`

`IValidatableObject` 介面提供了一個 `Validate` 方法，它會在所有屬性層級的驗證都完成後被呼叫。這使得它成為實作跨屬性驗證的理想位置。

**範例：驗證密碼和確認密碼一致**
```csharp
public class ChangePasswordViewModel : IValidatableObject
{
    [Required]
    [DataType(DataType.Password)]
    [Display(Name = "新密碼")]
    public required string NewPassword { get; set; }

    [Required]
    [DataType(DataType.Password)]
    [Display(Name = "確認新密碼")]
    public required string ConfirmPassword { get; set; }

    // IValidatableObject 的實作
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (NewPassword != ConfirmPassword)
        {
            // 使用 yield return 來回傳多個錯誤
            // 第二個參數是一個陣列，指定這個錯誤與哪些欄位相關
            yield return new ValidationResult(
                "新密碼與確認密碼不相符。",
                new[] { nameof(NewPassword), nameof(ConfirmPassword) }
            );
        }
    }
}
```
當驗證失敗時，錯誤訊息會同時顯示在「新密碼」和「確認新密碼」兩個欄位的驗證摘要中。

### 方法二：建立一個可比較屬性的自訂驗證屬性

`IValidatableObject` 的缺點是驗證邏輯和 ViewModel 耦合在一起。如果這個邏輯需要在多個地方重用，更好的方法是建立一個更通用的自訂驗證屬性。

ASP.NET Core 內建了一個 `[Compare]` 屬性，可以直接用來處理這種情況，但為了教學目的，我們來看看如何自己實現一個類似的。

**`CompareToAttribute.cs`**
```csharp
public class CompareToAttribute : ValidationAttribute
{
    private readonly string _otherPropertyName;

    public CompareToAttribute(string otherPropertyName)
    {
        _otherPropertyName = otherPropertyName;
    }

    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        // 取得要比較的另一個屬性的資訊
        var otherProperty = validationContext.ObjectType.GetProperty(_otherPropertyName);

        if (otherProperty == null)
        {
            return new ValidationResult($"未知的屬性: {_otherPropertyName}");
        }

        // 取得另一個屬性的值
        var otherPropertyValue = otherProperty.GetValue(validationContext.ObjectInstance);

        // 比較兩個屬性的值
        if (!Equals(value, otherPropertyValue))
        {
            return new ValidationResult(ErrorMessage ?? $"{validationContext.DisplayName} 與 {_otherPropertyName} 不相符。");
        }

        return ValidationResult.Success;
    }
}
```
**使用方式：**
```csharp
public class ChangePasswordViewModel
{
    [Required]
    [DataType(DataType.Password)]
    [Display(Name = "新密碼")]
    public required string NewPassword { get; set; }

    [Required]
    [DataType(DataType.Password)]
    [Display(Name = "確認新密碼")]
    [CompareTo(nameof(NewPassword), ErrorMessage = "密碼不一致")] // 使用自訂的比較屬性
    public required string ConfirmPassword { get; set; }
}
```

## 3. 遠端驗證 (Remote Validation)

在某些情況下，驗證需要查詢後端資料庫才能完成。例如，在註冊時檢查「使用者名稱」是否已經被他人使用。如果等到整個表單提交後才驗證，使用者體驗會很差。

「遠端驗證」允許我們在使用者輸入完成後（例如，焦點離開輸入框時），透過 AJAX 非同步地向伺服器發送一個請求來進行即時驗證。

**步驟：**

1.  **在 Controller 中建立一個驗證用的 Action**
    這個 Action 必須回傳 `JsonResult`。如果驗證成功，回傳 `true`；如果失敗，回傳錯誤訊息字串。

    **`AccountController.cs`**
    ```csharp
    [AcceptVerbs("GET", "POST")] // 允許 GET 或 POST 請求
    public async Task<IActionResult> IsUserNameInUse(string userName)
    {
        var user = await _userManager.FindByNameAsync(userName);
        if (user == null)
        {
            // 使用者名稱可用
            return Json(true);
        }
        // 使用者名稱已被使用，回傳錯誤訊息
        return Json($"使用者名稱 '{userName}' 已經被註冊了。");
    }
    ```

2.  **在 ViewModel 中使用 `[Remote]` 屬性**
    將 `[Remote]` 屬性加到需要遠端驗證的欄位上，並指定對應的 Action 和 Controller。

    **`RegisterViewModel.cs`**
    ```csharp
    public class RegisterViewModel
    {
        [Required]
        [Display(Name = "使用者名稱")]
        [Remote(action: "IsUserNameInUse", controller: "Account")] // 設定遠端驗證
        public required string UserName { get; set; }

        // ...
    }
    ```

3.  **在 View 中確保引用了 jQuery Unobtrusive Validation**
    這個 JavaScript 函式庫會自動處理 `[Remote]` 屬性，發起 AJAX 請求。通常在 `_ValidationScriptsPartial.cshtml` 中已經包含了。

## 結論

ASP.NET Core MVC 的驗證框架提供了極大的彈性。

1.  **自訂驗證屬性**：當業務規則相對簡單且可在單一屬性上完成時，繼承 `ValidationAttribute` 是最直接的方法。
2.  **`IValidatableObject`**：當需要進行跨屬性驗證，且該驗證邏輯只在單一 ViewModel 中使用時，這是一個快速的解決方案。
3.  **可重用的比較屬性**：對於可重用的跨屬性驗證邏輯，建立一個通用的自訂驗證屬性是更佳的選擇。
4.  **遠端驗證 `[Remote]`**：當驗證需要即時查詢後端資料時，遠端驗證可以提供絕佳的使用者體驗。

靈活運用這些技巧，可以確保你的應用程式只接收乾淨、有效、安全的資料。
