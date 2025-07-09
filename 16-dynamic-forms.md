# ASP.NET Core MVC 教學 (.NET 8)：動態表單開發技巧

在許多應用場景中，我們需要的表單並不是靜態的，而是需要使用者能夠動態地增加或刪除項目。例如，一個可以讓使用者新增多個電話號碼的聯絡人表單、一個可以動態增減商品的訂單，或是一個問卷調查中可以新增多個選項的問題。

處理這種動態集合的表單提交，關鍵在於如何產生正確的 HTML `name` 屬性，以便 ASP.NET Core 的模型繫結器 (Model Binder) 能夠在後端將這些零散的表單資料正確地重新組合成一個集合 (List 或 Array)。

## 1. 模型繫結器如何處理集合

要讓模型繫結器能夠識別一個集合，HTML `input` 元素的 `name` 屬性必須遵循一個特定的格式：
`CollectionPropertyName[index].PropertyName`

例如，假設我們有以下的 ViewModel：
```csharp
public class SurveyViewModel
{
    public string Title { get; set; }
    public List<QuestionOption> Options { get; set; } = new();
}

public class QuestionOption
{
    public int Id { get; set; }
    public string OptionText { get; set; }
}
```

為了讓模型繫結器能正確繫結 `Options` 這個 List，HTML 應該長得像這樣：
```html
<input type="text" name="Options[0].OptionText" value="選項A">
<input type="hidden" name="Options[0].Id" value="1">

<input type="text" name="Options[1].OptionText" value="選項B">
<input type="hidden" name="Options[1].Id" value="2">

...
```
**關鍵點**：
*   `Options` 是 ViewModel 中的集合屬性名稱。
*   `[0]`, `[1]` 是集合的索引。**這個索引必須是連續的、從 0 開始的整數**，否則模型繫結器可能會中斷處理。
*   `OptionText`, `Id` 是集合內物件的屬性名稱。

## 2. 使用 `for` 迴圈產生靜態集合表單

對於一個已存在的集合（例如，編輯一個已儲存的問卷），我們可以使用標準的 `for` 迴圈（而不是 `foreach`）來產生正確的 `name` 屬性。

**`EditSurvey.cshtml`**
```html
@model SurveyViewModel

<form asp-action="EditSurvey" method="post">
    <input asp-for="Title" class="form-control" />

    <div id="options-container">
        @for (var i = 0; i < Model.Options.Count; i++)
        {
            <div class="option-item">
                <label>選項 @(i + 1)</label>
                <input asp-for="Options[i].OptionText" class="form-control" />
                <input type="hidden" asp-for="Options[i].Id" />
            </div>
        }
    </div>
    
    <button type="submit">儲存</button>
</form>
```
使用 `for` 迴圈和索引 `i`，`asp-for` Tag Helper 會自動產生 `name="Options[0].OptionText"`, `name="Options[1].OptionText"` 這樣符合規範的 `name`。

## 3. 使用 JavaScript 實現動態新增

動態表單的挑戰在於，當使用者點擊「新增」按鈕時，我們需要用 JavaScript 在前端產生一組新的 `input` 欄位，並且它們的 `name` 屬性中的索引必須是正確的、不重複的。

**策略**：
1.  在頁面上建立一個 HTML「範本 (template)」，這個範本包含了一組新項目所需的所有 `input` 欄位。
2.  在範本中，使用一個特殊的預留位置 (placeholder) 來代替索引，例如 `__prefix__`。
3.  當使用者點擊「新增」時：
    a. 獲取當前容器中已有多少個項目，以確定新的索引值。
    b. 複製範本的 HTML。
    c. 將範本中的所有 `__prefix__` 預留位置替換為新的索引值。
    d. 將處理過後的 HTML 附加到容器中。

**`CreateSurvey.cshtml` (包含 JavaScript)**
```html
@model SurveyViewModel

<form asp-action="CreateSurvey" method="post">
    <input asp-for="Title" class="form-control" />

    <div id="options-container">
        @* 初始時可以顯示一個或多個預設選項 *@
    </div>

    <button type="button" id="add-option-btn" class="btn btn-secondary mt-2">新增選項</button>
    <hr />
    <button type="submit" class="btn btn-primary">建立問卷</button>
</form>

@* 
  這是一個隱藏的 HTML 範本，用於新增項目。
  注意 name 屬性中的 __prefix__ 預留位置。
*@
<div id="option-template" style="display: none;">
    <div class="option-item mb-2">
        <input type="text" name="Options[__prefix__].OptionText" class="form-control" placeholder="請輸入選項文字" />
        <button type="button" class="btn btn-danger btn-sm remove-option-btn">移除</button>
    </div>
</div>


@section Scripts {
    <script>
        $(document).ready(function () {
            $("#add-option-btn").click(function () {
                // 1. 獲取當前有多少個選項，以決定新索引
                var newIndex = $("#options-container .option-item").length;

                // 2. 複製範本
                var templateHtml = $("#option-template").html();

                // 3. 替換預留位置
                //    使用正則表達式 /__prefix__/g 來確保替換所有出現的地方
                var newRowHtml = templateHtml.replace(/__prefix__/g, newIndex);

                // 4. 將新的一行附加到容器中
                $("#options-container").append(newRowHtml);
            });

            // 處理移除按鈕的點擊事件 (使用事件委派)
            $("#options-container").on("click", ".remove-option-btn", function () {
                $(this).closest(".option-item").remove();
                // **重要**: 移除後需要重新索引，以確保索引是連續的
                updateOptionIndexes();
            });

            function updateOptionIndexes() {
                $("#options-container .option-item").each(function (index) {
                    $(this).find("input, select, textarea").each(function () {
                        // this.name 可能是 "Options[2].OptionText"
                        // 我們要將 [2] 更新為正確的 index
                        this.name = this.name.replace(/\[\d+\]/, "[" + index + "]");
                    });
                });
            }
        });
    </script>
}
```

## 4. 處理刪除與重新索引

從上面的 JavaScript 程式碼中可以看到，僅僅移除一個 HTML 元素是不夠的。如果我們移除了索引為 `[1]` 的項目，剩下的項目索引就會變成 `[0]`, `[2]`, `[3]`... 這是不連續的，會導致模型繫結失敗。

因此，在每次**移除**一個項目後，**必須**重新遍歷所有現存的項目，並更新它們 `name` 屬性中的索引，確保它們是從 0 開始的連續數字。`updateOptionIndexes()` 函式就是為此而生。

## 結論

開發動態集合表單需要前端 JavaScript 和後端模型繫結的緊密配合。

1.  **理解繫結格式**：牢記 `Collection[index].Property` 的 `name` 屬性格式是成功的關鍵。
2.  **使用 `for` 迴圈**：在編輯現有集合資料時，使用 `for` 迴圈（而非 `foreach`）來產生初始表單，以確保索引正確。
3.  **範本與預留位置**：在前端使用帶有預留位置（如 `__prefix__`）的 HTML 範本，是實現動態新增的標準做法。
4.  **新增時計算新索引**：根據當前已有項目的數量來決定新項目的索引。
5.  **刪除後重新索引**：在刪除任何一個項目後，必須用 JavaScript 重新整理所有剩餘項目的索引，以保持其連續性。

雖然這比靜態表單要複雜一些，但掌握了這個技巧，你就能夠應對各種複雜的資料輸入場景，提供更靈活、更強大的使用者體驗。
