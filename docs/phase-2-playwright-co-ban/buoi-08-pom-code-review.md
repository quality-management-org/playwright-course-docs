---
description: >-
  Refactor toàn bộ test sang POM và học cách review code như một automation
  engineer thực thụ.
---

# Buổi 8 · Hoàn thiện POM & Code Review

{% hint style="info" %}
**Sau buổi này bạn sẽ:** có bộ page objects hoàn chỉnh cho Saucedemo, toàn bộ test cũ được refactor sạch sẽ, và — quan trọng nhất — biết review code của người khác theo checklist chuẩn. Kỹ năng review là thứ phân biệt junior và mid-level.
{% endhint %}

## 1. Hoàn thiện bộ page objects

Buổi này chúng ta chốt 2 trang cuối của flow mua hàng. Khung `CheckoutPage` — bạn tự điền phần còn thiếu tại lớp:

```typescript
import { Page, Locator } from "@playwright/test";

export class CheckoutPage {
  readonly page: Page;
  readonly firstNameInput: Locator;
  readonly lastNameInput: Locator;
  readonly zipInput: Locator;
  readonly continueButton: Locator;
  readonly finishButton: Locator;
  readonly completeMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.firstNameInput = page.getByPlaceholder("First Name");
    this.lastNameInput = page.getByPlaceholder("Last Name");
    this.zipInput = page.getByPlaceholder("Zip/Postal Code");
    this.continueButton = page.locator("[data-test='continue']");
    this.finishButton = page.locator("[data-test='finish']");
    this.completeMessage = page.locator(".complete-header");
  }

  async fillCustomerInfo(firstName: string, lastName: string, zip: string) {
    // TODO: bạn tự viết — 3 fill + 1 click Continue
  }

  async finishOrder() {
    // TODO: bạn tự viết
  }
}
```

Sau buổi này, cấu trúc project của bạn trông như một framework thật:

```
tests/
  login.spec.ts
  cart.spec.ts
  checkout.spec.ts
pages/
  login-page.ts
  inventory-page.ts
  cart-page.ts
  checkout-page.ts
```

## 2. Refactor an toàn — quy trình 3 bước

Refactor = đổi cách tổ chức code mà **không đổi hành vi**. Làm ẩu là từ "test pass" thành "test fail hàng loạt không hiểu vì sao". Quy trình an toàn:

1. **Chạy toàn bộ test cũ, xác nhận đang pass** — đây là "ảnh chụp" trạng thái đúng.
2. **Chuyển từng phần nhỏ một**: đưa phần login của 1 file test sang dùng `LoginPage` → chạy lại ngay → pass thì mới chuyển phần tiếp theo. Đừng đại tu cả project rồi mới chạy.
3. **Xóa code cũ chỉ sau khi bản mới đã pass.** Commit sau mỗi phần chuyển thành công — có chuyện gì còn quay đầu được.

{% hint style="success" %}
**Mẹo:** đây chính là lúc Git phát huy — mỗi bước refactor là 1 commit nhỏ (`refactor: chuyển login.spec sang dùng LoginPage`). PR của bạn sẽ dễ review hơn hẳn một commit khổng lồ "sửa hết mọi thứ".
{% endhint %}

## 3. Checklist Code Review — dùng từ hôm nay đến hết khóa

Từ buổi này, mỗi PR đều được một bạn học review trước khi trainer duyệt. Review theo đúng 8 câu hỏi:

* [ ] Locator có ưu tiên `getByRole` / `getByLabel` / `getByText` không?
* [ ] Không có XPath cứng (`//div[@class='...']`) nào chứ?
* [ ] Không có `page.waitForTimeout()` (hardcoded sleep) nào chứ?
* [ ] Mỗi test có chạy độc lập không? (không phụ thuộc thứ tự, không dùng chung dữ liệu để lại từ test trước)
* [ ] Test data được tạo và dọn sạch đúng cách chưa?
* [ ] Mỗi method trong page object có đúng một trách nhiệm không?
* [ ] Có magic string/number nào lặp lại chưa được đưa thành hằng số không?
* [ ] Assertion có đủ và nằm đúng chỗ (trong test, không trong page object) không?

## 4. Review code sao cho tử tế và hiệu quả

Review không phải là "bắt lỗi nhau" — là cùng nâng chất lượng code chung. Vài nguyên tắc từ môi trường làm việc thật:

| Nên                                                        | Tránh                                    |
| ---------------------------------------------------------- | ---------------------------------------- |
| Comment vào **đúng dòng code** cụ thể trên PR              | Nhận xét chung chung "code chưa ổn lắm"  |
| Hỏi để hiểu: _"Vì sao bạn chọn CSS selector ở đây?"_       | Phán xét: _"Sai rồi, dùng getByRole đi"_ |
| Khen điểm tốt trước khi góp ý: _"Tách method này hay đấy"_ | Chỉ liệt kê lỗi                          |
| Gợi ý kèm ví dụ code ngắn                                  | Bắt sửa mà không nói sửa thế nào         |

Và ở vai người **được** review: mọi comment là góp ý cho code, không phải chê con người bạn. Trả lời từng comment (sửa rồi / giải thích lý do giữ nguyên) — kỹ năng phản hồi review chuyên nghiệp cũng là thứ nhà tuyển dụng để ý khi xem GitHub của bạn.

## 5. Tổng kết anti-patterns — bảng "cấm kỵ" của automation

| Anti-pattern                | Vì sao tệ                                                 | Thay bằng                                             |
| --------------------------- | --------------------------------------------------------- | ----------------------------------------------------- |
| `page.waitForTimeout(3000)` | Chậm khi thừa, flaky khi thiếu — thua đủ đường            | Assertion neo trạng thái (`toHaveURL`, `toBeVisible`) |
| XPath bám cấu trúc HTML     | UI đổi nhẹ là vỡ, không ai đọc hiểu                       | `getByRole` + `filter` (Buổi 5)                       |
| Test phụ thuộc thứ tự chạy  | Chạy song song hoặc chạy lẻ 1 test là fail                | Mỗi test tự chuẩn bị dữ liệu của mình                 |
| Magic string rải khắp nơi   | `"standard_user"` xuất hiện 25 chỗ, đổi 1 lần sửa 25 chỗ  | Hằng số / test data tập trung (học kỹ ở Phase 4)      |
| Method ôm đồm làm 5 việc    | `loginAndAddToCartAndCheckout()` — không tái sử dụng được | Tách nhỏ theo hành vi, test tự ghép các bước          |

## 6. Lỗi thường gặp khi refactor

| Triệu chứng                         | Nguyên nhân                                         | Cách xử lý                                                        |
| ----------------------------------- | --------------------------------------------------- | ----------------------------------------------------------------- |
| Refactor xong fail hàng loạt        | Đổi quá nhiều thứ cùng lúc rồi mới chạy test        | Quay lại commit gần nhất còn pass, chuyển lại từng phần nhỏ       |
| Hai class cùng khai báo 1 locator   | Locator dùng chung (header, menu) bị copy nhiều nơi | Tạm chấp nhận ở Phase 2; Phase 4 sẽ học cách tách component chung |
| Import vòng: A import B, B import A | Hai page object gọi lẫn nhau                        | Page object không import page object khác — test là nơi ghép nối  |

## 7. Bài tập về nhà (45–60 phút)

1. Hoàn thiện `CheckoutPage` (2 method TODO ở mục 1)
2. Refactor **toàn bộ** test từ Buổi 4–6 sang dùng page objects — kể cả bài checkout flow của Buổi 6
3. Tạo PR, sau đó **review 1 PR của bạn học** theo checklist mục 3: tối thiểu 3 comment, trong đó ít nhất 1 comment khen điểm làm tốt
4. Nhánh: `ten-cua-ban/buoi8`

**Checklist trước khi nộp:** toàn bộ test pass · không còn file test nào đụng trực tiếp locator · đã review PR của bạn học · đã phản hồi các comment trên PR của mình.

{% hint style="info" %}
**Cột mốc:** hoàn thành buổi này, bạn đã có một mini-framework POM đúng chuẩn đi làm — thứ mà nhiều tester 1–2 năm kinh nghiệm vẫn chưa từng tự xây. Phase 3 sẽ nâng nó lên mức production: config đa browser, fixtures, chạy song song và API testing.
{% endhint %}
