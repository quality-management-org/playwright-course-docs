# Đề cương khóa học (tham chiếu nội bộ, không hiển thị trên GitBook)

File này tóm tắt nội dung 15 buổi, dùng làm căn cứ khi viết tài liệu các buổi tiếp theo. Nguồn: trang Notion "Automation Testing từ Zero đến Hero". Khi đề cương trên Notion thay đổi, cập nhật lại file này.

Thông tin chung: 15 buổi, 2 buổi/tuần, 2 giờ/buổi, khoảng 8 tuần. Bài tập về nhà 45-60 phút mỗi buổi, nộp qua Pull Request vào repo chung của lớp. Trang luyện tập: Saucedemo (Phase 2), DemoQA, The Internet, ReqRes (Phase 3).

## Phase 1: Nền tảng TypeScript (Buổi 1-3)

### Buổi 1: Setup & TypeScript cơ bản
Cài đặt Node.js, VS Code (ESLint, Prettier, Playwright extension), Git. Chạy file .ts đầu tiên. Variables (let/const), các kiểu cơ bản. Toán tử quan hệ, toán tử bằng (=== và !==), toán tử logic. Câu lệnh if/else, else if, switch, toán tử ba ngôi. Functions, arrow functions, optional parameter và default parameter. Arrays, objects. Vòng lặp for...of, for với biến đếm, while, break/continue. Destructuring (trọng tâm, nền tảng của cú pháp `async ({ page })`). Coding convention: camelCase, tên có ý nghĩa.

### Buổi 2: Git cơ bản & Async/Await
Async/await, try/catch, gọi API bằng fetch (dạy trước, dành nhiều thời gian thực hành nhất buổi). Git: clone, tạo branch theo quy ước ten-hoc-vien/buoi-X, add, commit, push, Pull Request. Nhánh main có branch protection, học viên cần Personal Access Token.

### Buổi 3: Interfaces, Classes, npm & Array methods
Interface vs type alias, optional, union type. Class, constructor, public/private (ví dụ UserAccount). Convention PascalCase. import/export, package.json, tsconfig cơ bản. Array methods: map, filter, find. Bài checkpoint cuối Phase 1: mini-script gọi API, filter, map, log kết quả. Học viên chưa tự làm được cần kèm thêm trước Buổi 4.

## Phase 2: Playwright cơ bản (Buổi 4-8)

### Buổi 4: Cài đặt, test đầu tiên & Codegen
So sánh Playwright, Selenium, Cypress. Cài bằng npm init playwright@latest. Test login Saucedemo: goto, fill, click, expect. Chạy headed/headless, đọc HTML report. Codegen, Inspector, debug.

### Buổi 5: Locators
Thứ tự ưu tiên: getByRole, getByLabel, getByPlaceholder, getByText, getByTestId, CSS selector; tránh XPath. Chaining và filter. Config testIdAttribute cho Saucedemo (data-test).

### Buổi 6: Actions, Assertions & Auto-wait
Actions: click, fill, selectOption, check, hover, dblclick, press, setInputFiles. Assertions: toBeVisible, toHaveText, toContainText, toHaveValue, toHaveURL, toBeChecked, toBeEnabled, toHaveCount. Soft assertions. Cơ chế auto-wait; không dùng waitForTimeout.

### Buổi 7: Page Object Model
Vấn đề lặp code và chi phí bảo trì. Xây LoginPage, InventoryPage. Quy ước: method đặt tên theo hành vi người dùng, assertion nằm trong test, mỗi trang một file.

### Buổi 8: Hoàn thiện POM & Code Review
CartPage, CheckoutPage. Quy trình refactor an toàn 3 bước. Checklist code review 8 mục, áp dụng từ buổi này đến hết khóa. Tổng kết anti-patterns.

## Phase 3: Nâng cao (Buổi 9-12)

### Buổi 9: playwright.config.ts, Fixtures & Chạy song song
Multiple projects (Chromium, Firefox, WebKit, mobile emulation). baseURL, timeout, retries, workers. Screenshot và video khi test fail. beforeAll/Each, afterAll/Each. Custom fixture (ví dụ authenticatedPage tự động login).

### Buổi 10: Tổ chức Test Suite
test.describe, test.skip, test.only, test.fixme. Tags @smoke @regression @critical, chạy theo tag với --grep. Parameterized tests với nhiều bộ dữ liệu.

### Buổi 11: API Testing với Playwright
APIRequestContext: get, post, put, delete. Headers, body, Bearer token. Assert status code, response body, JSON fields. Thực hành CRUD trên ReqRes hoặc JSONPlaceholder, không cần mở browser.

### Buổi 12: API + UI kết hợp & Network Mocking
Chiến lược tạo test data qua API rồi verify trên UI; dọn data sau test để test độc lập. page.route mock API response. Giả lập lỗi 500, empty state, timeout.

## Phase 4: Framework, Mini Project & AI (Buổi 13-15)

### Buổi 13: Framework hoàn chỉnh
Cấu trúc folder chuẩn: tests, pages, fixtures, utils, data, types. File .env cho credentials, tsconfig paths alias. Test data: inline, JSON file, factory function, Faker. Trace Viewer. HTML Reporter chuyên sâu. Tổng kết anti-patterns. Cuối buổi giao đề mini project: nhóm 2-3 người, tự chọn app thực tế, nộp test plan trước Buổi 14.

### Buổi 14: Mini Project thực chiến
Implement test suite theo test plan đã nộp, áp dụng đủ: POM, fixtures, API testing, config đa browser, tagging. Trainer hỗ trợ định hướng, không làm thay.

### Buổi 15: Áp dụng AI vào Automation & Tốt nghiệp
Demo mini project (khoảng 30 phút, mỗi nhóm 5 phút). AI trong automation testing: dùng GitHub Copilot, ChatGPT, Claude sinh test, locator, test data; kỹ thuật prompt kèm context; review code AI sinh ra bằng checklist của lớp; demo AI agent điều khiển browser qua Playwright MCP; giới hạn và rủi ro. Tổng kết: roadmap tiếp theo (k6, Appium, CI/CD), tips phỏng vấn vị trí Automation QA.