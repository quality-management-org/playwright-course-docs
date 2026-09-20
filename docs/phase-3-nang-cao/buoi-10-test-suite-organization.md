---
description: "Nhóm test bằng test.describe, điều khiển test chạy hay bỏ qua bằng test.only, test.skip, test.fixme, gắn tag để chạy theo nhóm với --grep, và viết parameterized test cho nhiều bộ dữ liệu."
icon: layer-group
---

# Buổi 10 · Tổ chức Test Suite

{% hint style="info" %}
**Sau buổi này bạn sẽ:** nhóm test theo tính năng bằng `test.describe` và đọc được cấu trúc nhóm trong report, dùng đúng `test.only`, `test.skip`, `test.fixme` trong lúc phát triển và trong code review, gắn tag `@smoke`, `@regression`, `@critical` cho test và chạy đúng nhóm cần chạy bằng `--grep`, và viết được parameterized test để một logic kiểm tra chạy với nhiều bộ dữ liệu mà không chép code.
{% endhint %}

## 1. Bộ test lớn lên thì gặp vấn đề gì

Sau Buổi 9, project của bạn có khoảng 10-15 test trong ba file `login.spec.ts`, `cart.spec.ts`, `checkout.spec.ts`, chạy trên bốn project. Mỗi lần `npx playwright test` là 40-60 lượt chạy, mất vài phút. Ở quy mô này, bốn câu hỏi bắt đầu xuất hiện hằng ngày:

| Tình huống                                                            | Nhu cầu                                                    | Công cụ trong buổi này  |
| --------------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------- |
| Report có 50 dòng, cần biết test nào thuộc tính năng nào              | Nhóm test theo tính năng, tên nhóm hiện trong report       | `test.describe`         |
| Đang sửa một test, không muốn chờ cả bộ chạy xong                     | Chỉ chạy một test, hoặc tạm bỏ qua test chưa sửa kịp       | `test.only`, `test.skip`, `test.fixme` |
| Trước khi tạo PR chỉ cần chạy các flow chính trong 1-2 phút            | Đánh dấu test theo mức độ quan trọng và chạy theo nhãn     | Tag và `--grep`         |
| Cùng một flow login cần kiểm tra với 4 loại tài khoản khác nhau       | Một logic, nhiều bộ dữ liệu, mỗi bộ là một dòng trong report | Parameterized test    |

Toàn bộ ví dụ trong buổi dùng lại `LoginPage`, `InventoryPage`, `CartPage` (Buổi 7-8) và file fixture `fixtures/test-fixtures.ts` (Buổi 9).

## 2. Nhóm test với test.describe

### Cú pháp cơ bản

`test.describe` nhận một tên nhóm và một hàm chứa các test thuộc nhóm đó:

```typescript
import { test, expect } from "../fixtures/test-fixtures";
import { LoginPage } from "../pages/login-page";

test.describe("Đăng nhập", () => {
  test("login thành công với tài khoản hợp lệ", async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login("standard_user", "secret_sauce");
    await expect(page).toHaveURL(/inventory/);
  });

  test("login thất bại khi sai password", async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login("standard_user", "wrong_password");
    await expect(loginPage.errorMessage).toContainText("do not match");
  });
});
```

Tên nhóm được ghép vào trước tên test ở mọi nơi Playwright hiển thị test: terminal, HTML report, Trace Viewer. Chạy file này, terminal in ra:

```
  ✓  1 [chromium] › login.spec.ts:5:7 › Đăng nhập › login thành công với tài khoản hợp lệ (1.2s)
  ✓  2 [chromium] › login.spec.ts:12:7 › Đăng nhập › login thất bại khi sai password (0.9s)
```

### Nhóm lồng nhau và hook theo nhóm

`test.describe` lồng được nhiều cấp. Hook `beforeEach` (Buổi 9, mục 6) khai báo bên trong một nhóm chỉ chạy cho các test của nhóm đó:

```typescript
import { test, expect } from "../fixtures/test-fixtures";

test.describe("Giỏ hàng", () => {
  test.describe("Thêm sản phẩm", () => {
    test("badge hiện số 1 sau khi thêm một sản phẩm", async ({ inventoryPage }) => {
      await inventoryPage.addProductToCart("Sauce Labs Backpack");
      await expect(inventoryPage.cartBadge).toHaveText("1");
    });
  });

  test.describe("Xóa sản phẩm", () => {
    test.beforeEach(async ({ inventoryPage }) => {
      // Chỉ các test trong nhóm "Xóa sản phẩm" cần có sẵn sản phẩm trong giỏ
      await inventoryPage.addProductToCart("Sauce Labs Backpack");
      await inventoryPage.openCart();
    });

    test("giỏ hàng trống sau khi xóa sản phẩm duy nhất", async ({ cartPage }) => {
      await cartPage.removeProduct("Sauce Labs Backpack");
      // cartItems là locator danh sách sản phẩm trong giỏ, bạn đã tạo ở bài tập Buổi 7
      await expect(cartPage.cartItems).toHaveCount(0);
    });
  });
});
```

Quy ước đặt tên của lớp từ buổi này:

| Thành phần         | Quy ước                                                                 | Ví dụ                                                  |
| ------------------ | ----------------------------------------------------------------------- | ------------------------------------------------------ |
| Tên file           | Một tính năng một file, `<feature>.spec.ts`, tiếng Anh kebab-case       | `checkout.spec.ts`                                     |
| Tên `describe`     | Danh từ chỉ tính năng hoặc màn hình, tiếng Việt                         | `"Thanh toán"`, `"Giỏ hàng"`                           |
| Tên `test`         | Câu mô tả kết quả mong đợi, đọc được mà không cần mở code                | `"hiện lỗi khi bỏ trống Zip Code"`                     |
| Độ sâu lồng nhau   | Tối đa hai cấp `describe`                                               | `"Giỏ hàng" › "Xóa sản phẩm"`                          |

{% hint style="warning" %}
`test.describe.configure({ mode: "serial" })` buộc các test trong nhóm chạy tuần tự và dừng cả nhóm khi một test fail. Cấu hình này không phải cách hợp lệ để cho test A chuẩn bị dữ liệu cho test B: quy tắc "mỗi test tự chuẩn bị trạng thái" của Buổi 9 vẫn áp dụng. Lớp không dùng `serial` trong khóa học.
{% endhint %}

### Chia nhỏ một test dài bằng test.step

Test có flow dài (login, thêm sản phẩm, checkout, xác nhận) khi fail chỉ báo dòng lỗi, khó biết đang ở giai đoạn nào. `test.step` đặt tên cho từng giai đoạn, và tên này xuất hiện trong report cùng Trace Viewer:

```typescript
import { test, expect } from "../fixtures/test-fixtures";
import { CheckoutPage } from "../pages/checkout-page";

test("mua một sản phẩm hoàn tất", async ({ authenticatedPage, inventoryPage, cartPage }) => {
  await test.step("Thêm sản phẩm vào giỏ", async () => {
    await inventoryPage.addProductToCart("Sauce Labs Backpack");
    await inventoryPage.openCart();
  });

  await test.step("Nhập thông tin nhận hàng", async () => {
    await cartPage.startCheckout();
    const checkoutPage = new CheckoutPage(authenticatedPage);
    await checkoutPage.fillCustomerInfo("Linh", "Nguyen", "70000");
    await checkoutPage.finishOrder();
  });

  await test.step("Xác nhận đơn hàng thành công", async () => {
    await expect(authenticatedPage.getByText("Thank you for your order!")).toBeVisible();
  });
});
```

`test.step` không thay thế `describe`: `describe` nhóm nhiều test, `step` chia một test. Chỉ dùng `step` khi test có từ ba giai đoạn trở lên.

## 3. Điều khiển test chạy: only, skip, fixme

| Lệnh                             | Tác dụng                                                                  | Khi dùng                                                            |
| -------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `test.only(...)`                 | Chỉ chạy test này (và các test khác cũng có `.only`) trong file            | Đang sửa một test, cần phản hồi nhanh                               |
| `test.skip(...)` dạng khai báo   | Bỏ qua test, report đánh dấu skipped                                      | Tính năng đang tắt tạm, test còn đúng nhưng chưa chạy được          |
| `test.skip(condition, reason)`   | Bỏ qua có điều kiện, đặt ở đầu thân test                                  | Test không áp dụng cho một browser hoặc một project cụ thể          |
| `test.fixme(...)`                | Như skip, nhưng mang nghĩa "test sai, cần sửa"                            | Test fail do lỗi của test, chưa kịp sửa trong PR hiện tại           |
| `test.slow()`                    | Nhân ba timeout của test                                                  | Flow dài hợp lệ, không phải để che test chậm bất thường             |
| `test.describe.skip / .only`     | Áp dụng cho cả nhóm                                                       | Cả tính năng đang tắt                                               |

### test.only

```typescript
test.only("login thành công với tài khoản hợp lệ", async ({ page }) => {
  // Chỉ test này chạy khi bạn gõ npx playwright test tests/login.spec.ts
});
```

Terminal báo `1 passed, 3 skipped`: các test còn lại trong file không chạy. `.only` chỉ dành cho máy của bạn trong lúc sửa test. Commit một file còn `.only` làm đồng nghiệp chạy bộ test và chỉ thấy một test, các test khác âm thầm bị bỏ qua. Dòng `forbidOnly: !!process.env.CI` trong config (Buổi 9) tồn tại chính vì lỗi này: trên CI, Playwright báo lỗi ngay khi còn sót `.only`.

### test.skip có điều kiện

Từ Buổi 9, bộ test chạy trên bốn project. Một số test hợp lệ trên desktop nhưng không áp dụng cho project khác. Thay vì để test fail, bỏ qua nó kèm lý do:

```typescript
test("badge giỏ hàng hiện số 1 sau khi thêm sản phẩm", async ({ inventoryPage, browserName }) => {
  test.skip(browserName === "webkit", "Lỗi đã biết trên WebKit, đang theo dõi ở ticket QA-123");

  await inventoryPage.addProductToCart("Sauce Labs Backpack");
  await expect(inventoryPage.cartBadge).toHaveText("1");
});
```

`browserName` và `isMobile` là fixture có sẵn của Playwright, giá trị do project đang chạy quyết định. Lệnh `test.skip(condition, reason)` phải đứng ở đầu thân test: các dòng phía trước nó vẫn được thực thi trước khi test bị bỏ qua.

### test.fixme

```typescript
test.fixme("xóa sản phẩm khỏi giỏ hàng cập nhật tổng tiền", async ({ cartPage }) => {
  // Locator tổng tiền đổi sau khi Saucedemo cập nhật giao diện, sửa ở PR sau
});
```

Trong report, `skip` và `fixme` đều hiện là skipped, nhưng khi đọc code hai từ này nói hai điều khác nhau: `skip` là "không cần chạy lúc này", `fixme` là "có lỗi cần sửa". Cả hai luôn phải kèm lý do: comment ngay dưới dòng khai báo, hoặc tham số `reason` với dạng có điều kiện.

{% hint style="success" %}
Bổ sung vào checklist code review của Buổi 8 hai mục: không còn `test.only` trong PR, và mọi `test.skip` hoặc `test.fixme` đều ghi rõ lý do. Trước khi commit, tìm nhanh bằng chức năng Search của VS Code với từ khóa `.only(`.
{% endhint %}

## 4. Tag và chạy test theo nhóm với --grep

### Gắn tag

Tag là nhãn gắn vào test, luôn bắt đầu bằng `@`. Tag khai báo trong object tùy chọn đặt giữa tên test và hàm test:

```typescript
test("login thành công với tài khoản hợp lệ", { tag: ["@smoke", "@critical"] }, async ({ page }) => {
  // ...
});
```

Tag đặt ở `describe` áp dụng cho mọi test bên trong:

```typescript
test.describe("Thanh toán", { tag: "@regression" }, () => {
  test("hoàn tất đơn hàng với thông tin hợp lệ", { tag: "@smoke" }, async ({ cartPage }) => {
    // Test này có cả @regression (từ describe) và @smoke
  });

  test("hiện lỗi khi bỏ trống First Name", async ({ cartPage }) => {
    // Test này chỉ có @regression
  });
});
```

Quy ước tag của lớp, áp dụng từ buổi này đến hết mini project (Buổi 14):

| Tag           | Ý nghĩa                                                                          | Số lượng dự kiến                   |
| ------------- | -------------------------------------------------------------------------------- | ---------------------------------- |
| `@smoke`      | Flow chính của sản phẩm, cả nhóm chạy xong trong 1-2 phút, chạy trước mỗi PR      | 5-10% số test                      |
| `@critical`   | Test fail thì chặn release: login, thanh toán                                    | Ít, thường trùng với `@smoke`      |
| `@regression` | Toàn bộ test kiểm tra chi tiết, chạy trước release hoặc hằng đêm                 | Phần lớn test còn lại              |
| `@api`        | Test gọi API không mở browser (Buổi 11)                                          | Theo số endpoint                   |

Cú pháp cũ đặt tag ngay trong tên test (`test("login thành công @smoke", ...)`) vẫn chạy được, nhưng lớp thống nhất dùng object `{ tag }` để tên test và tag tách bạch. Cú pháp này cần Playwright từ phiên bản 1.42; kiểm tra bằng `npx playwright --version`.

### Chạy theo tag với --grep

`--grep` nhận một biểu thức chính quy (regex) và chỉ chạy các test có tên đầy đủ (gồm tên describe, tên test và các tag) khớp với biểu thức đó:

| Lệnh                                                        | Ý nghĩa                                                            |
| ----------------------------------------------------------- | ------------------------------------------------------------------ |
| `npx playwright test --grep @smoke`                         | Chỉ chạy test có tag `@smoke`                                      |
| `npx playwright test --grep "@smoke\|@critical"`            | Chạy test có `@smoke` hoặc `@critical`                             |
| `npx playwright test --grep-invert @regression`             | Chạy mọi test trừ test có `@regression`                            |
| `npx playwright test --grep "Đăng nhập"`                    | Chạy test có chữ "Đăng nhập" trong tên describe hoặc tên test      |
| `npx playwright test --grep @smoke --project=chromium`      | Kết hợp với `--project` (Buổi 9): smoke test trên Chromium         |
| `npx playwright test --grep @smoke --list`                  | Liệt kê test khớp mà không chạy, dùng để kiểm tra tag trước khi chạy |

Trên PowerShell và Git Bash, đặt biểu thức có ký tự `|` trong dấu ngoặc kép như bảng trên. Thuộc tính `grep` trong `playwright.config.ts` đặt được bộ lọc mặc định (ví dụ `grep: /@smoke/`), nhưng lớp không dùng cách này để tránh quên rằng bộ test đang bị lọc.

HTML report hiển thị tag dưới dạng nhãn nhỏ cạnh tên test. Bấm vào nhãn để lọc report theo tag đó, hoặc gõ `@smoke` vào ô tìm kiếm phía trên danh sách.

<!-- TODO(image): HTML report sau khi chạy npx playwright test: danh sách test có nhãn tag @smoke, @critical, @regression hiện cạnh tên test; khoanh một nhãn @smoke và ô tìm kiếm phía trên đang chứa "@smoke" làm danh sách chỉ còn các test smoke. -->

## 5. Parameterized test: một logic, nhiều bộ dữ liệu

### Anti-pattern: vòng lặp bên trong một test

Cần kiểm tra bốn tình huống login thất bại của Saucedemo. Cách viết đầu tiên nhiều người nghĩ đến:

```typescript
// Anti-pattern: bốn tình huống gói trong một test
test("login thất bại với dữ liệu không hợp lệ", async ({ page }) => {
  const cases = [
    ["locked_out_user", "secret_sauce"],
    ["standard_user", "wrong_password"],
    ["", "secret_sauce"],
    ["standard_user", ""],
  ];
  for (const [username, password] of cases) {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login(username, password);
    await expect(loginPage.errorMessage).toBeVisible();
  }
});
```

Ba vấn đề: tình huống thứ hai fail thì hai tình huống sau không được chạy; report chỉ có một dòng nên không biết tình huống nào hỏng; và assertion `toBeVisible()` quá chung, không kiểm tra đúng thông báo của từng tình huống.

### Cách đúng: vòng lặp sinh ra nhiều test

Đặt vòng lặp bên ngoài `test`, mỗi vòng khai báo một test riêng. Dữ liệu mô tả bằng interface (Buổi 3) để VS Code kiểm tra từng trường:

```typescript
import { test, expect } from "../fixtures/test-fixtures";
import { LoginPage } from "../pages/login-page";

interface LoginCase {
  title: string;
  username: string;
  password: string;
  expectedError: string;
}

const invalidLoginCases: LoginCase[] = [
  {
    title: "tài khoản bị khóa",
    username: "locked_out_user",
    password: "secret_sauce",
    expectedError: "Sorry, this user has been locked out.",
  },
  {
    title: "sai password",
    username: "standard_user",
    password: "wrong_password",
    expectedError: "Username and password do not match any user in this service",
  },
  {
    title: "bỏ trống username",
    username: "",
    password: "secret_sauce",
    expectedError: "Username is required",
  },
  {
    title: "bỏ trống password",
    username: "standard_user",
    password: "",
    expectedError: "Password is required",
  },
];

test.describe("Đăng nhập thất bại", { tag: "@regression" }, () => {
  for (const data of invalidLoginCases) {
    test(`hiện thông báo lỗi khi ${data.title}`, async ({ page }) => {
      const loginPage = new LoginPage(page);
      await loginPage.goto();
      await loginPage.login(data.username, data.password);
      await expect(loginPage.errorMessage).toContainText(data.expectedError);
    });
  }
});
```

| Phần                                         | Ý nghĩa                                                                                                    |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `interface LoginCase`                        | Mỗi bộ dữ liệu bắt buộc có đủ bốn trường; thiếu trường nào VS Code báo đỏ ngay                             |
| `title`                                      | Phần mô tả ngắn để ghép vào tên test, vì tên test phải là duy nhất trong file                              |
| `for (const data of invalidLoginCases)`      | Chạy tại thời điểm Playwright đọc file, sinh ra bốn test độc lập trước khi bất kỳ test nào chạy            |
| Template string trong tên test               | Tên test chứa dữ liệu của chính nó, nhìn report biết ngay tình huống nào fail                              |
| `data.expectedError`                         | Assertion kiểm tra đúng thông báo của từng tình huống, thay cho `toBeVisible()` chung chung                |

Kiểm tra kết quả bằng `--list`:

```
npx playwright test tests/login-invalid.spec.ts --project=chromium --list

Listing tests:
  [chromium] › login-invalid.spec.ts:40:9 › Đăng nhập thất bại › hiện thông báo lỗi khi tài khoản bị khóa
  [chromium] › login-invalid.spec.ts:40:9 › Đăng nhập thất bại › hiện thông báo lỗi khi sai password
  [chromium] › login-invalid.spec.ts:40:9 › Đăng nhập thất bại › hiện thông báo lỗi khi bỏ trống username
  [chromium] › login-invalid.spec.ts:40:9 › Đăng nhập thất bại › hiện thông báo lỗi khi bỏ trống password
Total: 4 tests in 1 file
```

Bốn test này chạy song song trên bốn worker như mọi test khác, và mỗi test có một dòng riêng trong report.

### Kết hợp với fixture

Parameterized test dùng fixture của Buổi 9 như bình thường. Ví dụ kiểm tra ba sản phẩm đều thêm được vào giỏ:

```typescript
import { test, expect } from "../fixtures/test-fixtures";

const productNames = ["Sauce Labs Backpack", "Sauce Labs Bike Light", "Sauce Labs Bolt T-Shirt"];

test.describe("Thêm sản phẩm vào giỏ", () => {
  for (const productName of productNames) {
    test(`giỏ hàng chứa "${productName}" sau khi thêm`, { tag: "@smoke" }, async ({ authenticatedPage, inventoryPage }) => {
      await inventoryPage.addProductToCart(productName);
      await inventoryPage.openCart();
      await expect(authenticatedPage.getByText(productName)).toBeVisible();
    });
  }
});
```

### Khi nào nên và không nên parameterize

| Nên                                                                                  | Không nên                                                                           |
| ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| Các bước giống hệt nhau, chỉ khác dữ liệu đầu vào và kết quả mong đợi                | Các tình huống có luồng thao tác khác nhau, phải thêm `if` trong thân test           |
| Kiểm tra validation của form: thiếu trường, sai định dạng, vượt độ dài               | Chỉ có một hoặc hai tình huống, viết hai test thường dễ đọc hơn                       |
| Cùng một flow trên nhiều tài khoản có quyền khác nhau                                | Bộ dữ liệu quá lớn (hàng chục dòng) làm bộ test chạy lâu mà không thêm giá trị       |

Mảng dữ liệu hiện đặt ngay trong file test. Khi dữ liệu dùng chung cho nhiều file hoặc cần sinh ngẫu nhiên, Buổi 13 trình bày cách chuyển sang file JSON trong folder `data/` và thư viện Faker.

## 6. Tổng hợp cách chạy đúng phạm vi

| Nhu cầu                                          | Lệnh                                                        |
| ------------------------------------------------ | ----------------------------------------------------------- |
| Một file                                         | `npx playwright test tests/login.spec.ts`                   |
| Một test theo tên                                | `npx playwright test --grep "sai password"`                 |
| Một nhóm theo tag                                | `npx playwright test --grep @smoke`                         |
| Chạy lại các test fail ở lần chạy trước           | `npx playwright test --last-failed`                         |
| Xem danh sách test mà không chạy                 | `npx playwright test --list`                                |
| Một test, một browser, xem trực tiếp             | `npx playwright test --grep "sai password" --project=chromium --headed` |

Thói quen làm việc được khuyến nghị: khi viết test, chạy đúng file hoặc đúng test đang sửa trên Chromium; trước khi tạo PR, chạy `--grep @smoke` trên mọi project rồi chạy toàn bộ bộ test một lần.

## 7. Lỗi thường gặp

| Triệu chứng                                                          | Nguyên nhân                                                                      | Cách xử lý                                                                                       |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `Error: duplicate test title`                                        | Vòng lặp sinh nhiều test cùng tên                                                | Ghép dữ liệu hoặc trường `title` vào tên test bằng template string                               |
| `Error: No tests found` khi chạy `--grep @smoke`                     | Chưa có test nào gắn tag đó, hoặc gõ thiếu `@`                                   | Chạy `--grep @smoke --list` để xem test khớp; kiểm tra lại object `{ tag }`                      |
| `Error: Tag must start with "@" symbol`                              | Khai báo `tag: "smoke"` thiếu `@`                                                | Sửa thành `tag: "@smoke"`                                                                        |
| Đồng nghiệp chạy bộ test chỉ thấy 1 test                             | File commit còn sót `test.only`                                                  | Tìm `.only(` trong toàn project trước khi commit; xóa `.only`                                    |
| Test bị skip nhưng vài dòng đầu vẫn chạy                             | `test.skip(condition, reason)` đặt giữa thân test                                | Đưa dòng `test.skip` lên đầu thân test                                                           |
| Report có test skipped mà không ai biết vì sao                        | `test.skip` hoặc `test.fixme` không kèm lý do                                    | Bổ sung tham số `reason` hoặc comment ngay dưới dòng khai báo                                    |
| Test trong `describe` không nhận `beforeEach` mong đợi              | Hook khai báo ở nhóm khác hoặc ngoài nhóm                                        | Đặt hook bên trong đúng `describe`; hook ngoài mọi `describe` áp dụng cho cả file                |

## 8. Bài tập về nhà (45-60 phút)

1. Tổ chức lại ba file test hiện có: mỗi file có ít nhất một `test.describe` đặt tên theo tính năng, tối đa hai cấp lồng nhau. Tên test viết theo quy ước ở mục 2.
2. Gắn tag: `@smoke` cho tối đa 5 test thuộc flow chính (login thành công, thêm sản phẩm vào giỏ, hoàn tất đơn hàng), `@critical` cho test login thành công và test hoàn tất đơn hàng, `@regression` cho toàn bộ nhóm `describe` của checkout. Chạy `npx playwright test --grep @smoke --project=chromium` và dán dòng tổng kết vào mô tả PR.
3. Tạo `tests/login-invalid.spec.ts` với parameterized test cho bốn tình huống login thất bại như mục 5.
4. Tạo `tests/checkout-validation.spec.ts` với parameterized test cho ba tình huống bỏ trống First Name, Last Name, Zip/Postal Code ở bước nhập thông tin nhận hàng. Bổ sung locator `errorMessage` (dùng `data-test="error"`) vào `CheckoutPage` nếu chưa có. Thông báo mong đợi lần lượt là `First Name is required`, `Last Name is required`, `Postal Code is required`.
5. Chạy `npx playwright test --list`, dán dòng `Total: ... tests in ... files` vào mô tả PR. Push lên repo lớp qua Pull Request, nhánh `your-name/lesson-10`.

**Checklist trước khi nộp:** không còn `test.only` trong project · mọi `skip`/`fixme` có lý do · mỗi file test có `describe` · có ít nhất 3 test `@smoke` và lệnh `--grep @smoke` chạy pass · hai file parameterized sinh đúng 4 và 3 test trong `--list` · toàn bộ test pass trên project chromium.
