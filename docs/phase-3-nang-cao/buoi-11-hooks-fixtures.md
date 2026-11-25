---
description: "Nhóm test theo tính năng bằng test.describe, chuẩn bị trạng thái cho test bằng hook và custom fixture, và viết parameterized test để một logic kiểm tra chạy với nhiều bộ dữ liệu."
icon: layer-group
---

# Buổi 11 · Hooks, Fixtures & Tổ chức Test Suite

Khi số lượng test tăng, code test bắt đầu lặp lại: cùng một bước chuẩn bị ở đầu nhiều test, cùng một logic kiểm tra cho nhiều bộ dữ liệu. Buổi này trình bày cách tổ chức code test để loại bỏ phần lặp lại.

## 1. Nhóm test với test.describe

### Cú pháp

`test.describe` nhận một tên nhóm và một hàm chứa các test thuộc nhóm đó. Tên nhóm được ghép vào trước tên test ở mọi nơi Playwright hiển thị test: terminal, HTML report, Trace Viewer. Lệnh `--list` liệt kê tên đầy đủ của các test mà không chạy test:

![login.spec.ts nhóm hai test bằng test.describe và kết quả của --list](../.gitbook/assets/buoi-11-describe.png)

<!-- TODO(image): HTML report sau khi chạy npx playwright test với các file test đã nhóm bằng test.describe: mỗi dòng hiển thị đường dẫn nhóm trước tên test, ví dụ "Login › Logs in successfully with a valid account" và "Cart › Remove product › Cart is empty after removing the only product". Khoanh đỏ phần tên nhóm ở hai dòng, một dòng có một cấp describe và một dòng có hai cấp. -->

### Nhóm lồng nhau

Một `describe` có thể đặt bên trong một `describe` khác để tạo nhóm con:

![describe lồng nhau trong cart.spec.ts](../.gitbook/assets/buoi-11-nested-describe.png)

Tên đầy đủ của test là `Cart › Remove product › Cart is empty after removing the only product`.

### Quy ước đặt tên

| Thành phần     | Quy ước                                                                 | Ví dụ                                             |
| -------------- | ----------------------------------------------------------------------- | ------------------------------------------------- |
| Tên file       | Một tính năng một file, `<feature>.spec.ts`, tiếng Anh kebab-case       | `checkout.spec.ts`                                |
| Tên `describe` | Danh từ chỉ tính năng hoặc thao tác, tiếng Anh, viết hoa chữ cái đầu    | `"Checkout"`, `"Remove product"`                  |
| Tên `test`     | Câu tiếng Anh mô tả kết quả mong đợi, viết hoa chữ cái đầu             | `"Cart is empty after removing the only product"` |

## 2. Hooks

### Code chuẩn bị lặp lại

Mọi test trong `cart.spec.ts` và `checkout.spec.ts` bắt đầu bằng cùng một đoạn code đăng nhập:

![Đoạn code đăng nhập lặp lại ở đầu mỗi test](../.gitbook/assets/buoi-11-repeated-login.png)

**Hook** là hàm Playwright tự động gọi tại các thời điểm xác định trong quá trình chạy test.

### beforeEach

Hook `beforeEach` tập trung đoạn đăng nhập về một vị trí trong `tests/cart.spec.ts`:

![Hook beforeEach đăng nhập trước mỗi test trong nhóm Cart](../.gitbook/assets/buoi-11-before-each.png)

Hai test trong nhóm không còn đoạn đăng nhập ở đầu thân test:

![Hai test của nhóm Cart sau khi dùng beforeEach](../.gitbook/assets/buoi-11-before-each-tests.png)

* `page` trong `beforeEach` và `page` trong test là cùng một object: hook đăng nhập xong, test nhận trang đã đăng nhập.
* `page.waitForURL()` là bước chờ cần thiết. `login()` kết thúc ngay sau khi bấm nút Login, trong khi Toolshop cần thêm thời gian để xử lý đăng nhập. Thiếu bước chờ này, `homePage.goto()` có thể chuyển trang trước khi đăng nhập hoàn tất, và test fail không ổn định.

### Hook trong API test

Các test giỏ hàng qua API cũng bắt đầu bằng cùng hai bước chuẩn bị: `findProductId()` và `createCart()`. Chuyển hai bước này vào `beforeEach` trong `tests/api/carts.spec.ts`:

![beforeEach chuẩn bị sản phẩm và giỏ hàng cho các test giỏ hàng qua API](../.gitbook/assets/buoi-11-api-before-each.png)

* Biến `productId` và `cartId` khai báo bằng `let` trong `describe` để hook và các test trong nhóm cùng sử dụng.
* Hook gán giá trị mới trước mỗi test, nên mỗi test vẫn nhận một giỏ hàng mới.
* Hook nhận fixture giống test: `request` của hook và của test là cùng một object.
* Tag `@api` khai báo ở `describe` áp dụng cho mọi test trong nhóm.

### Phạm vi của hook

Hook có hiệu lực trong phạm vi nơi nó được khai báo:

| Vị trí khai báo          | Phạm vi áp dụng                                    |
| ------------------------ | -------------------------------------------------- |
| Ngoài mọi `describe`     | Mọi test trong file                                |
| Bên trong một `describe` | Chỉ các test của nhóm đó, kể cả nhóm con bên trong |

Ví dụ, chỉ các test xóa sản phẩm cần có sẵn sản phẩm trong giỏ. Bước chuẩn bị này đặt trong `beforeEach` của nhóm "Remove product", không ảnh hưởng tới các test khác của nhóm "Cart":

![beforeEach của nhóm Cart và nhóm con Remove product](../.gitbook/assets/buoi-11-hook-scope.png)

Thứ tự chạy với test "Cart is empty after removing the only product":

1. `beforeEach` của nhóm "Cart".
2. `beforeEach` của nhóm "Remove product".
3. Test.

### Các loại hook

| Hook              | Thời điểm chạy                                                | Fixture sử dụng được                    |
| ----------------- | ------------------------------------------------------------- | --------------------------------------- |
| `test.beforeEach` | Trước mỗi test trong phạm vi                                  | `page` và mọi fixture cấp test          |
| `test.afterEach`  | Sau mỗi test trong phạm vi, kể cả khi test fail               | `page` và mọi fixture cấp test          |
| `test.beforeAll`  | Một lần trước các test trong phạm vi, lặp lại trên mỗi worker | `browser`, không có `page` và `context` |
| `test.afterAll`   | Một lần sau các test trong phạm vi, lặp lại trên mỗi worker   | `browser`, không có `page` và `context` |

Thứ tự chạy với một nhóm gồm hai test, trên cùng một worker:

```
beforeAll
  beforeEach → test 1 → afterEach
  beforeEach → test 2 → afterEach
afterAll
```

Hook `afterEach` nhận tham số thứ hai `testInfo`, chứa thông tin về test vừa chạy. Ví dụ ghi lại URL cuối cùng của các test không pass:

![Hook afterEach dùng testInfo để ghi log khi test không pass](../.gitbook/assets/buoi-11-after-each.png)

{% hint style="warning" %}
**Không dùng `beforeAll` để đăng nhập một lần rồi dùng chung cho mọi test:**

* `beforeAll` không có `page`. Khai báo `page` trong `beforeAll` làm Playwright báo lỗi `"context" and "page" fixtures are not supported in "beforeAll"`.
* Với nhiều worker, `beforeAll` chạy một lần trên mỗi worker, không phải một lần cho cả bộ test.
* Tự tạo `page` trong `beforeAll` để dùng chung giữa các test làm các test phụ thuộc nhau.

`beforeAll` chỉ phù hợp với việc chuẩn bị không thao tác trên trang web.
{% endhint %}

### Hạn chế của hook

* `beforeEach` chỉ có hiệu lực trong file khai báo nó. `cart.spec.ts` và `checkout.spec.ts` đều cần đăng nhập, nên đoạn `beforeEach` phải sao chép sang cả hai file.
* Mọi test trong phạm vi đều chạy `beforeEach`, kể cả test không cần đăng nhập.
* Các dòng `new ...Page(page)` vẫn lặp lại trong từng test.

Custom fixture giải quyết các hạn chế này.

## 3. Custom fixture

### Khái niệm fixture

Mỗi test nhận `page` qua cú pháp destructuring `async ({ page })`. API test khai báo `request` và `playwright` theo cùng cú pháp. Các đối tượng này là **fixture**: Playwright chuẩn bị trước khi test chạy và dọn dẹp sau khi test kết thúc.

| Giai đoạn | Việc Playwright thực hiện với fixture `page` |
| --------- | -------------------------------------------- |
| Setup     | Tạo browser context mới, mở một tab mới      |
| Sử dụng   | Truyền `page` vào test                       |
| Teardown  | Đóng browser context sau khi test kết thúc   |

Playwright cho phép định nghĩa fixture riêng cho project bằng `test.extend`. Fixture được định nghĩa một lần trong một file và sử dụng được ở mọi file test.

### Bước 1: tạo fixture đầu tiên

Tạo folder `fixtures/` ngang hàng với `tests/` và `pages/`, sau đó tạo file `fixtures/test-fixtures.ts`:

![Cấu trúc project sau khi thêm folder fixtures](../.gitbook/assets/buoi-11-project-structure.png)

Nội dung file `fixtures/test-fixtures.ts`:

![fixtures/test-fixtures.ts với fixture loginPage đầu tiên](../.gitbook/assets/buoi-11-fixture-first.png)

Các thành phần của một fixture:

![Cấu tạo của một fixture: test as base, base.extend, tên fixture, các fixture cần dùng và hàm use](../.gitbook/assets/buoi-11-fixture-anatomy.png)

### Bước 2: sử dụng fixture trong test

File test import `test` và `expect` từ file fixture thay vì từ `@playwright/test`, sau đó khai báo fixture cần dùng trong tham số của test:

![Test đăng nhập dùng fixture loginPage](../.gitbook/assets/buoi-11-use-fixture.png)

So với cách viết trước:

* Test không còn dòng `const loginPage = new LoginPage(page);` và không cần import `LoginPage`.
* Playwright chỉ tạo fixture mà test khai báo: test không khai báo `loginPage` thì fixture `loginPage` không được tạo.

{% hint style="warning" %}
Sau khi có file fixture, mọi file test phải import `test` từ `@fixtures/test-fixtures`, kể cả các file API test trong `tests/api/`. File còn import từ `@playwright/test` mà khai báo fixture mới sẽ gặp hai lỗi:

* VS Code báo lỗi `Property 'loginPage' does not exist on type ...`.
* Khi chạy, Playwright báo lỗi `Test has unknown parameter "loginPage"`.
{% endhint %}

![VS Code báo lỗi loginPage khi file test còn import test từ @playwright/test](../.gitbook/assets/buoi-11-fixture-import-error.png)

### Bước 3: fixture có setup và sử dụng fixture khác

Bổ sung fixture `authenticatedPage`: một `page` đã đăng nhập bằng tài khoản chung. Cập nhật `fixtures/test-fixtures.ts`:

![Fixture authenticatedPage dùng page và loginPage để đăng nhập trước khi test chạy](../.gitbook/assets/buoi-11-authenticated-page.png)

`authenticatedPage` cần hai fixture khác: `page` của Playwright và `loginPage` vừa tạo. Khi một test khai báo `authenticatedPage`, Playwright:

* tạo các fixture theo thứ tự phụ thuộc: `page`, `loginPage`, rồi `authenticatedPage`;
* chạy test;
* dọn dẹp các fixture theo thứ tự ngược lại.

![Vòng đời của ba fixture page, loginPage, authenticatedPage: setup từ ngoài vào trong, teardown theo thứ tự ngược lại](../.gitbook/assets/buoi-11-fixture-lifecycle.png)

`authenticatedPage` và `page` là cùng một object, khác nhau ở trạng thái: khi test bắt đầu, `authenticatedPage` đã đăng nhập.

### Bước 4: fixture cho page object

Tương tự `loginPage`, khai báo fixture cho các page object còn lại. Nội dung hoàn chỉnh của `fixtures/test-fixtures.ts`:

![Nội dung hoàn chỉnh của fixtures/test-fixtures.ts](../.gitbook/assets/buoi-11-fixtures-complete.png)

Viết lại `tests/cart.spec.ts` bằng fixture:

![cart.spec.ts viết lại bằng fixture](../.gitbook/assets/buoi-11-cart-with-fixtures.png)

Test thứ nhất khai báo `authenticatedPage` nhưng không sử dụng biến này trong thân test:

* Việc khai báo tương đương mục **điều kiện tiên quyết** "Đã đăng nhập" của một test case thủ công: Playwright đăng nhập trước khi test chạy.
* Test không khai báo `authenticatedPage` bắt đầu ở trạng thái chưa đăng nhập.

### So sánh trước và sau khi dùng fixture

| Tiêu chí                                      | Trước khi dùng fixture                           | Sau khi dùng fixture                                   |
| --------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------ |
| Bước đăng nhập                                | Ba dòng code ở đầu mọi test cần đăng nhập        | Khai báo `authenticatedPage` trong tham số của test    |
| Tạo page object                               | `new HomePage(page)` trong từng test             | Khai báo `homePage` trong tham số của test             |
| Thay đổi cách đăng nhập (ví dụ thêm bước chờ) | Sửa mọi test, trong mọi file                     | Sửa một fixture                                        |
| Test không cần đăng nhập                      | Vẫn chạy `beforeEach` nếu nằm trong phạm vi hook | Không khai báo `authenticatedPage`, không bị đăng nhập |

### Hook hay fixture

| Tiêu chí           | Hook (`beforeEach`)                                            | Fixture (`test.extend`)                                                  |
| ------------------ | -------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Phạm vi            | Một file hoặc một nhóm `describe`                              | Mọi file import file fixture                                             |
| Test bị ảnh hưởng  | Mọi test trong phạm vi, kể cả test không cần                   | Chỉ test khai báo fixture                                                |
| Kết hợp            | Không dùng lại được ở file khác                                | Fixture sử dụng được fixture khác                                        |
| Trường hợp sử dụng | Bước chuẩn bị chỉ dành cho các test của một file hoặc một nhóm | Trạng thái dùng chung toàn project: đăng nhập, page object, dữ liệu test |

Quy ước của khóa học:

* Trạng thái dùng chung toàn project, như đăng nhập và page object, khai báo bằng fixture.
* Hook chỉ dùng cho bước chuẩn bị riêng của một file hoặc một nhóm, ví dụ bước thêm một sản phẩm vào giỏ của nhóm "Remove product".

## 4. Thực hành tại lớp: fixture cho checkout

Áp dụng fixture cho file `tests/checkout.spec.ts`.

### Các bước thực hiện

1. Đọc lại các test trong `tests/checkout.spec.ts`. Với mỗi bước trong test, xác định bước đó thuộc **điều kiện tiên quyết** hay thuộc **nội dung đang được kiểm tra**.
2. Bổ sung fixture `cartPage` và `checkoutPage` vào `fixtures/test-fixtures.ts`.
3. Chuyển `tests/checkout.spec.ts` sang import `test`, `expect` từ file fixture, nhóm test bằng `describe` và sử dụng các fixture đã có, theo quy trình refactor 3 bước.
4. Chạy `npx playwright test tests/checkout.spec.ts`.

### Câu hỏi thiết kế

Trả lời các câu hỏi sau trước khi viết code:

* Fixture `cartPage` có nên tự thêm sản phẩm vào giỏ trước khi chuyển cho test không? Xét trường hợp một test cần kiểm tra giỏ hàng trống.
* Fixture `homePage` nên được tạo từ `page` hay từ `authenticatedPage`? Nêu hệ quả của mỗi lựa chọn đối với test không cần đăng nhập.
* Bước Sign in của trang checkout nên nằm trong fixture, trong `beforeEach` hay trong test? Căn cứ vào câu trả lời ở bước 1.

## 5. Parameterized test

### Một test chứa nhiều bộ dữ liệu

Cần kiểm tra trang chi tiết hiển thị đúng tên với ba sản phẩm khác nhau. Cách viết đơn giản nhất là duyệt mảng dữ liệu bên trong một test:

![Anti-pattern: duyệt ba bộ dữ liệu trong cùng một test](../.gitbook/assets/buoi-11-loop-anti-pattern.png)

Cách viết này có ba hạn chế:

| Hạn chế                              | Hệ quả                                                        |
| ------------------------------------ | ------------------------------------------------------------- |
| Assertion fail dừng cả test          | Sản phẩm thứ hai fail thì sản phẩm thứ ba không được kiểm tra |
| Report chỉ có một dòng               | Phải đọc thông báo lỗi mới biết bộ dữ liệu nào fail           |
| Không chạy riêng được một bộ dữ liệu | Muốn kiểm tra lại một sản phẩm phải chạy cả ba                |

### Vòng lặp sinh ra nhiều test

Đặt vòng lặp bên ngoài `test`, mỗi vòng khai báo một test riêng:

* Vòng `for` chạy khi Playwright đọc file, trước khi bất kỳ test nào chạy.
* Tên test là template string chứa dữ liệu của chính nó, nên `--list` và report cho biết ngay bộ dữ liệu nào đang chạy.

![Vòng lặp đặt ngoài test sinh ra ba test riêng, kiểm tra bằng --list](../.gitbook/assets/buoi-11-parameterized.png)

Ba test có cùng vị trí khai báo (dòng 7) nhưng là ba test độc lập:

* Các test chạy song song trên các worker khác nhau.
* Mỗi test có browser context riêng và một dòng riêng trong report.
* Một test cụ thể chạy riêng được bằng `--grep "Bolt Cutters"`.

### Tên test phải duy nhất

Playwright yêu cầu tên đầy đủ của mỗi test là duy nhất. Nếu tên test trong vòng lặp không chứa dữ liệu, ví dụ `test("Shows the correct name", ...)`, ba vòng lặp sinh ra ba test cùng tên và Playwright báo lỗi `duplicate test title` ngay khi đọc file, trước khi chạy bất kỳ test nào.

Khi mỗi bộ dữ liệu là một object gồm nhiều trường, ví dụ email, password và kết quả mong đợi của một lần đăng nhập:

* Khai báo kiểu cho object bằng `type` hoặc `interface`.
* Ghép trường mô tả tình huống vào tên test.

### Trường hợp nên và không nên parameterize

| Nên                                                                   | Không nên                                                                 |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Các bước giống hệt nhau, chỉ khác dữ liệu đầu vào và kết quả mong đợi | Các tình huống có luồng thao tác khác nhau, phải thêm `if` trong thân test |
| Kiểm tra validation của form: thiếu trường, sai định dạng, sai độ dài | Chỉ có một hoặc hai tình huống, viết hai test thông thường dễ đọc hơn      |
| Cùng một luồng với nhiều sản phẩm hoặc nhiều loại tài khoản           | Bộ dữ liệu quá lớn làm bộ test chạy lâu mà không thêm giá trị kiểm thử    |

## 6. Tóm tắt

| Nội dung           | Điểm chính                                                                                           |
| ------------------ | ---------------------------------------------------------------------------------------------------- |
| `test.describe`    | Nhóm test theo tính năng, tên nhóm ghép vào tên test                                                 |
| Hook               | Chuẩn bị và dọn dẹp cho các test trong một file hoặc một nhóm. `beforeAll` không có `page`           |
| Fixture            | Chuẩn bị trạng thái dùng chung cho toàn project. Test chỉ nhận fixture mà test khai báo              |
| Hook hay fixture   | Trạng thái dùng chung dùng fixture, bước chuẩn bị riêng của một nhóm dùng hook                       |
| Parameterized test | Vòng lặp đặt ngoài `test`, mỗi bộ dữ liệu là một test riêng, tên test chứa dữ liệu                   |

## 7. Bài tập về nhà

Không nhập password sai cho tài khoản chung.

### Bài 1: chuyển bộ test sang fixture

Xây dựng `fixtures/test-fixtures.ts` và chuyển toàn bộ file test sang sử dụng fixture, theo quy trình refactor 3 bước. Kết quả sau khi refactor:

* Mọi test cần đăng nhập nhận trạng thái đã đăng nhập từ fixture. Không file test nào tự gọi `login()`, trừ các test kiểm tra chức năng đăng nhập.
* Test kiểm tra chức năng đăng nhập bắt đầu ở trạng thái chưa đăng nhập.
* Page object sử dụng trong test được cung cấp qua fixture, file test không còn dòng `new ...Page(page)`.
* Không còn `beforeEach` nào chỉ dùng để đăng nhập.
* Toàn bộ test vẫn pass.

### Bài 2: tổ chức bộ test theo tính năng

* Mỗi file test nhóm các test theo tính năng bằng `test.describe`. File API test trong `tests/api/` nhóm theo endpoint.
* Tên nhóm và tên test theo quy ước đặt tên của khóa học: tiếng Anh, viết hoa chữ cái đầu.
* Không còn bước chuẩn bị nào lặp lại ở đầu nhiều test trong cùng một file, kể cả các file API test.
* Đọc report hoặc kết quả `--list` biết được mỗi test thuộc tính năng nào.
* Các tag đã gắn trước đó vẫn chọn đúng nhóm test khi chạy `--grep`.

### Bài 3: parameterized test cho đăng nhập

Refactor test đăng nhập với nhiều bộ dữ liệu trong project để mỗi bộ dữ liệu là một test riêng. Kết quả:

* `--list` hiển thị một test cho mỗi bộ dữ liệu, tên test cho biết tình huống được kiểm tra.
* Một bộ dữ liệu fail không làm các bộ dữ liệu khác dừng.
* Chạy riêng được một bộ dữ liệu bằng `--grep`.
