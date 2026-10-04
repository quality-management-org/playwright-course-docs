---
description: >-
  Thao tác với mọi loại phần tử, viết assertion kiểm tra kết quả và hiểu cơ chế
  auto-wait giúp vĩnh biệt sleep().
icon: hand-pointer
---

# Buổi 6 · Actions, Assertions & Auto-wait

{% hint style="info" %}
**Sau buổi này bạn sẽ:** thao tác được với mọi loại phần tử (form, dropdown, checkbox, upload file...), viết assertion đúng chỗ đúng loại, và hiểu vì sao Playwright không bao giờ cần `sleep()`.
{% endhint %}

## 1. Actions — bộ thao tác của Playwright

Một test = chuỗi **thao tác** (actions) + các điểm **kiểm tra** (assertions). Đây là các action bạn sẽ dùng hằng ngày:

| Action                    | Tác dụng                               | Ví dụ                                                |
| ------------------------- | -------------------------------------- | ---------------------------------------------------- |
| `.click()`                | Bấm chuột trái                         | `await loginButton.click()`                          |
| `.fill(text)`             | Xóa sạch rồi điền text vào ô nhập      | `await usernameInput.fill("standard_user")`          |
| `.selectOption(value)`    | Chọn giá trị trong dropdown `<select>` | `await sortDropdown.selectOption("lohi")`            |
| `.check()` / `.uncheck()` | Tick / bỏ tick checkbox, radio         | `await agreeCheckbox.check()`                        |
| `.hover()`                | Di chuột lên phần tử (mở menu ẩn)      | `await menuItem.hover()`                             |
| `.dblclick()`             | Bấm đúp                                | `await cell.dblclick()`                              |
| `.press(key)`             | Gõ phím (Enter, Tab, Escape...)        | `await searchBox.press("Enter")`                     |
| `.setInputFiles(path)`    | Upload file vào input file             | `await uploadInput.setInputFiles("data/avatar.png")` |

Thử nhanh với dropdown sắp xếp của Saucedemo (sau khi login):

```typescript
// Sắp xếp sản phẩm theo giá thấp → cao
await page.locator(".product_sort_container").selectOption("lohi");
await expect(page.locator(".inventory_item_name").first()).toHaveText("Sauce Labs Onesie");
```

## 2. Assertions — trái tim của test

Action mà không có assertion thì chưa phải test — chỉ là "máy bấm nút". Assertion trả lời câu hỏi: _"kết quả có đúng như mong đợi không?"_

| Assertion                          | Kiểm tra điều gì                       |
| ---------------------------------- | -------------------------------------- |
| `toBeVisible()`                    | Phần tử hiển thị trên trang            |
| `toHaveText(text)`                 | Text **khớp chính xác toàn bộ**        |
| `toContainText(text)`              | Text **chứa** đoạn này (khớp một phần) |
| `toHaveValue(value)`               | Giá trị hiện tại của ô nhập            |
| `toHaveURL(url hoặc regex)`        | URL trang hiện tại                     |
| `toBeChecked()`                    | Checkbox/radio đang được tick          |
| `toBeEnabled()` / `toBeDisabled()` | Nút bấm được / bị khóa                 |
| `toHaveCount(n)`                   | Số lượng phần tử mà locator tìm thấy   |

```typescript
test("thêm sản phẩm vào giỏ hàng", async ({ page }) => {
  await page.goto("https://www.saucedemo.com");
  await page.getByPlaceholder("Username").fill("standard_user");
  await page.getByPlaceholder("Password").fill("secret_sauce");
  await page.getByRole("button", { name: "Login" }).click();

  await page.locator("[data-test='add-to-cart-sauce-labs-backpack']").click();

  // Badge giỏ hàng hiện số 1
  await expect(page.locator(".shopping_cart_badge")).toHaveText("1");
  // Nút đổi thành Remove — chứng tỏ đã thêm thành công
  await expect(page.locator("[data-test='remove-sauce-labs-backpack']")).toBeVisible();
});
```

{% hint style="info" %}
**Điểm cực hay của assertion trong Playwright:** `expect(...).toBeVisible()` không kiểm tra đúng 1 lần rồi kết luận — nó **tự thử lại liên tục** cho đến khi điều kiện đúng hoặc hết timeout. Vậy nên trang load chậm 1–2 giây cũng không làm test fail oan. Đây gọi là _web-first assertions_.
{% endhint %}

## 3. Soft assertions — kiểm tra nhiều điều, không dừng giữa chừng

Assertion thường: fail ở đâu, test **dừng ngay** ở đó — các kiểm tra phía sau không chạy. Khi cần kiểm tra nhiều field trên cùng 1 trang, điều này bất tiện: sửa lỗi thứ nhất, chạy lại mới thấy lỗi thứ hai...

`expect.soft` giải quyết: ghi nhận lỗi nhưng **chạy tiếp**, đến cuối test mới tổng kết fail:

```typescript
// Kiểm tra cả 3 thứ trong 1 lần chạy — thấy hết mọi lỗi cùng lúc
await expect.soft(page.locator(".title")).toHaveText("Products");
await expect.soft(page.locator(".shopping_cart_link")).toBeVisible();
await expect.soft(page.locator(".inventory_item")).toHaveCount(6);
```

Dùng khi: kiểm tra nhiều chi tiết độc lập của cùng 1 màn hình. **Không** dùng cho điều kiện sống còn (login thành công chưa) — chỗ đó phải dùng assertion thường để test dừng sớm, khỏi phí thời gian chạy tiếp trong trạng thái sai.

## 4. Auto-wait — vĩnh biệt sleep()

Ở các tool cũ, tester phải tự đoán thời gian chờ:

```typescript
// Kiểu cũ (Selenium-style) — ĐỪNG mang thói quen này sang Playwright
await page.click("#login-button");
await page.waitForTimeout(3000); // "chờ 3 giây cho chắc" 😱
```

Vấn đề: 3 giây là đoán mò. Mạng nhanh thì phí 3 giây mỗi test; mạng chậm hơn 3 giây thì test fail oan. Nhân với hàng trăm test là thảm họa.

Playwright làm khác: **trước mỗi action, nó tự chờ phần tử sẵn sàng** — hiển thị, không bị che, không bị disable, đứng yên (không đang trượt/animation) — rồi mới thao tác. Chờ đúng đến lúc cần, không thừa một giây.

{% hint style="warning" %}
**`page.waitForTimeout()` là anti-pattern số 1** trong checklist code review của lớp. Thấy nó trong PR là bị yêu cầu sửa. 99% trường hợp bạn "cần sleep" thực ra là cần một assertion đúng chỗ — ví dụ chờ trang sau login: đừng sleep, hãy `await expect(page).toHaveURL(/inventory/)`.
{% endhint %}

## 5. Lỗi thường gặp

| Triệu chứng                                       | Nguyên nhân                                        | Cách xử lý                                                                   |
| ------------------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `toHaveText` fail dù text nhìn đúng               | Text trên trang có thêm phần khác (giá, đơn vị...) | Dùng `toContainText` khi chỉ cần khớp một phần                               |
| Lỗi "element intercepts pointer events" khi click | Phần tử bị che bởi popup/banner khác               | Đóng popup trước, hoặc kiểm tra lại flow — có thể thiếu 1 bước               |
| Test thỉnh thoảng pass thỉnh thoảng fail (flaky)  | Thiếu assertion "neo" giữa các bước chuyển trang   | Thêm assertion xác nhận trạng thái (URL, element chính) trước bước tiếp theo |
| Quên `await` trước `expect(...)`                  | Assertion không được chờ, kết quả sai lệch         | Mọi assertion với locator đều cần `await` phía trước                         |

## 6. Bài tập về nhà (45–60 phút)

Viết test cho **toàn bộ checkout flow** của Saucedemo, trong 1 file `checkout.spec.ts`:

1. Login → thêm 2 sản phẩm vào giỏ → mở giỏ hàng → **assert đủ 2 sản phẩm đúng tên**
2. Bấm Checkout → điền thông tin (First Name, Last Name, Zip) → Continue
3. Ở trang tổng kết: **assert tổng tiền** (dùng `toContainText` với giá trị Item total)
4. Bấm Finish → **assert thông báo hoàn tất** "Thank you for your order!"
5. Yêu cầu kỹ thuật: chỉ dùng locator chuẩn đã học Buổi 5, tuyệt đối không có `waitForTimeout`, có ít nhất 1 chỗ dùng `expect.soft` hợp lý
6. Push lên repo lớp qua Pull Request, nhánh `ten-cua-ban/buoi6`

**Checklist trước khi nộp:** test pass ổn định khi chạy 3 lần liên tiếp · không có sleep · mỗi bước chuyển trang có assertion neo · tên test mô tả đúng hành vi.
