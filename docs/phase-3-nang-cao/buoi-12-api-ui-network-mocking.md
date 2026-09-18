---
description: >-
  Kết hợp API và UI trong một test: tạo dữ liệu qua API rồi kiểm tra trên giao
  diện, dọn dữ liệu bằng fixture, và dùng page.route để giả lập response API cho
  các tình huống lỗi 500, empty state và timeo
icon: network-wired
---

# Buổi 12 · API + UI kết hợp & Network Mocking

{% hint style="info" %}
**Sau buổi này bạn sẽ:** dùng API để chuẩn bị dữ liệu cho UI test thay vì bấm qua nhiều màn hình, viết fixture tự tạo dữ liệu trước test và tự xóa sau test để mọi test độc lập, dùng UI để thao tác rồi API để kiểm tra kết quả và ngược lại, chặn và giả lập response API bằng `page.route` để kiểm tra giao diện trong các tình huống server lỗi 500, danh sách rỗng và mạng chậm mà không cần server thật xảy ra lỗi.
{% endhint %}

## 1. Vì sao kết hợp API và UI

Yêu cầu: kiểm tra trang profile của DemoQA Book Store hiển thị đúng các sách trong bộ sưu tập của người dùng. Nếu chỉ dùng UI, một test phải: mở trang đăng ký, điền form và giải captcha, đăng nhập, mở cửa hàng, mở từng sách, bấm Add To Your Collection, quay lại profile. Khoảng 25 thao tác, 30-40 giây, và bất kỳ bước nào trong số đó hỏng thì test fail dù trang profile không có lỗi.

Cách làm chuẩn tách test thành ba phần, mỗi phần dùng lớp phù hợp:

| Phần        | Việc cần làm                       | Lớp nên dùng | Lý do                                                     |
| ----------- | ---------------------------------- | ------------ | --------------------------------------------------------- |
| Arrange     | Tạo user, thêm sách vào bộ sưu tập | API          | Nhanh, ổn định, không phải thứ đang được kiểm tra         |
| Act, Assert | Đăng nhập và xem trang profile     | UI           | Đây chính là hành vi cần kiểm tra                         |
| Cleanup     | Xóa user vừa tạo                   | API          | Lần chạy sau bắt đầu sạch, chạy song song không đụng nhau |

Cùng một nguyên tắc áp dụng theo chiều ngược lại: thao tác trên UI (xóa sách trong profile) rồi gọi API để kiểm tra dữ liệu đã đổi ở server.

### DemoQA Book Store

Buổi này dùng DemoQA vì có cả giao diện và API công khai cho cùng một dữ liệu. Các thành phần dùng đến:

| Thành phần     | Địa chỉ                      | Ghi chú                                                         |
| -------------- | ---------------------------- | --------------------------------------------------------------- |
| Trang login    | `https://demoqa.com/login`   | Ô UserName, ô Password, nút Login                               |
| Trang profile  | `https://demoqa.com/profile` | Hiện username và bảng sách trong bộ sưu tập                     |
| Trang cửa hàng | `https://demoqa.com/books`   | Bảng toàn bộ sách, dữ liệu lấy từ API `GET /BookStore/v1/Books` |

| Method | Endpoint                          | Cần token | Mục đích                                                                            |
| ------ | --------------------------------- | --------- | ----------------------------------------------------------------------------------- |
| POST   | `/Account/v1/User`                | Không     | Tạo user, body `{ userName, password }`, trả 201 kèm `userID`                       |
| POST   | `/Account/v1/GenerateToken`       | Không     | Lấy token, body `{ userName, password }`, trả `{ token }`                           |
| GET    | `/Account/v1/User/{userId}`       | Có        | Thông tin user và mảng `books` của user                                             |
| DELETE | `/Account/v1/User/{userId}`       | Có        | Xóa user, trả 204                                                                   |
| GET    | `/BookStore/v1/Books`             | Không     | Danh sách 8 sách, mỗi sách có `isbn` và `title`                                     |
| POST   | `/BookStore/v1/Books`             | Có        | Thêm sách vào bộ sưu tập, body `{ userId, collectionOfIsbns: [{ isbn }] }`, trả 201 |
| DELETE | `/BookStore/v1/Books?UserId={id}` | Có        | Xóa toàn bộ sách khỏi bộ sưu tập, trả 204                                           |

Hai ràng buộc của DemoQA: `userName` phải chưa tồn tại (trùng thì trả 406 `User exists!`), và password phải có ít nhất 8 ký tự gồm chữ hoa, chữ thường, chữ số và ký tự đặc biệt, ví dụ `Student@123`.

## 2. API và UI trong cùng một test

Fixture `request` (Buổi 11) và `page` dùng được cùng lúc trong một test. Phiên bản đầu tiên viết thẳng mọi bước để thấy rõ luồng, tạo file `tests/profile.spec.ts`:

```typescript
import { test, expect } from "../fixtures/test-fixtures";

const DEMOQA_URL = "https://demoqa.com";

test("sách thêm qua API hiển thị trên trang profile", async ({ page, request }) => {
  // Arrange: tạo user và thêm sách bằng API
  const userName = `student_${Date.now()}`;
  const password = "Student@123";

  const createResponse = await request.post(`${DEMOQA_URL}/Account/v1/User`, {
    data: { userName, password },
  });
  expect(createResponse.status()).toBe(201);
  const { userID } = await createResponse.json();

  const tokenResponse = await request.post(`${DEMOQA_URL}/Account/v1/GenerateToken`, {
    data: { userName, password },
  });
  const { token } = await tokenResponse.json();

  const addBookResponse = await request.post(`${DEMOQA_URL}/BookStore/v1/Books`, {
    headers: { Authorization: `Bearer ${token}` },
    data: { userId: userID, collectionOfIsbns: [{ isbn: "9781449325862" }] },
  });
  expect(addBookResponse.status()).toBe(201);

  // Act: đăng nhập trên giao diện
  await page.goto(`${DEMOQA_URL}/login`);
  await page.getByPlaceholder("UserName").fill(userName);
  await page.getByPlaceholder("Password").fill(password);
  await page.getByRole("button", { name: "Login" }).click();

  // Assert: sách xuất hiện trong bảng của trang profile
  await expect(page).toHaveURL(/profile/);
  await expect(page.getByRole("link", { name: "Git Pocket Guide" })).toBeVisible();

  // Cleanup: xóa user để lần chạy sau không gặp dữ liệu cũ
  const deleteResponse = await request.delete(`${DEMOQA_URL}/Account/v1/User/${userID}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  expect(deleteResponse.status()).toBe(204);
});
```

| Phần                               | Ý nghĩa                                                                                                         |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `` `student_${Date.now()}` ``      | Username duy nhất cho mỗi lần chạy, vì DemoQA từ chối username trùng                                            |
| `expect(createResponse.status())`  | Assert ngay ở bước chuẩn bị: nếu API tạo user hỏng, test fail với lý do rõ ràng thay vì fail mơ hồ ở bước login |
| `${DEMOQA_URL}` đầy đủ             | Test này chạy dưới project browser có `baseURL` Saucedemo (Buổi 9), nên địa chỉ DemoQA ghi đầy đủ               |
| `getByRole("link", { name: ... })` | Trang profile hiển thị tên sách dạng link, locator theo role như Buổi 5                                         |

Chạy `npx playwright test tests/profile.spec.ts --project=chromium`. Test mất khoảng 5 giây, trong đó ba lời gọi API chiếm chưa đến một giây.

### Vấn đề của phiên bản này

Test đúng nhưng có ba điểm yếu. Bước Cleanup nằm cuối thân test: nếu một assertion phía trên fail, các dòng sau không chạy và user rác ở lại server. Ba lời gọi API tạo user sẽ lặp lại ở mọi test cần user. Và test dài 30 dòng trong khi phần "Act, Assert" thực sự chỉ có 6 dòng. Mục 3 xử lý cả ba.

## 3. Tách API helper và fixture tạo dữ liệu

### API helper: page object cho API

Tương tự page object gom locator và thao tác của một trang (Buổi 7), một class helper gom các lời gọi API của một nhóm chức năng. Tạo folder `utils/` ngang hàng với `pages/` và file `utils/book-store-api.ts`:

```typescript
import { APIRequestContext } from "@playwright/test";

const DEMOQA_URL = "https://demoqa.com";

export interface BookStoreUser {
  userName: string;
  password: string;
  userId: string;
  token: string;
}

export class BookStoreApi {
  readonly request: APIRequestContext;

  constructor(request: APIRequestContext) {
    this.request = request;
  }

  async createUser(userName: string, password: string): Promise<BookStoreUser> {
    const createResponse = await this.request.post(`${DEMOQA_URL}/Account/v1/User`, {
      data: { userName, password },
    });
    if (createResponse.status() !== 201) {
      // Ném lỗi kèm body để test fail với thông tin đủ dùng, thay vì assert trong helper
      throw new Error(`Tạo user thất bại: ${createResponse.status()} ${await createResponse.text()}`);
    }
    const { userID } = await createResponse.json();

    const tokenResponse = await this.request.post(`${DEMOQA_URL}/Account/v1/GenerateToken`, {
      data: { userName, password },
    });
    const { token } = await tokenResponse.json();

    return { userName, password, userId: userID, token };
  }

  async addBooks(user: BookStoreUser, isbns: string[]) {
    const response = await this.request.post(`${DEMOQA_URL}/BookStore/v1/Books`, {
      headers: { Authorization: `Bearer ${user.token}` },
      // map (Buổi 3) chuyển ["isbn1", "isbn2"] thành [{ isbn: "isbn1" }, { isbn: "isbn2" }]
      data: { userId: user.userId, collectionOfIsbns: isbns.map((isbn) => ({ isbn })) },
    });
    if (response.status() !== 201) {
      throw new Error(`Thêm sách thất bại: ${response.status()} ${await response.text()}`);
    }
  }

  async getUserBooks(user: BookStoreUser): Promise<{ isbn: string; title: string }[]> {
    const response = await this.request.get(`${DEMOQA_URL}/Account/v1/User/${user.userId}`, {
      headers: { Authorization: `Bearer ${user.token}` },
    });
    const body = await response.json();
    return body.books;
  }

  async deleteUser(user: BookStoreUser) {
    await this.request.delete(`${DEMOQA_URL}/Account/v1/User/${user.userId}`, {
      headers: { Authorization: `Bearer ${user.token}` },
    });
  }
}
```

Quy ước cho helper: không có `expect` bên trong, giữ nguyên nguyên tắc "assertion nằm trong test" của Buổi 7. Bước chuẩn bị thất bại thì `throw new Error` với thông tin đủ để đọc lỗi. Method trả về dữ liệu (`getUserBooks`) để test tự assert.

### Page object cho DemoQA

Hai trang của DemoQA cần dùng, tạo `pages/book-store-login-page.ts` và `pages/profile-page.ts` theo đúng pattern Buổi 7:

```typescript
import { Page, Locator } from "@playwright/test";

export class BookStoreLoginPage {
  readonly page: Page;
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.usernameInput = page.getByPlaceholder("UserName");
    this.passwordInput = page.getByPlaceholder("Password");
    this.loginButton = page.getByRole("button", { name: "Login" });
  }

  async goto() {
    await this.page.goto("https://demoqa.com/login");
  }

  async login(userName: string, password: string) {
    await this.usernameInput.fill(userName);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }
}
```

```typescript
import { Page, Locator } from "@playwright/test";

export class ProfilePage {
  readonly page: Page;
  readonly userNameLabel: Locator;
  readonly bookTitles: Locator;

  constructor(page: Page) {
    this.page = page;
    this.userNameLabel = page.locator("#userName-value");
    // Mỗi sách trong bảng là một link mang tên sách
    this.bookTitles = page.locator(".rt-tbody").getByRole("link");
  }

  async goto() {
    await this.page.goto("https://demoqa.com/profile");
  }
}
```

### Fixture tạo user và tự dọn

Mở `fixtures/test-fixtures.ts` của Buổi 9 và bổ sung hai fixture. Phần sau `await use(...)` là teardown, Playwright luôn chạy phần này kể cả khi test fail:

```typescript
import { test as base, expect, Page } from "@playwright/test";
import { LoginPage } from "../pages/login-page";
import { InventoryPage } from "../pages/inventory-page";
import { CartPage } from "../pages/cart-page";
import { BookStoreApi, BookStoreUser } from "../utils/book-store-api";

type CourseFixtures = {
  authenticatedPage: Page;
  inventoryPage: InventoryPage;
  cartPage: CartPage;
  bookStoreApi: BookStoreApi;
  bookStoreUser: BookStoreUser;
};

export const test = base.extend<CourseFixtures>({
  // ... ba fixture của Buổi 9 giữ nguyên ...

  // Helper API dựng trên fixture request có sẵn
  bookStoreApi: async ({ request }, use) => {
    await use(new BookStoreApi(request));
  },

  // User mới cho mỗi test, xóa ngay khi test kết thúc
  bookStoreUser: async ({ bookStoreApi }, use) => {
    const suffix = `${Date.now()}_${Math.floor(Math.random() * 1000)}`;
    const user = await bookStoreApi.createUser(`student_${suffix}`, "Student@123");

    await use(user);

    // Teardown: chạy cả khi test fail, nên không còn user rác trên server
    await bookStoreApi.deleteUser(user);
  },
});

export { expect };
```

`Math.random()` thêm vào sau `Date.now()` vì với bốn worker (Buổi 9), hai test có thể tạo user trong cùng một mili giây.

### Test sau khi tách

```typescript
import { test, expect } from "../fixtures/test-fixtures";
import { BookStoreLoginPage } from "../pages/book-store-login-page";
import { ProfilePage } from "../pages/profile-page";

test.describe("Trang profile", { tag: "@regression" }, () => {
  test("hiển thị đúng các sách đã thêm qua API", async ({ page, bookStoreApi, bookStoreUser }) => {
    await bookStoreApi.addBooks(bookStoreUser, ["9781449325862", "9781449331818"]);

    const loginPage = new BookStoreLoginPage(page);
    await loginPage.goto();
    await loginPage.login(bookStoreUser.userName, bookStoreUser.password);

    const profilePage = new ProfilePage(page);
    await expect(profilePage.userNameLabel).toHaveText(bookStoreUser.userName);
    await expect(profilePage.bookTitles).toHaveText(["Git Pocket Guide", "Learning JavaScript Design Patterns"]);
  });

  test("user mới không có sách nào", async ({ page, bookStoreUser }) => {
    const loginPage = new BookStoreLoginPage(page);
    await loginPage.goto();
    await loginPage.login(bookStoreUser.userName, bookStoreUser.password);

    const profilePage = new ProfilePage(page);
    await expect(profilePage.bookTitles).toHaveCount(0);
    await expect(page.getByText("No rows found")).toBeVisible();
  });
});
```

Test thứ nhất còn 9 dòng và đọc được như một kịch bản kiểm thử. Test thứ hai không gọi `addBooks` nên user không có sách; hai test chạy song song với hai user khác nhau và không ảnh hưởng nhau. `toHaveText` với một mảng kiểm tra đồng thời số lượng và nội dung từng phần tử theo thứ tự.

### Chiều ngược lại: UI thao tác, API kiểm tra

Kết quả của một thao tác trên giao diện nằm ở server. Kiểm tra bằng API chính xác hơn đọc lại màn hình:

```typescript
test("xóa toàn bộ sách trên profile cập nhật dữ liệu ở server", async ({ page, bookStoreApi, bookStoreUser }) => {
  await bookStoreApi.addBooks(bookStoreUser, ["9781449325862"]);

  const loginPage = new BookStoreLoginPage(page);
  await loginPage.goto();
  await loginPage.login(bookStoreUser.userName, bookStoreUser.password);

  // DemoQA hỏi xác nhận bằng modal, sau đó hiện alert của trình duyệt; Playwright tự đóng alert
  await page.getByRole("button", { name: "Delete All Books" }).click();
  await page.getByRole("button", { name: "OK" }).click();

  const books = await bookStoreApi.getUserBooks(bookStoreUser);
  expect(books).toHaveLength(0);
});
```

Với alert của trình duyệt (`window.alert`, `window.confirm`), Playwright mặc định tự đóng nên test không bị treo. Khi cần đọc nội dung alert, đăng ký `page.on("dialog", ...)` trước thao tác.

## 4. Network mocking với page.route

### Cơ chế

Mọi request mà trang web gửi đi (tải trang, gọi API, tải ảnh) đều đi qua Playwright trước khi ra mạng. `page.route(pattern, handler)` đăng ký một hàm xử lý cho các request khớp `pattern`. Trong hàm đó, `route` có ba lựa chọn:

| Lệnh                   | Kết quả                                               | Dùng khi                                                      |
| ---------------------- | ----------------------------------------------------- | ------------------------------------------------------------- |
| `route.fulfill({...})` | Trả về response do test tự đặt, request không ra mạng | Giả lập dữ liệu, lỗi 500, danh sách rỗng                      |
| `route.continue()`     | Cho request đi tiếp bình thường                       | Chỉ muốn quan sát, hoặc mock có điều kiện                     |
| `route.abort()`        | Request thất bại như mất mạng                         | Giả lập timeout, chặn quảng cáo và tài nguyên không cần thiết |

`pattern` là chuỗi glob hoặc regex, so với địa chỉ đầy đủ của request. `**/BookStore/v1/Books` khớp `https://demoqa.com/BookStore/v1/Books`. Dấu `**` thay cho phần địa chỉ gốc bất kỳ. Phải đăng ký `page.route` trước `page.goto`, vì trang gọi API ngay khi tải.

### Giả lập dữ liệu

Trang `https://demoqa.com/books` tải danh sách sách từ `GET /BookStore/v1/Books`. Thay response bằng một sách do test tự đặt, tạo file `tests/book-store-mock.spec.ts`:

```typescript
import { test, expect } from "../fixtures/test-fixtures";

const BOOKS_API = "**/BookStore/v1/Books";

test.describe("Trang Book Store với API giả lập", { tag: "@mock" }, () => {
  test("hiển thị sách do API trả về", async ({ page }) => {
    await page.route(BOOKS_API, async (route) => {
      await route.fulfill({
        json: {
          books: [
            {
              isbn: "9990000000001",
              title: "Playwright cho tester",
              subTitle: "",
              author: "Lớp Automation",
              publish_date: "2026-01-01T00:00:00.000Z",
              publisher: "Nội bộ",
              pages: 120,
              description: "",
              website: "",
            },
          ],
        },
      });
    });

    await page.goto("https://demoqa.com/books");

    await expect(page.getByRole("link", { name: "Playwright cho tester" })).toBeVisible();
    await expect(page.getByRole("link", { name: "Git Pocket Guide" })).toHaveCount(0);
  });
});
```

`route.fulfill({ json })` tự đặt status 200 và `Content-Type: application/json`. Object trong `json` phải có đúng cấu trúc mà giao diện mong đợi: mở DevTools, tab Network, bấm vào request thật để chép cấu trúc, hoặc dùng `route.fetch()` ở phần sau.

### Danh sách rỗng, lỗi 500, mạng chậm

Ba tình huống khó tái hiện với server thật, nhưng giao diện bắt buộc phải xử lý ổn:

```typescript
test("hiện thông báo khi không có sách", async ({ page }) => {
  await page.route(BOOKS_API, (route) => route.fulfill({ json: { books: [] } }));

  await page.goto("https://demoqa.com/books");

  await expect(page.getByText("No rows found")).toBeVisible();
  await expect(page.locator(".rt-tbody").getByRole("link")).toHaveCount(0);
});

test("không hiện sách và vẫn dùng được ô tìm kiếm khi server lỗi 500", async ({ page }) => {
  await page.route(BOOKS_API, (route) =>
    route.fulfill({
      status: 500,
      contentType: "application/json",
      body: JSON.stringify({ message: "Internal Server Error" }),
    }),
  );

  await page.goto("https://demoqa.com/books");

  await expect(page.locator(".rt-tbody").getByRole("link")).toHaveCount(0);
  await expect(page.getByPlaceholder("Type to search")).toBeVisible();
});

test("không hiện sách khi request bị timeout", async ({ page }) => {
  // "timedout" là mã lỗi mạng, trình duyệt nhận lỗi net::ERR_TIMED_OUT
  await page.route(BOOKS_API, (route) => route.abort("timedout"));

  await page.goto("https://demoqa.com/books");

  await expect(page.locator(".rt-tbody").getByRole("link")).toHaveCount(0);
});

test("hiện sách sau khi server phản hồi chậm", async ({ page }) => {
  await page.route(BOOKS_API, async (route) => {
    // Độ trễ này mô phỏng server chậm, nằm trong mock chứ không phải trong test;
    // test vẫn dùng auto-wait của assertion, không dùng waitForTimeout
    await new Promise((resolve) => setTimeout(resolve, 3000));
    await route.fulfill({ json: { books: [{ isbn: "9990000000002", title: "Sách tải chậm", author: "", publisher: "" }] } });
  });

  await page.goto("https://demoqa.com/books");

  await expect(page.getByRole("link", { name: "Sách tải chậm" })).toBeVisible();
});
```

Bốn test này chạy ổn định vì không phụ thuộc dữ liệu thật của DemoQA và không cần server thật gặp sự cố. Assertion `toBeVisible` ở test cuối chờ tối đa 5 giây (Buổi 9), đủ cho độ trễ 3 giây của mock.

### Sửa response thật với route.fetch

Khi chỉ cần thay đổi một phần dữ liệu (cắt bớt, đổi một trường), lấy response thật rồi sửa thay vì tự viết toàn bộ body:

```typescript
test("chỉ hiện hai sách đầu khi API trả về hai sách", async ({ page }) => {
  await page.route(BOOKS_API, async (route) => {
    // Gửi request thật và nhận response thật
    const response = await route.fetch();
    const json = await response.json();
    json.books = json.books.slice(0, 2);
    // Trả về response thật với body đã sửa, giữ nguyên status và header
    await route.fulfill({ response, json });
  });

  await page.goto("https://demoqa.com/books");

  await expect(page.locator(".rt-tbody").getByRole("link")).toHaveCount(2);
});
```

### Quan sát request mà không can thiệp

`page.waitForResponse` chờ một response khớp pattern và trả về nó để test assert. Cách này kiểm tra giao diện có gọi đúng API và API trả đúng dữ liệu, không thay đổi gì:

```typescript
test("trang Book Store gọi API danh sách sách khi tải", async ({ page }) => {
  // Đăng ký chờ trước khi goto, nhưng chưa await
  const responsePromise = page.waitForResponse(BOOKS_API);

  await page.goto("https://demoqa.com/books");

  const response = await responsePromise;
  expect(response.status()).toBe(200);
  const body = await response.json();
  expect(body.books.length).toBeGreaterThan(0);
});
```

{% hint style="success" %}
DemoQA chèn quảng cáo, đôi khi đè lên nút và làm click bị chặn với lỗi `intercepts pointer events`. Chặn các request quảng cáo bằng `route.abort()` trong fixture hoặc `beforeEach`:

```typescript
await page.route(/googlesyndication|doubleclick|adsbygoogle/, (route) => route.abort());
```

Đây là ứng dụng thực tế phổ biến nhất của `route.abort()`: bỏ tài nguyên bên thứ ba để test nhanh và ổn định hơn.
{% endhint %}

{% hint style="warning" %}
Mock che đi server thật. Một test mock pass không chứng minh hệ thống chạy đúng end-to-end. Chỉ mock các tình huống khó tạo ra với server thật (lỗi 500, rỗng, chậm) hoặc API bên thứ ba không kiểm soát được (cổng thanh toán, gửi SMS). Flow chính của sản phẩm luôn phải có ít nhất một test chạy với API thật. Đặt tag `@mock` cho mọi test dùng `route.fulfill` để phân biệt trong report.
{% endhint %}

## 5. Chọn lớp cho từng bước

| Bước trong test                              | Nên dùng                 | Tránh                                         |
| -------------------------------------------- | ------------------------ | --------------------------------------------- |
| Tạo tài khoản, dữ liệu nền                   | API qua fixture          | Bấm qua form đăng ký ở mỗi test               |
| Hành vi đang được kiểm tra                   | UI                       | Mock API của chính chức năng đó               |
| Kiểm tra dữ liệu đã lưu ở server             | API (`getUserBooks`)     | Reload trang rồi đọc lại màn hình             |
| Trạng thái lỗi, rỗng, chậm                   | `page.route` với `@mock` | Chờ server thật gặp sự cố                     |
| Dọn dữ liệu                                  | Teardown của fixture     | Dòng cuối thân test, không chạy khi test fail |
| Tài nguyên bên thứ ba (quảng cáo, analytics) | `route.abort()`          | Tăng timeout để chờ chúng tải xong            |

Từ Buổi 13, `utils/` và `fixtures/` là hai folder bắt buộc trong cấu trúc framework chuẩn, và mini project ở Buổi 14 yêu cầu có cả API test, test kết hợp API-UI và ít nhất một test mock.

## 6. Lỗi thường gặp

| Triệu chứng                                                                       | Nguyên nhân                                                                | Cách xử lý                                                                               |
| --------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Tạo user trả 406 `User exists!`                                                   | Username trùng với lần chạy trước hoặc với worker khác                     | Ghép `Date.now()` và `Math.random()` vào username như fixture ở mục 3                    |
| Tạo user trả 400 `Passwords must have at least one non alphanumeric character...` | Password không đủ quy tắc của DemoQA                                       | Dùng password có chữ hoa, chữ thường, số và ký tự đặc biệt, ví dụ `Student@123`          |
| API trả 401 `User not authorized!`                                                | Thiếu header `Authorization`, token sai, hoặc `userId` không khớp token    | Kiểm tra `Bearer ${token}` và `userId` lấy từ đúng user                                  |
| Mock không có tác dụng, trang vẫn hiện dữ liệu thật                               | `page.route` đăng ký sau `page.goto`, hoặc pattern không khớp địa chỉ thật | Đưa `page.route` lên trước `goto`; so pattern với địa chỉ trong tab Network của DevTools |
| Click trên DemoQA báo `intercepts pointer events`                                 | Quảng cáo đè lên phần tử                                                   | Chặn request quảng cáo bằng `route.abort()` như mục 4; chạy headless                     |
| DemoQA trả 502 hoặc trang tải rất lâu                                             | Server DemoQA quá tải tạm thời                                             | Chạy lại sau vài phút; không tăng timeout hoặc thêm retry vĩnh viễn để che sự cố         |
| Server còn nhiều user `student_...` sau khi chạy test                             | Cleanup đặt ở cuối thân test nên không chạy khi test fail                  | Chuyển việc xóa sang teardown của fixture `bookStoreUser`                                |
| Test mock pass nhưng trang thật hiện sai                                          | Body giả lập không đúng cấu trúc API thật                                  | Chép cấu trúc từ DevTools hoặc dùng `route.fetch()` rồi sửa                              |

## 7. Bài tập về nhà (45-60 phút)

1. Tạo `utils/book-store-api.ts`, hai page object `BookStoreLoginPage` và `ProfilePage`, và bổ sung hai fixture `bookStoreApi`, `bookStoreUser` vào `fixtures/test-fixtures.ts` như mục 3.
2. Tạo `tests/profile.spec.ts` với ba test trong `describe` "Trang profile", tag `@regression`: user mới đăng nhập thấy đúng username và không có sách; user có hai sách thêm qua API thấy đúng hai tên sách; user có một sách, xóa toàn bộ sách trên giao diện bằng nút Delete All Books, sau đó API `getUserBooks` trả về mảng rỗng.
3. Tạo `tests/book-store-mock.spec.ts` với tag `@mock` gồm bốn test: API trả về một sách tự đặt thì trang hiện đúng tên sách đó; API trả về danh sách rỗng thì trang hiện `No rows found`; API trả 500 thì trang không hiện sách và ô tìm kiếm vẫn hiển thị; dùng `route.fetch()` cắt còn ba sách thì bảng hiện đúng ba link.
4. Chạy `npx playwright test tests/profile.spec.ts tests/book-store-mock.spec.ts --project=chromium` hai lần liên tiếp. Cả hai lần đều phải pass: lần hai chứng minh dữ liệu của lần một đã được dọn. Dán dòng tổng kết của lần chạy thứ hai vào mô tả PR.
5. Push lên repo lớp qua Pull Request, nhánh `your-name/lesson-12`.

**Checklist trước khi nộp:** `BookStoreApi` không chứa `expect` · fixture `bookStoreUser` xóa user trong phần sau `use()` · không test nào tự gọi API tạo user trong thân test · mọi `page.route` đứng trước `page.goto` · test mock có tag `@mock` · không dùng `waitForTimeout` · hai lần chạy liên tiếp đều pass trên project chromium.
