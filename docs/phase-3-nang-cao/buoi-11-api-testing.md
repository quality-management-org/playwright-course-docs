---
description: "Viết test API bằng APIRequestContext của Playwright: gọi GET, POST, PUT, DELETE, gửi header và Bearer token, assert status code và JSON body mà không cần mở browser."
icon: plug
---

# Buổi 11 · API Testing với Playwright

{% hint style="info" %}
**Sau buổi này bạn sẽ:** hiểu API test đứng ở đâu so với UI test và khi nào nên viết loại nào, gọi được GET, POST, PUT, PATCH, DELETE bằng fixture `request` của Playwright, gửi header và Bearer token, assert status code, header và từng trường trong JSON body, cấu hình project `api` riêng trong `playwright.config.ts`, và tổ chức bộ test API theo quy ước của lớp.
{% endhint %}

## 1. API test đứng ở đâu

Mọi test từ Buổi 4 đến nay đều đi qua giao diện: mở browser, điền form, bấm nút, đọc màn hình. Phía sau giao diện, trình duyệt gọi các API của server để login, lấy danh sách sản phẩm, tạo đơn hàng. API test gọi thẳng vào các API này, bỏ qua giao diện.

| Tiêu chí                    | UI test (Buổi 4-10)                                | API test (buổi này)                                       |
| --------------------------- | -------------------------------------------------- | --------------------------------------------------------- |
| Thời gian một test          | Vài giây                                           | Vài chục mili giây                                        |
| Nguyên nhân fail phổ biến   | Locator đổi, trang tải chậm, animation             | Logic nghiệp vụ sai, hợp đồng dữ liệu đổi                 |
| Kiểm tra được               | Trải nghiệm thật của người dùng, bố cục, luồng màn hình | Logic xử lý, validation, mã lỗi, cấu trúc dữ liệu     |
| Không kiểm tra được         | Chi tiết từng trường hợp lỗi của server (quá chậm) | Giao diện có hiển thị đúng dữ liệu đó không               |
| Số lượng nên có             | Ít, tập trung flow chính                           | Nhiều, phủ các trường hợp biên                            |

Hai loại bổ sung cho nhau. Một dự án lành mạnh có nhiều API test cho logic và ít UI test cho flow chính. Buổi 12 kết hợp cả hai trong cùng một test.

### fetch và fixture request

Ở Buổi 2 bạn đã gọi API bằng `fetch`. Playwright cung cấp fixture `request` (kiểu `APIRequestContext`) làm cùng việc nhưng phù hợp hơn cho việc viết test:

| Tiêu chí                | `fetch` (Buổi 2)                                     | Fixture `request`                                                 |
| ----------------------- | ---------------------------------------------------- | ----------------------------------------------------------------- |
| Gửi JSON                | Tự `JSON.stringify` và đặt header `Content-Type`     | Truyền object vào `data`, Playwright lo phần còn lại              |
| Địa chỉ gốc             | Ghi đầy đủ ở mỗi lời gọi                             | Dùng `baseURL` trong config như UI test (Buổi 9)                  |
| Assertion               | Tự viết `if`                                         | `expect(response).toBeOK()` và các matcher quen thuộc             |
| Report                  | Không có                                             | Mỗi lời gọi được ghi vào report và trace như một action           |
| Header dùng chung       | Lặp lại ở mỗi lời gọi                                | Khai báo một lần bằng `extraHTTPHeaders`                          |

### Test API đầu tiên

Tạo folder `tests/api/` và file `tests/api/users.spec.ts`:

```typescript
import { test, expect } from "../../fixtures/test-fixtures";

test("GET /users/1 trả về đúng user", async ({ request }) => {
  const response = await request.get("https://jsonplaceholder.typicode.com/users/1");

  expect(response.status()).toBe(200);
  const user = await response.json();
  expect(user.name).toBe("Leanne Graham");
});
```

| Dòng                                   | Ý nghĩa                                                                                     |
| -------------------------------------- | ------------------------------------------------------------------------------------------- |
| `async ({ request })`                  | Khai báo fixture `request` thay cho `page`; test không mở browser                            |
| `await request.get(url)`               | Gửi GET và chờ server trả lời, kết quả là object `APIResponse`                              |
| `response.status()`                    | Mã trạng thái HTTP dạng số                                                                  |
| `await response.json()`                | Đọc body và chuyển từ chuỗi JSON thành object, giống `response.json()` của `fetch`          |
| `expect(user.name).toBe(...)`          | `expect` với giá trị thường: so sánh ngay, không có auto-wait như assertion trên locator (Buổi 6) |

Chạy `npx playwright test tests/api --project=chromium`. Test pass trong dưới một giây và không có cửa sổ browser nào mở dù thêm `--headed`. Đường dẫn import fixture có hai cấp `../../` vì file nằm trong `tests/api/`.

## 2. Project api trong playwright.config.ts

Hiện tại `baseURL` trong config là Saucedemo, và mọi test trong `tests/` chạy trên bốn project browser. API test không cần browser, cũng không cần chạy bốn lần. Thêm một project riêng cho API và loại folder `tests/api/` khỏi các project UI:

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
    baseURL: "https://www.saucedemo.com",
    testIdAttribute: "data-test",
    screenshot: "only-on-failure",
    video: "retain-on-failure",
    trace: "on-first-retry",
  },
  projects: [
    {
      name: "api",
      testMatch: "**/api/**/*.spec.ts",
      use: { baseURL: "https://jsonplaceholder.typicode.com" },
    },
    { name: "chromium", use: { ...devices["Desktop Chrome"] }, testIgnore: "**/api/**" },
    { name: "firefox", use: { ...devices["Desktop Firefox"] }, testIgnore: "**/api/**" },
    { name: "webkit", use: { ...devices["Desktop Safari"] }, testIgnore: "**/api/**" },
    { name: "mobile-chrome", use: { ...devices["Pixel 5"] }, testIgnore: "**/api/**" },
  ],
});
```

| Thuộc tính                          | Ý nghĩa                                                                                                     |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `testMatch: "**/api/**/*.spec.ts"`  | Project `api` chỉ nhận file `.spec.ts` nằm trong folder `api`                                                |
| `use: { baseURL: ... }`             | `baseURL` khai báo trong project ghi đè `baseURL` chung; project `api` trỏ về JSONPlaceholder                |
| `testIgnore: "**/api/**"`           | Bốn project browser bỏ qua folder `api`, tránh chạy API test bốn lần                                         |
| Không có `devices`                  | Project `api` không mở browser nên không cần cấu hình thiết bị                                               |

Sau khi sửa config, test đầu tiên rút gọn thành đường dẫn tương đối và chạy bằng lệnh mới:

```typescript
const response = await request.get("/users/1");
```

```bash
npx playwright test --project=api
```

Từ đây, mọi lệnh trong buổi này ngầm hiểu là chạy với `--project=api`. Terminal báo `Running N tests using M workers` như thường lệ: API test cũng chạy song song.

## 3. GET và assertion trên response

### Cấu trúc một test API

Mỗi test API đi qua ba bước giống nhau: gửi request, kiểm tra status, kiểm tra body. Khai báo interface (Buổi 3) cho body để VS Code gợi ý tên trường và bắt lỗi gõ sai:

```typescript
import { test, expect } from "../../fixtures/test-fixtures";

interface User {
  id: number;
  name: string;
  username: string;
  email: string;
}

interface Post {
  userId: number;
  id: number;
  title: string;
  body: string;
}

test.describe("GET /users", { tag: "@api" }, () => {
  test("trả về danh sách 10 user", async ({ request }) => {
    const response = await request.get("/users");

    await expect(response).toBeOK();
    expect(response.headers()["content-type"]).toContain("application/json");

    const users: User[] = await response.json();
    expect(users).toHaveLength(10);
    expect(users[0]).toMatchObject({ id: 1, username: "Bret" });
  });

  test("trả về 404 khi user không tồn tại", async ({ request }) => {
    const response = await request.get("/users/9999");

    expect(response.status()).toBe(404);
  });
});

test.describe("GET /posts", { tag: "@api" }, () => {
  test("lọc bài viết theo userId bằng query string", async ({ request }) => {
    const response = await request.get("/posts", { params: { userId: 1 } });

    await expect(response).toBeOK();
    const posts: Post[] = await response.json();
    expect(posts).toHaveLength(10);
    // every() đã học ở Buổi 3 cùng nhóm với map, filter, find
    expect(posts.every((post) => post.userId === 1)).toBe(true);
  });
});
```

`params: { userId: 1 }` được Playwright ghép thành `/posts?userId=1`. `toBeOK()` pass khi status nằm trong khoảng 200-299 và là matcher bất đồng bộ, bắt buộc có `await`.

### Các method của response

| Method                  | Trả về                                                | Ghi chú                                                           |
| ----------------------- | ----------------------------------------------------- | ----------------------------------------------------------------- |
| `status()`              | Số, ví dụ `200`, `404`                                | Assertion phổ biến nhất                                           |
| `ok()`                  | `true` khi status từ 200 đến 299                       | Dùng trong `if`; trong assertion ưu tiên `toBeOK()`               |
| `statusText()`          | Chuỗi, ví dụ `"OK"`, `"Not Found"`                    |                                                                   |
| `headers()`             | Object các header, tên header viết thường             | `headers()["content-type"]`                                       |
| `json()`                | Body đã parse thành object hoặc mảng                  | Ném lỗi nếu body không phải JSON                                  |
| `text()`                | Body dạng chuỗi nguyên bản                            | Dùng để log khi `json()` lỗi                                      |
| `url()`                 | Địa chỉ đầy đủ đã gọi                                 | Kiểm tra `baseURL` và `params` ghép đúng                           |

### Các matcher dùng cho dữ liệu

| Matcher                             | Kiểm tra                                                            | Ví dụ                                                     |
| ----------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------- |
| `toBe(value)`                       | Bằng chính xác (số, chuỗi, boolean)                                 | `expect(response.status()).toBe(201)`                     |
| `toEqual(object)`                   | Object hoặc mảng có nội dung giống hệt                              | `expect(body).toEqual({})`                                |
| `toMatchObject(object)`             | Object chứa các trường liệt kê, các trường khác bỏ qua              | `expect(user).toMatchObject({ id: 1 })`                   |
| `toHaveLength(n)`                   | Mảng hoặc chuỗi có đúng độ dài                                      | `expect(users).toHaveLength(10)`                          |
| `toContain(item)`                   | Chuỗi chứa chuỗi con, hoặc mảng chứa phần tử                        | `expect(email).toContain("@")`                            |
| `toHaveProperty(path)`              | Object có trường tên đó                                              | `expect(user).toHaveProperty("address.city")`             |
| `toBeGreaterThan(n)`                | So sánh số                                                          | `expect(posts.length).toBeGreaterThan(0)`                 |
| `await expect(response).toBeOK()`   | Status trong khoảng 200-299                                          | Dùng cho response, không dùng cho giá trị thường          |

{% hint style="success" %}
Assert đúng những trường nghiệp vụ quan tâm bằng `toMatchObject`, thay vì `toEqual` toàn bộ body. Server thêm một trường mới (ví dụ `updatedAt`) là chuyện bình thường và không nên làm test fail.
{% endhint %}

## 4. POST, PUT, PATCH, DELETE

JSONPlaceholder chấp nhận đủ các method ghi dữ liệu và trả về response giống server thật, nhưng không lưu thay đổi. Điều này giúp bài thực hành không cần dọn dữ liệu; Buổi 12 làm việc với API lưu thật.

```typescript
import { test, expect } from "../../fixtures/test-fixtures";

interface Post {
  userId: number;
  id: number;
  title: string;
  body: string;
}

test.describe("CRUD /posts", { tag: "@api" }, () => {
  test("POST tạo bài viết mới trả về 201 kèm id", async ({ request }) => {
    const newPost = { title: "Bài viết từ Playwright", body: "Nội dung thử nghiệm", userId: 1 };

    const response = await request.post("/posts", { data: newPost });

    expect(response.status()).toBe(201);
    const created: Post = await response.json();
    expect(created.id).toBe(101);
    expect(created).toMatchObject(newPost);
  });

  test("PUT thay thế toàn bộ bài viết", async ({ request }) => {
    const updatedPost = { userId: 1, id: 1, title: "Tiêu đề mới", body: "Nội dung mới" };

    const response = await request.put("/posts/1", { data: updatedPost });

    await expect(response).toBeOK();
    const body: Post = await response.json();
    expect(body).toEqual(updatedPost);
  });

  test("PATCH chỉ cập nhật trường được gửi", async ({ request }) => {
    const response = await request.patch("/posts/1", { data: { title: "Chỉ đổi tiêu đề" } });

    await expect(response).toBeOK();
    const body: Post = await response.json();
    expect(body.title).toBe("Chỉ đổi tiêu đề");
    // Các trường không gửi giữ nguyên giá trị cũ
    expect(body.userId).toBe(1);
    expect(body.id).toBe(1);
  });

  test("DELETE trả về 200 và body rỗng", async ({ request }) => {
    const response = await request.delete("/posts/1");

    await expect(response).toBeOK();
    expect(await response.json()).toEqual({});
  });
});
```

Tham số thứ hai của `get`, `post`, `put`, `patch`, `delete` là object tùy chọn:

| Tùy chọn            | Kiểu                       | Ý nghĩa                                                                                       |
| ------------------- | -------------------------- | --------------------------------------------------------------------------------------------- |
| `data`              | Object, mảng hoặc chuỗi    | Body của request. Truyền object thì Playwright tự chuyển sang JSON và đặt `Content-Type: application/json` |
| `params`            | Object                     | Query string, ghép vào sau dấu `?`                                                             |
| `headers`           | Object                     | Header bổ sung cho riêng lời gọi này                                                          |
| `form`              | Object                     | Body dạng `application/x-www-form-urlencoded`, cho form đăng nhập kiểu cũ                     |
| `timeout`           | Số mili giây               | Thời gian chờ tối đa cho lời gọi, mặc định 30 giây                                             |
| `failOnStatusCode`  | Boolean                    | `true` để lời gọi ném lỗi khi status không phải 2xx; mặc định `false` để test tự assert status |

{% hint style="warning" %}
Không tự gọi `JSON.stringify` rồi truyền chuỗi vào `data`. Khi nhận chuỗi, Playwright gửi nguyên chuỗi với `Content-Type: text/plain`, server trả 400 hoặc 415. Truyền object và để Playwright xử lý.
{% endhint %}

## 5. Header và Bearer token

Phần lớn API thật yêu cầu xác thực: client gọi endpoint login, nhận token, và gửi token này trong header `Authorization: Bearer <token>` ở các lời gọi sau. ReqRes cung cấp endpoint login để luyện quy trình này. Tài liệu của ReqRes yêu cầu gửi kèm header `x-api-key` với giá trị của gói miễn phí; tùy thời điểm, server có thể bỏ qua hoặc từ chối request thiếu header này. Lớp luôn gửi header để có thêm một ví dụ về header bắt buộc của API.

Tạo file `tests/api/reqres-auth.spec.ts`. Project `api` có `baseURL` là JSONPlaceholder nên các lời gọi tới ReqRes ghi địa chỉ đầy đủ:

```typescript
import { test, expect } from "../../fixtures/test-fixtures";

const REQRES_URL = "https://reqres.in";
const REQRES_HEADERS = { "x-api-key": "reqres-free-v1" };

test.describe("Xác thực trên ReqRes", { tag: "@api" }, () => {
  test("login thành công trả về token", async ({ request }) => {
    const response = await request.post(`${REQRES_URL}/api/login`, {
      headers: REQRES_HEADERS,
      data: { email: "eve.holt@reqres.in", password: "cityslicka" },
    });

    await expect(response).toBeOK();
    const body = await response.json();
    expect(typeof body.token).toBe("string");
    expect(body.token.length).toBeGreaterThan(0);
  });

  test("login thiếu password trả về 400 kèm thông báo", async ({ request }) => {
    const response = await request.post(`${REQRES_URL}/api/login`, {
      headers: REQRES_HEADERS,
      data: { email: "eve.holt@reqres.in" },
    });

    expect(response.status()).toBe(400);
    const body = await response.json();
    expect(body.error).toBe("Missing password");
  });

  test("gọi API kèm Bearer token nhận được sau khi login", async ({ request }) => {
    const loginResponse = await request.post(`${REQRES_URL}/api/login`, {
      headers: REQRES_HEADERS,
      data: { email: "eve.holt@reqres.in", password: "cityslicka" },
    });
    const { token } = await loginResponse.json();

    const response = await request.get(`${REQRES_URL}/api/users/2`, {
      headers: { ...REQRES_HEADERS, Authorization: `Bearer ${token}` },
    });

    await expect(response).toBeOK();
    const body = await response.json();
    expect(body.data).toMatchObject({ id: 2, email: "janet.weaver@reqres.in" });
  });
});
```

| Phần                                           | Ý nghĩa                                                                                              |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `const REQRES_HEADERS`                         | Header dùng chung khai báo một lần ở đầu file, tên viết hoa theo quy ước hằng số                     |
| `{ ...REQRES_HEADERS, Authorization: ... }`    | Spread (Buổi 9) gộp header chung với header riêng của lời gọi                                        |
| `` `Bearer ${token}` ``                        | Định dạng chuẩn: chữ `Bearer`, một dấu cách, rồi token                                               |
| `const { token } = await loginResponse.json()` | Destructuring (Buổi 1) lấy đúng trường cần từ body                                                   |

ReqRes là API mẫu nên không thật sự kiểm tra token; quy trình ba bước (login, lấy token, gửi token) đúng với API thật của dự án bạn. Buổi 12 dùng quy trình này trên DemoQA, nơi thiếu token thì server trả 401.

### Context có sẵn header với newContext

Khi nhiều lời gọi trong một test cùng cần token, lặp lại `headers` ở từng dòng làm test dài và dễ sót. Tạo một `APIRequestContext` riêng có sẵn `baseURL` và header:

```typescript
test("liệt kê user bằng context đã đăng nhập", async ({ request, playwright }) => {
  const loginResponse = await request.post(`${REQRES_URL}/api/login`, {
    headers: REQRES_HEADERS,
    data: { email: "eve.holt@reqres.in", password: "cityslicka" },
  });
  const { token } = await loginResponse.json();

  const authorizedContext = await playwright.request.newContext({
    baseURL: REQRES_URL,
    extraHTTPHeaders: { ...REQRES_HEADERS, Authorization: `Bearer ${token}` },
  });

  const response = await authorizedContext.get("/api/users", { params: { page: 2 } });
  await expect(response).toBeOK();
  const body = await response.json();
  expect(body.page).toBe(2);
  expect(body.data).toHaveLength(6);

  // Context tự tạo thì tự đóng, fixture request có sẵn được Playwright đóng giúp
  await authorizedContext.dispose();
});
```

`playwright` là fixture có sẵn, cho phép tạo context mới bằng `playwright.request.newContext()`. `extraHTTPHeaders` áp dụng cho mọi lời gọi qua context đó. Buổi 12 đưa cách tạo context này vào file fixture để mọi test dùng chung.

Header dùng chung cho toàn bộ project cũng khai báo được trong config, trong khối `use` của project `api`:

```typescript
{
  name: "api",
  testMatch: "**/api/**/*.spec.ts",
  use: {
    baseURL: "https://jsonplaceholder.typicode.com",
    extraHTTPHeaders: { Accept: "application/json" },
  },
},
```

Không đặt token hoặc API key thật vào config, vì file này được commit lên repo. Buổi 13 trình bày cách đọc giá trị bí mật từ file `.env`.

## 6. Tổ chức bộ test API

| Quy ước                               | Nội dung                                                                                                          |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Vị trí                                | `tests/api/`, mỗi resource một file: `users.spec.ts`, `posts.spec.ts`, `reqres-auth.spec.ts`                       |
| Nhóm                                  | `describe` theo endpoint hoặc nhóm chức năng: `"GET /users"`, `"CRUD /posts"`, `"Xác thực"`                        |
| Tên test                              | Kết quả mong đợi, nêu rõ status khi có: `"trả về 404 khi user không tồn tại"`                                       |
| Tag                                   | Mọi test API có `@api` (Buổi 10), thêm `@smoke` cho endpoint cốt lõi                                                |
| Mỗi test                              | Ít nhất một assertion về status và một assertion về body                                                          |
| Trường hợp âm                         | Mỗi endpoint có ít nhất một test cho dữ liệu sai hoặc thiếu quyền                                                 |
| Độc lập                               | Không dựa vào dữ liệu do test khác tạo; với API lưu thật, tự tạo và tự xóa (Buổi 12)                               |

API test có `@api` chạy được cùng UI test smoke trong một lệnh trước khi tạo PR:

```bash
npx playwright test --grep "@smoke|@api"
```

Lệnh này chạy project `api` cho các test API và bốn project browser cho các test UI có `@smoke`, vì `testMatch` và `testIgnore` đã phân chia file cho từng project.

## 7. Lỗi thường gặp

| Triệu chứng                                                          | Nguyên nhân                                                                        | Cách xử lý                                                                                          |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `apiRequestContext.get: Invalid URL` với đường dẫn `/users/1`        | Test chạy dưới project browser (baseURL Saucedemo) hoặc chưa có project `api`      | Chạy `--project=api`; kiểm tra `testMatch` và `testIgnore` trong config                              |
| Mỗi API test chạy 4-5 lần trong report                               | Các project browser chưa có `testIgnore: "**/api/**"`                              | Thêm `testIgnore` cho từng project browser                                                          |
| `SyntaxError: Unexpected token <` khi gọi `response.json()`          | Server trả HTML (trang 404, trang bảo trì) thay vì JSON                             | Log `await response.text()` và `response.url()` để xem server thực sự trả gì                         |
| Test pass dù status sai                                              | Quên `await` trước `expect(response).toBeOK()`                                      | Luôn `await` matcher `toBeOK`; các matcher giá trị thường (`toBe`, `toEqual`) không cần `await`      |
| ReqRes trả 401 `Missing API key`                                     | Server đang bắt buộc header `x-api-key` mà lời gọi không gửi                        | Gửi `REQRES_HEADERS` ở mọi lời gọi tới ReqRes hoặc đặt vào `extraHTTPHeaders` của context             |
| Server trả 400 hoặc 415 khi POST                                     | Truyền chuỗi `JSON.stringify(...)` vào `data`                                       | Truyền object trực tiếp vào `data`                                                                  |
| VS Code không báo lỗi khi gõ sai tên trường, test fail lúc chạy với `undefined` | `response.json()` trả về kiểu `any`, TypeScript không kiểm tra được | Gán kết quả vào biến có kiểu: `const user: User = await response.json()`                              |
| Lời gọi treo rồi báo `Request timed out after 30000ms`               | API công cộng phản hồi chậm hoặc mất mạng                                           | Kiểm tra mạng, thử lại; không tăng `timeout` lên quá lớn để che sự cố                                 |

## 8. Bài tập về nhà (45-60 phút)

1. Cập nhật `playwright.config.ts` theo mục 2: thêm project `api` với `testMatch` và `baseURL` JSONPlaceholder, thêm `testIgnore` cho bốn project browser. Chuyển các file API test vào `tests/api/`.
2. Tạo `tests/api/todos.spec.ts` với interface `Todo` (`userId`, `id`, `title`, `completed`) và các test: GET `/todos/1` trả về `title` là `delectus aut autem` và `completed` là `false`; GET `/todos` với `params: { userId: 1 }` trả về 20 todo và mọi phần tử có `userId` bằng 1; POST `/todos` trả về 201 và `id` bằng 201; PUT `/todos/1` trả về body giống dữ liệu gửi lên; DELETE `/todos/1` trả về 200. Gắn tag `@api` cho cả file.
3. Bổ sung vào `tests/api/reqres-auth.spec.ts` hai test cho endpoint đăng ký `POST /api/register`: đăng ký thành công với `email: "eve.holt@reqres.in"` và `password: "pistol"` trả về 200 kèm `id` và `token`; đăng ký thiếu password trả về 400 với `error` là `Missing password`. Nhớ gửi `REQRES_HEADERS`.
4. Chạy `npx playwright test --project=api` và dán dòng tổng kết vào mô tả PR. Chạy thêm `npx playwright test --grep @api --list` và dán dòng `Total`.
5. Push lên repo lớp qua Pull Request, nhánh `your-name/lesson-11`.

**Checklist trước khi nộp:** config có project `api` và bốn project browser có `testIgnore` · mọi file API nằm trong `tests/api/` và import `test` từ file fixture · mọi test API có tag `@api`, có assertion status và assertion body · body được gán vào biến có interface · không có token hoặc API key ghi trong config · `npx playwright test --project=api` pass toàn bộ.
