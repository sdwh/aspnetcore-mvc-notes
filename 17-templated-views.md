# ASP.NET Core MVC 教學 (.NET 8)：範本檢視表單開發技巧

在開發大型應用程式時，我們經常會遇到需要在不同地方使用相同或相似的表單 UI。例如，「地址」輸入區塊可能會在使用者註冊、訂單填寫、個人資料修改等多個頁面中出現。如果我們在每個 View 中都重複編寫相同的表單 HTML，將會導致程式碼冗餘和維護困難。

ASP.NET Core MVC 提供了「範本檢視 (Templated Views)」或稱為「顯示與編輯範本 (Display and Editor Templates)」的強大功能，讓我們可以為特定的資料型別或屬性定義可重用的 Razor 範本。當 MVC 框架需要渲染該型別的資料時，它會自動尋找並使用我們定義好的範本。

## 1. 顯示與編輯範本 (Display and Editor Templates)

ASP.NET Core 提供了兩個核心的 HTML 輔助方法來觸發這個機制：

*   **`@Html.DisplayFor(m => m.PropertyName)`**: 用於**顯示**一個屬性的值。它會尋找對應的「顯示範本」。
*   **`@Html.EditorFor(m => m.PropertyName)`**: 用於產生一個**編輯**該屬性的表單輸入欄位。它會尋找對應的「編輯範本」。

這些方法比標準的 `asp-for` Tag Helper 更進一步，它們不僅僅是繫結到一個屬性，而是為整個屬性（或複雜物件）尋找一個對應的 `.cshtml` 範本來進行渲染。

## 2. 範本的存放位置與命名慣例

為了讓 MVC 框架能夠自動找到你的範本，它們必須放在特定的資料夾下：

*   `Views/Shared/DisplayTemplates/{TemplateName}.cshtml`
*   `Views/Shared/EditorTemplates/{TemplateName}.cshtml`
*   `Views/{ControllerName}/DisplayTemplates/{TemplateName}.cshtml`
*   `Views/{ControllerName}/EditorTemplates/{TemplateName}.cshtml`

通常，我們會將共用的範本放在 `Views/Shared` 下。

`{TemplateName}` 通常是**資料型別的名稱**。例如，對於 `DateTime` 型別，你可以建立一個名為 `DateTime.cshtml` 的範本。

## 3. 為特定資料型別建立範本

### 範例：自訂 `DateTime` 的編輯範本

預設情況下，`EditorFor` 一個 `DateTime` 屬性會產生一個 `type="datetime-local"` 的輸入框。假設我們希望網站上所有的日期選擇器都是 `type="date"`，並且有固定的格式。

**步驟 1：建立編輯範本**

在 `Views/Shared/EditorTemplates/` 資料夾下，建立一個新檔案 `DateTime.cshtml`。

**`Views/Shared/EditorTemplates/DateTime.cshtml`**
```html
@* 
  這個範本的模型 (Model) 就是傳入的 DateTime? 值。
  注意這裡我們使用的是 @Html 輔助方法，因為它們能更好地與範本系統整合。
*@
@model DateTime?

@* 
  Html.TextBoxFor 會產生 <input type="text">。
  第一個參數是空字串，代表繫結到模型本身。
  第二個參數是格式化字串。
  第三個參數是 HTML 屬性。
*@
@Html.TextBoxFor(
    m => m, 
    "{0:yyyy-MM-dd}", 
    new { @class = "form-control", type = "date" }
)
```

**步驟 2：在 ViewModel 中使用**

```csharp
public class EventViewModel
{
    [Display(Name = "活動名稱")]
    public string EventName { get; set; }

    [Display(Name = "開始日期")]
    public DateTime StartDate { get; set; }
}
```

**步驟 3：在 View 中呼叫**

```html
@model EventViewModel

<form asp-action="Create" method="post">
    <div class="form-group">
        <label asp-for="EventName"></label>
        <input asp-for="EventName" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="StartDate"></label>
        @* 
          當 MVC 看到要為 StartDate (一個 DateTime) 產生編輯器時，
          它會自動去 EditorTemplates 資料夾尋找 DateTime.cshtml
        *@
        @Html.EditorFor(m => m.StartDate)
        <span asp-validation-for="StartDate" class="text-danger"></span>
    </div>
    <button type="submit">建立</button>
</form>
```
現在，網站中所有使用 `@Html.EditorFor()` 來渲染 `DateTime` 屬性的地方，都會自動套用我們定義好的 `type="date"` 樣式，實現了全站 UI 的一致性。

## 4. 為複雜型別 (物件) 建立範本

範本檢視最強大的地方在於它可以處理複雜的物件。這對於像「地址」這樣可重用的資料結構特別有用。

### 範例：一個可重用的地址編輯範本

**步驟 1：定義地址 ViewModel**
```csharp
public class AddressViewModel
{
    [Display(Name = "國家")]
    public string Country { get; set; }
    [Display(Name = "城市")]
    public string City { get; set; }
    [Display(Name = "詳細地址")]
    public string Street { get; set; }
}
```

**步驟 2：建立地址的編輯範本**

**`Views/Shared/EditorTemplates/AddressViewModel.cshtml`**
```html
@model AddressViewModel

@* 
  這個範本的 @model 是 AddressViewModel。
  注意這裡的 asp-for 會自動產生正確的 name 屬性，
  例如 "ShippingAddress.Country", "BillingAddress.City"
  這取決於父模型中屬性的名稱。
*@
<div class="address-box p-3 border rounded">
    <div class="form-group">
        <label asp-for="Country"></label>
        <input asp-for="Country" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="City"></label>
        <input asp-for="City" class="form-control" />
    </div>
    <div class="form-group">
        <label asp-for="Street"></label>
        <input asp-for="Street" class="form-control" />
    </div>
</div>
```

**步驟 3：在父 ViewModel 中使用地址模型**
```csharp
public class OrderViewModel
{
    public int OrderId { get; set; }

    [Display(Name = "運送地址")]
    public AddressViewModel ShippingAddress { get; set; }

    [Display(Name = "帳單地址")]
    public AddressViewModel BillingAddress { get; set; }
}
```

**步驟 4：在父 View 中呼叫**
```html
@model OrderViewModel

<form asp-action="PlaceOrder" method="post">
    <h3>運送地址</h3>
    @Html.EditorFor(m => m.ShippingAddress)

    <hr />

    <h3>帳單地址</h3>
    @Html.EditorFor(m => m.BillingAddress)

    <button type="submit">下單</button>
</form>
```
現在，我們用一行 `@Html.EditorFor(m => m.ShippingAddress)` 就渲染出了一整組地址輸入欄位。如果未來需要修改地址表單的樣式，只需要修改 `AddressViewModel.cshtml` 這一個檔案即可。

## 5. 使用 `[UIHint]` 指定範本

有時候，你可能不想根據資料型別來選擇範本，而是想為某個特定的屬性指定一個特殊的範本。這可以透過 `[UIHint]` 屬性來實現。

**ViewModel:**
```csharp
public class ProductViewModel
{
    [Display(Name = "產品描述")]
    [UIHint("MarkdownEditor")] // 指定使用名為 "MarkdownEditor" 的範本
    public string Description { get; set; }
}
```
現在，當你呼叫 `@Html.EditorFor(m => m.Description)` 時，MVC 會去 `EditorTemplates` 資料夾尋找 `MarkdownEditor.cshtml` 這個檔案，而不是預設的 `String.cshtml`。這讓你能夠為富文本、星級評分等特殊 UI 元件建立可重用的範本。

## 結論

範本檢視是 ASP.NET Core MVC 中實現 DRY (Don't Repeat Yourself) 原則的利器。

1.  **按型別複用**：為基礎資料型別（如 `DateTime`, `bool`, `decimal`）建立範本，可以統一它們在整個網站中的外觀和行為。
2.  **按物件複用**：為可重用的複雜物件（如 `AddressViewModel`）建立範本，可以極大地簡化包含這些物件的父表單。
3.  **`@Html.EditorFor` 和 `@Html.DisplayFor`** 是啟用此機制的關鍵。
4.  **`[UIHint]`** 提供了一種覆寫預設範本選擇邏輯的靈活方式。

熟練運用範本檢視技巧，可以讓你的表單開發工作事半功倍，並建立出一個高度一致且易於維護的 Web 應用程式。
