# ASP.NET Core MVC 教學 (.NET 8)：自訂 Tag Helper 標籤輔助程式

雖然 ASP.NET Core 提供了許多強大的內建 Tag Helper，但在真實世界的專案中，我們經常會遇到需要重複產生特定 HTML 結構或行為的場景。這時，建立屬於我們自己的「自訂 Tag Helper」就成了一個非常有用的選擇。

自訂 Tag Helper 讓我們可以將可重用的 UI 邏輯封裝成一個乾淨、語意化的 HTML 標籤，從而簡化 View 的程式碼，並提高整個專案的一致性和可維護性。

## 1. 建立你的第一個 Tag Helper

建立一個 Tag Helper 本質上就是建立一個繼承自 `Microsoft.AspNetCore.Razor.TagHelpers.TagHelper` 的 C# 類別。

### 範例：一個簡單的電子郵件連結 Tag Helper

假設我們需要一個能自動產生 `mailto:` 連結的 Tag Helper。我們希望在 View 中可以這樣寫：
`<email-link address="support@example.com">聯繫客服</email-link>`

最終它應該被渲染成：
`<a href="mailto:support@example.com">聯繫客服</a>`

**步驟：**

1.  **建立 Tag Helper 類別**
    在專案中建立一個資料夾 (例如 `TagHelpers`)，並新增一個類別檔案 `EmailLinkTagHelper.cs`。

    ```csharp
    using Microsoft.AspNetCore.Razor.TagHelpers;

    // 預設情況下，類別名稱 "EmailLinkTagHelper" 會對應到 HTML 標籤 "email-link"
    // (PascalCase 會被轉換成 kebab-case)
    public class EmailLinkTagHelper : TagHelper
    {
        // 這是一個公開屬性，它會對應到 HTML 標籤的 "address" 屬性
        public string Address { get; set; }

        // 覆寫 Process 或 ProcessAsync 方法來處理標籤
        public override void Process(TagHelperContext context, TagHelperOutput output)
        {
            // 1. 設定要輸出的 HTML 標籤名稱
            output.TagName = "a";

            // 2. 設定標籤的屬性
            output.Attributes.SetAttribute("href", "mailto:" + Address);

            // 3. 內容會自動從 <email-link>...</email-link> 的內容傳遞過來
            // 如果需要，也可以手動設定內容：
            // output.Content.SetContent("點此聯繫");
        }
    }
    ```

2.  **註冊 Tag Helper**
    為了讓 MVC 能夠發現並使用你的自訂 Tag Helper，你需要在 `_ViewImports.cshtml` 中註冊它所在的組件 (Assembly)。

    **`_ViewImports.cshtml`**
    ```csharp
    @using MyWebApp
    @using MyWebApp.Models
    @addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
    @addTagHelper *, MyWebApp // "MyWebApp" 是你專案的組件名稱
    ```
    第二個 `@addTagHelper` 指令告訴 Razor 引擎去 `MyWebApp` 這個組件中尋找所有的 Tag Helper。

3.  **在 View 中使用**
    現在你就可以在任何 `.cshtml` 檔案中使用你的新標籤了！

    ```html
    <p>
        有任何問題嗎？ <email-link address="support@example.com">請隨時聯繫我們</email-link>。
    </p>
    ```

## 2. 進階技巧與屬性

### a. 控制目標標籤

預設情況下，`ClassNameTagHelper` 會對應到 `class-name` 標籤。但有時候我們不想建立新標籤，而是想為現有的 HTML 標籤（如 `<span>`, `<div>`）添加新功能。這可以透過 `[HtmlTargetElement]` 屬性來實現。

**範例：一個根據條件顯示/隱藏內容的 Tag Helper**

我們希望可以這樣寫：`<div asp-visible="false">這段內容會被隱藏</div>`

**`ConditionalDisplayTagHelper.cs`**
```csharp
using Microsoft.AspNetCore.Razor.TagHelpers;

// 指定這個 Tag Helper 只作用於具有 "asp-visible" 屬性的 <div> 標籤
[HtmlTargetElement("div", Attributes = "asp-visible")]
public class ConditionalDisplayTagHelper : TagHelper
{
    // 屬性名稱 "AspVisible" 會對應到 "asp-visible"
    public bool AspVisible { get; set; }

    public override void Process(TagHelperContext context, TagHelperOutput output)
    {
        if (!AspVisible)
        {
            // 如果條件為 false，抑制輸出，整個 <div> 標籤都不會被渲染
            output.SuppressOutput();
        }
    }
}
```

### b. 非同步處理

如果你的 Tag Helper 需要執行非同步操作（例如，從資料庫或快取中讀取資料），你應該覆寫 `ProcessAsync` 方法。

**範例：顯示目前線上人數的 Tag Helper**
```csharp
// 假設有一個 IOnlineUsersService 服務可以獲取線上人數
[HtmlTargetElement("online-users-count")]
public class OnlineUsersCountTagHelper(IOnlineUsersService usersService) : TagHelper
{
    public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
    {
        // 1. 設定標籤名稱，例如 <span>
        output.TagName = "span";
        output.TagMode = TagMode.StartTagAndEndTag; // 確保有開始和結束標籤

        // 2. 非同步獲取資料
        var count = await usersService.GetOnlineCountAsync();

        // 3. 設定內容
        output.Content.SetContent($"目前有 {count} 人在線");
    }
}
```
> **注意**：這個 Tag Helper 使用了建構函式注入 `IOnlineUsersService`。要讓 DI 生效，你需要在使用它的地方（例如 Controller）將其注入，或者將其註冊為 Scoped/Transient 服務。

### c. 處理子內容

`Process` 和 `ProcessAsync` 方法可以讓你存取和修改 Tag Helper 內部包裹的子內容。

**範例：一個將內容轉為大寫的 Tag Helper**
```html
<shout>這段文字會變大寫！</shout>
```

**`ShoutTagHelper.cs`**
```csharp
[HtmlTargetElement("shout")]
public class ShoutTagHelper : TagHelper
{
    public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
    {
        // 獲取原始的子內容
        var childContent = await output.GetChildContentAsync();
        var originalText = childContent.GetContent();

        // 設定標籤為 <strong>
        output.TagName = "strong";
        
        // 將處理過的內容放回去
        output.Content.SetContent(originalText.ToUpper());
    }
}
```

## 3. 為什麼要用自訂 Tag Helper？

1.  **簡化 View**：將複雜的、重複的 HTML 產生邏輯封裝起來，讓 View 保持乾淨、語意化。
2.  **提高可維護性**：當你需要修改某個共通 UI 元件的結構時，只需要修改一個 Tag Helper 類別，而不是幾十個 View 檔案。
3.  **可測試性**：Tag Helper 是標準的 C# 類別，可以對其進行單元測試，確保其行為符合預期。
4.  **伺服器端渲染**：與前端框架（如 Vue, React）的元件不同，Tag Helper 是在伺服器端執行的，它產生的就是最終的 HTML。這對於 SEO 和首屏加載速度是有益的。

## 結論

自訂 Tag Helper 是 ASP.NET Core MVC 提供的一個強大擴展點。它讓後端開發者能夠以一種對前端友善的方式，來建立可重用的 UI 元件。

*   **從簡單開始**：從一個簡單的需求（如郵件連結）開始練習。
*   **命名很重要**：為你的 Tag Helper 和它的屬性取一個有意義的、符合 HTML 風格的名稱。
*   **善用 `[HtmlTargetElement]`**：精確地控制你的 Tag Helper 要作用在哪種 HTML 標籤上。
*   **考慮非同步**：如果需要 I/O 操作，務必使用 `ProcessAsync`。
*   **不要濫用**：對於非常複雜的、有大量互動的 UI，使用前端元件框架可能仍然是更好的選擇。Tag Helper 最適合用來處理那些需要在伺服器端產生 HTML 的可重用 UI 片段。
