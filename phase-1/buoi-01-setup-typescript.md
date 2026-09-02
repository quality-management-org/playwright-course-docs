# Buổi 1 · Setup môi trường & TypeScript cơ bản

{% hint style="info" %}
**Sau buổi này bạn sẽ:** chạy được file TypeScript trên máy mình, viết được hàm có kiểu dữ liệu, và hiểu destructuring — cú pháp mà mọi test Playwright đều dùng.
{% endhint %}

## 1. Chuẩn bị trước buổi học (Prerequisites)

Trước khi bắt đầu, hãy cài đầy đủ các công cụ sau trên máy:

| Công cụ | Phiên bản tối thiểu           | Link tải                                                 | Cách kiểm tra    |
| ------- | ----------------------------- | -------------------------------------------------------- | ---------------- |
| Node.js | ≥ 22.x (nên dùng bản LTS)     | [nodejs.org/en/download](https://nodejs.org/en/download) | `node -v`        |
| npm     | ≥ 10.x (có sẵn theo Node.js)  | Cài cùng Node.js                                         | `npm -v`         |
| Git     | Mới nhất                      | [git-scm.com/downloads](https://git-scm.com/downloads)   | `git --version`  |
| VS Code | Mới nhất                      | [code.visualstudio.com](https://code.visualstudio.com)   | Mở được ứng dụng |

{% hint style="success" %}
**Tip cho Node.js:** ở trang tải, chọn bản **LTS (Long Term Support)** — đây là bản ổn định nhất. npm được cài tự động kèm Node.js nên bạn không cần cài riêng. Git buổi này chỉ cần cài sẵn — Buổi 2 chúng ta mới học cách dùng.
{% endhint %}

**Extensions cho VS Code** — mở VS Code, nhấn `Ctrl+Shift+X` (macOS: `Cmd+Shift+X`), tìm và cài:

1. **ESLint** — tự động phát hiện lỗi và code chưa chuẩn
2. **Prettier** — tự động format code cho đẹp
3. **Playwright Test for VSCode** — sẽ dùng từ Buổi 4

## 2. Tạo project TypeScript đầu tiên

Mở terminal trong VS Code (menu Terminal → New Terminal) và chạy lần lượt:

```bash
mkdir buoi-01
cd buoi-01
npm init -y
npm install typescript tsx --save-dev
```

Giải thích từng lệnh:

| Lệnh                                     | Ý nghĩa                                                                                                                                                    |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mkdir buoi-01` / `cd buoi-01`           | Tạo folder mới và di chuyển vào folder đó                                                                                                                   |
| `npm init -y`                            | Tạo file `package.json` — "giấy khai sinh" của project, ghi lại tên project và các thư viện đã cài                                                           |
| `npm install typescript tsx --save-dev`  | Cài 2 thư viện: `typescript` (bộ kiểm tra kiểu dữ liệu) và `tsx` (công cụ chạy trực tiếp file .ts). Cờ `--save-dev` nghĩa là chỉ dùng khi phát triển         |

Tạo file `hello.ts` với nội dung:

```typescript
const courseName: string = "Automation Testing từ Zero đến Hero";
const sessionNumber: number = 1;

console.log(`Chào mừng đến với ${courseName} — buổi ${sessionNumber}`);
```

Chạy file:

```bash
npx tsx hello.ts
```

> `npx` = chạy một công cụ đã cài trong project. Nếu terminal in ra dòng chào mừng — chúc mừng, bạn đã chạy thành công chương trình TypeScript đầu tiên! 🎉

## 3. TypeScript là gì và vì sao tester nên dùng?

TypeScript = JavaScript + **kiểu dữ liệu (type)**. Khi bạn khai báo một biến là `string`, TypeScript sẽ báo lỗi ngay nếu bạn gán nhầm số vào — **trước cả khi chạy chương trình**.

Thử ngay: trong `hello.ts`, gán `sessionNumber = "hai"` — VS Code sẽ gạch đỏ ngay lập tức. Với tester, đây chính là triết lý quen thuộc: **bắt bug càng sớm, sửa càng rẻ**. Playwright chọn TypeScript làm ngôn ngữ mặc định cũng vì lý do này.

## 4. Biến và kiểu dữ liệu

```typescript
// const: không gán lại được — dùng cho hầu hết trường hợp
const appUrl: string = "https://www.saucedemo.com";

// let: có thể gán lại — chỉ dùng khi giá trị thực sự thay đổi
let retryCount: number = 0;
retryCount = 1; // OK

const isLoggedIn: boolean = false;
```

{% hint style="warning" %}
**Tránh dùng kiểu `any`.** Khai báo `any` tức là tắt kiểm tra kiểu — bạn mất toàn bộ lợi ích của TypeScript. Chỉ dùng khi thực sự bất đắc dĩ.
{% endhint %}

## 5. Function và Arrow function

Có 2 cách viết hàm — cả hai đều khai báo kiểu cho tham số và giá trị trả về:

```typescript
// Cách 1: function truyền thống
function add(a: number, b: number): number {
  return a + b;
}

// Cách 2: arrow function — cú pháp bạn sẽ gặp khắp nơi trong Playwright
const multiply = (a: number, b: number): number => {
  return a * b;
};

console.log(add(2, 3));      // 5
console.log(multiply(2, 3)); // 6
```

Hai cách gần như tương đương ở giai đoạn này. Điều quan trọng: **làm quen với dấu `=>`** vì mọi test Playwright đều viết bằng arrow function.

## 6. Array và Object

```typescript
// Array — danh sách các giá trị cùng kiểu
const browsers: string[] = ["chromium", "firefox", "webkit"];
browsers.push("edge");        // thêm phần tử vào cuối
console.log(browsers.length); // 4

// Duyệt qua từng phần tử
for (const browser of browsers) {
  console.log(`Đang test trên: ${browser}`);
}

// Object — nhóm nhiều thông tin liên quan vào 1 chỗ
const user = {
  username: "standard_user",
  password: "secret_sauce",
  role: "customer",
};
console.log(user.username);
```

## 7. Destructuring — cú pháp quan trọng nhất buổi hôm nay

Destructuring là cách "bóc" nhiều field ra khỏi object trong 1 dòng:

```typescript
const user = {
  username: "standard_user",
  password: "secret_sauce",
};

// Cách cũ: lấy từng field một
const usernameOld = user.username;
const passwordOld = user.password;

// Destructuring: lấy nhiều field cùng lúc — ngắn gọn hơn hẳn
const { username, password } = user;
console.log(username); // "standard_user"
```

{% hint style="info" %}
**Vì sao phải học kỹ phần này?** Đây là một dòng test Playwright thật mà bạn sẽ viết ở Buổi 4: `test('login', async ({ page }) => {...})`. Phần `{ page }` chính là destructuring bạn vừa học — Playwright đưa cho bạn một object chứa nhiều công cụ, và bạn "bóc" ra đúng cái cần dùng là `page`. Còn `async` là gì — Buổi 2 sẽ giải đáp.
{% endhint %}

## 8. Quy tắc đặt tên (Coding convention)

| Loại      | Quy tắc                    | Đúng                            | Sai                              |
| --------- | -------------------------- | ------------------------------- | -------------------------------- |
| Biến, hàm | camelCase                  | `getUserData`                   | `GetUserData`, `get_user_data`   |
| Tên biến  | Có ý nghĩa, tự giải thích  | `loginButton`, `productPrice`   | `data`, `temp`, `a`, `b`         |
| Comment   | Giải thích "tại sao"       | // retry 3 lần vì API hay chậm  | // gán x bằng 3                  |

Code đặt tên tốt là code người khác (và chính bạn 2 tuần sau) đọc hiểu ngay không cần hỏi. Trainer sẽ review điểm này trong mọi bài nộp.

## 9. Lỗi thường gặp

| Triệu chứng                                     | Nguyên nhân                              | Cách xử lý                                                                       |
| ----------------------------------------------- | ---------------------------------------- | -------------------------------------------------------------------------------- |
| Gõ `node -v` báo không tìm thấy lệnh            | Terminal mở trước khi cài Node           | Đóng mở lại terminal hoặc khởi động lại VS Code                                   |
| `npx tsx hello.ts` báo lỗi không tìm thấy file  | Terminal đang đứng sai folder            | Kiểm tra bằng `pwd` (macOS) / `cd` (Windows), di chuyển vào đúng folder chứa file |
| VS Code không gạch đỏ khi sai kiểu              | File chưa lưu hoặc chưa cài đủ extension | Lưu file (`Ctrl+S`), kiểm tra lại mục 1                                           |

## 10. Bài tập về nhà (45–60 phút)

Viết các hàm sau vào file `baitap-buoi1.ts`, đặt tên đúng convention:

1. `isValidPassword(password: string): boolean` — trả về true khi password dài từ 8 ký tự
2. `getDiscountPrice(price: number, percent: number): number` — tính giá sau giảm
3. `formatUserInfo(user: { name: string; age: number }): string` — trả về chuỗi dạng "Linh (25 tuổi)", **bắt buộc dùng destructuring** trong thân hàm
4. (Không bắt buộc) hàm nhận mảng số, trả về số lớn nhất

Chạy thử từng hàm bằng `console.log` để kiểm tra kết quả. **Lưu file lại trong folder `buoi-01`** — Buổi 2 học Git xong bạn sẽ push bài này lên GitHub.
