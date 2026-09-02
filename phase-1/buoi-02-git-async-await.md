# Buổi 2 · Git cơ bản & Async/Await

{% hint style="info" %}
**Sau buổi này bạn sẽ:** hiểu và viết được async/await — trái tim của mọi test Playwright, gọi được API thật, và tạo Pull Request đầu tiên trên GitHub như một engineer thực thụ.
{% endhint %}

## 1. Vì sao cần Async/Await?

Hãy tưởng tượng bạn vào quán phở: bạn gọi món, rồi **ngồi lướt điện thoại trong lúc chờ** — chứ không đứng chặn ở bếp nhìn người ta nấu. Code cũng vậy: khi gọi API hay mở trang web, máy tính gửi yêu cầu đi rồi làm việc khác trong lúc chờ phản hồi. Đó gọi là **bất đồng bộ (asynchronous)**.

Vấn đề: nếu code dòng dưới cần kết quả của dòng trên (ví dụ cần dữ liệu API trả về để xử lý tiếp) thì phải có cách nói "**chờ món ra rồi hãy ăn**". Đó chính là `await`.

Hai từ khóa đi cặp với nhau:

* `async` đặt trước hàm — đánh dấu "hàm này có chỗ phải chờ"
* `await` đặt trước thao tác cần chờ — "đợi xong mới chạy dòng tiếp theo"

> Trong Playwright, **mọi thao tác với trang web đều phải chờ** (trang tải, nút hiện ra...) nên bạn sẽ gõ `await` nhiều đến mức thành phản xạ. Học chắc hôm nay = đỡ vất vả toàn bộ phần sau.

## 2. Gọi API thật với fetch()

```typescript
async function getUsers() {
  try {
    const response = await fetch("https://jsonplaceholder.typicode.com/users");
    const users = await response.json();

    console.log(`Lấy được ${users.length} user`);
    console.log(users[0].name);
  } catch (error) {
    console.error("Gọi API thất bại:", error);
  }
}

getUsers();
```

Giải thích từng dòng:

| Dòng code                            | Ý nghĩa                                                                                                              |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `async function getUsers()`          | Khai báo hàm có chứa thao tác cần chờ                                                                                 |
| `await fetch(...)`                   | Gọi API và **chờ** server phản hồi                                                                                    |
| `await response.json()`              | Chuyển phản hồi thành dữ liệu dùng được — bước này **cũng phải await**                                                |
| `try { ... } catch (error) { ... }`  | Nếu bất kỳ dòng nào trong `try` gặp lỗi (mất mạng, sai URL...), chương trình nhảy vào `catch` thay vì sập              |

{% hint style="warning" %}
**Lỗi kinh điển số 1 của người mới:** quên `await`. Nếu bạn log ra và thấy `Promise { <pending> }` thay vì dữ liệu — nghĩa là bạn "chưa chờ món ra đã đòi ăn". Quay lại kiểm tra xem thiếu `await` ở đâu (hay gặp nhất: quên ở `response.json()`).
{% endhint %}

**Promise là gì?** Là "phiếu hẹn lấy đồ" mà thao tác bất đồng bộ trả về ngay lập tức — cầm phiếu chưa phải có đồ. `await` = đứng chờ đến khi phiếu đổi được đồ thật. Trong khóa này chúng ta chỉ dùng async/await, không học cách viết `.then()` cũ hơn.

## 3. Git là gì?

Git giúp lưu lịch sử mọi thay đổi của code — giống Google Docs có "Version history", nhưng mạnh hơn nhiều. Các khái niệm cần nhớ:

| Thuật ngữ          | Hiểu đơn giản                                                          |
| ------------------ | ---------------------------------------------------------------------- |
| Repository (repo)  | "Kho" chứa toàn bộ code + lịch sử thay đổi                              |
| Commit             | Lưu một "mốc" thay đổi, kèm lời mô tả                                   |
| Branch (nhánh)     | Bản nháp riêng của bạn — sửa thoải mái không ảnh hưởng ai                |
| Push               | Đẩy code từ máy bạn lên GitHub                                          |
| Pull Request (PR)  | "Đơn xin gộp code" — nhờ người khác review trước khi đưa vào bản chính  |

## 4. Quy trình nộp bài của lớp

Lớp dùng chung 1 repo GitHub. Bạn làm theo đúng 5 bước sau mỗi lần nộp bài:

```bash
# Bước 1 — chỉ làm LẦN ĐẦU: tải repo về máy
git clone <link-repo-lớp>
cd <ten-repo>

# Bước 2: tạo nhánh riêng theo quy ước ten-cua-ban/buoi-X
git checkout -b linh/buoi2

# Bước 3: báo Git ghi nhận các file thay đổi
git add .

# Bước 4: lưu mốc kèm mô tả ngắn gọn
git commit -m "Buoi 2: bai tap async/await"

# Bước 5: đẩy nhánh lên GitHub
git push -u origin linh/buoi2
```

{% hint style="info" %}
**Vì sao không push thẳng vào `main`?** Nhánh `main` của lớp đã được bảo vệ — mọi thay đổi bắt buộc đi qua Pull Request và được review. Đây chính là cách 99% công ty vận hành: bạn đang tập quy trình thật ngay từ buổi thứ hai.
{% endhint %}

## 5. Tạo Pull Request trên GitHub

1. Mở repo lớp trên GitHub — sẽ thấy banner vàng gợi ý nhánh vừa push, bấm **Compare & pull request**
2. Đặt tiêu đề rõ ràng, ví dụ: `[Linh] Buoi 2 - Async/Await`
3. Bấm **Create pull request**
4. Chờ trainer review — nếu có comment, sửa code rồi push lại lên đúng nhánh đó (PR tự cập nhật)

## 6. Lỗi thường gặp

| Triệu chứng                                    | Nguyên nhân                             | Cách xử lý                                                          |
| ---------------------------------------------- | --------------------------------------- | -------------------------------------------------------------------- |
| Log ra `Promise { <pending> }`                 | Quên `await`                            | Thêm `await` trước fetch hoặc response.json()                        |
| Lỗi "await is only valid in async functions"   | Dùng `await` ngoài hàm async            | Thêm `async` vào khai báo hàm                                        |
| Push bị từ chối (rejected)                     | Đang đứng ở nhánh `main`                | Kiểm tra nhánh bằng `git branch`, tạo nhánh riêng rồi push lại       |
| Không thấy banner tạo PR                       | Banner chỉ hiện ít phút sau khi push    | Vào tab Pull requests → New pull request → chọn nhánh của bạn        |

## 7. Bài tập về nhà (45–60 phút)

1. Viết hàm `getProducts()` gọi API `https://fakestoreapi.com/products`: log ra **tổng số sản phẩm** và **tên sản phẩm đầu tiên**, có try/catch đầy đủ
2. Push cả bài Buổi 1 và bài này lên repo lớp qua Pull Request, nhánh đặt tên `ten-cua-ban/buoi2`

**Checklist trước khi nộp:** code chạy không lỗi · có try/catch · tên biến đúng convention · đúng quy ước tên nhánh · PR có tiêu đề rõ ràng.
