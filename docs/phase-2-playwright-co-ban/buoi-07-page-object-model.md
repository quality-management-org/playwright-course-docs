---
description: "Tổ chức test theo Page Object Model: mỗi trang web là một class, khi giao diện thay đổi chỉ cần sửa đúng một chỗ."
icon: sitemap
---

# Buổi 7 · Page Object Model: Khái niệm & Xây dựng

{% hint style="info" %}
**Sau buổi này bạn sẽ:**

* Phân tích được hạn chế về bảo trì của test viết trực tiếp bằng locator khi số lượng test tăng.
* Hiểu khái niệm Page Object Model (POM) và các thành phần của một page object.
* Thiết kế được page object cho một trang bất kỳ theo 3 câu hỏi phân tích trang.
* Viết test chỉ gọi method của page object, không thao tác trực tiếp với locator.
{% endhint %}

Toàn bộ ví dụ và bài tập của buổi này chạy trên [Toolshop](https://practicesoftwaretesting.com/).

Nội dung buổi học được trình bày theo trình tự sau:

| Mục | Nội dung                       | Mục tiêu                                                         |
| --- | ------------------------------ | ---------------------------------------------------------------- |
| 1   | Vì sao cần Page Object Model   | Phân tích hạn chế của test viết trực tiếp bằng locator           |
| 2   | Khái niệm Page Object Model    | Hiểu nguyên tắc tổ chức và thành phần của page object            |
| 3   | Xây dựng `LoginPage`           | Thiết kế và xây dựng page object theo từng bước                  |
| 4   | Sử dụng page object trong test | Viết test với page object và đánh giá hiệu quả bảo trì           |
| 5   | Quy ước thiết kế page object   | Nắm các quy ước áp dụng trong khóa học                           |
## 1. Vì sao cần Page Object Model

Bài tập Buổi 6 gồm nhiều test trên trang đăng nhập của Toolshop: đăng nhập với nhiều bộ dữ liệu, kiểm tra giao diện của form, kiểm tra trạng thái lỗi. Xét ba test sau:

```typescript
import { test, expect } from "@playwright/test";

test("đăng nhập thành công", async ({ page }) => {
  await page.goto("/auth/login");
  await page.getByLabel("Email address").fill("customer@practicesoftwaretesting.com");
  await page.getByRole("textbox", { name: "Password" }).fill("welcome01");
  await page.getByRole("button", { name: "Login" }).click();
  await expect(page).toHaveURL(/\/account/);
});

test("đăng nhập thất bại với email chưa đăng ký", async ({ page }) => {
  await page.goto("/auth/login");
  await page.getByLabel("Email address").fill("not-registered@practicesoftwaretesting.com");
  await page.getByRole("textbox", { name: "Password" }).fill("welcome01");
  await page.getByRole("button", { name: "Login" }).click();
  await expect(page.getByText("Invalid email or password")).toBeVisible();
});

test("form đăng nhập hiển thị đủ ô nhập và nút Login", async ({ page }) => {
  await page.goto("/auth/login");
  await expect(page.getByLabel("Email address")).toBeVisible();
  await expect(page.getByRole("textbox", { name: "Password" })).toBeVisible();
  await expect(page.getByRole("button", { name: "Login" })).toBeVisible();
});
```

Giả sử trong phiên bản tiếp theo, nhãn "Email address" được đổi thành "Email". Chức năng đăng nhập vẫn hoạt động đúng, nhưng:

* Cả 3 test đều fail, mặc dù sản phẩm không có lỗi.
* Locator `getByLabel("Email address")` phải sửa tại 3 vị trí trong file này và trong mọi file test khác có bước đăng nhập (giỏ hàng, checkout...). Bỏ sót một vị trí, test tương ứng tiếp tục fail.

Đối chiếu với bài làm Buổi 6: xác định số lần locator của ô Email được khai báo và số file chứa locator đó.

Ba test trên có 3 hạn chế:

| Hạn chế      | Biểu hiện                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------ |
| Lặp locator  | Cùng một locator được khai báo lại trong mọi test sử dụng phần tử đó                             |
| Lặp thao tác | Chuỗi thao tác điền email, điền password, bấm Login lặp lại trong mọi test cần đăng nhập          |
| Khó đọc hiểu | Người đọc phải phân tích từng locator để xác định bước nghiệp vụ mà test đang thực hiện          |

Bài 1 của Buổi 6 đã tách thao tác đăng nhập thành một hàm. Hàm khắc phục được hạn chế lặp thao tác, nhưng chưa khắc phục được hạn chế lặp locator: test thứ 3 không đăng nhập mà chỉ kiểm tra giao diện, nên vẫn phải khai báo lại locator. Do đó cần một cách tổ chức code tập trung **cả locator và thao tác** của một trang vào một vị trí duy nhất.

## 2. Khái niệm Page Object Model

Trong kiểm thử thủ công, các bước của test case được viết ở mức nghiệp vụ, ví dụ "Đăng nhập bằng tài khoản hợp lệ", "Kiểm tra trang tài khoản hiển thị". Test case không mô tả chi tiết giao diện như ô nhập nào, thẻ HTML nào. Chi tiết giao diện thuộc về bước thực hiện, không thuộc về kịch bản kiểm thử.

Page Object Model (POM) áp dụng nguyên tắc phân tách tương tự cho automation test. Code được chia thành hai lớp:

| Lớp                    | Nội dung                          | Vai trò                                       | Ví dụ                                                                          |
| ---------------------- | --------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------ |
| Test (`tests/`)        | Kịch bản và assertion             | Mô tả các bước nghiệp vụ và kết quả mong đợi  | Mở trang đăng nhập, đăng nhập, kiểm tra trang tài khoản hiển thị               |
| Page object (`pages/`) | Locator và thao tác của một trang | Mô tả cách thực hiện từng bước trên giao diện | Ô Email là `getByLabel("Email address")`, đăng nhập gồm điền 2 ô và bấm Login  |

Mỗi trang web tương ứng với một class, gọi là **page object**. Một page object gồm hai thành phần:

* **Locator:** các phần tử của trang.
* **Method:** các hành vi của người dùng trên trang.

Cách tổ chức này khắc phục 3 hạn chế ở mục 1:

* Locator chỉ khai báo một lần trong class. Khi giao diện thay đổi, chỉ cần cập nhật tại một vị trí.
* Thao tác chỉ viết một lần trong method. Các test sử dụng lại bằng cách gọi method.
* File test chỉ chứa các bước nghiệp vụ, có cấu trúc tương tự test case thủ công.

Về cú pháp, page object là một class gồm constructor, field và method, tương tự class `UserAccount` đã học ở [Buổi 3](../phase-1-nen-tang-typescript/buoi-03-interfaces-classes-array-methods.md) (mục 3). Cách tách class ra file riêng và `import` vào file test đã học ở Buổi 3 (mục 4).

## 3. Xây dựng LoginPage

### Bước 1: phân tích trang theo 3 câu hỏi

Trước khi viết code, mở [trang đăng nhập](https://practicesoftwaretesting.com/auth/login) và trả lời 3 câu hỏi sau. Đây là phương pháp thiết kế áp dụng cho mọi page object trong khóa học.

| Câu hỏi                                        | Trả lời cho trang đăng nhập                | Thành phần tương ứng trong class                     |
| ---------------------------------------------- | ------------------------------------------ | ---------------------------------------------------- |
| Người dùng thực hiện hành vi nào trên trang?   | Mở trang, đăng nhập bằng email và password | Method `goto()`, `login(email, password)`            |
| Hành vi đó cần thao tác với phần tử nào?       | Ô Email, ô Password, nút Login             | Locator `emailInput`, `passwordInput`, `loginButton` |
| Test cần kiểm tra thông tin nào trên trang?    | Thông báo lỗi khi đăng nhập sai            | Locator `errorMessage`                               |

Câu hỏi thứ nhất là cơ sở của thiết kế: page object được xây dựng theo **hành vi của người dùng**, không theo danh sách phần tử trên trang. Page object chỉ khai báo những phần tử test đang cần. Khi có test mới cần thêm phần tử, bổ sung vào class tại thời điểm đó.

### Bước 2: tạo class và khai báo locator

Tạo folder `pages/` ngang hàng với `tests/`, sau đó tạo file `pages/login-page.ts` với các locator đã xác định ở bước 1:

```typescript
import { Page, Locator } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel("Email address");
    // Không dùng getByLabel("Password"): link "Forgot your Password?" cũng khớp (Buổi 5, mục 12)
    this.passwordInput = page.getByRole("textbox", { name: "Password" });
    this.loginButton = page.getByRole("button", { name: "Login" });
    this.errorMessage = page.getByText("Invalid email or password");
  }
}
```

So với class ở Buổi 3, có ba điểm mới:

| Thành phần                 | Ý nghĩa                                                                                                                                 |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `import { Page, Locator }` | Hai kiểu dữ liệu của Playwright: `Page` là trang trình duyệt, `Locator` là kết quả của các hàm `getBy...()`                              |
| `readonly`                 | Field chỉ được gán giá trị một lần trong constructor và không thể gán lại. Locator của trang không thay đổi trong suốt quá trình chạy test |
| `constructor(page: Page)`  | Nhận `page` do test truyền vào. Mỗi test có một `page` riêng, page object sử dụng `page` đó để tạo locator và thực hiện thao tác         |

Locator được tạo trong constructor nhưng chưa tìm phần tử tại thời điểm đó. Playwright chỉ tìm phần tử khi có action hoặc assertion sử dụng locator (Buổi 6, mục 4.2), vì vậy có thể tạo page object trước khi mở trang.

### Bước 3: khai báo method theo hành vi của người dùng

Thêm hai method đã xác định ở bước 1 vào class, sau constructor:

```typescript
  async goto() {
    await this.page.goto("/auth/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }
```

Method sử dụng lại các locator đã khai báo thông qua `this`. Tên method mô tả hành vi (`login`), không mô tả thao tác kỹ thuật (`fillAndClick`), giúp người đọc test nhận biết ngay mục đích của bước. Test gọi `goto()` mà không cần biết URL của trang.

## 4. Sử dụng page object trong test

Viết lại 3 test ở mục 1 trong file `tests/login.spec.ts`:

```typescript
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/login-page";

test("đăng nhập thành công", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login("customer@practicesoftwaretesting.com", "welcome01");

  await expect(page).toHaveURL(/\/account/);
});

test("đăng nhập thất bại với email chưa đăng ký", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login("not-registered@practicesoftwaretesting.com", "welcome01");

  await expect(loginPage.errorMessage).toBeVisible();
});

test("form đăng nhập hiển thị đủ ô nhập và nút Login", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();

  await expect(loginPage.emailInput).toBeVisible();
  await expect(loginPage.passwordInput).toBeVisible();
  await expect(loginPage.loginButton).toBeVisible();
});
```

{% hint style="warning" %}
Tài khoản khách hàng của Toolshop bị khóa sau 3 lần đăng nhập sai password, và tài khoản bị khóa thì cả lớp không sử dụng được nữa. Không nhập password sai cho `customer@practicesoftwaretesting.com`. Với trường hợp sai thông tin đăng nhập, dùng một email chưa đăng ký như ví dụ trên.
{% endhint %}

{% hint style="info" %}
Ví dụ trên ghi email và password trực tiếp trong test để tập trung vào page object. [Buổi 8](buoi-08-pom-code-review.md) trình bày cách đưa tài khoản ra file `.env`.
{% endhint %}

Xét lại tình huống ở mục 1: nhãn "Email address" được đổi thành "Email".

| Tiêu chí                       | Test viết trực tiếp (mục 1)                         | Test sử dụng `LoginPage`                                   |
| ------------------------------ | --------------------------------------------------- | ---------------------------------------------------------- |
| Vị trí khai báo locator ô Email | Mọi test sử dụng ô Email                            | Một dòng trong constructor của `LoginPage`                 |
| Phạm vi cần sửa                | Mọi test, trong mọi file                            | Một dòng: `this.emailInput = page.getByLabel("Email")`     |
| Số file test cần sửa           | Tất cả                                              | Không có                                                   |
| Khả năng đọc hiểu              | Phải phân tích từng locator để xác định bước nghiệp vụ | Đọc như test case: mở trang, đăng nhập, kiểm tra kết quả   |

Chi tiết thao tác trên giao diện được đóng gói trong class. Người đọc test nắm được nghiệp vụ mà không cần biết cấu trúc giao diện.

## 5. Quy ước thiết kế page object

### Assertion đặt trong test, không đặt trong page object

Xét ví dụ đặt assertion trong method `login()`:

```typescript
// Chưa đúng: method login() tự kiểm tra kết quả
async login(email: string, password: string) {
  await this.emailInput.fill(email);
  await this.passwordInput.fill(password);
  await this.loginButton.click();
  await expect(this.page).toHaveURL(/\/account/);
}
```

Với cách viết này, test "đăng nhập thất bại" ở mục 4 không sử dụng được `login()`: method luôn yêu cầu trang chuyển sang trang tài khoản, nên test fail ngay trong method, trước khi kiểm tra thông báo lỗi. Method chứa assertion chỉ sử dụng được cho một kết quả mong đợi duy nhất.

Quy ước của khóa học: page object chỉ **thực hiện thao tác** và **cung cấp locator**, việc kết luận đúng hay sai là trách nhiệm của test. Một số dự án áp dụng cách khác, nhưng đây là cách phổ biến nhất.

### Các quy ước khác

| Quy ước                                 | Lý do                                                                                          | Ví dụ                                                                            |
| --------------------------------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Đặt tên method theo hành vi người dùng  | Test có cấu trúc như kịch bản nghiệp vụ                                                        | `login()`, `addToCart()`, không dùng `clickButton1()`, `fillInput()`             |
| Mỗi trang một file                      | Dễ xác định vị trí locator của từng trang                                                      | `login-page.ts` chứa `LoginPage`, `product-page.ts` chứa `ProductPage`           |
| Chọn locator theo thứ tự ưu tiên của Buổi 5 | Locator kém ổn định trong class khiến mọi test sử dụng class đó fail                       | Ưu tiên `getByRole`, `getByLabel`, `getByText` (Buổi 5, mục 7), không dùng XPath |
| Thành phần chỉ dùng nội bộ khai báo `private` | Tránh để test gọi các thành phần không dành cho test (Buổi 3, mục 3)                     | Locator chỉ được sử dụng bên trong một method của class                          |

## 6. Bài tập về nhà

Toàn bộ bài tập làm trên [Toolshop](https://practicesoftwaretesting.com/). Không nhập password sai cho tài khoản chung (xem cảnh báo ở mục 4).

Với mỗi page object, áp dụng phương pháp ở mục 3: mở trang trên trình duyệt, trả lời 3 câu hỏi phân tích trang, sau đó khai báo locator và viết method.

### Bài 1: HomePage và ProductPage

Xây dựng hai page object `HomePage` (file `pages/home-page.ts`) và `ProductPage` (file `pages/product-page.ts`). Trang chủ hiển thị danh sách sản phẩm, nút "Add to cart" chỉ có ở trang chi tiết sản phẩm. Hai page object phải cho phép test:

| Page object   | Test cần thực hiện được                                                                                                |
| ------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `HomePage`    | Mở trang chủ. Mở trang chi tiết của một sản phẩm theo tên                                                              |
| `ProductPage` | Thêm sản phẩm vào giỏ với số lượng tùy chọn. Mở giỏ hàng. Kiểm tra tên sản phẩm đang xem và số trên biểu tượng giỏ hàng |

Tạo file `tests/add-to-cart.spec.ts`, viết một test sử dụng hai page object trên:

1. Mở trang chủ, mở trang chi tiết của sản phẩm "Combination Pliers".
2. Thêm sản phẩm vào giỏ.
3. Kiểm tra số trên biểu tượng giỏ hàng là 1.

Các đặc điểm của Toolshop cần lưu ý:

| Đặc điểm                                                                                 | Kiến thức liên quan                                              |
| ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Tên sản phẩm khớp một phần: "Pliers" khớp cả "Combination Pliers", "Long Nose Pliers"... | Tham số `exact: true` (Buổi 5, mục 7)                            |
| Trang chủ có nhiều thẻ sản phẩm, cần chọn đúng một thẻ theo tên                          | Chaining và `filter()` (Buổi 5, mục 9)                           |
| Số trên biểu tượng giỏ hàng là thẻ `<span>` không có role                                | Thứ tự ưu tiên locator khi phần tử không có role (Buổi 5, mục 7) |
| Sau khi thêm vào giỏ, số trên biểu tượng giỏ hàng cập nhật sau một khoảng thời gian      | Assertion neo trước khi sang bước tiếp theo (Buổi 6, mục 2.3)    |

{% hint style="info" %}
Thanh menu, bao gồm biểu tượng giỏ hàng, xuất hiện ở mọi trang nên không thuộc riêng page object nào. Trong buổi này, các locator đó được đặt tạm trong page object đang sử dụng. [Buổi 8](buoi-08-pom-code-review.md) trình bày cách tập trung phần dùng chung vào `BasePage`.
{% endhint %}

### Bài 2: CartPage

Xây dựng page object `CartPage` trong file `pages/cart-page.ts` cho trang giỏ hàng (mở bằng biểu tượng giỏ trên thanh menu). Theo kịch bản của Bài 3, page object này phải cho phép test:

* Đọc danh sách tên sản phẩm đang có trong giỏ.
* Đọc tổng tiền ở ô Total.
* Xóa một sản phẩm khỏi giỏ theo tên sản phẩm.
* Bắt đầu checkout (nút "Proceed to checkout").

{% hint style="info" %}
Nút xóa sản phẩm trong giỏ là một phần tử không có accessible name. Xem lại cách xử lý phần tử thiếu tên ở Buổi 5 (mục 5 và mục 7).
{% endhint %}

### Bài 3: test giỏ hàng sử dụng page objects

Tạo file `tests/cart.spec.ts`, viết một test sử dụng `LoginPage`, `HomePage`, `ProductPage` và `CartPage`:

1. Đăng nhập bằng tài khoản chung.
2. Thêm "Combination Pliers" với số lượng 2 và thêm một sản phẩm khác với số lượng 1.
3. Mở giỏ hàng, kiểm tra giỏ có đúng 2 sản phẩm, đúng tên.
4. Xóa "Combination Pliers" khỏi giỏ.
5. Kiểm tra giỏ còn đúng 1 sản phẩm, tổng tiền bằng giá của sản phẩm còn lại.

Yêu cầu chung:

* File test không dùng locator trực tiếp, mọi thao tác và locator đi qua page object.
* Không có assertion nào nằm trong page object.
* Không dùng XPath, không dùng `page.waitForTimeout()`.

### Bài 4: nộp bài qua Pull Request

Commit và tạo Pull Request trên nhánh `your-name/lesson-7` (ví dụ `linh/lesson-7`) theo quy trình ở mục 5 và mục 6 của [Buổi 2](../phase-1-nen-tang-typescript/buoi-02-git-async-await.md). Commit message ngắn gọn bằng tiếng Anh, ví dụ `lesson 7: add cart page object`.

**Checklist trước khi nộp:**

* [ ] Test pass khi chạy `npx playwright test tests/add-to-cart.spec.ts tests/cart.spec.ts`
* [ ] File test không dùng locator trực tiếp, mọi thao tác đi qua page object
* [ ] Tên method mô tả hành vi người dùng
* [ ] Không có assertion nào nằm trong page object
* [ ] Mỗi page object nằm trong một file riêng trong folder `pages/`, tên file kebab-case
* [ ] Không có XPath, không có `page.waitForTimeout()`
* [ ] Đúng quy ước tên nhánh `your-name/lesson-7`
