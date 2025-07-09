# ASP.NET Core MVC 教學 (.NET 8)：深入理解 Razor 語法 (含速記法則)

Razor 是一種為在網頁中嵌入伺服器端程式碼而設計的標記語法。它是 ASP.NET Core MVC 的預設檢視引擎 (View Engine)，以其簡潔、流暢且易於學習的特性而聞名。Razor 的核心理念是讓 C# 程式碼和 HTML 標記能夠無縫地結合，同時保持檢視檔案的高度可讀性。

本文將帶你深入 Razor 的世界，從基礎語法到能顯著提升開發效率的速記法則。

## 1. Razor 的核心符號：`@`

`@` 符號是 Razor 的魔法棒，它負責啟動 Razor 解析器，告訴它「接下來是 C# 程式碼」。

### a. 顯式運算式 (Explicit Expressions)

當你需要明確地輸出來自一個運算式的結果時，使用 `@()`。

```html
<p>現在時間是: @(DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"))</p>
<p>您的會員等級是: @(user.IsAdmin ? "管理員" : "一般會員")</p>
<p>您的購物車內有 @(cart.Items.Count()) 件商品。</p>
```

### b. 隱式運算式 (Implicit Expressions)

對於簡單的變數或屬性存取，可以直接使用 `@`。Razor 足夠聰明，能夠識別變數的邊界。

```html
@{
    var product = new { Name = "無線鍵盤", Price = 1200 };
}

<h1>產品名稱: @product.Name</h1>
<p>價格: @product.Price 元</p>
```
**注意**：如果變數後面緊跟著沒有空格的文字，Razor 可能會混淆。這時需要使用顯式運算式。
```html
<!-- 錯誤的寫法 -->
<p>歡迎, @User.Name! 今天天氣很好。</p>

<!-- 正確的寫法 -->
<p>歡迎, @(User.Name)! 今天天氣很好。</p>
<p>歡迎, @User.Name! 今天天氣很好。</p> <!-- 或者在變數後加一個空格 -->
```

## 2. Razor 程式碼區塊 (Code Blocks)

當你需要編寫超過一行的 C# 程式碼時，使用 `@if`, `@for`, `@foreach`, `@while`, `@switch` 等控制結構，或者用 `@{ ... }` 包裹起來。

### a. 控制結構

Razor 的控制結構不需要用 `{}` 包裹 HTML 內容，它會自動識別標籤的範圍。

```html
@if (Model.Products.Any())
{
    <ul>
        @foreach (var product in Model.Products)
        {
            <li>@product.Name - @product.Price 元</li>
        }
    </ul>
}
else
{
    <p>目前沒有任何產品。</p>
}
```

### b. 通用程式碼區塊

使用 `@{ ... }` 來定義變數或執行不直接輸出 HTML 的複雜邏輯。

```html
@{
    var discountRate = 0.8;
    var finalPrice = Model.OriginalPrice * discountRate;
    var message = "今日特價!";
}

<p>@message</p>
<p>最終價格: @finalPrice</p>
```

## 3. Razor 速記法則與技巧

掌握以下技巧，能讓你的 Razor 程式碼更精簡。

### a. 條件式屬性 (Conditional Attributes)

當一個 HTML 屬性需要根據條件來決定是否輸出時，Razor 提供了非常優雅的寫法。如果 C# 運算式的結果是 `null` 或 `false`，Razor 將不會呈現該屬性。

```csharp
// Controller 或 ViewModel 中的變數
ViewData["IsDisabled"] = true;
ViewData["CssClass"] = null;
```

```html
<!-- 
  如果 ViewData["IsDisabled"] 是 true，會呈現 disabled="disabled"
  如果是 false，則整個 disabled 屬性都不會出現
-->
<button type="submit" disabled="@ViewData["IsDisabled"]">送出</button>

<!-- 
  如果 ViewData["CssClass"] 是 null，class 屬性將不會被呈現
  如果是有值的字串，則會呈現 class="some-class"
-->
<div class="@ViewData["CssClass"]">...</div>
```

### b. 隱式 `using`

在 `.NET 6` 之後，專案預設啟用隱式 `using`。對於 View，你可以在專案根目錄下的 `_ViewImports.cshtml` 檔案中定義全域的 `using` 指令，這樣就不需要在每個 View 檔案中重複引用命名空間。

**`_ViewImports.cshtml`**
```csharp
@using MyWebApp
@using MyWebApp.Models
@using MyWebApp.ViewModels
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```
這個檔案中定義的內容會自動應用到所有 View。

### c. 函式與輔助方法 (`@functions` / `@helper`)

雖然現在更推薦使用 View Components 或 Tag Helpers 來處理複雜的 UI 邏輯，但在 Razor Pages (`.cshtml`) 中，你仍然可以使用 `@functions` 區塊來定義頁面內的 C# 方法。

```html
@functions {
    public string FormatPrice(decimal price)
    {
        return price.ToString("C"); // "C" 代表貨幣格式
    }
}

<p>價格: @FormatPrice(Model.Price)</p>
```
> **注意**：在 MVC 的 View 中，`@functions` 的使用已不常見。更好的做法是將這種邏輯放在 ViewModel 或建立一個擴充方法。

### d. 處理電子郵件地址

Razor 特別處理了電子郵件地址。當 `@` 後面跟著的字串符合郵件格式時，它不會被當作程式碼來解析。

```html
<!-- Razor 會正確地將這整段視為純文字 -->
<a href="mailto:support@example.com">聯繫我們</a>
```

### e. 轉義 `@` 符號

如果你需要在 HTML 中顯示一個字面上的 `@` 符號，而不是用它來啟動 Razor，只需要使用兩個 `@` (`@@`) 即可。

```html
<!-- 輸出: My Twitter handle is @MyUsername -->
<p>My Twitter handle is @@MyUsername</p>
```

## 4. 註解 (Comments)

Razor 提供了伺服器端的註解語法 `@* ... *@`。這種註解**不會**被傳送到客戶端的瀏覽器，因此適合用來註解伺服器端邏輯或暫時移除程式碼。

```html
@*
  這裡是伺服器端註解。
  下面的這段 HTML 將不會被執行或傳送到瀏覽器。
  @if (false) {
      <p>這段不會出現</p>
  }
*@

<!-- 這是標準的 HTML 註解，它會被傳送到瀏覽器 -->
<div>
    <p>一些內容</p>
</div>
```

## 結論

Razor 語法以其優雅和高效著稱，是 ASP.NET Core 開發者必須熟練掌握的技能。

*   **`@` 是核心**：所有 Razor 魔法都始於 `@`。
*   **簡潔優先**：盡可能使用隱式運算式和控制結構，讓程式碼保持乾淨。
*   **善用速記**：利用條件式屬性等技巧來簡化 HTML 的產生。
*   **邏輯分離**：避免在 Razor 檔案中編寫過於複雜的業務邏輯。將它們移至 ViewModel、Controller、Tag Helper 或 View Component 中，讓 View 只專注於「呈現」。

精通 Razor，你就能夠以最少的程式碼，創造出最富動態性的網頁。
