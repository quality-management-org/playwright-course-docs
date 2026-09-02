---
description: >-
  Học cách chọn phần tử web đúng chuẩn với getByRole, getByText... — kỹ năng
  quyết định test của bạn bền hay flaky.
---

# Buổi 5 · Locators: Chọn phần tử đúng cách

{% hint style="info" %}
**Sau buổi này bạn sẽ:** biết thứ tự ưu tiên khi chọn locator, hiểu vì sao XPath cứng là kẻ thù của test bền vững, và biết cách kết hợp locator để trỏ chính xác phần tử mình cần.
{% endhint %}

## 1. Locator là gì và vì sao quan trọng đến vậy?

Locator là "địa chỉ" bạn đưa cho Playwright để tìm một phần tử trên trang: cái nút này, ô nhập kia. Kinh nghiệm thực tế: **phần lớn test fail không phải do logic sai, mà do locator kém** — UI đổi một chút là locator vỡ. Chọn locator tốt ngay từ đầu là kỹ năng đáng tiền nhất của automation tester.

Nguyên tắc vàng: **chọn locator theo cách người dùng nhìn trang web** (nút có chữ "Login", ô có nhãn "Username") thay vì theo cấu trúc HTML bên trong (div thứ 3, class `btn_primary_2x`). Người dùng không quan tâm HTML — và HTML là thứ dev đổi thường xuyên nhất.

## 2. Thứ tự ưu tiên khi chọn locator

Đi từ trên xuống — chỉ xuống mức dưới khi mức trên không dùng được:

| Ưu tiên | Locator              | Dùng khi                                    | Ví dụ                                         |
| ------- | -------------------- | ------------------------------------------- | --------------------------------------------- |
| 1       | `getByRole()`        | Hầu hết mọi trường hợp — nút, link, heading | `page.getByRole("button", { name: "Login" })` |
| 2       | `getByLabel()`       | Ô nhập liệu có nhãn (label)                 | `page.getByLabel("Password")`                 |
| 3       | `getByPlaceholder()` | Ô nhập chỉ có chữ gợi ý bên trong           | `page.getByPlaceholder("Username")`           |
| 4       | `getByText()`        | Phần tử nhận diện bằng nội dung chữ         | `page.getByText("Products")`                  |
| 5       | `getByTestId()`      | Team dev có gắn thuộc tính test riêng       | `page.getByTestId("login-button")`            |
| 6       | CSS selector         | Không còn cách nào ở trên dùng được         | `page.locator("#login-button")`               |
| 7       | XPath                | Gần như không bao giờ — xem mục 3           | —                                             |

Thử ngay trên Saucedemo:

```typescript
import { test, expect } from "@playwright/test";

test("login bằng locator chuẩn", async ({ page }) => {
  await page.goto("https://www.saucedemo.com");

  // Thay vì #user-name, #password như Buổi 4:
  await page.getByPlaceholder("Username").fill("standard_user");
  await page.getByPlaceholder("Password").fill("secret_sauce");
  await page.getByRole("button", { name: "Login" }).click();

  await expect(page).toHaveURL(/inventory/);
});
```

Đọc code này gần như đọc văn nói: "điền vào ô Username, điền vào ô Password, bấm nút Login" — người chưa học code cũng đoán được test làm gì. Đó chính là dấu hiệu của locator tốt.

{% hint style="success" %}
**Mẹo cho Saucedemo:** trang này gắn thuộc tính `data-test` cho hầu hết phần tử (thay vì `data-testid` mặc định). Muốn dùng `getByTestId()`, thêm 1 dòng vào `playwright.config.ts`: `use: { testIdAttribute: "data-test" }` — sau đó `page.getByTestId("login-button")` sẽ chạy được.
{% endhint %}

## 3. Vì sao tránh XPath cứng?

So sánh 2 cách trỏ vào cùng 1 nút "Add to cart":

```typescript
// Cách 1: XPath theo cấu trúc HTML — ĐỪNG làm thế này
page.locator("//div[@class='inventory_list']/div[3]/div[2]/div[2]/button");

// Cách 2: theo cách người dùng nhìn
page.locator(".inventory_item")
  .filter({ hasText: "Sauce Labs Backpack" })
  .getByRole("button", { name: "Add to cart" });
```

Cách 1 vỡ ngay khi: dev thêm 1 sản phẩm (thứ tự đổi), đổi tên class, bọc thêm 1 div. Cách 2 chỉ vỡ khi chính sản phẩm hoặc nút biến mất — tức là khi _thực sự_ có thay đổi đáng để test fail. XPath không "sai" — nó chỉ mong manh, và test mong manh (flaky) là thứ khiến cả team mất niềm tin vào automation.

## 4. Chaining và filter — trỏ chính xác trong danh sách

Khi trang có nhiều phần tử giống nhau (danh sách sản phẩm, bảng dữ liệu), kết hợp locator để khoanh vùng:

```typescript
// Bước 1: khoanh vùng đúng "thẻ" sản phẩm cần thao tác
const backpackCard = page
  .locator(".inventory_item")
  .filter({ hasText: "Sauce Labs Backpack" });

// Bước 2: tìm tiếp BÊN TRONG vùng đó
await backpackCard.getByRole("button", { name: "Add to cart" }).click();

// Kiểm tra giá của đúng sản phẩm đó
await expect(backpackCard.locator(".inventory_item_price")).toHaveText("$29.99");
```

Tư duy: giống như chỉ đường — "vào tòa nhà A (filter), lên phòng 302 (chaining)" thay vì "cái cửa thứ 47 tính từ cổng".

## 5. Công cụ tìm locator nhanh

* **Codegen** (đã học Buổi 4): `npx playwright codegen <url>` — di chuột lên phần tử, Playwright gợi ý ngay locator tốt nhất.
* **Chế độ debug**: `npx playwright test --debug` — tab "Pick locator" cho phép click vào phần tử để lấy locator.
* **DevTools của trình duyệt**: chuột phải → Inspect để xem HTML khi cần đối chiếu.

## 6. Lỗi thường gặp

| Triệu chứng                                         | Nguyên nhân                                            | Cách xử lý                                                                                 |
| --------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| Lỗi "strict mode violation: resolved to N elements" | Locator trúng nhiều phần tử cùng lúc                   | Khoanh vùng bằng `.filter()` hoặc thêm điều kiện `{ name: ... }` cho cụ thể hơn            |
| Timeout dù phần tử nhìn thấy rõ trên trang          | Sai chữ hoa/thường hoặc thừa khoảng trắng trong `name` | Copy chính xác text trên trang; hoặc dùng regex: `{ name: /login/i }` để bỏ qua hoa thường |
| `getByText` không tìm thấy dù text có trên trang    | Text bị tách trong nhiều thẻ HTML con                  | Tìm bằng phần text ngắn hơn, hoặc dùng `getByRole` của phần tử cha                         |
| `getByTestId` không hoạt động trên Saucedemo        | Trang dùng `data-test`, không phải `data-testid`       | Config `testIdAttribute: "data-test"` — xem hint ở mục 2                                   |

## 7. Bài tập về nhà (45–60 phút)

Vào [DemoQA](https://demoqa.com) (mục Elements và Forms) và viết locator cho **ít nhất 15 phần tử khác nhau**, yêu cầu:

1. Không dùng XPath — mỗi locator kèm 1 dòng comment giải thích vì sao chọn cách đó
2. Có đủ ít nhất 4 loại: `getByRole`, `getByText`, `getByPlaceholder`, và 1 trường hợp phải dùng `.filter()` hoặc chaining
3. Viết thành file test chạy được: mỗi locator kèm 1 assertion đơn giản (`toBeVisible()` là đủ) để chứng minh locator trúng đích
4. Push lên repo lớp qua Pull Request, nhánh `ten-cua-ban/buoi5`

**Checklist trước khi nộp:** tất cả test pass · không có XPath nào · mỗi locator có comment lý do · đặt tên test có ý nghĩa.
