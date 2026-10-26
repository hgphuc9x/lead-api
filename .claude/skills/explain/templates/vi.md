# Vietnamese Templates

## Giải thích Code

```markdown
## 📁 File: {filename}

### Mục đích
[1-2 câu mô tả file này làm gì]

### Phân tích

**Imports**
- Import X từ Y để [mục đích]

**Class/function chính**
- [Giải thích logic]
- [Dùng ví dụ đời thường nếu cần]

**Các methods**
- `methodX()`: [làm gì, khi nào được gọi]

### 💡 Điểm cần nhớ
- Điểm 1
- Điểm 2

### 🔗 Liên kết
- Được gọi bởi: ...
- Gọi tới: ...
```

---

## Giải thích Concept

```markdown
## 🎯 {Tên Concept}

### Là gì?
[Giải thích đơn giản, 2-3 câu]

### Ví dụ đời thường
[So sánh với thứ quen thuộc]

Ví dụ: Repository Pattern giống như **thủ thư** trong thư viện:
- Bạn không tự vào kho tìm sách
- Bạn nhờ thủ thư (Repository) tìm giúp
- Thủ thư biết sách ở đâu, cách tìm nhanh nhất

### Trong code

```typescript
// Ví dụ code ngắn
```

### Tại sao cần?
- Lý do 1
- Lý do 2

### Không dùng thì sao?
[Vấn đề sẽ gặp]

### 📚 Tìm hiểu thêm
- Từ khóa để search
```

---

## Giải thích Flow

```markdown
## 🔄 Flow: {Tên Action}

### Tổng quan
[1-2 câu mô tả flow này]

### Sơ đồ

```
[Client]
    │
    ▼
[Controller] ──── Nhận request, validate input
    │
    ▼
[Service] ──────── Xử lý logic nghiệp vụ
    │
    ▼
[Repository] ───── Truy vấn database
    │
    ▼
[Database]
    │
    ▼
[Response] ◄────── Transform & trả về
```

### Chi tiết từng bước

**Bước 1: Client gửi request**
- Endpoint: POST /products
- Body: { name, price, ... }

**Bước 2: Controller nhận**
- Validate DTO
- Lấy user từ JWT (nếu cần)
- Gọi Service

**Bước 3: Service xử lý**
- Logic nghiệp vụ
- Gọi Repository

**Bước 4: Repository truy vấn**
- Tạo query
- Thực thi

**Bước 5: Response**
- Transform data
- Trả về client

### 🔍 Mẹo debug
- Nếu lỗi ở bước X, kiểm tra...
```

---

## Giải thích Why

```markdown
## ❓ Tại sao: {Câu hỏi}

### Trả lời ngắn
[1-2 câu, trả lời trực tiếp]

### Giải thích chi tiết

**Lý do 1: ...**
[Giải thích]

**Lý do 2: ...**
[Giải thích]

### Không làm vậy thì sao?
[Vấn đề sẽ gặp]

### Đánh đổi
| Ưu điểm | Nhược điểm |
|---------|------------|
| ... | ... |

### Có cách khác không?
[Các lựa chọn khác và khi nào dùng]

### 📌 Kết luận
[Tóm tắt khuyến nghị]
```