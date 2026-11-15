---
description: "Tách phần dùng chung vào BasePage, hoàn thiện bộ page objects cho luồng mua hàng, refactor test cũ sang POM và đưa tài khoản đăng nhập ra file .env."
icon: clipboard-check
---

# Buổi 8 · Hoàn thiện Page Object Model

{% hint style="info" %}
**Sau buổi này bạn sẽ:**

* Xác định phần lặp lại giữa các page object và tách phần đó vào `BasePage` bằng kế thừa class.
* Thiết kế được page object cho một trang gồm nhiều bước, áp dụng cho trang checkout của [Toolshop](https://practicesoftwaretesting.com/).
* Refactor test cũ sang POM theo quy trình 3 bước an toàn.
* Đưa tài khoản đăng nhập ra file `.env` và đọc qua biến môi trường, không ghi password trong code.
{% endhint %}

Toàn bộ ví dụ và bài tập của buổi này chạy trên [Toolshop](https://practicesoftwaretesting.com/).

Buổi 7 đã xây dựng 4 page object cho luồng mua hàng. Nội dung buổi này được trình bày theo trình tự sau:

| Mục | Nội dung                          | Mục tiêu                                                       |
| --- | --------------------------------- | -------------------------------------------------------------- |
| 1   | Code lặp lại giữa các page object | Xác định phần code lặp lại giữa các page object                |
| 2   | `BasePage` và kế thừa class       | Tập trung phần dùng chung vào class cha                        |
| 3   | Thực hành tại lớp: `CheckoutPage` | Thiết kế page object cho một trang gồm nhiều bước              |
| 4   | Refactor an toàn                  | Chuyển test cũ sang POM mà không làm thay đổi hành vi của test |
| 5   | Tách tài khoản đăng nhập khỏi code | Quản lý tài khoản và địa chỉ trang bằng biến môi trường        |

## 1. Code lặp lại giữa các page object

So sánh `LoginPage` với một page object khác trong folder `pages/` của bài làm Buổi 7, ví dụ `HomePage`. Các phần thường lặp lại:

| Phần lặp lại                                           | Vị trí xuất hiện                                                  |
| ------------------------------------------------------ | ----------------------------------------------------------------- |
| Field `readonly page: Page` và dòng `this.page = page` | Constructor của mọi page object                                   |
| Locator của thanh menu và biểu tượng giỏ hàng          | Mọi page object cần mở giỏ hàng hoặc chuyển trang qua thanh menu  |

Thanh menu là trường hợp điển hình. Menu xuất hiện ở mọi trang, do đó mỗi page object mới lại khai báo lặp lại locator của menu. Khi biểu tượng giỏ hàng thay đổi, cần cập nhật ở nhiều class.

Đây là vấn đề lặp code đã phân tích ở mục 1 của [Buổi 7](buoi-07-page-object-model.md), ở cấp độ page object thay vì cấp độ test. Hướng giải quyết tương tự: tập trung phần dùng chung vào một vị trí duy nhất.

## 2. BasePage và kế thừa class

Kế thừa class (đã học tại lớp) cho phép class con sử dụng mọi field và method của class cha, chỉ cần khai báo thêm phần riêng. Trong POM, kế thừa được áp dụng như sau: class cha `BasePage` chứa phần có ở mọi trang, mọi page object kế thừa `BasePage`.

```
BasePage           page, phần có ở mọi trang
├── LoginPage      phần riêng của trang đăng nhập
├── HomePage       phần riêng của trang chủ
├── ProductPage    phần riêng của trang chi tiết sản phẩm
├── CartPage       phần riêng của trang giỏ hàng
└── CheckoutPage   phần riêng của trang checkout
```

Mục này minh họa với `LoginPage`. Các page object còn lại được chuyển đổi trong bài tập về nhà.

### Bước 1: tạo BasePage

Tạo file `pages/base-page.ts`, chứa phần lặp lại đầu tiên trong bảng ở mục 1:

```typescript
import { Page } from "@playwright/test";

export class BasePage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }
}
```

### Bước 2: cho LoginPage kế thừa BasePage

So với `LoginPage` của Buổi 7, có 3 thay đổi:

| Vị trí         | Buổi 7                    | Sau khi kế thừa                              |
| -------------- | ------------------------- | -------------------------------------------- |
| Khai báo class | `export class LoginPage`  | `export class LoginPage extends BasePage`    |
| Field `page`   | `readonly page: Page;`    | Xóa, field đã được khai báo trong `BasePage` |
| Constructor    | `this.page = page;`       | `super(page);`                               |

File `pages/login-page.ts` sau khi sửa:

```typescript
import { Page, Locator } from "@playwright/test";
import { BasePage } from "./base-page";

export class LoginPage extends BasePage {
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    super(page);
    this.emailInput = page.getByLabel("Email address");
    // Không dùng getByLabel("Password"): link "Forgot your Password?" cũng khớp (Buổi 5, mục 12)
    this.passwordInput = page.getByRole("textbox", { name: "Password" });
    this.loginButton = page.getByRole("button", { name: "Login" });
    this.errorMessage = page.getByText("Invalid email or password");
  }

  async goto() {
    await this.page.goto("/auth/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }
}
```

Các từ khóa kế thừa được sử dụng:

| Từ khóa                                  | Ý nghĩa                                                                                                                          |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `extends BasePage`                       | `LoginPage` kế thừa mọi field và method của `BasePage` (hiện tại là field `page`)                                                |
| `super(page)`                            | Gọi constructor của `BasePage` để gán `page`. Phải gọi `super(...)` trước khi dùng `this` trong constructor của class con         |
| `import { BasePage } from "./base-page"` | Hai file cùng nằm trong folder `pages/` nên đường dẫn bắt đầu bằng `./`                                                           |

### Bước 3: chạy lại test của Buổi 7

```bash
npx playwright test tests/login.spec.ts
```

Test pass mà không cần sửa dòng nào trong file test, vì các thành phần test sử dụng (`goto()`, `login()`, các locator) giữ nguyên tên và tham số, chỉ cấu trúc bên trong class thay đổi. Thay đổi cấu trúc code mà không thay đổi hành vi được gọi là **refactor**, trình bày ở mục 4.

### Phạm vi của BasePage

Phần dùng chung thứ hai trong bảng ở mục 1 (thanh menu và biểu tượng giỏ hàng) cũng thuộc `BasePage`. Phần này được chuyển trong bài tập về nhà.

{% hint style="warning" %}
Chỉ đưa vào `BasePage` những gì có ở **mọi** trang, ví dụ thanh menu và biểu tượng giỏ hàng. Locator chỉ có ở một trang (nút "Add to cart", ô Email) đặt trong page object của trang đó. `BasePage` chứa nhiều thành phần không liên quan đến nhau sẽ khó bảo trì không kém code lặp lại.
{% endhint %}

## 3. Thực hành tại lớp: CheckoutPage

Trang cuối cùng của luồng mua hàng là trang checkout. Sau khi bấm "Proceed to checkout" ở giỏ hàng, Toolshop chuyển qua 3 bước trên cùng một URL `/checkout`:

| Bước            | Nội dung trên trang                                                                                         |
| --------------- | ----------------------------------------------------------------------------------------------------------- |
| Sign in         | Khi đã đăng nhập, trang hiển thị thông báo "you are already logged in" và nút "Proceed to checkout"         |
| Billing Address | Dropdown Country, các ô Postal code, House number, Street, City, State và nút "Proceed to checkout"         |
| Payment         | Dropdown Payment Method, nút "Check payment", sau đó là nút "Confirm" và thông báo "Thanks for your order!" |

<!-- TODO(image): trang https://practicesoftwaretesting.com/checkout ở bước Billing Address (thanh tiến trình Cart, Sign in, Billing Address, Payment ở trên cùng), đã chọn Country và điền Postal code, House number, các ô Street, City, State đã được tự điền. Khoanh đỏ thanh tiến trình 4 bước, khoanh các ô Street, City, State được tự điền và nút "Proceed to checkout". -->

### Xác định phạm vi page object

Câu hỏi thiết kế đầu tiên đối với trang này là phạm vi của page object: mỗi bước là một page object riêng, hay cả 3 bước thuộc cùng một page object. Ba bước dùng chung một URL, chung thanh tiến trình và lần lượt hiển thị trên cùng một trang. Theo quy ước mỗi trang một page object (Buổi 7, mục 5), cả 3 bước thuộc một page object `CheckoutPage` (file `pages/checkout-page.ts`, kế thừa `BasePage`).

### Các bước thực hiện

1. Đăng nhập, thêm một sản phẩm vào giỏ và thực hiện cả 3 bước checkout trên trình duyệt.
2. Trả lời 3 câu hỏi thiết kế (Buổi 7, mục 3) cho từng bước.
3. Xây dựng `CheckoutPage` kế thừa `BasePage`.
4. Viết test trong file `tests/checkout.spec.ts` cho toàn bộ luồng mua hàng bằng các page object đã có, kết thúc với thông báo "Thanks for your order!".

Sau khi hoàn thành, đối chiếu kết quả với các yêu cầu sau. Thông qua `CheckoutPage`, test cần thực hiện được:

* Chuyển tiếp ở bước Sign in khi đã đăng nhập.
* Điền Billing Address và chuyển sang bước Payment.
* Thanh toán bằng một phương thức cho trước và xác nhận đặt hàng.
* Kiểm tra thông báo đặt hàng thành công.

{% hint style="info" %}
**Ô địa chỉ tự điền:** sau khi chọn Country và điền Postal code, House number, Toolshop tự điền Street, City và State. Nút "Proceed to checkout" ở bước này bị vô hiệu hóa (disabled) cho đến khi các ô bắt buộc có giá trị. `click()` tự chờ nút chuyển sang enabled (Buổi 6, mục 4.2), nên không cần thêm lệnh chờ. Nếu điền tay Street, City, State, giá trị tự điền có thể ghi đè lên giá trị đã nhập.
{% endhint %}

Sau phần thực hành và bài tập về nhà, cấu trúc project có dạng một framework nhỏ:

```
tests/
  login.spec.ts
  cart.spec.ts
  checkout.spec.ts
pages/
  base-page.ts
  login-page.ts
  home-page.ts
  product-page.ts
  cart-page.ts
  checkout-page.ts
```

## 4. Refactor an toàn: quy trình 3 bước

Refactor là thay đổi cách tổ chức code mà **không thay đổi hành vi** của code. Bước 2 ở mục 2 là một ví dụ refactor nhỏ: cấu trúc bên trong `LoginPage` thay đổi, test vẫn pass như trước. Với thay đổi nhỏ, chạy lại test một lần là đủ để kiểm chứng. Với thay đổi lớn, ví dụ chuyển toàn bộ bài tập Buổi 6 sang POM, nếu sửa nhiều phần cùng lúc thì khi test fail sẽ khó xác định nguyên nhân. Quy trình an toàn gồm 3 bước:

1. **Chạy toàn bộ test hiện có và xác nhận tất cả đang pass.** Đây là trạng thái chuẩn để so sánh sau khi refactor.
2. **Chuyển đổi từng phần nhỏ.** Chuyển một phần, chạy lại test ngay. Test pass mới chuyển phần tiếp theo. Không sửa toàn bộ project rồi mới chạy test.
3. **Chỉ xóa code cũ sau khi code mới đã pass.** Commit sau mỗi phần chuyển đổi thành công, để có thể quay lại trạng thái gần nhất khi phát sinh lỗi.

Ví dụ: chuyển các test đăng nhập của Buổi 6 sang sử dụng `LoginPage`.

| Bước | Thao tác                                                                                                          | Lệnh                                                   |
| ---- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 1    | Chạy toàn bộ test, xác nhận tất cả pass                                                                           | `npx playwright test`                                  |
| 2    | Trong một file test, thay phần đăng nhập bằng `loginPage.goto()` và `loginPage.login(...)`, chạy lại file đó      | `npx playwright test tests/<tên-file>.spec.ts`         |
| 3    | Test pass thì commit                                                                                              | `git commit -m "refactor login test to use LoginPage"` |
| 4    | Xóa hàm đăng nhập cũ của Buổi 6 khi không còn test nào sử dụng, chạy lại toàn bộ test, commit                     | `npx playwright test`                                  |

Lặp lại bước 2 và bước 3 cho các file test tiếp theo.

{% hint style="success" %}
**Mẹo:** mỗi bước refactor là một commit nhỏ, ví dụ `refactor login test to use LoginPage`. Pull Request gồm nhiều commit nhỏ dễ review hơn so với một commit lớn chứa mọi thay đổi. Quy trình commit đã học ở [Buổi 2](../phase-1-nen-tang-typescript/buoi-02-git-async-await.md) (mục 5).
{% endhint %}

## 5. Tách tài khoản đăng nhập khỏi code

Test đăng nhập ở Buổi 7 truyền email và password trực tiếp:

```typescript
await loginPage.login("customer@practicesoftwaretesting.com", "welcome01");
```

Cách viết này có 3 hạn chế:

| Hạn chế            | Biểu hiện                                                                                                        |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| Lộ mật khẩu        | File test được push lên Git, mọi người xem được repo đều đọc được password                                       |
| Lặp dữ liệu        | Email và password lặp lại trong mọi test cần đăng nhập. Đổi tài khoản phải sửa từng file                         |
| Khó đổi môi trường | Môi trường test và staging thường dùng tài khoản, địa chỉ khác nhau. Chạy trên môi trường khác phải sửa code     |

Hướng giải quyết: code chỉ ghi **tên** của giá trị cần dùng, ví dụ `USER_EMAIL`. Giá trị thật đặt trong file `.env` trên máy của từng người, và file này không được push lên Git.

**Biến môi trường** (environment variable) là cặp `TÊN=giá trị` nằm ngoài code, do hệ điều hành, terminal hoặc hệ thống CI cung cấp cho chương trình đang chạy. Trong Node.js, mọi biến môi trường nằm trong object `process.env`. Giá trị luôn có kiểu `string`, hoặc `undefined` nếu biến không tồn tại.

### Bước 1: tạo file .env và .env.example

Tạo file `.env` ở thư mục gốc project, cùng cấp với `playwright.config.ts`:

```
BASE_URL=https://practicesoftwaretesting.com

# Tài khoản customer dùng chung cho test đăng nhập
USER_EMAIL=customer@practicesoftwaretesting.com
USER_PASSWORD=welcome01
```

Mỗi dòng là một biến dạng `TÊN=giá trị`, không có khoảng trắng quanh dấu `=`, không cần dấu ngoặc kép. Dòng bắt đầu bằng `#` là comment. Tên biến viết hoa, các từ nối bằng dấu gạch dưới.

Thêm dòng sau vào file `.gitignore` để Git bỏ qua file `.env`:

```
.env
```

Người khác clone project về cần biết phải khai báo những biến nào. Tạo thêm file `.env.example` có cùng tên biến nhưng dùng giá trị mẫu:

```
BASE_URL=https://practicesoftwaretesting.com

# Tài khoản customer dùng chung cho test đăng nhập
USER_EMAIL=your-email@example.com
USER_PASSWORD=your-password
```

| File           | Push lên Git | Nội dung                   |
| -------------- | ------------ | -------------------------- |
| `.env`         | Không        | Giá trị thật, gồm password |
| `.env.example` | Có           | Tên biến và giá trị mẫu    |

Người mới clone project copy `.env.example` thành `.env` rồi điền giá trị thật. Khi thêm biến mới vào `.env`, thêm cả biến đó vào `.env.example`.

<!-- TODO(image): cửa sổ Explorer của VS Code ở thư mục gốc project, hiển thị .env, .env.example, .gitignore, playwright.config.ts và folder utils đang mở với file env.ts. File .env hiển thị màu xám do đã được .gitignore bỏ qua. Khoanh đỏ file .env (màu xám) và file .env.example để so sánh. -->

### Bước 2: nạp .env trong playwright.config.ts

Node.js không tự đọc file `.env`. Cài thư viện `dotenv`:

```bash
npm install -D dotenv
```

Mở `playwright.config.ts`, thêm phần nạp `.env` ở đầu file và đọc `BASE_URL` trong khối `use`:

```typescript
import { defineConfig, devices } from "@playwright/test";
import dotenv from "dotenv";
import path from "path";

// Đọc các biến trong file .env vào process.env
dotenv.config({ path: path.resolve(__dirname, ".env") });

export default defineConfig({
  // Các thiết lập hiện có giữ nguyên
  use: {
    baseURL: process.env.BASE_URL || "https://practicesoftwaretesting.com",
  },
});
```

| Lệnh                                | Ý nghĩa                                                                                     |
| ----------------------------------- | ------------------------------------------------------------------------------------------- |
| `dotenv.config(...)`                | Đọc từng dòng trong `.env` và gán vào `process.env`                                         |
| `path.resolve(__dirname, ".env")`   | File `.env` nằm cùng thư mục với file config. Chạy lệnh test từ thư mục nào cũng tìm đúng file |
| `process.env.BASE_URL \|\| "..."`   | Dùng `BASE_URL` trong `.env`. Nếu biến không tồn tại, dùng địa chỉ mặc định                  |

Playwright đọc config trước khi chạy test, nên chỉ cần gọi `dotenv.config()` một lần trong config. Mọi file test đều đọc được `process.env` mà không cần import `dotenv`.

### Bước 3: đọc biến qua utils/env.ts

Có thể dùng trực tiếp `process.env.USER_EMAIL` trong test, nhưng sẽ gặp 2 vấn đề:

1. Kiểu của `process.env.USER_EMAIL` là `string | undefined`, trong khi `login(email: string, password: string)` chỉ nhận `string`. VS Code báo lỗi `Argument of type 'string | undefined' is not assignable to parameter of type 'string'`.
2. Thiếu file `.env` hoặc gõ sai tên biến thì test vẫn chạy với giá trị `undefined` và fail ở bước điền form, với thông báo lỗi không liên quan tới file `.env`.

Giải pháp là gom việc đọc biến vào một file duy nhất. Trước tiên, khai báo kiểu dữ liệu cho tài khoản trong file `types/user.ts` (type alias đã học ở [Buổi 3](../phase-1-nen-tang-typescript/buoi-03-interfaces-classes-array-methods.md)):

```typescript
export type Account = {
  email: string;
  password: string;
};
```

Tạo file `utils/env.ts`:

```typescript
import { Account } from "../types/user";

// Báo lỗi rõ ràng khi thiếu biến, thay vì để test fail ở một bước khác
function getEnv(name: string): string {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Thiếu biến môi trường "${name}". Kiểm tra file .env (xem .env.example)`);
  }
  return value;
}

export const sharedCustomer: Account = {
  email: getEnv("USER_EMAIL"),
  password: getEnv("USER_PASSWORD"),
};
```

| Dòng                                   | Ý nghĩa                                                                                  |
| -------------------------------------- | ---------------------------------------------------------------------------------------- |
| `process.env[name]`                    | Lấy biến theo tên truyền vào. Dùng ngoặc vuông vì tên biến nằm trong tham số `name`      |
| `if (!value)`                          | Biến không tồn tại (`undefined`) hoặc là chuỗi rỗng                                      |
| `throw new Error(...)`                 | Dừng ngay và báo thiếu biến nào, cần sửa ở đâu                                           |
| `return value;`                        | Đến được dòng này thì `value` chắc chắn là `string`, TypeScript cũng xác định được như vậy |
| `export const sharedCustomer: Account` | Gom email và password thành một object kiểu `Account`, test chỉ cần import một tên        |

### Bước 4: sử dụng trong test

Sửa test đăng nhập thành công trong `tests/login.spec.ts`:

```typescript
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/login-page";
import { sharedCustomer } from "../utils/env";

test("đăng nhập thành công", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login(sharedCustomer.email, sharedCustomer.password);

  await expect(page).toHaveURL(/\/account/);
});
```

Luồng dữ liệu từ file `.env` đến test:

```
.env → dotenv (playwright.config.ts) → process.env → utils/env.ts → sharedCustomer → test
```

Để kiểm chứng, đổi tên file `.env` thành `.env.bak` rồi chạy lại test. Test dừng ngay với thông báo `Thiếu biến môi trường "USER_EMAIL"`. Đổi lại tên file sau khi kiểm tra.

`LoginPage.login()` vẫn nhận email và password qua tham số, không tự đọc `.env`. Test quyết định dùng tài khoản nào: test đăng nhập thất bại vẫn truyền email chưa đăng ký trực tiếp, vì đó là dữ liệu test, không phải thông tin bí mật.

| Đặt trong `.env`                                                  | Đặt trong code hoặc file test data                                         |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Thông tin bí mật: password, token                                 | Dữ liệu không bí mật: tên sản phẩm, từ khóa tìm kiếm, email chưa đăng ký   |
| Thông tin thay đổi theo môi trường: địa chỉ trang, tài khoản      | Dữ liệu giống nhau ở mọi môi trường                                        |

{% hint style="info" %}
Trên hệ thống CI không có file `.env`, vì file này không được push lên Git. Giá trị được khai báo trong phần Secrets của CI và truyền vào khi chạy test dưới dạng biến môi trường. `process.env` vẫn đọc được như bình thường, code test không cần sửa.
{% endhint %}

## 6. Bài tập về nhà

{% hint style="info" %}
Sau buổi này, bạn đã có một framework POM nhỏ theo cách tổ chức phổ biến trong dự án thực tế. Phase 3 tiếp tục mở rộng framework này: cấu hình nhiều trình duyệt, chạy song song ([Buổi 9](../phase-3-nang-cao/buoi-09-config-fixtures-parallel.md)), hooks và fixtures ([Buổi 10](../phase-3-nang-cao/buoi-10-test-suite-organization.md)) và API testing ([Buổi 11](../phase-3-nang-cao/buoi-11-api-testing.md)).
{% endhint %}

Toàn bộ bài tập làm trên [Toolshop](https://practicesoftwaretesting.com/). Không nhập password sai cho tài khoản chung: tài khoản bị khóa sau 3 lần sai và cả lớp sẽ không sử dụng được nữa.

### Bài 1: hoàn thiện bộ page objects

1. Chuyển `HomePage`, `ProductPage` và `CartPage` sang kế thừa `BasePage`, theo quy trình 3 bước ở mục 4.
2. Đưa phần dùng chung của mọi trang (thanh menu, biểu tượng giỏ hàng) vào `BasePage`.
3. Hoàn thiện `CheckoutPage` từ phần thực hành tại lớp (mục 3).

### Bài 2: refactor bài tập Buổi 6 sang POM

Refactor các test của bài tập [Buổi 6](buoi-6-actions-assertions-and-auto-wait.md) (Bài 1 đến Bài 4) sang sử dụng page objects, theo quy trình 3 bước ở mục 4. Bổ sung page object mới nếu trang được test chưa có page object.

Kết quả sau khi refactor:

* Test checkout flow (Bài 3 của Buổi 6) thực hiện toàn bộ luồng mua hàng bằng các page object `LoginPage`, `HomePage`, `ProductPage`, `CartPage`, `CheckoutPage`, kết thúc với thông báo "Thanks for your order!".
* Các test vẫn kiểm tra đúng những nội dung đã kiểm tra trước khi refactor.
* File test không dùng locator trực tiếp, không có assertion nào nằm trong page object.
* Tài khoản chung được đọc từ `.env` qua `utils/env.ts`, không xuất hiện trong file test. Pull Request có `.env.example`, không có `.env`.
