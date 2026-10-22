---
description: "Thao tác với các loại phần tử trên trang, viết assertion kiểm tra kết quả và hiểu cơ chế auto-wait thay thế cho thời gian chờ cố định."
icon: hand-pointer
---

# Buổi 6 · Actions, Assertions & Auto-wait

{% hint style="info" %}
**Sau buổi này bạn sẽ:**

* Thao tác được với các loại phần tử thường gặp: nút bấm, ô nhập, dropdown, checkbox, radio, upload file, kéo thả, iframe, popup.
* Chọn đúng action theo loại phần tử (native HTML element hay custom component).
* Viết assertion đúng loại, đúng chỗ, và biết khi nào dùng soft assertion.
* Hiểu cơ chế auto-wait và vì sao test Playwright không cần thời gian chờ cố định.
* Biết các hàm wait của Playwright và chọn đúng cách chờ cho từng tình huống.
{% endhint %}

## 1. Actions: bộ thao tác của Playwright

Một test gồm chuỗi **thao tác** (actions) và các điểm **kiểm tra** (assertions, trình bày ở mục 2). Phần lớn action được gọi trên locator (đã học ở [Buổi 5](buoi-05-locators.md)). Mọi action đều trả về Promise nên luôn cần `await` (Promise đã học ở [Buổi 2](../phase-1/buoi-02-git-async-await.md), mục 2).

Trước khi thực hiện action, Playwright tự chờ phần tử đáp ứng các điều kiện dưới đây, gọi là **actionability checks**. Nếu hết thời gian chờ mà phần tử vẫn chưa đạt, test fail kèm thông báo điều kiện nào chưa thỏa mãn.

| Điều kiện       | Phần tử đạt khi                                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------------------------------- |
| Visible         | Hiển thị trên trang: có kích thước lớn hơn 0 và không bị ẩn bằng CSS (`display: none`, `visibility: hidden`)      |
| Stable          | Không còn di chuyển, ví dụ menu đã trượt xuống xong, không còn đang chạy hiệu ứng                                 |
| Receives events | Không có phần tử khác nằm đè lên tại vị trí click, ví dụ popup, lớp loading phủ lên trang                         |
| Enabled         | Không có thuộc tính `disabled`, ví dụ nút "Submit" bị làm mờ, không bấm được khi form chưa nhập đủ                |
| Editable        | Ô nhập cho phép sửa: enabled và không có thuộc tính `readonly`. Chỉ áp dụng cho các action nhập liệu như `fill()` |

Mỗi action chỉ kiểm tra những điều kiện cần cho nó, ví dụ `click()` không kiểm tra Editable. Ngoài ra, mọi action đều tự scroll phần tử vào vùng nhìn thấy trước khi thao tác. Danh sách chi tiết theo từng action: [Actionability checks](https://playwright.dev/docs/actionability).

### 1.1 Click, double click, hover

`click()` chờ phần tử vượt qua actionability checks rồi mới click, nên không cần chờ thủ công trước action.

```typescript
test("mở trang chi tiết sản phẩm", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");

  await page.getByRole("heading", { name: "Combination Pliers" }).click();
});
```

`dblclick()` dùng khi chức năng chỉ chạy với double click, ví dụ double click vào một ô trong bảng để chuyển ô đó sang chế độ sửa:

```typescript
test("double click vào nút", async ({ page }) => {
  await page.goto("https://demoqa.com/buttons");

  await page.getByRole("button", { name: "Double Click Me" }).dblclick();
  await expect(page.getByText("You have done a double click")).toBeVisible();
});
```

`hover()` đặt con trỏ chuột lên phần tử mà không click. Dùng khi nội dung chỉ hiển thị lúc rê chuột lên phần tử: menu con, tooltip, chú thích ảnh.

```typescript
test("hover hiện thông tin người dùng", async ({ page }) => {
  await page.goto("https://the-internet.herokuapp.com/hovers");
  const firstAvatar = page.locator(".figure").first();

  // Trước khi hover, chú thích đang bị ẩn
  await expect(firstAvatar.getByText("name: user1")).toBeHidden();
  await firstAvatar.hover();
  await expect(firstAvatar.getByText("name: user1")).toBeVisible();
});
```

{% hint style="warning" %}
**Không dùng `{ force: true }` để vượt lỗi click.** Tùy chọn này bỏ qua actionability checks. Test có thể pass trong khi người dùng thật không bấm được phần tử (ví dụ phần tử đang bị popup che). Khi gặp lỗi `element intercepts pointer events`, hãy tìm và xử lý phần tử đang che (xem mục 1.8).
{% endhint %}

**Tham khảo:** [Mouse click](https://playwright.dev/docs/input#mouse-click), [locator.click()](https://playwright.dev/docs/api/class-locator#locator-click), [locator.dblclick()](https://playwright.dev/docs/api/class-locator#locator-dblclick), [locator.hover()](https://playwright.dev/docs/api/class-locator#locator-hover)

### 1.2 Dropdown

Dropdown có hai dạng, mỗi dạng dùng một cách thao tác riêng:

| Dạng            | Cấu tạo                                                                                                                                                         | Action                                     |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Native dropdown | Thẻ `<select>` do HTML định nghĩa sẵn, option đang chọn do trình duyệt quản lý                                                                                  | `selectOption()`                           |
| Custom dropdown | Do UI library (Bootstrap, Material UI, Ant Design...) dựng từ `<button>`, `<div>`, `<ul>`/`<li>`, trạng thái do JavaScript quản lý. [Buổi 5](buoi-05-locators.md), mục 5 gọi đây là thẻ "giả" | `click()` mở danh sách, `click()` chọn mục |

`selectOption()` chỉ hoạt động với thẻ `<select>`, kể cả khi custom dropdown có role `combobox` hoặc `listbox`. Để xác định dạng dropdown, xem tên thẻ ở tab Elements của DevTools ([Buổi 5](buoi-05-locators.md), mục 2).

#### Native dropdown: `selectOption()`

Dropdown "Sort" ở trang chủ và dropdown "Subject" ở trang Contact của Toolshop đều là thẻ `<select>`:

```html
<select data-test="sort" aria-label="sort" class="form-select">
  <option value=""></option>
  <option value="name,asc">Name (A - Z)</option>
  <option value="name,desc">Name (Z - A)</option>
  <option value="price,asc">Price (Low - High)</option>
  ...
</select>
```

`selectOption()` chọn option theo một trong các cách sau:

* Theo `value`: `selectOption("price,asc")`.
* Theo chữ hiển thị: `selectOption({ label: "Price (Low - High)" })`.
* Theo vị trí (bắt đầu từ 0): `selectOption({ index: 3 })`. Hạn chế dùng cách này: khi dev thêm hoặc đổi thứ tự option, test chọn sai option mà không báo lỗi (cùng lý do tránh XPath phụ thuộc cấu trúc ở [Buổi 5](buoi-05-locators.md), mục 8).
* Chọn nhiều option cùng lúc, chỉ dùng được với `<select multiple>`: truyền mảng, ví dụ `selectOption(["red", "green"])`. Các option không có trong mảng sẽ bị bỏ chọn.

```typescript
test("sắp xếp sản phẩm bằng dropdown Sort", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");
  const sortDropdown = page.getByRole("combobox", { name: "sort" });

  await sortDropdown.selectOption("price,asc");
  await expect(sortDropdown).toHaveValue("price,asc");

  // Chọn theo chữ hiển thị, assertion vẫn so sánh với value
  await sortDropdown.selectOption({ label: "Name (Z - A)" });
  await expect(sortDropdown).toHaveValue("name,desc");
});
```

{% hint style="info" %}
Ưu tiên chọn theo `label` khi chữ hiển thị là thứ người dùng nhìn thấy và tester cần kiểm tra. Chọn theo `value` khi chữ hiển thị thay đổi theo ngôn ngữ giao diện, vì `value` thường giữ nguyên.
{% endhint %}

#### Custom dropdown: `click()`

Nút chọn ngôn ngữ (biểu tượng quả địa cầu, chữ "EN") trên menu Toolshop là một Bootstrap dropdown: thẻ `<button>` điều khiển danh sách `<ul role="menu">`, không có thẻ `<select>`:

```html
<button id="language" data-test="language-select" aria-label="Select language" data-bs-toggle="dropdown">EN</button>
<ul role="menu" class="dropdown-menu">
  <li role="menuitem"><a data-test="lang-de" class="dropdown-item">DE</a></li>
  <li role="menuitem"><a data-test="lang-en" class="dropdown-item">EN</a></li>
  ...
</ul>
```

<!-- TODO(image): trang chủ https://practicesoftwaretesting.com, menu trên cùng sau khi click nút chọn ngôn ngữ (biểu tượng quả địa cầu, chữ "EN"), danh sách ngôn ngữ (DE, EN...) đang mở. Khoanh đỏ nút "EN" và danh sách ngôn ngữ, đánh dấu mục "DE". -->

Gọi `selectOption()` trên nút này sẽ nhận lỗi `Element is not a <select> element`. Thao tác đúng gồm hai lần `click()`: mở menu, sau đó chọn `menuitem`:

```typescript
test("đổi ngôn ngữ giao diện sang tiếng Đức", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");
  const languageButton = page.getByRole("button", { name: "Select language" });

  // Bước 1: mở danh sách. Bước 2: chọn mục cần thiết
  await languageButton.click();
  await page.getByRole("menuitem", { name: "DE" }).click();

  await expect(languageButton).toContainText("DE");
});
```

**Tham khảo:** [Select options](https://playwright.dev/docs/input#select-options), [locator.selectOption()](https://playwright.dev/docs/api/class-locator#locator-select-option)

### 1.3 Checkbox và radio

Checkbox và radio có hai dạng, mỗi dạng dùng một cách thao tác riêng:

| Dạng                   | Cấu tạo                                                                                                     | Action                                 |
| ---------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| Native checkbox, radio | Thẻ `<input type="checkbox">`, `<input type="radio">`, trạng thái `checked` do trình duyệt quản lý          | `check()`, `uncheck()`, `setChecked()` |
| Custom checkbox        | Do UI library dựng từ `<div>`, `<span>`, trạng thái do JavaScript quản lý                                   | Có ARIA: `check()`, `uncheck()`. Không có ARIA: `click()` |

Gọi `check()`, `uncheck()` trên phần tử không thuộc các dạng trên (`<div>` thường, `<button>`, ô nhập text...) sẽ báo lỗi `Not a checkbox or radio button`.

#### Native checkbox, radio: `check()` và `uncheck()`

Ví dụ với bộ lọc danh mục ở trang chủ Toolshop:

```typescript
test("lọc sản phẩm theo category Hammer", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");
  const hammerFilter = page.getByRole("checkbox", { name: "Hammer" });

  await hammerFilter.check();
  // Gọi lại lần nữa vẫn an toàn: checkbox đã được tick nên Playwright bỏ qua
  await hammerFilter.check();
  await expect(hammerFilter).toBeChecked();
  await expect(page.locator("[data-test='product-name']").first()).toContainText("Hammer");

  await hammerFilter.uncheck();
  await expect(hammerFilter).not.toBeChecked();
});
```

Khi trạng thái cần đặt nằm trong một biến boolean (ví dụ đọc từ dữ liệu test), dùng `setChecked()` thay cho câu lệnh `if`:

```typescript
const onlyEcoFriendly = true;
await page.getByRole("checkbox", { name: "Show only eco-friendly products" }).setChecked(onlyEcoFriendly);
```

| Hàm                 | Kết quả                                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| `check()`           | Phần tử luôn ở trạng thái được chọn sau khi gọi                                                            |
| `uncheck()`         | Phần tử luôn ở trạng thái không được chọn sau khi gọi. Chỉ dùng cho checkbox                               |
| `setChecked(value)` | Đặt trạng thái theo giá trị `true` hoặc `false`                                                            |
| `click()`           | Chỉ đảo trạng thái. Click hai lần sẽ quay về trạng thái ban đầu, nên kết quả phụ thuộc trạng thái trước đó |

Radio không bỏ chọn được bằng click, nên `uncheck()` trên radio báo lỗi `Cannot uncheck radio button`. Để đổi lựa chọn, `check()` một radio khác trong cùng nhóm:

```typescript
test("chọn radio Impressive", async ({ page }) => {
  await page.goto("https://demoqa.com/radio-button");

  await page.getByLabel("Yes").check();
  // Đổi lựa chọn bằng cách check radio khác, không dùng uncheck()
  await page.getByLabel("Impressive").check();

  await expect(page.getByLabel("Yes")).not.toBeChecked();
  await expect(page.locator(".text-success")).toHaveText("Impressive");
});
```

{% hint style="warning" %}
**Native input bị ẩn:** nhiều UI library ẩn thẻ `<input>` (ví dụ bằng `display: none`) và hiển thị một phần tử thay thế được style bằng CSS. Gọi `check()` trên input bị ẩn sẽ hết thời gian chờ vì phần tử không đạt điều kiện Visible (bảng actionability checks ở đầu mục 1). Cách xử lý: gọi `check()` trên `<label>` gắn với input đó, ví dụ `page.getByText("Show only eco-friendly products").check()`. Không dùng `{ force: true }` để vượt qua lỗi này.
{% endhint %}

#### Custom checkbox: `check()` hoặc `click()`

Cách thao tác phụ thuộc vào việc dev có khai báo thuộc tính ARIA hay không. ARIA là nhóm thuộc tính `role` và `aria-*` (ví dụ `aria-label` đã gặp ở [Buổi 5](buoi-05-locators.md), mục 4), mô tả vai trò và trạng thái của phần tử. Role và trạng thái hiển thị ở tab Accessibility của DevTools.

**Có ARIA (`role="checkbox"` kèm `aria-checked`): dùng `check()`, `uncheck()`.** Cây thư mục ở trang [DemoQA Check Box](https://demoqa.com/checkbox) dựng checkbox bằng thẻ `<span>`:

```html
<span class="rc-tree-checkbox" role="checkbox" aria-checked="false" aria-label="Select Home"></span>
```

<!-- TODO(image): Chrome DevTools trên trang https://demoqa.com/checkbox, tab Elements đang chọn thẻ <span class="rc-tree-checkbox" role="checkbox"> của thư mục Home, khung bên phải mở tab Accessibility. Khoanh đỏ các dòng Name: "Select Home", Role: checkbox và Checked: false trong mục Computed Properties. -->

```typescript
test("tick thư mục Home trong cây thư mục", async ({ page }) => {
  await page.goto("https://demoqa.com/checkbox");
  const homeCheckbox = page.getByRole("checkbox", { name: "Select Home" });

  await homeCheckbox.check();
  await expect(homeCheckbox).toBeChecked();
});
```

**Không có ARIA: dùng `click()`.** Trạng thái chỉ thể hiện qua class CSS, ví dụ toggle sau có class `toggle-on` khi bật:

```html
<div class="toggle toggle-on" data-test="newsletter-toggle"><span class="toggle-knob"></span></div>
```

```typescript
const newsletterToggle = page.locator("[data-test='newsletter-toggle']");
await newsletterToggle.click();
await expect(newsletterToggle).toHaveClass(/toggle-on/);
```

Vì `click()` chỉ đảo trạng thái, hãy kiểm tra trạng thái ban đầu trước khi click nếu test phụ thuộc vào nó.

**Tham khảo:** [Checkboxes and radio buttons](https://playwright.dev/docs/input#checkboxes-and-radio-buttons), [locator.check()](https://playwright.dev/docs/api/class-locator#locator-check), [locator.uncheck()](https://playwright.dev/docs/api/class-locator#locator-uncheck), [locator.setChecked()](https://playwright.dev/docs/api/class-locator#locator-set-checked)

### 1.4 Upload file

`setInputFiles(path)` gán file cho thẻ `<input type="file">` mà không cần mở hộp thoại chọn file của hệ điều hành. Đường dẫn tương đối được tính từ thư mục chạy lệnh `npx playwright test` (thư mục gốc của project).

Form Contact của Toolshop có ô Attachment chỉ nhận file `.txt` có dung lượng 0 KB. Tạo file rỗng `data/empty-note.txt` trong project, sau đó chạy:

```typescript
test("đính kèm file vào form Contact", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/contact");
  const attachment = page.getByLabel("Attachment");

  await attachment.setInputFiles("data/empty-note.txt");
  await expect(attachment).toHaveValue(/empty-note\.txt/);

  // Truyền mảng rỗng để bỏ file đã chọn
  await attachment.setInputFiles([]);
  await expect(attachment).toHaveValue("");
});
```

Với input có thuộc tính `multiple`, truyền mảng đường dẫn để chọn nhiều file cùng lúc: `setInputFiles(["data/a.txt", "data/b.txt"])`.

#### Ngoại lệ: không có `<input type="file">` để trỏ tới

`setInputFiles()` vẫn hoạt động khi `<input type="file">` bị ẩn bằng CSS, miễn là thẻ đó có trong HTML. Một số trang chỉ tạo `<input type="file">` bằng JavaScript tại thời điểm người dùng bấm nút "Upload", nên trước đó không có phần tử để `setInputFiles()` trỏ tới. Khi đó, chờ sự kiện `filechooser` (phát sinh khi trang mở hộp thoại chọn file) rồi gán file:

```typescript
// Bắt đầu chờ trước khi click, chưa await ở bước này
const fileChooserPromise = page.waitForEvent("filechooser");
await page.getByRole("button", { name: "Upload" }).click();
const fileChooser = await fileChooserPromise;
await fileChooser.setFiles("data/empty-note.txt");
```

**Tham khảo:** [Upload files](https://playwright.dev/docs/input#upload-files), [locator.setInputFiles()](https://playwright.dev/docs/api/class-locator#locator-set-input-files)

### 1.5 Scroll

Mọi action đều tự scroll phần tử vào vùng nhìn thấy trước khi thao tác. Vì vậy trong phần lớn trường hợp, test không cần lệnh scroll riêng.

Scroll thủ công chỉ cần trong các trường hợp sau:

* Kiểm tra phần tử có nằm trong vùng nhìn thấy (viewport) hay không.
* Trang chỉ tải thêm nội dung khi người dùng cuộn xuống (lazy load, infinite scroll).
* Chụp screenshot tại một vị trí cụ thể của trang.

Hai cách scroll:

* `locator.scrollIntoViewIfNeeded()`: cuộn trang cho đến khi phần tử nằm trong viewport.
* `page.mouse.wheel(deltaX, deltaY)`: cuộn trang theo số pixel, tương đương lăn chuột. `deltaY` dương là cuộn xuống, âm là cuộn lên.

```typescript
test("cuộn xuống footer", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");
  const footerTitle = page.getByText("Learn & Explore");

  // Footer có trong HTML nhưng nằm ngoài viewport khi trang vừa mở
  await expect(footerTitle).not.toBeInViewport();
  await footerTitle.scrollIntoViewIfNeeded();
  await expect(footerTitle).toBeInViewport();
});
```

Với trang infinite scroll, cuộn bằng bánh xe chuột rồi chờ nội dung mới bằng assertion:

```typescript
// Cuộn xuống 1000 pixel theo chiều dọc
await page.mouse.wheel(0, 1000);
```

{% hint style="info" %}
`toBeVisible()` chỉ kiểm tra phần tử có hiển thị trên trang, không quan tâm phần tử đang ở trong hay ngoài viewport. Khi cần kiểm tra vị trí trên màn hình, dùng `toBeInViewport()`.
{% endhint %}

**Tham khảo:** [Scrolling](https://playwright.dev/docs/input#scrolling), [locator.scrollIntoViewIfNeeded()](https://playwright.dev/docs/api/class-locator#locator-scroll-into-view-if-needed), [mouse.wheel()](https://playwright.dev/docs/api/class-mouse#mouse-wheel)

### 1.6 Kéo thả (drag and drop)

`dragTo(target)` thực hiện đủ chuỗi thao tác: di chuột tới phần tử nguồn, nhấn giữ, di chuyển tới phần tử đích, thả chuột.

```typescript
test("đổi chỗ hai cột bằng kéo thả", async ({ page }) => {
  await page.goto("https://the-internet.herokuapp.com/drag_and_drop");

  await page.locator("#column-a").dragTo(page.locator("#column-b"));

  await expect(page.locator("#column-a")).toHaveText("B");
  await expect(page.locator("#column-b")).toHaveText("A");
});
```

**Tham khảo:** [Drag and drop](https://playwright.dev/docs/input#drag-and-drop), [locator.dragTo()](https://playwright.dev/docs/api/class-locator#locator-drag-to)

### 1.7 Iframe

Thẻ `<iframe>` nhúng một trang web khác vào bên trong trang hiện tại, thường gặp với trình soạn thảo văn bản, khung thanh toán, video, bản đồ. Nội dung trong iframe là một tài liệu HTML riêng, nên locator bắt đầu từ `page` **không nhìn thấy** các phần tử bên trong iframe.

Để thao tác trong iframe, trước hết trỏ tới thẻ `<iframe>`, sau đó gọi `contentFrame()` để đi vào nội dung bên trong:

```html
<iframe id="frame1" src="/sample.html"></iframe>
<!-- Nội dung của /sample.html: <h1 id="sampleHeading">This is a sample page</h1> -->
```

<!-- TODO(image): Chrome DevTools tab Elements trên trang https://demoqa.com/frames, thẻ <iframe id="frame1"> đang được mở rộng, bên dưới là nút #document chứa <html> riêng và thẻ <h1 id="sampleHeading">. Khoanh đỏ dòng #document để thể hiện iframe là một tài liệu HTML riêng. -->

```typescript
test("đọc tiêu đề bên trong iframe", async ({ page }) => {
  await page.goto("https://demoqa.com/frames");

  // Locator từ page không thấy phần tử nằm trong iframe
  await expect(page.getByRole("heading", { name: "This is a sample page" })).toHaveCount(0);

  const sampleFrame = page.locator("#frame1").contentFrame();
  await expect(sampleFrame.getByRole("heading", { name: "This is a sample page" })).toBeVisible();
});
```

Sau `contentFrame()`, mọi hàm locator (`getByRole`, `getByLabel`, `locator`...) và action dùng như trên `page`. Một số lưu ý:

* Cách viết tương đương: `page.frameLocator("#frame1")`. Tài liệu Playwright hiện ưu tiên `locator(...).contentFrame()`.
* Không cần chờ iframe tải xong. Locator bên trong iframe vẫn có auto-wait như bình thường.

**Tham khảo:** [Frames](https://playwright.dev/docs/frames), [locator.contentFrame()](https://playwright.dev/docs/api/class-locator#locator-content-frame), [FrameLocator](https://playwright.dev/docs/api/class-framelocator)

### 1.8 Popup

"Popup" là tên gọi chung cho ba loại khác nhau, mỗi loại xử lý theo cách riêng:

| Loại                                          | Bản chất                                                                           | Cách xử lý              |
| --------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------- |
| Hộp thoại JavaScript (alert, confirm, prompt) | Do trình duyệt hiển thị, không nằm trong HTML của trang nên locator không tìm được | Sự kiện `dialog`        |
| Tab hoặc cửa sổ mới                           | Mở khi click link có `target="_blank"` hoặc khi trang gọi `window.open()`          | Sự kiện `popup`         |
| Modal HTML                                    | Các thẻ HTML của trang (thường là `<div>`), hiển thị đè lên nội dung trang         | Locator như bình thường |

#### Hộp thoại JavaScript: sự kiện `dialog`

Mặc định Playwright **tự động bấm Cancel** (dismiss) mọi hộp thoại. Muốn bấm OK, nhập dữ liệu vào prompt, hoặc kiểm tra nội dung hộp thoại, hãy đăng ký xử lý sự kiện `dialog` **trước** khi thực hiện action làm hộp thoại xuất hiện:

<!-- TODO(image): trang https://the-internet.herokuapp.com/javascript_alerts sau khi click nút "Click for JS Confirm", hộp thoại confirm của Chrome đang mở với nội dung "I am a JS Confirm" và hai nút OK, Cancel. Khoanh đỏ hộp thoại để học viên nhận diện hộp thoại do trình duyệt hiển thị. -->

```typescript
test("bấm OK trên hộp thoại confirm", async ({ page }) => {
  await page.goto("https://the-internet.herokuapp.com/javascript_alerts");

  // Đăng ký trước khi click, vì hộp thoại xuất hiện ngay trong lúc click
  page.once("dialog", async (dialog) => {
    expect(dialog.message()).toBe("I am a JS Confirm");
    await dialog.accept();
  });
  await page.getByRole("button", { name: "Click for JS Confirm" }).click();

  await expect(page.locator("#result")).toHaveText("You clicked: Ok");
});
```

Tham số thứ hai của `page.once()` là một arrow function ([Buổi 1](../phase-1/buoi-01-setup-typescript.md), mục 7), Playwright gọi hàm này khi hộp thoại xuất hiện. Các hàm của đối tượng `dialog`:

* `dialog.accept()`: bấm OK. Với prompt, truyền thêm chữ cần nhập: `dialog.accept("Playwright")`.
* `dialog.dismiss()`: bấm Cancel.
* `dialog.message()`: đọc nội dung hộp thoại.

`page.once()` chỉ xử lý hộp thoại xuất hiện đầu tiên. `page.on()` xử lý mọi hộp thoại cho đến hết test.

{% hint style="warning" %}
Hàm xử lý đã đăng ký **bắt buộc** gọi `accept()` hoặc `dismiss()`. Nếu không, trang bị treo vì hộp thoại vẫn đang mở, và action gây ra hộp thoại không bao giờ hoàn tất.
{% endhint %}

#### Tab hoặc cửa sổ mới: sự kiện `popup`

Link "GitHub" ở footer Toolshop có thuộc tính `target="_blank"`, nên mở trang GitHub trong tab mới. Biến `page` vẫn trỏ vào tab cũ. Để làm việc với tab mới, hãy chờ sự kiện `popup`:

```typescript
test("mở link GitHub ở tab mới", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");

  // Bắt đầu chờ trước khi click, chưa await ở bước này
  const popupPromise = page.waitForEvent("popup");
  await page.getByRole("link", { name: /GitHub/ }).click();
  const githubPage = await popupPromise;

  await expect(githubPage).toHaveURL(/github\.com\/testsmith-io\/practice-software-testing/);
  await githubPage.close();

  // Tab gốc không thay đổi
  await expect(page).toHaveURL("https://practicesoftwaretesting.com/");
});
```

Thứ tự ba dòng giữa là bắt buộc. Nếu viết `await page.waitForEvent("popup")` sau khi click, tab mới có thể đã mở xong trước khi bắt đầu chờ, và test bị treo đến hết thời gian chờ. Dòng đầu tạo Promise (khái niệm Promise đã học ở [Buổi 2](../phase-1/buoi-02-git-async-await.md), mục 2) và bắt đầu lắng nghe sự kiện, dòng thứ ba mới `await` để lấy tab mới. Đối tượng `githubPage` dùng được mọi hàm như `page`.

#### Modal HTML: locator như phần tử bình thường

Modal HTML là một phần của trang, nên tìm và thao tác bằng locator như mọi phần tử khác. Trang The Internet hiển thị một modal quảng cáo khi vừa mở:

<!-- TODO(image): trang https://the-internet.herokuapp.com/entry_ad ngay sau khi tải, modal "This is a modal window" đang hiển thị đè lên trang, bên cạnh mở Chrome DevTools tab Elements đang chọn thẻ <div id="modal">. Khoanh đỏ modal trên trang và dòng <div id="modal"> trong DevTools để thể hiện modal là thẻ HTML của trang. -->

```typescript
test("đóng modal quảng cáo", async ({ page }) => {
  await page.goto("https://the-internet.herokuapp.com/entry_ad");
  const modal = page.locator("#modal");

  await expect(modal.getByText("This is a modal window")).toBeVisible();
  await modal.getByText("Close", { exact: true }).click();
  await expect(modal).toBeHidden();
});
```

Với modal xây dựng đúng chuẩn (có `role="dialog"`), có thể dùng `page.getByRole("dialog")`.

{% hint style="info" %}
**Popup xuất hiện ngẫu nhiên** (banner cookie, khảo sát, quảng cáo) làm test lúc pass lúc fail vì có lúc che mất phần tử cần click. Playwright cung cấp `page.addLocatorHandler()` để đăng ký trước cách đóng popup, áp dụng bất cứ khi nào popup đó xuất hiện. Chi tiết: [page.addLocatorHandler()](https://playwright.dev/docs/api/class-page#page-add-locator-handler).
{% endhint %}

**Tham khảo:** [Dialogs](https://playwright.dev/docs/dialogs), [Handling popups](https://playwright.dev/docs/pages#handling-popups), [page.waitForEvent()](https://playwright.dev/docs/api/class-page#page-wait-for-event)

### 1.9 Nhập liệu và bàn phím

`fill(text)` xóa toàn bộ nội dung cũ của ô nhập rồi điền giá trị mới trong một lần. Dùng được với `<input>`, `<textarea>` và phần tử có thuộc tính `contenteditable`.

`press(key)` nhấn một phím hoặc tổ hợp phím: `"Enter"`, `"Tab"`, `"Escape"`, `"ArrowDown"`, `"Control+A"`, `"Shift+Tab"`...

```typescript
test("tìm kiếm sản phẩm bằng phím Enter", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");
  const searchBox = page.getByPlaceholder("Search");

  await searchBox.fill("Pliers");
  await expect(searchBox).toHaveValue("Pliers");

  await searchBox.press("Enter");
  await expect(page.locator("[data-test='search-caption']")).toContainText("Pliers");
});
```

Các hàm liên quan:

* `clear()`: xóa nội dung ô nhập, tương đương `fill("")`.
* `pressSequentially(text)`: nhập từng ký tự một, mỗi ký tự là một lần nhấn phím, giống người dùng gõ bàn phím.

`fill()` điền cả chuỗi vào ô nhập trong một lần, không có lần nhấn phím nào. Phần lớn trang chỉ quan tâm giá trị cuối cùng của ô nhập, nên `fill()` là đủ và nhanh hơn. Chỉ dùng `pressSequentially()` khi trang phản ứng theo từng phím được nhấn, ví dụ ô tìm kiếm hiện danh sách gợi ý trong lúc người dùng đang gõ: với `fill()`, ô nhập có chữ nhưng danh sách gợi ý không xuất hiện.

**Tham khảo:** [Text input](https://playwright.dev/docs/input#text-input), [Type characters](https://playwright.dev/docs/input#type-characters), [Keys and shortcuts](https://playwright.dev/docs/input#keys-and-shortcuts), [locator.fill()](https://playwright.dev/docs/api/class-locator#locator-fill), [locator.pressSequentially()](https://playwright.dev/docs/api/class-locator#locator-press-sequentially)

### 1.10 Timeout của action

Có thể đặt timeout riêng cho action theo bảng dưới đây (đơn vị mili giây). Nếu không đặt, action chờ tối đa bằng timeout của cả test, mặc định 30 giây.

| Phạm vi                        | Cách đặt                                                                                                          |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| Một action                     | `await page.getByRole("button", { name: "Login" }).click({ timeout: 10000 });`                                    |
| Một lần chuyển trang           | `await page.goto("https://practicesoftwaretesting.com/", { timeout: 15000 });`                                    |
| Mọi action trong project       | `use: { actionTimeout: 10000, navigationTimeout: 15000 }` trong `playwright.config.ts` ([Buổi 9](../phase-3/buoi-09-config-fixtures-parallel.md), mục 2) |

**Tham khảo:** [Timeouts](https://playwright.dev/docs/test-timeouts)

## 2. Assertions: kiểm tra kết quả của test

Action chỉ mô phỏng thao tác của người dùng. Assertion kiểm tra kết quả có đúng như mong đợi hay không: assertion sai thì test fail. Một test không có assertion thì không phát hiện được lỗi nào.

### 2.1 Cú pháp và hai loại assertion

Assertion có dạng `expect(đối tượng).matcher(giá trị mong đợi)`:

| Thành phần          | Ý nghĩa                                                                 |
| ------------------- | ----------------------------------------------------------------------- |
| `expect(đối tượng)` | Thứ cần kiểm tra: một locator, `page`, hoặc một giá trị (chuỗi, số...) |
| `.matcher(...)`     | Điều kiện cần kiểm tra, ví dụ `toBeVisible()`, `toHaveText("1")`        |
| `.not`              | Đảo ngược điều kiện: kiểm tra điều kiện **không** xảy ra                |

Tùy đối tượng truyền vào `expect()`, assertion chia thành hai loại:

| Loại                   | Đối tượng                     | Cách kiểm tra                                                        | `await`  | Ví dụ                                                |
| ---------------------- | ----------------------------- | -------------------------------------------------------------------- | -------- | ---------------------------------------------------- |
| Web-first assertion    | Locator, `page`               | Kiểm tra lặp lại đến khi đúng hoặc hết thời gian chờ (mặc định 5 giây) | Bắt buộc | `await expect(page).toHaveURL(/account/)`            |
| Assertion trên giá trị | Chuỗi, số, mảng, object       | So sánh một lần, có kết quả ngay                                     | Không cần | `expect(dialog.message()).toBe("I am a JS Confirm")` |

Web-first assertion trả về Promise, nên cần `await` để test chờ kết quả. Thiếu `await`, test chạy tiếp ngay mà không chờ assertion, lỗi (nếu có) không được báo đúng chỗ. Assertion trên giá trị so sánh ngay tại dòng code đó, không trả về Promise, nên không cần `await`.

Khi kiểm tra dữ liệu trên trang, luôn dùng web-first assertion. Đọc chữ trên trang ra biến rồi so sánh bằng assertion trên giá trị sẽ mất cơ chế tự chờ, test dễ fail khi trang tải chậm:

```typescript
// Không nên: đọc chữ một lần, nếu badge chưa cập nhật thì test fail
const cartCount = await page.locator("[data-test='cart-quantity']").textContent();
expect(cartCount).toBe("1");

// Nên: Playwright chờ đến khi badge hiển thị số 1
await expect(page.locator("[data-test='cart-quantity']")).toHaveText("1");
```

Assertion trên giá trị dùng cho dữ liệu không nằm trên trang, ví dụ nội dung hộp thoại (mục 1.8), status code của response (mục 4.4), dữ liệu API trả về ở [Buổi 11](../phase-3/buoi-11-api-testing.md).

### 2.2 Các assertion thường dùng

**Web-first assertion:**

| Nhóm       | Assertion                                            | Kiểm tra                                                |
| ---------- | ---------------------------------------------------- | ------------------------------------------------------- |
| Hiển thị   | `toBeVisible()`, `toBeHidden()`                      | Phần tử đang hiển thị hoặc đang bị ẩn                   |
| Hiển thị   | `toBeInViewport()`                                   | Phần tử nằm trong vùng nhìn thấy của màn hình (mục 1.5) |
| Nội dung   | `toHaveText(text)`                                   | Chữ của phần tử khớp **toàn bộ** với giá trị mong đợi   |
| Nội dung   | `toContainText(text)`                                | Chữ của phần tử **chứa** giá trị mong đợi               |
| Nội dung   | `toHaveValue(value)`                                 | Giá trị hiện tại của ô nhập hoặc `<select>`             |
| Trạng thái | `toBeChecked()`                                      | Checkbox hoặc radio đang được chọn                      |
| Trạng thái | `toBeEnabled()`, `toBeDisabled()`                    | Phần tử bấm được hoặc đang bị vô hiệu hóa (`disabled`)  |
| Thuộc tính | `toHaveAttribute(name, value)`, `toHaveClass(value)` | Giá trị của một thuộc tính, hoặc của thuộc tính `class` |
| Số lượng   | `toHaveCount(n)`                                     | Số phần tử mà locator tìm thấy                          |
| Trang      | `toHaveURL(url)`, `toHaveTitle(title)`               | URL hoặc tiêu đề (tab title) của trang hiện tại         |

Với phần tử có nội dung `Combination Pliers $14.15`: `toHaveText("Combination Pliers")` fail vì thiếu phần giá, `toContainText("Combination Pliers")` pass. Dùng `toHaveText()` khi biết chính xác toàn bộ nội dung (số lượng trong giỏ, thông báo lỗi), `toContainText()` khi phần tử có thêm nội dung không cần kiểm tra.

Các assertion nhận chữ (`toHaveText`, `toContainText`, `toHaveValue`, `toHaveURL`, `toHaveClass`...) chấp nhận cả chuỗi và regex. Regex như `/account/` nghĩa là "có chứa `account`", đã gặp ở [Buổi 4](buoi-04-playwright-first-test.md), mục 3.

**Assertion trên giá trị:**

| Assertion                 | Kiểm tra                                              | Ví dụ                                        |
| ------------------------- | ----------------------------------------------------- | -------------------------------------------- |
| `toBe(value)`             | Bằng chính xác (chuỗi, số, boolean)                   | `expect(response.status()).toBe(200)`        |
| `toEqual(value)`          | Bằng nhau về nội dung (mảng, object)                  | `expect(names).toEqual(["Pliers", "Hammer"])` |
| `toContain(item)`         | Mảng có phần tử, hoặc chuỗi có chứa đoạn chữ          | `expect(names).toContain("Thor Hammer")`     |
| `toBeGreaterThan(n)`      | Lớn hơn một số                                        | `expect(names.length).toBeGreaterThan(0)`    |
| `toBeTruthy()`            | Giá trị đúng (không phải `false`, `0`, `""`, `null`, `undefined`) | `expect(isLoggedIn).toBeTruthy()`  |

Ví dụ kết hợp nhiều web-first assertion sau khi thêm sản phẩm vào giỏ hàng:

```typescript
test("thêm sản phẩm vào giỏ hàng", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");
  await page.getByRole("heading", { name: "Combination Pliers" }).click();

  await page.getByRole("button", { name: "Add to cart" }).click();

  // Thông báo xác nhận xuất hiện
  await expect(page.getByText("Product added to shopping cart.")).toBeVisible();
  // Badge giỏ hàng hiển thị số 1
  await expect(page.locator("[data-test='cart-quantity']")).toHaveText("1");
});
```

### 2.3 Assertion neo giữa các bước

Khi một action làm trang chuyển sang trạng thái mới (chuyển trang, mở form, tải danh sách), hãy thêm một assertion xác nhận trạng thái mới **trước** action tiếp theo. Assertion này gọi là **assertion neo**:

```typescript
await page.getByRole("button", { name: "Login" }).click();
// Neo: chờ trang tài khoản tải xong rồi mới thao tác tiếp
await expect(page.getByRole("heading", { name: "My account" })).toBeVisible();

await page.getByRole("link", { name: "Favorites" }).click();
```

Thiếu assertion neo, action tiếp theo có thể chạy khi trang cũ chưa chuyển đi, hoặc tìm thấy phần tử trùng tên ở trang cũ. Đây là nguyên nhân phổ biến nhất khiến test lúc pass lúc fail (flaky).

### 2.4 Timeout của assertion

Web-first assertion mặc định chờ tối đa 5 giây. Assertion trên giá trị không có timeout vì chỉ so sánh một lần. Cách đặt timeout, đơn vị mili giây:

| Phạm vi                  | Cách đặt                                                                                                          |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| Một assertion            | `await expect(page.getByText("Product added to shopping cart.")).toBeVisible({ timeout: 10000 });`                |
| Mọi assertion trong project | `expect: { timeout: 10000 }` trong `playwright.config.ts` ([Buổi 9](../phase-3/buoi-09-config-fixtures-parallel.md), mục 2) |

**Tham khảo:** [Assertions](https://playwright.dev/docs/test-assertions), [Generic assertions](https://playwright.dev/docs/api/class-genericassertions), [Timeouts](https://playwright.dev/docs/test-timeouts)

## 3. Soft assertions: kiểm tra nhiều điều, không dừng giữa chừng

Assertion thông thường fail ở dòng nào thì test dừng ngay ở dòng đó, các assertion phía sau không chạy. Khi cần kiểm tra nhiều chi tiết trên cùng một màn hình, điều này bất tiện: sửa lỗi thứ nhất, chạy lại mới thấy lỗi thứ hai.

`expect.soft()` ghi nhận lỗi nhưng **cho test chạy tiếp**. Test vẫn bị đánh dấu fail, nhưng report liệt kê đủ mọi assertion fail trong một lần chạy:

```typescript
test("trang chủ hiển thị đủ thành phần chính", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");

  // Ba kiểm tra độc lập, một kiểm tra fail không làm dừng các kiểm tra còn lại
  await expect.soft(page.getByRole("combobox", { name: "sort" })).toBeVisible();
  await expect.soft(page.getByRole("link", { name: "Sign in" })).toBeVisible();
  await expect.soft(page.locator("[data-test='product-name']")).toHaveCount(9);
});
```

| Nên dùng soft assertion                                                  | Không dùng soft assertion                                                                             |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| Kiểm tra nhiều chi tiết độc lập trên cùng một màn hình (nhãn, giá, ảnh)  | Điều kiện mà các bước sau phụ thuộc vào, ví dụ đăng nhập thành công, đã chuyển sang trang thanh toán |
| Kết quả của kiểm tra này không ảnh hưởng đến kiểm tra khác               | Assertion neo giữa các bước (mục 2.3)                                                                 |

Với điều kiện mà các bước sau phụ thuộc vào, dùng assertion thông thường để test dừng sớm, tránh chạy tiếp trong trạng thái sai và sinh ra hàng loạt lỗi không liên quan.

**Tham khảo:** [Soft assertions](https://playwright.dev/docs/test-assertions#soft-assertions)

## 4. Auto-wait: không cần sleep()

### 4.1 Vấn đề của thời gian chờ cố định

Trang web không phản hồi ngay lập tức: sau một cú click, trang cần thời gian gọi API, tải dữ liệu, chạy hiệu ứng. Với nhiều công cụ automation cũ, tester phải tự thêm thời gian chờ cố định (sleep):

```typescript
// Anti-pattern: không dùng cách viết này
await page.getByRole("button", { name: "Login" }).click();
await page.waitForTimeout(3000);
```

Con số 3 giây là ước lượng, nên luôn sai theo một trong hai hướng:

* Trang phản hồi trong 0,5 giây: test lãng phí 2,5 giây. Với vài trăm test, thời gian lãng phí cộng dồn lên hàng chục phút mỗi lần chạy.
* Trang phản hồi chậm hơn 3 giây (server chậm, mạng chậm): test fail dù chức năng không có lỗi.

### 4.2 Cách Playwright tự chờ

Playwright tự chờ đến đúng thời điểm trang sẵn sàng, không chờ thừa. Cơ chế này gọi là **auto-wait**, có ở hai nơi:

| Nơi             | Playwright chờ điều gì                                                                  | Đã trình bày ở |
| --------------- | --------------------------------------------------------------------------------------- | -------------- |
| Trước mỗi action | Phần tử đạt các điều kiện actionability checks: hiển thị, đứng yên, không bị che, enabled | Đầu mục 1      |
| Trong mỗi assertion | Điều kiện của assertion trở thành đúng (web-first assertion)                         | Mục 2.1        |

Vì vậy, thay cho `waitForTimeout()`, hãy viết assertion mô tả trạng thái cần chờ:

```typescript
await page.getByRole("button", { name: "Login" }).click();
// Chờ đến khi chuyển sang trang tài khoản, không cần sleep
await expect(page).toHaveURL(/account/);
```

### 4.3 Thời gian chờ tối đa mặc định

| Loại                 | Thời gian chờ tối đa mặc định                                                                     |
| -------------------- | ------------------------------------------------------------------------------------------------- |
| Assertion (`expect`) | 5 giây cho mỗi assertion                                                                          |
| Action               | Không giới hạn riêng, nhưng cả test chỉ được chạy tối đa 30 giây (lỗi `Timeout 30000ms exceeded`) |

{% hint style="warning" %}
Không dùng `page.waitForTimeout()` trong test. Khi cần chờ, dùng assertion mô tả trạng thái cần chờ.
{% endhint %}

### 4.4 Các hàm wait của Playwright

Auto-wait xử lý được phần lớn trường hợp. Một số tình huống cần chờ một thứ không phải phần tử trên trang (API trả về, tab mới mở, trang tải xong), hoặc cần chờ trước khi gọi hàm không có auto-wait. Khi đó dùng các hàm wait chủ động:

| Hàm                                | Chờ điều gì                                                                 | Khi nào dùng                                                                  |
| ---------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `locator.waitFor()`                | Phần tử đạt trạng thái `visible` (mặc định), `hidden`, `attached`, `detached` | Trước các hàm không có auto-wait, ví dụ `allTextContents()`, `count()`        |
| `page.waitForURL(url)`             | URL của trang khớp với giá trị mong đợi và trang tải xong                   | Chờ chuyển trang trong code dùng chung (Page Object ở Buổi 7). Trong test, ưu tiên `toHaveURL()` |
| `page.waitForLoadState(state)`     | Trang đạt trạng thái tải: `load` (mặc định), `domcontentloaded`, `networkidle` | Hiếm khi cần, vì `goto()` và `click()` dẫn tới chuyển trang đã tự chờ `load` |
| `page.waitForResponse(url)`        | Trình duyệt nhận được response của một request khớp điều kiện               | Chờ API trả về sau một action, kiểm tra status code hoặc dữ liệu của API      |
| `page.waitForRequest(url)`         | Trình duyệt gửi đi một request khớp điều kiện                               | Kiểm tra action có gọi đúng API hay không                                    |
| `page.waitForEvent(event)`         | Một sự kiện của trang: `popup`, `filechooser`, `download`, `dialog`...      | Chờ tab mới (mục 1.8), hộp thoại chọn file (mục 1.4), file tải về            |
| `page.waitForTimeout(ms)`          | Một khoảng thời gian cố định                                                | Không dùng trong test (mục 4.1). Chỉ dùng tạm khi debug                       |

#### `locator.waitFor()`: chờ trước hàm không có auto-wait

Các hàm đọc dữ liệu như `allTextContents()`, `count()` trả kết quả ngay tại thời điểm gọi, không chờ phần tử xuất hiện. Nếu danh sách chưa tải xong, kết quả là mảng rỗng hoặc số 0 (lưu ý đã gặp ở Bài 4, bài tập về nhà của [Buổi 5](buoi-05-locators.md)):

```typescript
test("lấy tên các sản phẩm ở trang chủ", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");
  const productNames = page.locator("[data-test='product-name']");

  // waitFor() chỉ nhận locator trỏ tới một phần tử, nên dùng first()
  await productNames.first().waitFor();
  const names = await productNames.allTextContents();

  expect(names.length).toBeGreaterThan(0);
});
```

Có thể thay `waitFor()` bằng một assertion, ví dụ `await expect(productNames).toHaveCount(9)`. Cách này vừa chờ vừa kiểm tra, nên được ưu tiên khi biết trước số lượng.

#### `page.waitForResponse()`: chờ API trả về

Chọn option trong dropdown "Sort" của Toolshop làm trang gọi API lấy lại danh sách sản phẩm. Để chờ và kiểm tra API đó:

```typescript
test("sắp xếp sản phẩm gọi API thành công", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/");

  // Bắt đầu chờ trước action, chưa await ở bước này
  const responsePromise = page.waitForResponse(
    (response) => response.url().includes("/products") && response.url().includes("sort=price,asc"),
  );
  await page.getByRole("combobox", { name: "sort" }).selectOption("price,asc");
  const response = await responsePromise;

  expect(response.status()).toBe(200);
});
```

Giống sự kiện `popup` ở mục 1.8, Promise phải được tạo **trước** action. Nếu tạo sau, response có thể đã về trước khi bắt đầu chờ và test bị treo đến hết thời gian chờ. `waitForRequest()` dùng theo cùng cách. [Buổi 12](../phase-3/buoi-12-api-ui-network-mocking.md) trình bày chi tiết cách làm việc với request và response của trang.

#### `page.waitForURL()` và `page.waitForLoadState()`

```typescript
await page.getByRole("button", { name: "Login" }).click();
// Chờ URL chuyển sang trang tài khoản và trang tải xong
await page.waitForURL(/account/);
```

`waitForURL()` chỉ chờ, không phải assertion. Trong test, `await expect(page).toHaveURL(/account/)` cho cùng kết quả và hiển thị rõ hơn trong report khi fail. `waitForURL()` phù hợp hơn trong code dùng chung như Page Object (Buổi 7).

`waitForLoadState()` hiếm khi cần, vì `page.goto()` mặc định đã chờ trạng thái `load`. Không dùng `waitForLoadState("networkidle")` (chờ đến khi không còn request nào trong 500 ms): tài liệu Playwright không khuyến nghị cách này vì trang có request chạy nền liên tục sẽ làm test chờ đến hết thời gian.

### 4.5 Chọn cách chờ nào

Áp dụng theo thứ tự sau, dừng ở bước đầu tiên phù hợp:

1. **Không viết gì thêm:** action đã tự chờ phần tử sẵn sàng (đầu mục 1).
2. **Assertion:** khi cần chờ một trạng thái nhìn thấy trên trang (phần tử hiển thị, chữ thay đổi, URL đổi).
3. **`locator.waitFor()`:** khi cần chờ trước hàm không có auto-wait như `allTextContents()`.
4. **`waitForEvent()`, `waitForResponse()`, `waitForRequest()`:** khi cần chờ thứ không nằm trên trang (tab mới, file, API).
5. **Không bao giờ dùng `waitForTimeout()`** trong test nộp bài.

**Tham khảo:** [Auto-waiting](https://playwright.dev/docs/actionability), [locator.waitFor()](https://playwright.dev/docs/api/class-locator#locator-wait-for), [page.waitForURL()](https://playwright.dev/docs/api/class-page#page-wait-for-url), [page.waitForLoadState()](https://playwright.dev/docs/api/class-page#page-wait-for-load-state), [page.waitForResponse()](https://playwright.dev/docs/api/class-page#page-wait-for-response), [page.waitForEvent()](https://playwright.dev/docs/api/class-page#page-wait-for-event), [Navigations](https://playwright.dev/docs/navigations)

## 5. Bài tập về nhà

Toàn bộ bài tập làm trên [Toolshop](https://practicesoftwaretesting.com/) với tài khoản `customer@practicesoftwaretesting.com` / `welcome01`. Bài 1 và Bài 2 ôn tập kiến thức từ Buổi 1 đến Buổi 5 trên [trang đăng nhập](https://practicesoftwaretesting.com/auth/login).

### Bài 1: kiểm tra đăng nhập với nhiều bộ dữ liệu

Bài này ôn tập kiến thức của:

* [Buổi 1 · Setup môi trường & TypeScript cơ bản](../phase-1/buoi-01-setup-typescript.md)
* [Buổi 3 · Interfaces, Classes, npm & Array methods](../phase-1/buoi-03-interfaces-classes-array-methods.md)

{% hint style="warning" %}
Tài khoản khách hàng của Toolshop bị khóa sau 3 lần đăng nhập sai password, và tài khoản bị khóa thì cả lớp không dùng được nữa. Không nhập password sai cho `customer@practicesoftwaretesting.com`. Với trường hợp sai thông tin đăng nhập, dùng một email chưa đăng ký, ví dụ `not-registered@practicesoftwaretesting.com`.
{% endhint %}

1. Viết một hàm đăng nhập (mở trang, điền email và password, bấm "Login"), tham số password có giá trị mặc định là `"welcome01"`.
2. Khai báo một `type` ([Buổi 3](../phase-1/buoi-03-interfaces-classes-array-methods.md), mục 2) mô tả một bộ dữ liệu đăng nhập: email, password (tùy chọn) và kết quả mong đợi.
3. Khai báo một mảng có kiểu là `type` ở bước 2, gồm ít nhất 6 bộ dữ liệu:

   | #   | Dữ liệu                                               | Expected Result                        |
   | --- | ----------------------------------------------------- | -------------------------------------- |
   | 1   | Email để trống, password đúng                         | Thông báo thiếu email                  |
   | 2   | Email sai định dạng (không có `@`), password đúng     | Thông báo email sai định dạng          |
   | 3   | Email đúng, password để trống                         | Thông báo thiếu password               |
   | 4   | Email đúng, password có 2 ký tự                       | Thông báo độ dài password không hợp lệ |
   | 5   | Email chưa đăng ký, password `welcome01`              | Thông báo sai email hoặc password      |
   | 6   | `customer@practicesoftwaretesting.com`, password đúng | Chuyển sang trang tài khoản            |

4. Viết một test dùng `for...of` duyệt mảng và destructuring để lấy các field, kiểm tra kết quả của từng bộ dữ liệu khớp với cột "Expected Result".

{% hint style="info" %}
Trong bài này, nếu một bộ dữ liệu fail thì test dừng ngay và các bộ dữ liệu phía sau không chạy. [Buổi 10](../phase-3/buoi-10-test-suite-organization.md) trình bày parameterized test, cách viết để mỗi bộ dữ liệu là một test riêng trong report.
{% endhint %}

### Bài 2: kiểm tra giao diện và trạng thái của form đăng nhập

Bài này ôn tập kiến thức của:

* [Buổi 1 · Setup môi trường & TypeScript cơ bản](../phase-1/buoi-01-setup-typescript.md)
* [Buổi 3 · Interfaces, Classes, npm & Array methods](../phase-1/buoi-03-interfaces-classes-array-methods.md)
* [Buổi 5 · Locators: Chọn phần tử đúng cách](buoi-05-locators.md)

Viết 3 test:

1. **Giao diện trang đăng nhập:** tiêu đề "Login", ô Email, ô Password, nút "Login", link "Register your account", link "Forgot your Password?" đều hiển thị. Ô Email có `type="email"` và placeholder "Your email".
2. **Trạng thái lỗi khi bỏ trống:** bấm "Login" khi chưa nhập gì, hai thông báo lỗi của Email và Password hiển thị, ô Email có `aria-invalid="true"`. Sau khi điền email hợp lệ, lỗi của Email biến mất, `aria-invalid` chuyển thành `"false"`, lỗi của Password vẫn còn.
3. **Điều hướng từ trang đăng nhập:** dùng `type` cho link (tên link, URL mong đợi) và `for...of` để kiểm tra 2 link "Register your account" và "Forgot your Password?" chuyển đúng trang.

### Bài 3: test checkout flow

Viết một test cho toàn bộ luồng đặt hàng:

1. Đăng nhập, thêm 2 sản phẩm khác nhau vào giỏ, mở giỏ hàng (biểu tượng giỏ trên thanh menu). Kiểm tra giỏ hàng có đủ 2 sản phẩm, đúng tên.
2. Ở trang giỏ hàng, kiểm tra tổng tiền bằng `toContainText()` với giá trị trong ô Total.
3. Bấm "Proceed to checkout", qua bước Sign in (đã đăng nhập sẵn), điền đầy đủ Billing Address (Country là dropdown `<select>`), bấm "Proceed to checkout".
4. Ở bước Payment, chọn phương thức "Cash on Delivery" bằng `selectOption()`, bấm "Check payment" và kiểm tra thông báo "Payment was successful".
5. Bấm "Confirm", kiểm tra thông báo hoàn tất "Thanks for your order!".

Yêu cầu kỹ thuật:

* Locator theo thứ tự ưu tiên ở [Buổi 5](buoi-05-locators.md), mục 7. Không dùng XPath.
* Không có `page.waitForTimeout()`.
* Có assertion neo sau mỗi bước chuyển trang (mục 2.3).
* Có ít nhất một chỗ dùng `expect.soft()` hợp lý (mục 3).
* Tên biến tiếng Anh, camelCase (ví dụ `cartTotal`, `paymentMethodDropdown`).

### Bài 4: nộp bài qua Pull Request

Commit (ví dụ `lesson 6: homework`) và tạo Pull Request trên nhánh `your-name/lesson-6` (ví dụ `linh/lesson-6`) theo quy trình ở [Buổi 2](../phase-1/buoi-02-git-async-await.md), mục 5 và mục 6.

**Checklist trước khi nộp:**

* [ ] Tất cả test của bài tập Buổi 6 đều pass
* [ ] Bài 1: hàm đăng nhập có tham số password với giá trị mặc định
* [ ] Bài 1: có `type` cho bộ dữ liệu đăng nhập, ít nhất 6 bộ dữ liệu
* [ ] Bài 1: không có bộ dữ liệu nào nhập password sai cho tài khoản `customer@practicesoftwaretesting.com`
* [ ] Bài 2: có kiểm tra `aria-invalid` trước và sau khi điền email hợp lệ
* [ ] Bài 3: test pass ổn định khi chạy 3 lần liên tiếp
* [ ] Bài 3: mỗi bước chuyển trang có assertion neo
* [ ] Bài 3: có ít nhất một chỗ dùng `expect.soft()`, với lý do hợp lý
* [ ] Không có `page.waitForTimeout()`, XPath, `{ force: true }`
* [ ] Tên biến tiếng Anh, camelCase. Tên test mô tả đúng hành vi được kiểm tra
* [ ] Đúng quy ước tên nhánh `your-name/lesson-6`
