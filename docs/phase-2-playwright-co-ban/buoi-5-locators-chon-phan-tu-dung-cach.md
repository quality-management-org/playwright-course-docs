# Buổi 5 · Locators: Chọn phần tử đúng cách

{% hint style="info" %}
**Sau buổi này bạn sẽ:**

* Đọc được cấu tạo của một thẻ HTML và nhận ra các thuộc tính quan trọng cho việc viết locator.
* Hiểu khái niệm role và accessible name, nền tảng của `getByRole()`.
* Phân biệt thẻ đúng chuẩn với thẻ "giả" và biết cách xử lý khi gặp thẻ "giả".
* Nắm thứ tự ưu tiên khi chọn locator và lý do nên tránh XPath phụ thuộc cấu trúc.
* Kết hợp chaining và filter để trỏ chính xác một phần tử trong danh sách.
{% endhint %}

## 1. Locator là gì và vì sao quan trọng

Locator là thông tin bạn cung cấp cho Playwright để tìm một phần tử trên trang, ví dụ một nút bấm hoặc một ô nhập liệu. Trong thực tế, một tỷ lệ lớn test fail không đến từ logic kiểm thử mà đến từ locator không ổn định: giao diện thay đổi nhỏ là locator không còn trỏ đúng phần tử. Chọn locator tốt ngay từ đầu giúp bộ test bền vững và tốn ít công bảo trì.

Nguyên tắc chung: **chọn locator theo cách người dùng nhìn trang web** (nút có chữ "Login", ô có nhãn "Email address") thay vì theo cấu trúc HTML bên trong (thẻ `div` thứ 3, class `btn-outline-secondary`). Người dùng không nhìn thấy cấu trúc HTML, và cấu trúc HTML là phần được dev thay đổi thường xuyên nhất.

Để áp dụng nguyên tắc này, bạn cần đọc được HTML ở mức cơ bản. Các mục 2 đến 6 trình bày phần nền tảng đó trước khi đi vào cú pháp locator của Playwright ở mục 7.

{% hint style="info" %}
Ví dụ trong buổi này dùng trang thực hành [Toolshop](https://practicesoftwaretesting.com/) (practicesoftwaretesting.com), một cửa hàng dụng cụ mô phỏng dành cho việc học kiểm thử. Tài khoản khách hàng dùng chung: `customer@practicesoftwaretesting.com` / `welcome01`. Không đổi mật khẩu hay thông tin của tài khoản này, vì cả lớp cùng sử dụng.
{% endhint %}

## 2. Cấu tạo của một thẻ (tag)

Mỗi trang web là một tài liệu HTML, gồm nhiều **phần tử (element)**. Mỗi phần tử được viết bằng một **thẻ (tag)**. Ví dụ nút "Add to cart" trên trang chi tiết sản phẩm của Toolshop (đã rút gọn):

```html
<button id="btn-add-to-cart" class="btn btn-success" data-test="add-to-cart">Add to cart</button>
```

| Thành phần | Trong ví dụ                                                      | Ý nghĩa                                                   |
| ---------- | ---------------------------------------------------------------- | --------------------------------------------------------- |
| Thẻ mở     | `<button ...>`                                                   | Bắt đầu phần tử, cho biết loại phần tử (ở đây là nút bấm) |
| Tên thẻ    | `button`                                                         | Xác định phần tử là gì: nút, link, ô nhập, tiêu đề...     |
| Thuộc tính | `id="btn-add-to-cart"`, `class="..."`, `data-test="add-to-cart"` | Thông tin bổ sung, viết theo dạng `tên="giá trị"`         |
| Nội dung   | `Add to cart`                                                    | Phần chữ người dùng nhìn thấy trên màn hình               |
| Thẻ đóng   | `</button>`                                                      | Kết thúc phần tử, có dấu `/` trước tên thẻ                |

Một số thẻ không có nội dung và không có thẻ đóng, gọi là thẻ rỗng (void element), ví dụ `<input>` và `<img>`. Với các thẻ này, chữ hiển thị trên màn hình lấy từ thuộc tính. Nút Login trên [trang đăng nhập Toolshop](https://practicesoftwaretesting.com/auth/login) là một ví dụ:

```html
<input type="submit" class="btnSubmit" data-test="login-submit" aria-label="Login" value="Login">
```

Chữ "Login" trên nút đến từ thuộc tính `value`, không phải từ nội dung giữa hai thẻ.

### Thẻ lồng nhau

Các thẻ có thể lồng vào nhau: thẻ bên ngoài là **thẻ cha (parent)**, thẻ bên trong là **thẻ con (child)**. Toàn bộ trang tạo thành một cấu trúc cây, gọi là **DOM** (Document Object Model). Thẻ sản phẩm trên trang chủ Toolshop (đã rút gọn):

```html
<a class="card" href="/product/01M437BCK8E3SPK5T3YG91B7BC">
  <div class="card-img-wrapper">
    <img class="card-img-top" alt="Combination Pliers" src="assets/img/products/pliers01.avif">
    <button class="btn compare-btn" data-test="compare-btn" aria-label="Compare">
      <svg aria-hidden="true">...</svg>
    </button>
  </div>
  <div class="card-body">
    <h5 class="card-title" data-test="product-name">Combination Pliers</h5>
  </div>
  <div class="card-footer">
    <span data-test="product-price">$14.15</span>
  </div>
</a>
```

Thẻ `<a>` là thẻ cha của toàn bộ thẻ sản phẩm. Thẻ `<h5>` chứa tên sản phẩm và thẻ `<span>` chứa giá đều là thẻ con, nằm sâu hơn một hoặc hai tầng. Cả 9 sản phẩm trên mỗi trang đều có cấu trúc giống hệt nhau, chỉ khác tên, ảnh và giá. Đặc điểm này giải thích vì sao cần chaining và filter ở mục 9.

### Xem HTML của một phần tử

Mở [Toolshop](https://practicesoftwaretesting.com/) bằng Chrome, chuột phải vào tên một sản phẩm, chọn **Inspect**. Cửa sổ DevTools mở ra, tab **Elements** hiển thị cây DOM và tô sáng thẻ tương ứng với phần tử vừa chọn.

## 3. Thuộc tính quan trọng

Không phải thuộc tính nào cũng phù hợp để viết locator. Bảng sau liệt kê các thuộc tính thường gặp trên Toolshop và mức độ ổn định của chúng:

| Thuộc tính                 | Ví dụ trên Toolshop                 | Ý nghĩa                                                 | Dùng cho locator                                         |
| -------------------------- | ----------------------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| `id`                       | `id="email"`                        | Định danh, theo chuẩn là duy nhất trên trang            | Khá ổn định, nhưng một số framework sinh id ngẫu nhiên   |
| `class`                    | `class="btn btn-outline-secondary"` | Nhóm phần tử để áp dụng giao diện (CSS)                 | Kém ổn định, dev đổi class khi chỉnh giao diện           |
| `name`                     | `name="category_id"`                | Tên trường dữ liệu khi gửi form                         | Khá ổn định với ô nhập liệu, nhưng có thể trùng nhau     |
| `type`                     | `type="checkbox"`, `type="email"`   | Loại ô nhập: text, email, password, checkbox, submit... | Quyết định role của thẻ `<input>` (mục 4)                |
| `placeholder`              | `placeholder="Your email"`          | Chữ gợi ý mờ hiển thị trong ô nhập khi ô còn trống      | Dùng với `getByPlaceholder()`                            |
| `value`                    | `value="Login"`                     | Giá trị của ô nhập, hoặc chữ hiển thị trên nút `submit` | Tạo nên tên của nút `<input type="submit">`              |
| `href`                     | `href="/auth/register"`             | Đường dẫn của link                                      | Thẻ `<a>` có `href` mới được xem là link                 |
| `for`                      | `<label for="email">`               | Gắn nhãn với ô nhập có `id` tương ứng                   | Một trong hai cách giúp `getByLabel()` hoạt động (mục 4) |
| `aria-label`               | `aria-label="Compare"`              | Tên dành cho công cụ đọc màn hình, không hiển thị       | Tạo nên tên của phần tử chỉ có icon, không có chữ        |
| `data-testid`, `data-test` | `data-test="login-submit"`          | Thuộc tính dev gắn riêng cho mục đích kiểm thử          | Rất ổn định, dùng với `getByTestId()`                    |
| `disabled`, `checked`      | `<button disabled>`                 | Trạng thái của phần tử, không cần giá trị               | Dùng trong assertion (Buổi 6), không dùng để tìm phần tử |

Selector dạng `#user-name` đã dùng ở Buổi 4 chính là tìm theo thuộc tính `id` (dấu `#` nghĩa là id). Tương tự, `.btnSubmit` là tìm theo `class` (dấu `.` nghĩa là class), và `[data-test="login-submit"]` là tìm theo một thuộc tính bất kỳ.

{% hint style="warning" %}
**Cẩn thận với thuộc tính do framework tự sinh.** Toolshop được viết bằng Angular, nên trong DevTools bạn sẽ thấy các thuộc tính dạng `_ngcontent-ng-c1719315749`. Một số framework khác sinh class dạng `css-1x9f3k2`. Các giá trị này thay đổi mỗi lần dev build lại ứng dụng, nên locator dựa vào chúng sẽ vỡ mà không có thay đổi nào về chức năng.
{% endhint %}

## 4. Các phần tử chuẩn

HTML định nghĩa sẵn các thẻ cho từng loại phần tử. Trình duyệt dựa vào tên thẻ (và thuộc tính `type` với thẻ `<input>`) để gán cho mỗi phần tử một **role**, tức vai trò của phần tử đó với người dùng: nút bấm, link, ô nhập, tiêu đề...

| Thẻ HTML                                                    | Role                   | Ví dụ trên Toolshop                          |
| ----------------------------------------------------------- | ---------------------- | -------------------------------------------- |
| `<button>`, `<input type="submit">`                         | `button`               | Nút "Add to cart", nút "Login"               |
| `<a href="...">`                                            | `link`                 | Link "Register your account"                 |
| `<input type="text">`, `<input type="email">`, `<textarea>` | `textbox`              | Ô "Email address", ô "Message"               |
| `<input type="number">`                                     | `spinbutton`           | Ô "Quantity" ở trang chi tiết sản phẩm       |
| `<input type="checkbox">`                                   | `checkbox`             | Bộ lọc danh mục "Hammer", "Pliers"           |
| `<select>`                                                  | `combobox`             | Dropdown "Sort", dropdown "Subject"          |
| `<h1>` đến `<h6>`                                           | `heading`              | Tên sản phẩm (`<h5>`), "My account" (`<h1>`) |
| `<img alt="...">`                                           | `img`                  | Ảnh sản phẩm                                 |
| `<table>`, `<tr>`, `<td>`                                   | `table`, `row`, `cell` | Bảng "Specifications" ở trang chi tiết       |
| `<div>`, `<span>`                                           | không có role riêng    | Giá sản phẩm (`<span>`)                      |

Ô mật khẩu `<input type="password">` không có role chính thức trong chuẩn HTML, nhưng Playwright vẫn tìm được ô này bằng `getByRole("textbox")`. Bảng đầy đủ về role của từng thẻ nằm trong tài liệu W3C ở mục 11.

### Accessible name

Ngoài role, mỗi phần tử còn có một **accessible name**: cái tên mà công cụ đọc màn hình (dành cho người khiếm thị) đọc lên. Trình duyệt tính tên này theo quy tắc sau:

| Phần tử                    | Accessible name lấy từ                             | Ví dụ trên Toolshop                                                   |
| -------------------------- | -------------------------------------------------- | --------------------------------------------------------------------- |
| `<button>`, `<a>`, heading | Nội dung chữ bên trong thẻ                         | `<button>Add to cart</button>` có tên "Add to cart"                   |
| `<input type="submit">`    | Thuộc tính `value`                                 | `<input type="submit" value="Send">` có tên "Send"                    |
| Ô nhập liệu                | Thẻ `<label>` gắn bằng `for`                       | `<label for="email">Email address *</label>` cho ô có `id="email"`    |
| Ô nhập liệu                | Thẻ `<label>` bọc ngoài ô nhập                     | `<label><input type="checkbox"> Hammer </label>` có tên "Hammer"      |
| `<img>`                    | Thuộc tính `alt`                                   | `<img alt="Combination Pliers">` có tên "Combination Pliers"          |
| Mọi phần tử                | Thuộc tính `aria-label`, nếu có (ưu tiên cao nhất) | Nút so sánh chỉ có icon, `aria-label="Compare"` cho nút tên "Compare" |

Cặp role và accessible name chính là thông tin `getByRole()` sử dụng: `page.getByRole("button", { name: "Add to cart" })` nghĩa là "phần tử có role `button` và tên `Add to cart`". Vì role và tên phản ánh đúng cách người dùng nhận biết phần tử, locator dạng này ít bị ảnh hưởng khi dev đổi class hay cấu trúc thẻ.

Để xem role và tên của một phần tử trong Chrome: mở DevTools, chọn phần tử ở tab Elements, sau đó mở tab **Accessibility** ở khung bên phải.

## 5. Thẻ đúng chuẩn và thẻ "giả"

Không phải trang web nào cũng dùng đúng thẻ cho đúng mục đích. Dev có thể tạo một phần tử trông giống nút bấm nhưng thực chất là thẻ `<div>`:

```html
<!-- Thẻ đúng chuẩn -->
<button type="submit">Đăng nhập</button>

<!-- Thẻ "giả": trông giống nút nhờ CSS, bấm được nhờ JavaScript -->
<div class="btn" onclick="login()">Đăng nhập</div>
```

Trên màn hình, hai phần tử có thể trông giống hệt nhau. Khác biệt nằm ở role: thẻ thứ nhất có role `button`, thẻ thứ hai không có role nào. Hệ quả là `page.getByRole("button", { name: "Đăng nhập" })` chỉ tìm thấy thẻ thứ nhất.

Các dạng thẻ "giả" thường gặp:

| Trông giống    | Thực chất là                                        | Hệ quả với locator                                                |
| -------------- | --------------------------------------------------- | ----------------------------------------------------------------- |
| Nút bấm        | `<div>` hoặc `<span>` có xử lý click                | `getByRole("button")` không tìm thấy                              |
| Tiêu đề        | `<div>` hoặc `<span>` có chữ to, đậm                | `getByRole("heading")` không tìm thấy                             |
| Link           | `<a>` không có `href`, hoặc `<span>` có xử lý click | `getByRole("link")` không tìm thấy                                |
| Ô nhập có nhãn | `<label>` không có `for`, đặt cạnh `<input>`        | `getByLabel()` không tìm thấy                                     |
| Checkbox       | `<div>` có hình dấu tích vẽ bằng CSS                | `getByRole("checkbox")` không tìm thấy, `check()` không dùng được |

### Ví dụ thực tế trên Toolshop

**Tiêu đề "giả".** Chữ "Learn & Explore" ở footer trang chủ được in hoa, đậm, trông như tiêu đề, nhưng HTML thực tế là:

```html
<div class="ptt-explore-heading text-uppercase text-muted fw-semibold">Learn &amp; Explore</div>
```

**Role đi theo thẻ, không đi theo giao diện.** Mục "Categories" trên menu trông giống các link "Home", "Contact" bên cạnh, nhưng là một nút bấm (bấm vào để mở danh sách danh mục):

```html
<button type="button" class="nav-link dropdown-toggle btn-link-style" data-test="nav-categories">Categories</button>
```

Test sau xác nhận cả hai trường hợp:

```typescript
import { test, expect } from "@playwright/test";

test("Categories là button, Learn & Explore là thẻ div", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com");

  // "Categories" trên menu trông giống link nhưng là thẻ <button>
  await expect(page.getByRole("button", { name: "Categories" })).toBeVisible();
  await expect(page.getByRole("link", { name: "Categories" })).toHaveCount(0);

  // "Learn & Explore" ở footer trông giống tiêu đề nhưng là thẻ <div>
  await expect(page.getByText("Learn & Explore")).toBeVisible();
  await expect(page.getByRole("heading", { name: "Learn & Explore" })).toHaveCount(0);
});
```

Assertion `toHaveCount()` và `toBeVisible()` được trình bày chi tiết ở Buổi 6.

**Thẻ đúng nhưng thiếu tên.** Trên trang đăng nhập, nút hình con mắt cạnh ô Password (dùng để hiện mật khẩu) là thẻ `<button>` đúng chuẩn, nhưng bên trong chỉ có icon và không có `aria-label`:

```html
<button type="button" class="btn btn-outline-secondary">
  <svg aria-hidden="true">...</svg>
</button>
```

Nút này có role `button` nhưng accessible name rỗng, nên không thể viết `getByRole("button", { name: ... })` cho nó. So sánh với nút so sánh sản phẩm ở mục 2: cũng chỉ có icon, nhưng có `aria-label="Compare"` nên vẫn có tên.

{% hint style="warning" %}
**Khi gặp thẻ "giả" hoặc phần tử thiếu tên:** chuyển sang locator ở mức ưu tiên thấp hơn (`getByText()`, `getByTestId()`, CSS selector) theo bảng ở mục 7. Đồng thời ghi nhận đây là lỗi accessibility và báo lại cho team dev: phần tử không đúng chuẩn gây khó khăn cho người dùng công cụ đọc màn hình, không chỉ cho automation test.
{% endhint %}

## 6. Từ HTML đến locator

Quy trình chọn locator cho một phần tử bất kỳ:

1. Chuột phải vào phần tử, chọn **Inspect** để xem HTML.
2. Xác định tên thẻ và role (bảng ở mục 4). Kiểm tra phần tử có phải thẻ "giả" không (mục 5).
3. Xác định accessible name: nội dung chữ, `value`, `<label>`, `alt` hay `aria-label`.
4. Nếu có role và tên rõ ràng, dùng `getByRole()`. Nếu không, xét lần lượt các thông tin còn lại theo thứ tự ở mục 7.

Bảng sau áp dụng quy trình trên cho một số phần tử thật trên Toolshop:

| HTML (rút gọn)                                                       | Thông tin dùng được                              | Locator                                                   |
| -------------------------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------- |
| `<input type="submit" value="Login">`                                | Role `button`, tên "Login"                       | `page.getByRole("button", { name: "Login" })`             |
| `<label for="email">Email address *</label>` và `<input id="email">` | Có `<label>` gắn bằng `for`                      | `page.getByLabel("Email address")`                        |
| `<label><input type="checkbox"> Hammer </label>`                     | Role `checkbox`, tên "Hammer" từ label bọc ngoài | `page.getByRole("checkbox", { name: "Hammer" })`          |
| `<select aria-label="sort" data-test="sort">`                        | Role `combobox`, tên "sort" từ `aria-label`      | `page.getByRole("combobox", { name: "sort" })`            |
| `<div class="ptt-explore-heading">Learn &amp; Explore</div>`         | Thẻ "giả", không có role, có chữ                 | `page.getByText("Learn & Explore")`                       |
| `<span data-test="product-price">$14.15</span>`                      | Không có role, có `data-test`                    | `page.getByTestId("product-price")` (cần cấu hình, mục 7) |
| `<button aria-label="Compare">` (lặp lại 9 lần trên trang)           | Role `button`, tên trùng nhau                    | Cần chaining và filter, xem mục 9                         |

Dòng cuối cho thấy role và tên chưa đủ khi trang có nhiều phần tử giống nhau. Playwright sẽ báo lỗi "strict mode violation" nếu locator trỏ trúng nhiều phần tử trong khi thao tác chỉ áp dụng cho một phần tử. Mục 9 trình bày cách xử lý.

Áp dụng trên trang đăng nhập của Toolshop:

```typescript
import { test, expect } from "@playwright/test";

test("đăng nhập Toolshop bằng getByLabel", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com/auth/login");

  // Ô email có <label for="email"> nên dùng được getByLabel
  await page.getByLabel("Email address").fill("customer@practicesoftwaretesting.com");
  // Không dùng getByLabel("Password"): link "Forgot your Password?" cũng khớp (mục 12)
  await page.getByRole("textbox", { name: "Password" }).fill("welcome01");
  await page.getByRole("button", { name: "Login" }).click();

  await expect(page).toHaveURL(/account/);
  await expect(page.getByRole("heading", { name: "My account" })).toBeVisible();
});
```

| Dòng code                                           | Ý nghĩa                                                                      |
| --------------------------------------------------- | ---------------------------------------------------------------------------- |
| `page.getByLabel("Email address")`                  | Tìm ô nhập được gắn với nhãn "Email address \*"                              |
| `page.getByRole("textbox", { name: "Password" })`   | Tìm ô nhập có tên "Password \*". Cách này chỉ trúng ô nhập, không trúng link |
| `page.getByRole("button", { name: "Login" })`       | Tìm nút có tên "Login", tức thẻ `<input type="submit" value="Login">`        |
| `page.getByRole("heading", { name: "My account" })` | Tiêu đề `<h1>` của trang tài khoản, chỉ xuất hiện khi đăng nhập thành công   |

## 7. Thứ tự ưu tiên khi chọn locator

Xét từ trên xuống, chỉ chuyển xuống mức dưới khi mức trên không dùng được:

| Ưu tiên | Locator              | Dùng khi                                             | Ví dụ trên Toolshop                                 |
| ------- | -------------------- | ---------------------------------------------------- | --------------------------------------------------- |
| 1       | `getByRole()`        | Phần tử đúng chuẩn, có role và tên rõ ràng           | `page.getByRole("button", { name: "Add to cart" })` |
| 2       | `getByLabel()`       | Ô nhập liệu có `<label>` gắn đúng cách               | `page.getByLabel("Email address")`                  |
| 3       | `getByPlaceholder()` | Ô nhập không có label, chỉ có chữ gợi ý              | `page.getByPlaceholder("Your email")`               |
| 4       | `getByText()`        | Phần tử nhận diện bằng nội dung chữ, kể cả thẻ "giả" | `page.getByText("Learn & Explore")`                 |
| 5       | `getByTestId()`      | Team dev có gắn thuộc tính dành cho kiểm thử         | `page.getByTestId("login-submit")`                  |
| 6       | CSS selector         | Các cách ở trên đều không dùng được                  | `page.locator("#email")`                            |
| 7       | XPath                | Tránh dùng, xem mục 8                                | Không khuyến nghị                                   |

Ví dụ tìm kiếm sản phẩm trên trang chủ Toolshop, viết theo thứ tự ưu tiên:

```typescript
import { test, expect } from "@playwright/test";

test("tìm kiếm sản phẩm bằng locator chuẩn", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com");

  // Thay cho page.locator("#search-query") và page.locator("[data-test=search-submit]")
  await page.getByRole("textbox", { name: "Search" }).fill("pliers");
  await page.getByRole("button", { name: "Search" }).click();

  await expect(page.getByRole("heading", { name: "Searched for: pliers" })).toBeVisible();
  await expect(page.getByRole("heading", { name: "Long Nose Pliers" })).toBeVisible();
});
```

Code này đọc gần giống mô tả thao tác bằng lời: điền "pliers" vào ô Search, bấm nút Search, kiểm tra kết quả có sản phẩm "Long Nose Pliers". Người chưa đọc code vẫn hiểu được test làm gì, đây là dấu hiệu của locator tốt.

Ô Search có nhãn `<label for="search-query">Search</label>` nhưng nhãn này bị ẩn khỏi màn hình bằng class `visually-hidden`. Nhãn ẩn vẫn tạo nên accessible name cho ô nhập, nên `getByRole("textbox", { name: "Search" })` và `getByLabel("Search")` đều tìm được ô này.

{% hint style="info" %}
Mặc định, tham số `name` của `getByRole()` và chuỗi truyền vào `getByLabel()`, `getByText()` so khớp không phân biệt chữ hoa, chữ thường và chấp nhận khớp một phần. Ví dụ trên trang chủ Toolshop, `page.getByRole("heading", { name: "Pliers" })` khớp 4 tiêu đề: "Combination Pliers", "Pliers", "Long Nose Pliers" và "Slip Joint Pliers". Thêm `exact: true` khi cần khớp chính xác toàn bộ chuỗi: `page.getByRole("heading", { name: "Pliers", exact: true })` chỉ khớp 1 tiêu đề.
{% endhint %}

{% hint style="success" %}
**Dùng getByTestId trên Toolshop:** trang này gắn thuộc tính `data-test` thay vì `data-testid` mà Playwright tìm theo mặc định. Thêm cấu hình sau vào phần `use` trong `playwright.config.ts`: `testIdAttribute: "data-test"`. Sau đó `page.getByTestId("login-submit")` sẽ hoạt động. Buổi 9 trình bày chi tiết `playwright.config.ts`.
{% endhint %}

## 8. Vì sao tránh XPath phụ thuộc cấu trúc

So sánh hai cách trỏ vào cùng một nút "Compare" của sản phẩm "Combination Pliers":

```typescript
// Cách 1 (anti-pattern): XPath theo vị trí trong cây DOM
page.locator("//div[@class='container']/div[2]/div[2]/div[1]/a[1]/div[1]/button");

// Cách 2: theo thông tin người dùng nhìn thấy
page
  .locator(".card")
  .filter({ hasText: "Combination Pliers" })
  .getByRole("button", { name: "Compare" });
```

{% hint style="warning" %}
Cách 1 chỉ minh họa anti-pattern, không dùng trong bài làm. XPath dạng này mô tả đường đi qua từng tầng thẻ lồng nhau (mục 2), nên phụ thuộc hoàn toàn vào cấu trúc HTML.
{% endhint %}

Cách 1 vỡ khi dev thêm một sản phẩm hoặc người dùng đổi cách sắp xếp (thứ tự thay đổi), khi dev bọc thêm một thẻ `<div>`, hoặc đổi bố cục trang. Cách 2 chỉ vỡ khi chính sản phẩm hoặc nút bấm biến mất, tức là khi có thay đổi thực sự đáng để test fail. XPath không sai về mặt kỹ thuật, nhưng XPath phụ thuộc cấu trúc tạo ra test dễ vỡ, và test thường xuyên fail không rõ lý do làm giảm niềm tin của cả team vào automation.

## 9. Chaining và filter: trỏ chính xác trong danh sách

Khi trang có nhiều phần tử giống nhau (danh sách sản phẩm, bảng dữ liệu), kết hợp locator để khoanh vùng trước, sau đó tìm tiếp bên trong vùng đó. Trang chủ Toolshop có 9 nút "Compare" giống hệt nhau, mỗi nút thuộc một sản phẩm:

```typescript
import { test, expect } from "@playwright/test";

test("thêm Combination Pliers vào danh sách so sánh", async ({ page }) => {
  await page.goto("https://practicesoftwaretesting.com");

  // Bước 1: khoanh vùng đúng thẻ sản phẩm cần thao tác
  const pliersCard = page
    .locator(".card")
    .filter({ hasText: "Combination Pliers" });

  // Bước 2: tìm tiếp bên trong vùng đó
  await expect(pliersCard.locator('[data-test="product-price"]')).toHaveText("$14.15");
  await pliersCard.getByRole("button", { name: "Compare" }).click();

  // Sau khi chọn so sánh, trang hiển thị link "Compare Now"
  await expect(page.getByRole("link", { name: "Compare Now" })).toBeVisible();
});
```

| Thành phần                                          | Ý nghĩa                                                           |
| --------------------------------------------------- | ----------------------------------------------------------------- |
| `page.locator(".card")`                             | Tìm tất cả thẻ sản phẩm trên trang (9 phần tử, cùng class `card`) |
| `.filter({ hasText: "Combination Pliers" })`        | Giữ lại thẻ sản phẩm có chứa chữ "Combination Pliers"             |
| `pliersCard.locator('[data-test="product-price"]')` | Chaining: chỉ tìm giá bên trong thẻ sản phẩm đã khoanh vùng       |
| `pliersCard.getByRole("button", ...)`               | Chaining: chỉ tìm nút "Compare" bên trong thẻ sản phẩm đó         |

Cách tư duy tương tự việc chỉ đường theo địa danh ("vào tòa nhà A, lên phòng 302") thay vì đếm vị trí ("cửa thứ 47 tính từ cổng"). Chaining dựa trên quan hệ cha con đã trình bày ở mục 2.

## 10. Công cụ tìm locator nhanh

| Công cụ             | Cách dùng                                                                                          |
| ------------------- | -------------------------------------------------------------------------------------------------- |
| Codegen (Buổi 4)    | Chạy `npx playwright codegen <url>`, di chuột lên phần tử, Playwright gợi ý locator phù hợp        |
| Chế độ debug        | Chạy `npx playwright test --debug`, dùng nút **Pick locator** rồi click vào phần tử để lấy locator |
| DevTools của Chrome | Chuột phải, chọn **Inspect** để xem HTML (mục 2) và tab **Accessibility** để xem role, tên (mục 4) |

Locator do Codegen gợi ý cần được đối chiếu lại với thứ tự ưu tiên ở mục 7. Với thẻ "giả" hoặc phần tử thiếu tên, Codegen có thể gợi ý CSS selector dài; khi đó hãy kiểm tra xem `getByText()` hoặc `getByTestId()` có dùng được không.

## 11. Tài liệu tham khảo

| Tài liệu                                                                                                 | Nội dung                                                                                                                                                               |
| -------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [ARIA in HTML: Document conformance requirements](https://www.w3.org/TR/html-aria/#docconformance) (W3C) | Bảng chuẩn liệt kê role mặc định của từng thẻ HTML và các role, thuộc tính `aria-*` được phép dùng cho thẻ đó. Dùng để tra cứu khi cần biết một thẻ có role gì (mục 4) |

## 12. Lỗi thường gặp

| Triệu chứng                                                             | Nguyên nhân                                                                                                      | Cách xử lý                                                                                     |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Lỗi "strict mode violation: resolved to N elements"                     | Locator trúng nhiều phần tử cùng lúc, ví dụ 9 nút "Compare" trên trang chủ                                       | Khoanh vùng bằng `.filter()` và chaining (mục 9), hoặc bổ sung `name` cho cụ thể hơn           |
| `getByLabel("Password")` báo strict mode violation trên trang đăng nhập | Khớp một phần: link "Forgot your Password?" có `aria-label` chứa chữ "Password"                                  | Dùng `getByRole("textbox", { name: "Password" })` để chỉ tìm ô nhập (mục 6)                    |
| `getByRole()` timeout dù phần tử hiển thị rõ trên trang                 | Phần tử là thẻ "giả" (`<div>`, `<span>`) hoặc là thẻ khác loại, ví dụ "Categories" là button chứ không phải link | Inspect để xem tên thẻ (mục 5), dùng đúng role hoặc chuyển sang `getByText()`, `getByTestId()` |
| `getByLabel()` không tìm thấy ô nhập dù nhãn nằm ngay cạnh ô            | Thẻ `<label>` không gắn với ô nhập (thiếu `for` và không bọc ngoài ô nhập)                                       | Dùng `getByPlaceholder()` hoặc `getByTestId()` theo thứ tự ở mục 7                             |
| Locator chạy được hôm nay, vỡ sau khi dev cập nhật giao diện            | Locator dựa vào class, thuộc tính tự sinh hoặc vị trí trong cây DOM                                              | Chuyển sang role, label, text hoặc test id (mục 3, mục 7)                                      |
| `getByTestId()` không hoạt động trên Toolshop                           | Trang dùng `data-test`, không phải `data-testid`                                                                 | Cấu hình `testIdAttribute: "data-test"`, xem hint ở mục 7                                      |

## 13. Bài tập về nhà

Toàn bộ bài tập làm trên [Toolshop](https://practicesoftwaretesting.com/). Không cần đăng nhập; nếu đăng nhập, chỉ dùng tài khoản chung ở mục 1 và không thay đổi thông tin tài khoản.

### Bài 1: đọc HTML của phần tử

Mở trang [Contact](https://practicesoftwaretesting.com/contact), chọn 3 phần tử bất kỳ trong form và Inspect từng phần tử. Với mỗi phần tử, ghi vào comment đầu file test của Bài 2 các thông tin sau:

* Tên thẻ và các thuộc tính chính
* Role và accessible name (kiểm tra bằng tab Accessibility của DevTools)
* Phần tử là thẻ đúng chuẩn hay thẻ "giả", có accessible name hay không

### Bài 2: viết locator trên Toolshop

Tạo file `tests/homework-lesson-05.spec.ts`. Viết locator cho **ít nhất 15 phần tử khác nhau**, lấy từ các trang: trang chủ (ô tìm kiếm, dropdown sắp xếp, bộ lọc danh mục, thẻ sản phẩm), trang chi tiết sản phẩm, trang Contact và trang đăng nhập. Yêu cầu:

1. Không dùng XPath. Mỗi locator kèm một dòng comment giải thích lý do chọn cách đó.
2. Có đủ ít nhất 5 loại: `getByRole()`, `getByLabel()`, `getByPlaceholder()`, `getByText()`, và ít nhất một trường hợp dùng `.filter()` hoặc chaining.
3. Có ít nhất một phần tử là thẻ "giả" hoặc thiếu accessible name, kèm comment giải thích vì sao không dùng được `getByRole()` cho phần tử đó.
4. Mỗi locator kèm một assertion đơn giản (`toBeVisible()` là đủ) để chứng minh locator trỏ đúng phần tử.
5. Tên biến đặt bằng tiếng Anh, theo camelCase (ví dụ `searchInput`, `subjectDropdown`).

Chạy bằng `npx playwright test tests/homework-lesson-05.spec.ts`.

{% hint style="info" %}
Không cần bấm gửi form Contact. Bài tập chỉ yêu cầu tìm đúng phần tử và kiểm tra phần tử hiển thị.
{% endhint %}

### Bài 3: nộp bài qua Pull Request

Commit và tạo Pull Request trên nhánh `your-name/lesson-5` (ví dụ `linh/lesson-5`) theo quy trình ở mục 5 và mục 6 của Buổi 2.

**Checklist trước khi nộp:**

* [ ] Tất cả test pass khi chạy `npx playwright test tests/homework-lesson-05.spec.ts`
* [ ] Có comment phân tích HTML cho 3 phần tử của Bài 1
* [ ] Không có XPath nào trong file
* [ ] Có đủ 5 loại locator theo yêu cầu, gồm ít nhất một chỗ dùng `.filter()` hoặc chaining
* [ ] Có ít nhất một phần tử thẻ "giả" hoặc thiếu tên, kèm comment giải thích
* [ ] Mỗi locator có comment giải thích lý do chọn
* [ ] Tên test mô tả đúng hành vi được kiểm tra
* [ ] Đúng quy ước tên nhánh `your-name/lesson-5`
