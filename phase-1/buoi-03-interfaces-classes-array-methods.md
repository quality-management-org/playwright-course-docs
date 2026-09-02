# Buổi 3 · Interfaces, Classes, npm & Array methods

{% hint style="info" %}
**Sau buổi này bạn sẽ:** mô tả được dữ liệu bằng interface, viết được class (nền tảng của Page Object Model), và xử lý dữ liệu bằng map/filter/find. Cuối buổi có bài checkpoint tổng hợp toàn bộ Phase 1.
{% endhint %}

## 1. Interface — mô tả "hình dạng" của dữ liệu

Interface cho TypeScript biết một object **phải có những field nào, kiểu gì**. Thiếu field hay sai kiểu → báo đỏ ngay:

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  tag?: string; // dấu ? = optional, có cũng được không có cũng được
}

const backpack: Product = { id: 1, name: "Sauce Labs Backpack", price: 29.99 };
// const hat: Product = { id: 2 }; // LỖI: thiếu name và price
```

Với tester, interface cực hữu ích khi làm việc với test data và response API: bạn biết chính xác dữ liệu có gì, gõ sai tên field là biết liền thay vì đợi chạy mới phát hiện.

## 2. Type alias và Union type

```typescript
// Union type: giá trị chỉ được phép là 1 trong các lựa chọn
type PaymentMethod = "cash" | "card" | "momo";

const payment: PaymentMethod = "card"; // OK
// const wrong: PaymentMethod = "kard"; // LỖI — gõ sai là biết ngay
```

> **Quy ước của lớp:** mô tả object → dùng `interface`; kiểu dạng lựa chọn/union → dùng `type`. Đi làm bạn sẽ gặp cả hai — không cần tranh luận cái nào "đúng hơn".

## 3. Class — khuôn đúc ra object

Class là "khuôn" định nghĩa một lần, tạo ra nhiều object cùng cấu trúc:

```typescript
class UserAccount {
  public username: string;   // public: bên ngoài đọc được
  private password: string;  // private: chỉ dùng được BÊN TRONG class

  // constructor: hàm chạy khi tạo object mới bằng từ khóa new
  constructor(username: string, password: string) {
    this.username = username; // this = "chính object này"
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

{% hint style="info" %}
**Vì sao tester phải học class?** Ở Buổi 7, bạn sẽ học **Page Object Model** — cách tổ chức test chuẩn công nghiệp, trong đó mỗi trang web là 1 class và mỗi thao tác là 1 method. Học class hôm nay tức là bạn đã đi trước 50% quãng đường POM.
{% endhint %}

**Convention:** tên Class và Interface viết **PascalCase** (`UserAccount`, `Product`) — khác với camelCase của biến/hàm đã học Buổi 1.

## 4. Tách file với import/export

Project thật không viết hết vào 1 file. Cách tách:

```typescript
// file: utils.ts
export function formatPrice(price: number): string {
  return `${price} USD`;
}
```

```typescript
// file: main.ts
import { formatPrice } from "./utils";

console.log(formatPrice(29.99)); // "29.99 USD"
```

Những file/folder bạn sẽ thấy trong mọi project:

| Tên             | Vai trò                                                                                |
| --------------- | --------------------------------------------------------------------------------------- |
| `package.json`  | Danh sách thư viện project dùng + các lệnh tắt (scripts)                                 |
| `node_modules/` | Nơi chứa code của các thư viện đã cài. **Không bao giờ sửa tay, không commit lên Git**   |
| `tsconfig.json` | Cấu hình TypeScript — hiện tại chỉ cần biết nó tồn tại, không cần thuộc                  |

## 5. Ba array methods quan trọng nhất: map, filter, find

| Method   | Trả về                       | Dùng khi muốn                              |
| -------- | ---------------------------- | ------------------------------------------- |
| `filter` | Mảng mới                     | **Lọc** các phần tử thỏa điều kiện          |
| `map`    | Mảng mới (cùng độ dài)       | **Biến đổi** từng phần tử sang dạng khác    |
| `find`   | 1 phần tử (hoặc undefined)   | **Tìm** phần tử đầu tiên thỏa điều kiện     |

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

> Để ý: dữ liệu trên lấy từ [Saucedemo.com](https://www.saucedemo.com) — trang web bạn sẽ tự động hóa từ Buổi 4. Sau này đây chính là cách bạn xử lý test data: lọc user theo role, tìm sản phẩm theo tên, biến đổi response API trước khi kiểm tra.

## 6. Bài checkpoint cuối Phase 1 (làm tại lớp)

**Đề bài:** gọi API `https://jsonplaceholder.typicode.com/users`, lọc ra các user có email đuôi `.biz`, lấy danh sách tên của họ và log ra màn hình. Yêu cầu: có interface `User`, có try/catch.

Gợi ý các bước (tự làm trước khi nhìn gợi ý nhé):

1. Khai báo interface `User` với các field bạn cần dùng (id, name, email)
2. Viết hàm async, gọi API và lấy dữ liệu (nhớ await 2 lần!)
3. Dùng `filter` với điều kiện `u.email.endsWith(".biz")`
4. Dùng `map` để lấy ra mảng tên
5. Log số lượng và danh sách tên

Bài này gộp đủ kiến thức 3 buổi. **Tự làm được bài này = bạn đã sẵn sàng học Playwright.** Nếu còn vướng chỗ nào, nhắn trainer để được kèm thêm trước Buổi 4 — đừng ngại, đây là mục đích của bài checkpoint.

## 7. Bài tập về nhà (45–60 phút)

1. Viết class `Product` có: constructor, ít nhất 1 field private, và method `getDisplayPrice()` trả về chuỗi dạng `"29.99 USD"`
2. Hoàn thiện bài checkpoint nếu chưa xong tại lớp
3. Push tất cả lên repo lớp qua Pull Request, nhánh `ten-cua-ban/buoi3`
