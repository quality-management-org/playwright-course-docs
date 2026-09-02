# Buổi 4 · Cài đặt Playwright & Test đầu tiên

{% hint style="info" %}
**Sau buổi này bạn sẽ:** tự cài Playwright, viết và chạy test đầu tiên trên một trang web thật, xem browser tự động chạy theo code của mình — và biết dùng Codegen để máy tự sinh code test.
{% endhint %}

## 1. Playwright là gì?

Playwright là công cụ automation testing do Microsoft phát triển: bạn viết code mô tả thao tác (mở trang, điền form, bấm nút) và điều kiện kiểm tra — Playwright điều khiển browser thực hiện và báo kết quả pass/fail.

So với các tool phổ biến khác:

| Tiêu chí                          | Playwright                         | Selenium       | Cypress       |
| --------------------------------- | ---------------------------------- | -------------- | ------------- |
| Tự động chờ element (auto-wait)   | Có sẵn                             | Phải tự code   | Có sẵn        |
| Browser hỗ trợ                    | Chromium, Firefox, WebKit (Safari) | Đa dạng        | Hạn chế hơn   |
| Codegen & Trace Viewer            | Có sẵn, rất mạnh                   | Không          | Một phần      |
| API testing tích hợp              | Có                                 | Không          | Có            |

Không có tool nào "thắng tuyệt đối" — nhưng kỹ năng cốt lõi bạn học trong khóa (locator, assertion, POM, tư duy test design) dùng lại được ở mọi tool.

## 2. Cài đặt Playwright

{% hint style="success" %}
**Nên chạy lệnh này ở nhà trước buổi học** — bước tải browser nặng vài trăm MB, mạng yếu sẽ chờ lâu.
{% endhint %}

```bash
npm init playwright@latest
```

Trả lời các câu hỏi khi cài như sau:

| Câu hỏi                          | Chọn                   | Vì sao                    |
| -------------------------------- | ---------------------- | ------------------------- |
| TypeScript or JavaScript?        | **TypeScript**         | Ngôn ngữ của khóa học     |
| Where to put your tests?         | **tests** (mặc định)   | Theo chuẩn chung          |
| Add a GitHub Actions workflow?   | **No**                 | Khóa không dùng CI/CD     |
| Install Playwright browsers?     | **Yes**                | Cần browser để chạy test  |

Cấu trúc project sau khi cài:

| File / Folder          | Vai trò                                                    |
| ---------------------- | ----------------------------------------------------------- |
| `tests/`               | Nơi chứa các file test — bạn làm việc chủ yếu ở đây          |
| `playwright.config.ts` | Cấu hình chung (browser, timeout...) — Buổi 9 học sâu        |
| `playwright-report/`   | Report HTML sinh ra sau mỗi lần chạy test                    |

## 3. Test đầu tiên — login vào Saucedemo

Tạo file `tests/login.spec.ts`:

```typescript
import { test, expect } from "@playwright/test";

test("login thành công với tài khoản hợp lệ", async ({ page }) => {
  await page.goto("https://www.saucedemo.com");
  await page.fill("#user-name", "standard_user");
  await page.fill("#password", "secret_sauce");
  await page.click("#login-button");
  await expect(page).toHaveURL(/inventory/);
});
```

Giải thích từng dòng:

| Dòng code                                     | Ý nghĩa                                                                                                                          |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `import { test, expect }`                     | Lấy 2 công cụ từ thư viện Playwright (destructuring khi import)                                                                    |
| `async ({ page }) =>`                         | Kiến thức Buổi 1 + 2 hội tụ: destructuring lấy `page` (đại diện tab browser) + async vì thao tác web cần chờ                       |
| `await page.goto(...)`                        | Mở trang web, chờ trang tải xong                                                                                                   |
| `await page.fill("#user-name", ...)`          | Điền text vào ô có id là user-name (dấu # = tìm theo id)                                                                           |
| `await page.click(...)`                       | Bấm nút login                                                                                                                      |
| `await expect(page).toHaveURL(/inventory/)`   | **Assertion** — kiểm tra URL có chứa "inventory" (tức đã login thành công). Không đúng → test fail                                 |

> Selector dạng `#user-name` là cách nhanh để chạy được ngay hôm nay. Buổi 5 bạn sẽ học cách chọn locator chuẩn và bền hơn — đừng vội quen tay với cách này.

## 4. Các lệnh chạy test cần nhớ

| Lệnh                            | Tác dụng                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------- |
| `npx playwright test`           | Chạy toàn bộ test ở chế độ headless (không hiện browser — nhanh)                 |
| `npx playwright test --headed`  | Chạy và **hiện browser** để xem từng thao tác — hãy thử ngay, rất đã mắt!        |
| `npx playwright test --debug`   | Chạy từng bước một với Inspector — dùng khi test fail không hiểu vì sao          |
| `npx playwright show-report`    | Mở report HTML của lần chạy gần nhất                                             |
| `npx playwright install`        | Cài browser (khi gặp lỗi báo thiếu browser)                                      |

## 5. Codegen — để máy tự sinh code test

Codegen mở browser, bạn thao tác bằng tay và nó tự viết code tương ứng:

```bash
npx playwright codegen https://www.saucedemo.com
```

Cách dùng: thao tác flow bạn muốn (ví dụ login → thêm sản phẩm vào giỏ) → copy code sinh ra ở cửa sổ bên cạnh → dán vào file test trong project → chạy thử.

{% hint style="warning" %}
**Codegen là bản nháp, không phải bản cuối.** Code sinh ra thường thừa thao tác, tên test vô nghĩa và locator chưa tối ưu — luôn cần con người đọc lại, dọn dẹp và đặt tên lại. Tư duy "máy sinh code → người review" này cũng chính là cách làm việc với AI mà bạn sẽ học ở buổi cuối khóa.
{% endhint %}

## 6. Khi test fail — đọc lỗi thế nào?

Test fail là chuyện bình thường hằng ngày của automation tester. Cách đọc:

1. Nhìn dòng **Error** đầu tiên: thường ghi rõ thao tác nào fail và selector nào không tìm thấy
2. Thấy chữ **Timeout 30000ms exceeded** = Playwright đã chờ 30 giây mà element không xuất hiện — thường do selector sai hoặc trang chưa ở trạng thái mong đợi
3. Mở report (`npx playwright show-report`) — có **screenshot tại đúng thời điểm fail** để nhìn trang web lúc đó trông ra sao
4. Vẫn chưa rõ → chạy lại với `--debug` để đi từng bước

## 7. Lỗi thường gặp

| Triệu chứng                          | Nguyên nhân                        | Cách xử lý                                                     |
| ------------------------------------ | ---------------------------------- | --------------------------------------------------------------- |
| No tests found                       | Chạy lệnh sai folder               | cd vào folder gốc của project (nơi có playwright.config.ts)     |
| Báo thiếu browser executable         | Browser chưa được cài              | Chạy `npx playwright install`                                   |
| Test chạy loạn thứ tự, fail khó hiểu | Quên `await` trước thao tác page   | Rà lại: mọi dòng page.xxx đều phải có await                     |
| Timeout dù code đúng                 | Mạng chậm, trang tải lâu           | Chạy lại; kiểm tra mở được Saucedemo bằng tay không             |

## 8. Bài tập về nhà (45–60 phút)

1. Viết test **login thất bại khi sai password**: điền password sai, bấm login, và assert thông báo lỗi hiện ra (gợi ý: quan sát message lỗi trên trang, tìm selector của nó bằng chuột phải → Inspect)
2. Dùng Codegen record 1 user flow tùy chọn trên Saucedemo, sau đó **dọn code**: xóa thao tác thừa, đổi tên test có nghĩa
3. Push lên repo lớp qua Pull Request, nhánh `ten-cua-ban/buoi4`

**Checklist trước khi nộp:** cả 2 test đều pass khi chạy `npx playwright test` · tên test mô tả đúng hành vi · không còn thao tác thừa từ Codegen.
