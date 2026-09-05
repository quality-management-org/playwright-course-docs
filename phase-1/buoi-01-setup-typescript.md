---
description: "Cài đặt Node.js, VS Code và Git, chạy file TypeScript đầu tiên, làm quen với kiểu dữ liệu, function và destructuring."
icon: laptop-code
---

# Buổi 1 · Setup môi trường & TypeScript cơ bản

{% hint style="info" %}
**Sau buổi này bạn sẽ:** chạy được file TypeScript trên máy của mình, viết được hàm có khai báo kiểu dữ liệu, và sử dụng được destructuring, cú pháp xuất hiện trong mọi test Playwright.
{% endhint %}

## 1. Chuẩn bị trước buổi học

Cài đầy đủ các công cụ sau trước khi bắt đầu:

| Công cụ | Phiên bản tối thiểu           | Link tải                                                 | Cách kiểm tra    |
| ------- | ----------------------------- | -------------------------------------------------------- | ---------------- |
| Node.js | 22.x trở lên (nên dùng LTS)   | [nodejs.org/en/download](https://nodejs.org/en/download) | `node -v`        |
| npm     | 10.x trở lên (đi kèm Node.js) | Cài cùng Node.js                                         | `npm -v`         |
| Git     | Mới nhất                      | [git-scm.com/downloads](https://git-scm.com/downloads)   | `git --version`  |
| VS Code | Mới nhất                      | [code.visualstudio.com](https://code.visualstudio.com)   | Mở được ứng dụng |

{% hint style="success" %}
**Chọn bản Node.js:** ở trang tải, chọn bản LTS (Long Term Support) vì đây là bản ổn định nhất. npm được cài tự động kèm Node.js nên không cần cài riêng. Git chỉ cần cài sẵn trong buổi này, cách dùng Git được trình bày ở Buổi 2.
{% endhint %}

**Extensions cho VS Code:** mở VS Code, nhấn `Ctrl+Shift+X` (macOS: `Cmd+Shift+X`), tìm và cài:

1. **ESLint**: phát hiện lỗi và code chưa đúng chuẩn.
2. **Prettier**: tự động format code.
3. **Playwright Test for VSCode**: dùng từ Buổi 4.

## 2. Tạo project TypeScript đầu tiên

Mở terminal trong VS Code (menu Terminal > New Terminal) và chạy lần lượt các lệnh sau:

```bash
mkdir lesson-01
cd lesson-01
npm init -y
npm install typescript tsx --save-dev
```

| Lệnh                                    | Ý nghĩa                                                                                                                                                       |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mkdir lesson-01` / `cd lesson-01`      | Tạo folder mới và di chuyển vào folder đó                                                                                                                     |
| `npm init -y`                           | Tạo file `package.json`, nơi ghi tên project và danh sách thư viện đã cài                                                                                     |
| `npm install typescript tsx --save-dev` | Cài 2 thư viện: `typescript` (bộ kiểm tra kiểu dữ liệu) và `tsx` (công cụ chạy trực tiếp file .ts). Cờ `--save-dev` đánh dấu thư viện chỉ dùng khi phát triển |

Tạo file `hello.ts` với nội dung:

```typescript
const courseName: string = "Automation Testing từ Zero đến Hero";
const sessionNumber: number = 1;

console.log(`Chào mừng đến với ${courseName}, buổi ${sessionNumber}`);
```

Chạy file:

```bash
npx tsx hello.ts
```

Kết quả mong đợi trên terminal:

```
Chào mừng đến với Automation Testing từ Zero đến Hero, buổi 1
```

{% hint style="info" %}
`npx` chạy một công cụ đã cài trong project mà không cần cài toàn cục. Nếu terminal in ra đúng dòng trên, bạn đã chạy thành công chương trình TypeScript đầu tiên.
{% endhint %}

## 3. TypeScript là gì

TypeScript là JavaScript có bổ sung hệ thống kiểu dữ liệu (type). Khi một biến được khai báo là `string`, TypeScript báo lỗi ngay nếu biến đó bị gán một số, trước khi chương trình chạy.

Thử trực tiếp: trong `hello.ts`, gán `sessionNumber = "hai"`. VS Code gạch đỏ dòng này ngay lập tức. Với tester, đây là nguyên tắc quen thuộc: phát hiện lỗi càng sớm, chi phí sửa càng thấp. Playwright chọn TypeScript làm ngôn ngữ mặc định cũng vì lý do này.

## 4. Biến và kiểu dữ liệu

```typescript
// const: không gán lại được, dùng cho hầu hết trường hợp
const appUrl: string = "https://www.saucedemo.com";

// let: có thể gán lại, chỉ dùng khi giá trị thực sự thay đổi
let retryCount: number = 0;
retryCount = 1; // OK

const isLoggedIn: boolean = false;
```

{% hint style="warning" %}
**Tránh dùng kiểu `any`.** Khai báo `any` tương đương với tắt kiểm tra kiểu cho biến đó, và bạn mất toàn bộ lợi ích của TypeScript. Chỉ dùng khi không còn cách nào khác.
{% endhint %}

## 5. Function và Arrow function

Có 2 cách viết hàm. Cả hai đều khai báo kiểu cho tham số và giá trị trả về:

```typescript
// Cách 1: function truyền thống
function add(a: number, b: number): number {
  return a + b;
}

// Cách 2: arrow function, cú pháp được dùng trong mọi test Playwright
const multiply = (a: number, b: number): number => {
  return a * b;
};

console.log(add(2, 3));      // 5
console.log(multiply(2, 3)); // 6
```

Hai cách gần như tương đương ở giai đoạn này. Điểm cần ghi nhớ là ký hiệu `=>`, vì mọi test Playwright đều được viết bằng arrow function.

## 6. Array và Object

```typescript
// Array: danh sách các giá trị cùng kiểu
const browsers: string[] = ["chromium", "firefox", "webkit"];
browsers.push("edge");        // thêm phần tử vào cuối
console.log(browsers.length); // 4

// Duyệt qua từng phần tử
for (const browser of browsers) {
  console.log(`Đang test trên: ${browser}`);
}

// Object: nhóm nhiều thông tin liên quan vào một chỗ
const user = {
  username: "standard_user",
  password: "secret_sauce",
  role: "customer",
};
console.log(user.username);
```

## 7. Destructuring

Destructuring là cú pháp lấy nhiều field ra khỏi object và gán vào các biến riêng trong một câu lệnh:

```typescript
const user = {
  username: "standard_user",
  password: "secret_sauce",
};

// Cách cũ: lấy từng field một
const usernameOld = user.username;
const passwordOld = user.password;

// Destructuring: lấy nhiều field cùng lúc, tên biến trùng với tên field
const { username, password } = user;
console.log(username); // "standard_user"
```

{% hint style="info" %}
**Liên hệ với Playwright:** ở Buổi 4, bạn sẽ viết dòng `test("login", async ({ page }) => {...})`. Phần `{ page }` chính là destructuring: Playwright truyền vào một object chứa nhiều công cụ, và test chỉ lấy ra `page`. Từ khóa `async` được giải thích ở Buổi 2.
{% endhint %}

## 8. Quy tắc đặt tên (Coding convention)

| Loại      | Quy tắc                   | Đúng                             | Sai                            |
| --------- | ------------------------- | -------------------------------- | ------------------------------ |
| Biến, hàm | camelCase                 | `getUserData`                    | `GetUserData`, `get_user_data` |
| Tên biến  | Có ý nghĩa, tự giải thích | `loginButton`, `productPrice`    | `data`, `temp`, `a`, `b`       |
| Comment   | Giải thích lý do          | `// retry 3 lần vì API hay chậm` | `// gán x bằng 3`              |

Code đặt tên tốt là code người khác đọc hiểu mà không cần giải thích thêm. Trainer sẽ review tiêu chí này trong mọi bài nộp.

## 9. Lỗi thường gặp

| Triệu chứng                                    | Nguyên nhân                              | Cách xử lý                                                                           |
| ---------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------ |
| Gõ `node -v` báo không tìm thấy lệnh           | Terminal được mở trước khi cài Node.js   | Đóng và mở lại terminal, hoặc khởi động lại VS Code                                  |
| `npx tsx hello.ts` báo lỗi không tìm thấy file | Terminal đang đứng sai folder            | Kiểm tra bằng `pwd` (macOS) hoặc `cd` (Windows), di chuyển vào đúng folder chứa file |
| VS Code không gạch đỏ khi sai kiểu             | File chưa lưu hoặc chưa cài đủ extension | Lưu file (`Ctrl+S`), kiểm tra lại mục 1                                              |

## 10. Bài tập về nhà (45-60 phút)

Viết các hàm sau vào file `homework-lesson-01.ts` trong folder `lesson-01`, đặt tên đúng convention:

1. `isValidPassword(password: string): boolean`: trả về `true` khi password có từ 8 ký tự trở lên.
2. `getDiscountPrice(price: number, percent: number): number`: tính giá sau khi giảm.
3. `formatUserInfo(user: { name: string; age: number }): string`: trả về chuỗi dạng `"Linh (25 tuổi)"`, bắt buộc dùng destructuring trong thân hàm.
4. (Không bắt buộc) Hàm nhận một mảng số và trả về số lớn nhất.

Chạy thử từng hàm bằng `console.log` để kiểm tra kết quả. Bài này được nộp cùng bài tập Buổi 2 qua Pull Request trên nhánh `your-name/lesson-2`, sau khi bạn học Git ở Buổi 2. Giữ nguyên folder `lesson-01` trên máy để nộp sau.

**Checklist trước khi nộp:** file chạy được bằng `npx tsx homework-lesson-01.ts` · mọi tham số và giá trị trả về đều có khai báo kiểu · không dùng `any` · tên hàm và biến đúng camelCase · hàm số 3 có dùng destructuring.
