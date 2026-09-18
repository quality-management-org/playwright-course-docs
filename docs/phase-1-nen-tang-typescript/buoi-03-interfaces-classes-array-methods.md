---
description: "Mô tả dữ liệu bằng interface, viết class với constructor và private field, tách file bằng import/export, xử lý mảng bằng map, filter, find."
icon: cubes
---

# Buổi 3 · Interfaces, Classes, npm & Array methods

{% hint style="info" %}
**Sau buổi này bạn sẽ:** mô tả được cấu trúc dữ liệu bằng interface, viết được class (nền tảng của Page Object Model ở Buổi 7), và xử lý dữ liệu bằng map, filter, find. Cuối buổi có bài checkpoint tổng hợp toàn bộ Phase 1.
{% endhint %}

## 1. Interface: mô tả cấu trúc của dữ liệu

Interface khai báo một object phải có những field nào và mỗi field thuộc kiểu gì. Object thiếu field hoặc sai kiểu bị TypeScript báo lỗi ngay khi viết code:

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  tag?: string; // dấu ? đánh dấu field optional, có thể bỏ trống
}

const backpack: Product = { id: 1, name: "Sauce Labs Backpack", price: 29.99 };
// const hat: Product = { id: 2 }; // LỖI: thiếu name và price
```

Với tester, interface đặc biệt hữu ích khi làm việc với test data và response của API: cấu trúc dữ liệu được ghi rõ trong code, và lỗi gõ sai tên field được phát hiện ngay trong VS Code thay vì khi chạy.

## 2. Type alias và Union type

```typescript
// Union type: giá trị chỉ được phép là một trong các lựa chọn
type PaymentMethod = "cash" | "card" | "momo";

const payment: PaymentMethod = "card"; // OK
// const wrong: PaymentMethod = "kard"; // LỖI: giá trị không nằm trong danh sách cho phép
```

{% hint style="info" %}
**Quy ước của lớp:** dùng `interface` để mô tả object, dùng `type` cho union type và các kiểu dạng lựa chọn. Cả hai cách đều xuất hiện trong các project thực tế.
{% endhint %}

## 3. Class

Class là bản thiết kế được định nghĩa một lần, dùng để tạo ra nhiều object có cùng cấu trúc và hành vi:

```typescript
class UserAccount {
  public username: string;   // public: đọc được từ bên ngoài class
  private password: string;  // private: chỉ dùng được bên trong class

  // constructor: hàm chạy khi tạo object mới bằng từ khóa new
  constructor(username: string, password: string) {
    this.username = username; // this trỏ đến chính object đang được tạo
    this.password = password;
  }

  checkPassword(input: string): boolean {
    return input === this.password;
  }
}

const account = new UserAccount("standard_user", "secret_sauce");
console.log(account.username);            // OK
console.log(account.checkPassword("x"));  // false
// console.log(account.password);         // LỖI: password là private
```

<!-- TODO(image): VS Code với dòng console.log(account.password) đã bỏ comment và bị gạch đỏ, tooltip hiện Property 'password' is private and only accessible within class 'UserAccount'. -->

{% hint style="info" %}
**Liên hệ với Playwright:** ở Buổi 7, bạn học Page Object Model, cách tổ chức test trong đó mỗi trang web là một class và mỗi thao tác người dùng là một method. Kiến thức về class ở buổi này là nền tảng trực tiếp cho POM.
{% endhint %}

**Convention:** tên class và interface viết PascalCase (`UserAccount`, `Product`), khác với camelCase dùng cho biến và hàm đã học ở Buổi 1.

## 4. Tách file với import/export

Project thực tế không viết toàn bộ code trong một file. Mở project `playwright-course` trong repo lớp bằng VS Code (File > Open Folder), tạo folder `lesson-03` bằng Explorer (chuột phải > New Folder), rồi tạo 2 file sau trong folder đó:

```typescript
// file: lesson-03/utils.ts
export function formatPrice(price: number): string {
  return `${price} USD`;
}
```

```typescript
// file: lesson-03/main.ts
import { formatPrice } from "./utils";

console.log(formatPrice(29.99)); // "29.99 USD"
```

Chạy bằng `npx tsx lesson-03/main.ts`. Đường dẫn `./utils` trong `import` là đường dẫn tương đối tính từ file đang viết, nên hai file phải nằm cùng folder.

Cấu trúc project `playwright-course` đến thời điểm này:

```
playwright-course/
├── node_modules/
├── lesson-01/
├── lesson-02/
├── lesson-03/
│   ├── main.ts
│   └── utils.ts
├── .gitignore
├── package.json
├── package-lock.json
└── tsconfig.json
```

Vai trò của các file ở gốc project, đã tạo ở Buổi 1 và gặp lại trong mọi project Node.js:

| Tên                 | Vai trò                                                                                                   |
| ------------------- | --------------------------------------------------------------------------------------------------------- |
| `package.json`      | Danh sách thư viện project sử dụng và các lệnh tắt (scripts)                                              |
| `package-lock.json` | Phiên bản chính xác của từng thư viện, npm tự sinh. Không sửa tay, có commit lên Git                       |
| `node_modules/`     | Nơi chứa code của các thư viện đã cài. Không sửa tay, không commit lên Git                                 |
| `.gitignore`        | Danh sách file và folder Git bỏ qua, hiện chỉ có `node_modules/`                                           |
| `tsconfig.json`     | Cấu hình TypeScript. Buổi 13 trình bày thêm các tùy chọn nâng cao                                          |

## 5. Ba array methods quan trọng nhất: map, filter, find

| Method   | Trả về                     | Dùng khi muốn                            |
| -------- | -------------------------- | ---------------------------------------- |
| `filter` | Mảng mới                   | Lọc các phần tử thỏa điều kiện           |
| `map`    | Mảng mới (cùng độ dài)     | Biến đổi từng phần tử sang dạng khác     |
| `find`   | 1 phần tử (hoặc undefined) | Tìm phần tử đầu tiên thỏa điều kiện      |

Ví dụ sử dụng interface `Product` đã khai báo ở mục 1:

```typescript
const products: Product[] = [
  { id: 1, name: "Sauce Labs Backpack", price: 29.99 },
  { id: 2, name: "Sauce Labs Bike Light", price: 9.99 },
  { id: 3, name: "Sauce Labs Bolt T-Shirt", price: 15.99 },
  { id: 4, name: "Sauce Labs Fleece Jacket", price: 49.99 },
];

const cheapProducts = products.filter((p) => p.price < 20);         // 2 sản phẩm
const productNames = products.map((p) => p.name);                   // mảng 4 tên
const backpack = products.find((p) => p.name.includes("Backpack")); // 1 sản phẩm
```

{% hint style="info" %}
Dữ liệu trong ví dụ lấy từ [Saucedemo](https://www.saucedemo.com), trang web bạn sẽ tự động hóa từ Buổi 4. Đây cũng là cách xử lý test data về sau: lọc user theo role, tìm sản phẩm theo tên, biến đổi response API trước khi kiểm tra.
{% endhint %}

## 6. Bài checkpoint cuối Phase 1 (làm tại lớp)

**Đề bài:** tạo file `lesson-03/checkpoint-phase-1.ts`, gọi API `https://jsonplaceholder.typicode.com/users`, lọc ra các user có email kết thúc bằng `.biz`, lấy danh sách tên của họ và log ra màn hình. Yêu cầu: có interface `User`, có try/catch.

Các bước gợi ý (nên tự làm trước khi đọc phần này):

1. Khai báo interface `User` với các field cần dùng (`id`, `name`, `email`).
2. Viết hàm async gọi API và lấy dữ liệu, với `await` ở cả `fetch` và `response.json()` (đã học ở Buổi 2).
3. Dùng `filter` với điều kiện `u.email.endsWith(".biz")`.
4. Dùng `map` để lấy ra mảng tên.
5. Log số lượng và danh sách tên.

Bài này tổng hợp kiến thức của cả 3 buổi. Tự hoàn thành được bài này nghĩa là bạn đã sẵn sàng học Playwright ở Buổi 4. Nếu còn vướng ở bước nào, liên hệ trainer để được hỗ trợ thêm trước Buổi 4; đây là mục đích của bài checkpoint.

## 7. Lỗi thường gặp

| Triệu chứng                                                        | Nguyên nhân                                              | Cách xử lý                                                              |
| ------------------------------------------------------------------ | -------------------------------------------------------- | ----------------------------------------------------------------------- |
| Lỗi "Object literal may only specify known properties"             | Gán field không có trong interface, thường do gõ sai tên | So sánh tên field với khai báo interface                                |
| Lỗi "Property 'password' is private and only accessible within..." | Truy cập field private từ bên ngoài class                | Thêm method public trong class để đọc hoặc kiểm tra giá trị đó          |
| Lỗi "Cannot find module './utils'"                                 | Sai đường dẫn import hoặc thiếu `./`                     | Kiểm tra file tồn tại, import file cùng folder phải bắt đầu bằng `./`   |
| `find` trả về `undefined`, gọi `.name` báo lỗi                     | Không có phần tử nào thỏa điều kiện                      | Kiểm tra kết quả trước khi dùng: `if (backpack) { ... }`                |

## 8. Bài tập về nhà (45-60 phút)

1. Tạo file `lesson-03/product.ts`, viết class `Product` có constructor, ít nhất 1 field private, và method `getDisplayPrice()` trả về chuỗi dạng `"29.99 USD"`.
2. Hoàn thiện bài checkpoint ở mục 6 trong file `lesson-03/checkpoint-phase-1.ts` nếu chưa xong tại lớp.
3. Push project `playwright-course` lên repo lớp qua Pull Request theo quy trình ở Buổi 2, nhánh `your-name/lesson-3`.

**Checklist trước khi nộp:**

* [ ] Code chạy không lỗi bằng `npx tsx`
* [ ] Class và interface đặt tên PascalCase
* [ ] Class `Product` có ít nhất 1 field private và method `getDisplayPrice()`
* [ ] Bài checkpoint có interface `User` và try/catch
* [ ] Đúng quy ước tên nhánh `your-name/lesson-3`
