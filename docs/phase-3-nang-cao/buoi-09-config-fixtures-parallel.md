---
description: "Cấu hình playwright.config.ts cho nhiều browser, chạy song song, tự chụp screenshot khi test fail, và viết custom fixture để test tự động login."
icon: sliders
---

# Buổi 9 · playwright.config.ts, Fixtures & Chạy song song

{% hint style="info" %}
**Sau buổi này bạn sẽ:** đọc hiểu và tự chỉnh được `playwright.config.ts`, chạy cùng một bộ test trên Chromium, Firefox, WebKit và màn hình điện thoại, hiểu cách Playwright chạy song song và vì sao test phải độc lập, bật screenshot và video khi test fail, và viết được custom fixture để mọi test bắt đầu từ trạng thái đã login mà không lặp code.
{% endhint %}

## 1. Đọc hiểu playwright.config.ts

File `playwright.config.ts` được tạo sẵn ở gốc project khi chạy `npm init playwright@latest` ở Buổi 4. Mọi lệnh `npx playwright test` đều đọc file này trước khi chạy. Nội dung hiện tại của bạn tương tự đoạn sau (đã lược bớt comment), trong đó dòng `testIdAttribute` là dòng bạn thêm ở Buổi 5:

```typescript
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",
  use: {
    trace: "on-first-retry",
    testIdAttribute: "data-test",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "firefox", use: { ...devices["Desktop Firefox"] } },
    { name: "webkit", use: { ...devices["Desktop Safari"] } },
  ],
});
```

| Thuộc tính            | Ý nghĩa                                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `defineConfig({...})` | Hàm bọc object cấu hình để VS Code gợi ý tên thuộc tính và báo lỗi khi gõ sai                                            |
| `testDir`             | Folder chứa file test. Playwright chỉ tìm file `*.spec.ts` trong folder này                                              |
| `fullyParallel`       | `true`: các test trong cùng một file cũng được chạy song song. Mục 3 giải thích chi tiết                                 |
| `forbidOnly`          | Khi chạy trên CI, báo lỗi nếu còn sót `test.only` (Buổi 10 trình bày `test.only`)                                        |
| `retries`             | Số lần chạy lại một test khi fail. Mục 2 giải thích                                                                      |
| `workers`             | Số process chạy test đồng thời. `undefined` để Playwright tự chọn theo số nhân CPU                                       |
| `reporter`            | Định dạng báo cáo. `html` là report bạn đã mở bằng `npx playwright show-report` từ Buổi 4                                |
| `use`                 | Các thiết lập áp dụng cho mọi test: browser, baseURL, screenshot, video... Phần lớn buổi này làm việc với khối này       |
| `projects`            | Danh sách cấu hình chạy. Mỗi project chạy toàn bộ test một lần với thiết lập riêng. Mục 4 giải thích                     |

Biểu thức `process.env.CI ? 2 : 0` đọc biến môi trường tên `CI`, thứ mà các hệ thống tích hợp liên tục (GitHub Actions, GitLab CI...) tự đặt. Khi chạy trên máy cá nhân, biến này không tồn tại nên `retries` bằng 0 và `workers` do Playwright tự chọn. Khóa học không dùng CI, nhưng giữ nguyên các dòng này không gây hại. Buổi 13 trình bày cách đọc biến môi trường từ file `.env`.

{% hint style="info" %}
Từ Buổi 4 đến nay, mỗi lần chạy `npx playwright test`, bộ test của bạn thực ra đã chạy ba lần trên ba browser, vì config mặc định có sẵn ba project. Mở lại report gần nhất bằng `npx playwright show-report`: mỗi tên test xuất hiện ba dòng, kèm nhãn chromium, firefox, webkit.
{% endhint %}

## 2. baseURL, timeout và retries

### baseURL

Hiện tại `LoginPage.goto()` (Buổi 7) ghi cứng địa chỉ `https://www.saucedemo.com`. Khi dự án có môi trường test và môi trường staging với địa chỉ khác nhau, mỗi lần đổi môi trường phải sửa mọi page object. Cách chuẩn là khai báo địa chỉ gốc một lần trong config:

```typescript
use: {
  baseURL: "https://www.saucedemo.com",
  trace: "on-first-retry",
  testIdAttribute: "data-test",
},
```

Sau đó mọi lời gọi `page.goto()` chỉ cần ghi đường dẫn tương đối. Sửa `pages/login-page.ts`:

```typescript
async goto() {
  // Playwright ghép baseURL với đường dẫn này thành địa chỉ đầy đủ
  await this.page.goto("/");
}
```

Assertion `toHaveURL` cũng chấp nhận đường dẫn tương đối: `await expect(page).toHaveURL("/inventory.html")` tương đương với địa chỉ đầy đủ. Các test đang dùng regex như `toHaveURL(/inventory/)` giữ nguyên.

### Ba loại timeout

Playwright có ba mốc thời gian chờ độc lập. Xác định đúng loại nào đang hết hạn giúp đọc thông báo lỗi nhanh hơn:

| Thuộc tính       | Mặc định       | Áp dụng cho                                                            | Vị trí khai báo             |
| ---------------- | -------------- | ---------------------------------------------------------------------- | --------------------------- |
| `timeout`        | 30 giây        | Toàn bộ một test, tính cả hook `beforeEach` (mục 6)                    | Cấp cao nhất của config     |
| `expect.timeout` | 5 giây         | Mỗi assertion có auto-wait như `toBeVisible`, `toHaveText` (Buổi 6)    | `expect: { timeout: 5000 }` |
| `actionTimeout`  | Không giới hạn | Mỗi action như `click`, `fill`                                         | Trong khối `use`            |

Lỗi `Test timeout of 30000ms exceeded` nghĩa là toàn bộ test vượt quá 30 giây, thường do một action đang chờ phần tử không bao giờ xuất hiện. Lỗi `Timed out 5000ms waiting for expect(locator).toBeVisible()` nghĩa là riêng assertion đó hết 5 giây. Trong khóa học, giữ các giá trị mặc định. Chỉ tăng `timeout` khi test có flow thực sự dài, và tăng trong config thay vì rải `{ timeout: 60000 }` vào từng dòng.

### retries

`retries: 1` yêu cầu Playwright chạy lại một lần những test bị fail. Test fail lần đầu nhưng pass khi chạy lại được report đánh dấu là **flaky** (không ổn định) thay vì passed.

{% hint style="warning" %}
Retries không sửa được test. Một test flaky trên máy của bạn sẽ flaky trên máy của cả team. Giữ `retries: 0` khi phát triển để nhìn thấy lỗi ngay, và coi mọi nhãn flaky trong report là một bug cần điều tra. Nguyên nhân phổ biến nhất là thiếu assertion neo trạng thái trước khi thao tác tiếp, đã nói ở Buổi 6 và Buổi 8.
{% endhint %}

### Ghi đè config từ dòng lệnh

Mọi thuộc tính trên đều ghi đè được ngay khi chạy, không cần sửa file. Các lệnh dùng thường xuyên:

| Lệnh                                       | Ý nghĩa                                                                  |
| ------------------------------------------ | ------------------------------------------------------------------------ |
| `npx playwright test --workers=1`          | Chạy tuần tự từng test, dùng khi nghi ngờ test phụ thuộc nhau            |
| `npx playwright test --retries=2`          | Chạy lại tối đa 2 lần các test fail                                      |
| `npx playwright test --project=chromium`   | Chỉ chạy project tên `chromium`                                          |
| `npx playwright test --repeat-each=5`      | Chạy mỗi test 5 lần liên tiếp, dùng để tái hiện test flaky               |
| `npx playwright test --timeout=60000`      | Đặt timeout cho mỗi test là 60 giây trong lần chạy này                   |
| `npx playwright test tests/login.spec.ts`  | Chỉ chạy một file                                                        |

## 3. Chạy song song với workers

### Worker là gì

Khi chạy `npx playwright test`, dòng đầu tiên trên terminal cho biết Playwright đang dùng bao nhiêu worker:

```
Running 12 tests using 4 workers
```

Mỗi worker là một process riêng của Node.js, mở browser riêng và lần lượt nhận các test để chạy. Bốn worker nghĩa là bốn test chạy cùng lúc. Mặc định Playwright dùng một nửa số nhân CPU của máy. Với bộ test gồm 12 test, mỗi test 5 giây, chạy tuần tự mất 60 giây, chạy với 4 worker mất khoảng 15 giây.

Thiết lập `fullyParallel` quyết định đơn vị được chia cho worker:

| `fullyParallel`          | Cách chia                                                                         | Kết quả                                         |
| ------------------------ | --------------------------------------------------------------------------------- | ----------------------------------------------- |
| `false`                  | Mỗi file test là một đơn vị. Các test trong cùng file chạy tuần tự theo thứ tự viết | Ít song song hơn, thứ tự trong file được giữ    |
| `true` (config mặc định) | Mỗi test là một đơn vị, worker nào rảnh nhận test tiếp theo                       | Song song tối đa, thứ tự chạy không đảm bảo     |

### Mỗi test có một browser context riêng

Dù chạy song song hay tuần tự, Playwright tạo cho mỗi test một browser context mới: tương đương một cửa sổ ẩn danh riêng với cookie, localStorage và session trống. Test A login xong, test B vẫn bắt đầu ở trạng thái chưa login. Đây là lý do kỹ thuật đằng sau mục "Mỗi test có chạy độc lập không?" trong checklist code review của Buổi 8.

Đoạn sau là anti-pattern minh họa cho test phụ thuộc nhau:

```typescript
// Anti-pattern: test thứ hai trông đợi giỏ hàng do test thứ nhất để lại
test("thêm sản phẩm vào giỏ", async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login("standard_user", "secret_sauce");
  await new InventoryPage(page).addProductToCart("Sauce Labs Backpack");
});

test("giỏ hàng có đúng 1 sản phẩm", async ({ page }) => {
  // page ở đây là context mới: chưa login, giỏ hàng trống
  await page.goto("/cart.html");
  await expect(page.locator(".cart_item")).toHaveCount(1); // luôn fail
});
```

Với `fullyParallel: true`, hai test này còn có thể chạy trên hai worker khác nhau, thứ tự không đảm bảo. Cách sửa đúng là mỗi test tự chuẩn bị trạng thái của mình: login, thêm sản phẩm, rồi mới kiểm tra. Mục 7 cho thấy cách làm việc này gọn bằng fixture.

{% hint style="success" %}
Cách kiểm tra nhanh bộ test có phụ thuộc nhau không: chạy `npx playwright test --workers=1` rồi chạy lại với `--workers=4`. Hai lần cho kết quả khác nhau là dấu hiệu có test đang dựa vào test khác.
{% endhint %}

## 4. Multiple projects: đa browser và mobile emulation

Mỗi phần tử trong mảng `projects` là một cấu hình chạy riêng, có tên và khối `use` riêng. Playwright chạy toàn bộ test một lần cho mỗi project. Thêm project giả lập điện thoại vào config:

```typescript
projects: [
  { name: "chromium", use: { ...devices["Desktop Chrome"] } },
  { name: "firefox", use: { ...devices["Desktop Firefox"] } },
  { name: "webkit", use: { ...devices["Desktop Safari"] } },
  { name: "mobile-chrome", use: { ...devices["Pixel 5"] } },
],
```

`devices` là danh sách thiết bị có sẵn của Playwright. `devices["Pixel 5"]` là một object gồm kích thước màn hình, tỉ lệ pixel, user agent và cờ cảm ứng của điện thoại Pixel 5. Dấu `...` (spread) sao chép toàn bộ thuộc tính của object đó vào khối `use`. Các tên thiết bị khác dùng được ngay: `"iPhone 13"`, `"iPad Pro 11"`, `"Galaxy S9+"`.

Sau khi thêm project, số lượt chạy trong report nhân bốn: 12 test thành 48 lượt chạy. Lệnh `--project` giới hạn phạm vi:

```bash
# Chỉ chạy trên điện thoại
npx playwright test --project=mobile-chrome

# Chạy trên hai project, cờ này lặp lại được
npx playwright test --project=chromium --project=firefox

# Xem browser giả lập điện thoại hoạt động
npx playwright test --project=mobile-chrome --headed
```

Firefox và WebKit cần được tải về máy. Nếu ở Buổi 4 bạn đã trả lời Yes cho câu hỏi "Install Playwright browsers", cả ba đã có sẵn. Nếu chưa, chạy `npx playwright install`.

<!-- TODO(image): HTML report sau khi chạy npx playwright test với 4 project: danh sách test cho thấy cùng một tên test "login thành công với tài khoản hợp lệ" lặp lại 4 dòng với nhãn chromium, firefox, webkit, mobile-chrome; khoanh các nhãn project ở đầu mỗi dòng và bộ lọc project ở phía trên danh sách. -->

{% hint style="info" %}
Khi đang viết test, chạy `--project=chromium` để có phản hồi nhanh. Chạy đủ mọi project trước khi tạo PR. Lỗi chỉ xảy ra trên một browser (ví dụ WebKit xử lý khác một kiểu input) là loại lỗi có giá trị nhất mà việc chạy đa browser mang lại.
{% endhint %}

## 5. Screenshot, video và trace khi test fail

Khi test fail trên máy của người khác hoặc trong lần chạy ban đêm, dòng lỗi trên terminal thường không đủ để biết trang web khi đó trông thế nào. Ba thiết lập trong khối `use` giải quyết việc này:

```typescript
use: {
  baseURL: "https://www.saucedemo.com",
  testIdAttribute: "data-test",
  screenshot: "only-on-failure",
  video: "retain-on-failure",
  trace: "on-first-retry",
},
```

| Thuộc tính   | Giá trị nên dùng      | Các giá trị khác                                | Kết quả                                                                   |
| ------------ | --------------------- | ----------------------------------------------- | ------------------------------------------------------------------------- |
| `screenshot` | `"only-on-failure"`   | `"off"` (mặc định), `"on"`                      | Ảnh chụp màn hình tại thời điểm test fail                                 |
| `video`      | `"retain-on-failure"` | `"off"` (mặc định), `"on"`, `"on-first-retry"`  | Video toàn bộ test, chỉ giữ lại file của test fail                        |
| `trace`      | `"on-first-retry"`    | `"off"`, `"on"`, `"retain-on-failure"`          | Bản ghi chi tiết từng action, DOM và network, mở bằng Trace Viewer        |

Các file được lưu trong `test-results/`, mỗi test fail một folder con, và gắn vào report dưới mục Attachments của test đó. Folder `test-results/` và `playwright-report/` đã nằm trong `.gitignore` từ Buổi 4. Kiểm tra lại bằng `git status` trước khi commit để chắc chắn không có file ảnh hoặc video nào được đưa lên PR.

Thử ngay tại lớp: sửa tạm password trong một test login thành `"wrong_password"`, chạy `npx playwright test --project=chromium tests/login.spec.ts`, mở `npx playwright show-report`, bấm vào test fail và xem ảnh chụp cùng video. Sửa lại password sau khi xem xong.

<!-- TODO(image): Trang chi tiết một test fail trong HTML report: phần Errors ở trên, phần Attachments ở dưới gồm mục "screenshot" có ảnh trang login Saucedemo báo lỗi "Epic sadface" và mục "video"; khoanh khối Attachments. -->

Giá trị `"on"` cho screenshot và video làm mỗi lần chạy chậm hơn và tốn dung lượng, chỉ bật khi cần điều tra một lỗi cụ thể. Trace là công cụ mạnh nhất trong ba loại, Buổi 13 trình bày chi tiết cách đọc Trace Viewer.

## 6. Hooks: beforeEach, afterEach, beforeAll, afterAll

Nhìn lại các file test của Buổi 8: hầu hết test đều bắt đầu bằng ba dòng tạo `LoginPage`, gọi `goto()` và `login()`. Playwright cung cấp hook để gom các bước lặp lại này vào một chỗ trong file:

```typescript
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/login-page";
import { InventoryPage } from "../pages/inventory-page";

test.beforeEach(async ({ page }) => {
  // Chạy trước MỖI test trong file này, trên cùng page mà test sẽ dùng
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login("standard_user", "secret_sauce");
});

test("thêm một sản phẩm làm badge giỏ hàng hiện số 1", async ({ page }) => {
  const inventoryPage = new InventoryPage(page);
  await inventoryPage.addProductToCart("Sauce Labs Backpack");
  await expect(inventoryPage.cartBadge).toHaveText("1");
});

test("thêm hai sản phẩm làm badge giỏ hàng hiện số 2", async ({ page }) => {
  const inventoryPage = new InventoryPage(page);
  await inventoryPage.addProductToCart("Sauce Labs Backpack");
  await inventoryPage.addProductToCart("Sauce Labs Bike Light");
  await expect(inventoryPage.cartBadge).toHaveText("2");
});
```

`page` trong `beforeEach` và `page` trong test là cùng một object: hook login xong, test nhận đúng trang inventory đã login. Bốn hook có sẵn:

| Hook              | Thời điểm chạy                                                | Fixture dùng được                  |
| ----------------- | ------------------------------------------------------------- | ---------------------------------- |
| `test.beforeEach` | Trước mỗi test trong file                                     | `page` và mọi fixture cấp test     |
| `test.afterEach`  | Sau mỗi test trong file, kể cả khi test fail                  | `page` và mọi fixture cấp test     |
| `test.beforeAll`  | Một lần trước toàn bộ test trong file, trên mỗi worker        | Chỉ `browser`, không có `page`     |
| `test.afterAll`   | Một lần sau toàn bộ test trong file, trên mỗi worker          | Chỉ `browser`, không có `page`     |

Thứ tự chạy với một file có hai test:

```
beforeAll
  beforeEach → test 1 → afterEach
  beforeEach → test 2 → afterEach
afterAll
```

`afterEach` nhận thêm tham số thứ hai `testInfo` chứa thông tin về test vừa chạy. Ví dụ ghi lại URL cuối cùng của các test không pass để dễ điều tra:

```typescript
test.afterEach(async ({ page }, testInfo) => {
  // Chỉ log khi test không pass
  if (testInfo.status !== "passed") {
    console.log(`Test "${testInfo.title}" kết thúc tại ${page.url()}`);
  }
});
```

{% hint style="warning" %}
**Không dùng `beforeAll` để login một lần rồi dùng chung cho mọi test.** `beforeAll` không có `page`, và với 4 worker nó chạy 4 lần chứ không phải một lần. Cố tạo `page` thủ công trong `beforeAll` rồi chia sẻ cho các test là cách phổ biến nhất làm bộ test fail khi chạy song song. `beforeAll` chỉ phù hợp cho việc chuẩn bị không đụng đến trang web, ví dụ tạo dữ liệu qua API (Buổi 12).
{% endhint %}

Hạn chế của hook: `beforeEach` chỉ có tác dụng trong file khai báo nó. Project có `cart.spec.ts`, `checkout.spec.ts`, `inventory.spec.ts` đều cần login thì đoạn `beforeEach` phải được chép sang ba file. Mục 7 giải quyết vấn đề này.

## 7. Custom fixture với test.extend

### Fixture là gì

Từ Buổi 4, mỗi test đều nhận `page` qua cú pháp destructuring `async ({ page })` đã học ở Buổi 1. `page` là một fixture: thứ Playwright chuẩn bị sẵn trước test (mở browser context, mở tab mới) và dọn dẹp sau test (đóng context). Playwright cho phép tự định nghĩa fixture của riêng project bằng `test.extend`, và fixture định nghĩa một lần dùng được ở mọi file test.

### Tạo file fixture

Tạo folder `fixtures/` ngang hàng với `tests/` và `pages/`, rồi tạo file `fixtures/test-fixtures.ts`:

```typescript
import { test as base, expect, Page } from "@playwright/test";
import { LoginPage } from "../pages/login-page";
import { InventoryPage } from "../pages/inventory-page";

// Khai báo tên và kiểu của từng fixture mới, tương tự type alias ở Buổi 3
type CourseFixtures = {
  authenticatedPage: Page;
  inventoryPage: InventoryPage;
};

export const test = base.extend<CourseFixtures>({
  // Fixture 1: page đã login sẵn bằng tài khoản standard_user
  authenticatedPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login("standard_user", "secret_sauce");
    // Chờ chuyển sang trang inventory rồi mới trao page cho test
    await page.waitForURL(/inventory/);

    await use(page);
    // Code sau use() chạy khi test kết thúc; Saucedemo không cần dọn dẹp gì thêm
  },

  // Fixture 2: InventoryPage dựng sẵn trên page đã login
  inventoryPage: async ({ authenticatedPage }, use) => {
    await use(new InventoryPage(authenticatedPage));
  },
});

export { expect };
```

| Phần                                              | Ý nghĩa                                                                                                                                                  |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `import { test as base }`                         | Đổi tên `test` gốc thành `base`, để tên `test` dành cho phiên bản mở rộng sắp tạo                                                                        |
| `type CourseFixtures`                             | Cho TypeScript biết tên và kiểu của từng fixture, nhờ đó VS Code gợi ý `inventoryPage` khi gõ trong test                                                 |
| `base.extend<CourseFixtures>({...})`              | Tạo `test` mới có thêm các fixture liệt kê trong object                                                                                                  |
| `async ({ page }, use)`                           | Hàm định nghĩa fixture: nhận các fixture khác mà nó cần (ở đây là `page` có sẵn của Playwright) và hàm `use`                                             |
| `await use(page)`                                 | Mốc chia đôi fixture: code trước `use` là setup, giá trị truyền vào `use` là thứ test nhận được, code sau `use` là teardown                              |
| `inventoryPage: async ({ authenticatedPage }, use)` | Fixture được phép dùng fixture khác. `inventoryPage` yêu cầu `authenticatedPage`, nên test chỉ cần khai báo `inventoryPage` là tự động được login       |
| `export { expect }`                               | Xuất lại `expect` để file test import mọi thứ từ một chỗ                                                                                                 |

### Dùng fixture trong test

File test import `test` và `expect` từ file fixture thay vì từ `@playwright/test`, sau đó khai báo fixture cần dùng trong destructuring:

```typescript
import { test, expect } from "../fixtures/test-fixtures";

test("thêm một sản phẩm làm badge giỏ hàng hiện số 1", async ({ inventoryPage }) => {
  await inventoryPage.addProductToCart("Sauce Labs Backpack");
  await expect(inventoryPage.cartBadge).toHaveText("1");
});

test("mở giỏ hàng chuyển sang trang cart", async ({ authenticatedPage, inventoryPage }) => {
  await inventoryPage.openCart();
  await expect(authenticatedPage).toHaveURL(/cart/);
});
```

So với phiên bản dùng `beforeEach` ở mục 6, test không còn dòng login nào và không còn dòng `new InventoryPage(page)`. Test chỉ khai báo fixture nào thì fixture đó mới chạy: test kiểm tra màn hình login vẫn dùng `page` gốc và không bị login trước.

{% hint style="warning" %}
Sau khi có file fixture, mọi file test trong project phải import `test` từ `../fixtures/test-fixtures`. File nào còn import từ `@playwright/test` sẽ báo lỗi `Property 'inventoryPage' does not exist` khi khai báo fixture mới, vì `test` gốc không biết các fixture này.
{% endhint %}

### Hook hay fixture

| Tiêu chí               | Hook (`beforeEach`)                          | Fixture (`test.extend`)                                          |
| ---------------------- | -------------------------------------------- | ---------------------------------------------------------------- |
| Phạm vi                | Một file test                                | Mọi file import file fixture                                     |
| Test nào bị ảnh hưởng  | Mọi test trong file, kể cả test không cần    | Chỉ test khai báo fixture đó                                     |
| Kết hợp                | Không kết hợp được                           | Fixture gọi được fixture khác                                    |
| Nên dùng khi           | Bước chuẩn bị riêng cho một file             | Trạng thái dùng chung toàn project: login, page object, dữ liệu test |

Từ buổi này, quy ước của lớp: trạng thái dùng chung khai báo bằng fixture, hook chỉ dùng cho việc riêng của một file. Buổi 12 mở rộng file fixture với dữ liệu tạo qua API, và Buổi 13 đặt `fixtures/` vào cấu trúc folder chuẩn của framework.

## 8. Lỗi thường gặp

| Triệu chứng                                                    | Nguyên nhân                                                                 | Cách xử lý                                                                                          |
| -------------------------------------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `Cannot navigate to invalid URL` khi gọi `goto("/")`           | Chưa khai báo `baseURL` trong khối `use`, hoặc gõ sai chữ hoa (`baseUrl`)   | Kiểm tra dòng `baseURL` trong config, đúng chính tả với U, R, L viết hoa                            |
| `Executable doesn't exist` khi chạy project firefox hoặc webkit | Browser chưa được tải về máy                                                | Chạy `npx playwright install`                                                                       |
| Test pass khi chạy `--workers=1`, fail khi chạy song song      | Test phụ thuộc trạng thái do test khác để lại                               | Mỗi test tự chuẩn bị trạng thái bằng fixture `authenticatedPage`, không dựa vào thứ tự chạy         |
| `Property 'inventoryPage' does not exist on type`              | File test import `test` từ `@playwright/test` thay vì từ file fixture       | Sửa dòng import thành `from "../fixtures/test-fixtures"`                                            |
| Lỗi báo `page` không dùng được trong `beforeAll`               | `beforeAll` chạy ở cấp worker, không có fixture cấp test                    | Chuyển sang `beforeEach` hoặc fixture                                                               |
| Bộ test chạy lâu gấp bốn lần so với Buổi 8                     | Mỗi test chạy lại trên từng project                                         | Khi phát triển dùng `--project=chromium`, chạy đủ project trước khi tạo PR                          |
| Report đánh dấu test là flaky                                  | Test fail lần đầu và pass khi retry                                         | Chạy `npx playwright test --repeat-each=5 --retries=0` để tái hiện, rồi bổ sung assertion neo trạng thái |

## 9. Bài tập về nhà (45-60 phút)

1. Cập nhật `playwright.config.ts`: khai báo `baseURL` của Saucedemo, giữ ba project desktop và thêm project `mobile-chrome` dùng `devices["Pixel 5"]`, bật `screenshot: "only-on-failure"` và `video: "retain-on-failure"`. Sửa `LoginPage.goto()` sang đường dẫn `/`.
2. Tạo `fixtures/test-fixtures.ts` với hai fixture `authenticatedPage` và `inventoryPage` như mục 7. Thêm fixture thứ ba `cartPage` dựng `CartPage` (Buổi 7-8) trên `authenticatedPage`.
3. Refactor toàn bộ file test hiện có sang import `test` từ file fixture. Xóa mọi đoạn login lặp lại trong test và trong `beforeEach`. Riêng các test kiểm tra màn hình login (login thành công, login sai password) vẫn dùng `page` gốc.
4. Chạy `npx playwright test` trên toàn bộ project và dán dòng tổng kết của terminal (ví dụ `40 passed (52.1s)`) vào phần mô tả PR.
5. Push lên repo lớp qua Pull Request, nhánh `your-name/lesson-9`.

**Checklist trước khi nộp:** `LoginPage.goto()` dùng đường dẫn tương đối · config có ít nhất 4 project, gồm 1 project mobile · không còn test nào tự gọi `login()` ngoài các test kiểm tra màn hình login · toàn bộ test pass trên mọi project · `test-results/` và `playwright-report/` không xuất hiện trong PR.
