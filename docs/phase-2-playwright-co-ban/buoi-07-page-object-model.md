---
description: >-
  Tổ chức test theo chuẩn công nghiệp: mỗi trang web là một class, sửa UI chỉ
  cần sửa đúng một chỗ.
---

# Buổi 7 · Page Object Model: Khái niệm & Xây dựng

{% hint style="info" %}
**Sau buổi này bạn sẽ:** hiểu vì sao mọi công ty đều tổ chức test theo Page Object Model, và tự xây được LoginPage + InventoryPage cho Saucedemo. Đây là buổi biến bạn từ "người viết test" thành "người xây framework".
{% endhint %}

## 1. Vấn đề: test "trần" không sống nổi qua 20 test

Nhìn lại các test bạn đã viết ở Buổi 4–6 — để ý thấy gì?

```typescript
// test 1
await page.getByPlaceholder("Username").fill("standard_user");
await page.getByPlaceholder("Password").fill("secret_sauce");
await page.getByRole("button", { name: "Login" }).click();

// test 2 — y hệt
await page.getByPlaceholder("Username").fill("standard_user");
await page.getByPlaceholder("Password").fill("secret_sauce");
await page.getByRole("button", { name: "Login" }).click();

// test 3, 4, 5... — vẫn y hệt
```

Hai vấn đề chết người khi dự án lớn dần:

* **Lặp code:** 30 test đều login → 30 lần copy-paste cùng 3 dòng.
* **Sửa một, vá ba mươi:** dev đổi placeholder "Username" thành "Email" → bạn sửa 30 chỗ, sót 1 chỗ là có test fail "ma".

## 2. Page Object Model — mỗi trang web là một class

Ý tưởng của POM đơn giản đến bất ngờ: **gom mọi thứ liên quan đến 1 trang vào 1 class** — locator của trang đó, thao tác trên trang đó. Test chỉ gọi method, không đụng trực tiếp vào locator.

Kết quả: dev đổi UI → bạn sửa **đúng 1 chỗ** trong class → 30 test tự "lành". Còn nhớ class UserAccount ở Buổi 3 không? Hôm nay nó phát huy tác dụng: LoginPage cũng chỉ là một class có constructor, field và method — không có gì mới về cú pháp cả.

## 3. Xây LoginPage từng bước

Tạo folder `pages/` ngang hàng với `tests/`, rồi tạo file `pages/login-page.ts`:

```typescript
import { Page, Locator } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.usernameInput = page.getByPlaceholder("Username");
    this.passwordInput = page.getByPlaceholder("Password");
    this.loginButton = page.getByRole("button", { name: "Login" });
    this.errorMessage = page.locator("[data-test='error']");
  }

  async goto() {
    await this.page.goto("https://www.saucedemo.com");
  }

  async login(username: string, password: string) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }
}
```

Giải thích từng phần:

| Phần                      | Ý nghĩa                                                                                               |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| `export class LoginPage`  | `export` để file test import được; tên PascalCase đúng convention Buổi 3                              |
| `readonly ... : Locator`  | Khai báo trước các locator của trang; `readonly` = gán 1 lần trong constructor, không ai sửa được nữa |
| `constructor(page: Page)` | Nhận `page` từ test truyền vào — mỗi test có `page` riêng, class chỉ "mượn" dùng                      |
| `async login(...)`        | Method mô tả **hành vi người dùng**, không phải thao tác kỹ thuật — đọc tên hiểu ngay làm gì          |

## 4. Dùng POM trong test — trước và sau

```typescript
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/login-page";

test("login thành công với tài khoản hợp lệ", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login("standard_user", "secret_sauce");

  await expect(page).toHaveURL(/inventory/);
});

test("login thất bại khi sai password", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login("standard_user", "sai_password");

  await expect(loginPage.errorMessage).toContainText("Epic sadface");
});
```

So với bản "trần": test giờ đọc như một kịch bản kiểm thử — _mở trang, login, kiểm tra kết quả_. Chi tiết "điền ô nào, bấm nút nào" đã giấu gọn trong class. Người mới vào team đọc test hiểu ngay nghiệp vụ mà chưa cần biết UI.

## 5. Quy ước thiết kế Page Object của lớp

* **Method = hành vi người dùng**, đặt tên theo nghiệp vụ: `login()`, `addProductToCart(name)`, `checkout()` — không đặt kiểu kỹ thuật `clickBtn1()`, `fillInput()`.
* **Assertion để trong test, không để trong page object.** Page object chỉ _thao tác_ và _cung cấp locator_; việc _phán xét đúng sai_ là của test. (Đi làm bạn sẽ gặp team làm khác — không sao, đây là quy ước để lớp thống nhất và cũng là trường phái phổ biến nhất.)
* **Mỗi trang một file**, tên file kebab-case: `login-page.ts`, `inventory-page.ts`, `cart-page.ts`.
* Locator nào chỉ dùng nội bộ trong class → cân nhắc để `private` (kiến thức Buổi 3).

## 6. Trang thứ hai: InventoryPage

Tại lớp chúng ta xây tiếp `pages/inventory-page.ts` — khung gợi ý để bạn hoàn thiện:

```typescript
import { Page, Locator } from "@playwright/test";

export class InventoryPage {
  readonly page: Page;
  readonly title: Locator;
  readonly cartBadge: Locator;

  constructor(page: Page) {
    this.page = page;
    this.title = page.locator(".title");
    this.cartBadge = page.locator(".shopping_cart_badge");
  }

  // Thêm sản phẩm theo TÊN — dùng kỹ thuật filter đã học Buổi 5
  async addProductToCart(productName: string) {
    await this.page
      .locator(".inventory_item")
      .filter({ hasText: productName })
      .getByRole("button", { name: "Add to cart" })
      .click();
  }

  async openCart() {
    await this.page.locator(".shopping_cart_link").click();
  }
}
```

Để ý `addProductToCart(productName)` nhận tham số — một method phục vụ được mọi sản phẩm, thay vì viết 6 method cho 6 sản phẩm. Đó là sức mạnh của việc kết hợp POM + filter.

## 7. Lỗi thường gặp

| Triệu chứng                                     | Nguyên nhân                                     | Cách xử lý                                                               |
| ----------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------ |
| Lỗi "Cannot find module '../pages/login-page'"  | Sai đường dẫn import hoặc quên `export`         | Kiểm tra đường dẫn tương đối từ file test, và chữ `export` trước `class` |
| `this.page is undefined`                        | Quên truyền `page` khi tạo: `new LoginPage()`   | Luôn viết `new LoginPage(page)`                                          |
| Method chạy không làm gì cả                     | Quên `await` khi gọi: `loginPage.login(...)`    | Method async thì lời gọi phải có `await`                                 |
| Sửa locator trong class rồi mà test vẫn fail cũ | Test vẫn đang dùng locator "trần" chưa refactor | Rà lại test: mọi thao tác phải đi qua page object                        |

## 8. Bài tập về nhà (45–60 phút)

1. Hoàn thiện `InventoryPage`: thêm method `getCartCount()` trả về số hiển thị trên badge giỏ hàng (trả về `string` từ `cartBadge.textContent()` là đạt)
2. Tạo mới `pages/cart-page.ts` — class `CartPage` theo đúng pattern: locator danh sách sản phẩm trong giỏ, method `removeProduct(productName)` và `startCheckout()`
3. Viết 1 test dùng cả 3 page objects: login → thêm 2 sản phẩm → mở giỏ → xóa 1 sản phẩm → assert giỏ còn đúng 1 sản phẩm
4. Push lên repo lớp qua Pull Request, nhánh `ten-cua-ban/buoi7`

**Checklist trước khi nộp:** test không đụng trực tiếp locator nào (tất cả qua page object) · tên method theo hành vi người dùng · không có assertion nào nằm trong page object · test pass.
