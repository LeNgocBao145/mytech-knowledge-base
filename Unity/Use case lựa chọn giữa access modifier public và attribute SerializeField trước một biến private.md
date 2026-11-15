
### 1. `public float speed;` (Public Variable)

Khi bạn khai báo một biến `public`:

1. **Trạng thái trên Inspector:** Unity sẽ tự động serialize biến này và hiển thị nó trên bảng Inspector để bạn chỉnh sửa.
    
2. **Quyền truy cập trong Code:** **Bất kỳ script nào khác** trong dự án cũng có thể đọc và trực tiếp sửa đổi giá trị của biến này (`player.speed = 100;`).
    

#### Khi nào nên dùng `public`?

- Khi thuộc tính đó **cần được truy cập hoặc chỉnh sửa từ các script khác**.
    
- _Ví dụ:_ `public int maxHealth;` trên script `PlayerHealth` để script `HealItem` hoặc `UIController` có thể đọc và cập nhật giá trị.
    

### 2. `[SerializeField] private float speed;` (Serialized Private Variable)

Khi bạn thêm attribute `[SerializeField]` trước một biến `private`:

1. **Trạng thái trên Inspector:** Unity sẽ serialize biến này và hiển thị nó trên Inspector để bạn tinh chỉnh thông số y hệt như biến `public`.
    
2. **Quyền truy cập trong Code:** Biến vẫn giữ tính chất **private** — các script bên ngoài **không thể** đọc hay sửa đổi trực tiếp giá trị của nó.
    

#### Khi nào nên dùng `[SerializeField] private`?

- Khi thuộc tính đó là **dữ liệu nội bộ của riêng class**, bạn muốn chỉnh sửa thông số trên Inspector nhưng **không muốn các script khác can thiệp/sửa đổi làm lỗi logic**.
    
- Đây là **quy chuẩn viết code (Best Practice)** được khuyến nghị hàng đầu trong Unity.
    
- _Ví dụ:_ `[SerializeField] private RigidBody2D rb;` hoặc `[SerializeField] private float moveSpeed;` trên script `PlayerController`.
    

### Bảng so sánh nhanh

|**Tiêu chí**|**public**|**[SerializeField] private**|
|---|---|---|
|**Hiển thị trên Inspector**|Có|Có|
|**Class khác đọc/sửa được**|Có (Dễ gây mất kiểm soát dữ liệu)|Không (Bảo vệ dữ liệu an toàn)|
|**Tuân thủ nguyên lý OOP**|Tùy trường hợp|Rất tốt (Giữ tính đóng gói)|
|**Mục đích sử dụng**|Giao tiếp giữa các Script|Tinh chỉnh thông số nội bộ từ Inspector|

### Lời khuyên thiết kế Code chuẩn trong Unity

1. **Mặc định dùng `[SerializeField] private`:** Cho tất cả các thuộc tính chỉ dùng nội bộ trong script đó (từ tốc độ, thời gian hồi chiêu, đến các reference Prefab, UI [[Component]],...).
    
2. **Chỉ dùng `public` khi thực sự cần:** Nếu một script khác cần đọc giá trị của biến private, hãy dùng **Property (Getter)** thay vì biến public:
    

C#

```
[SerializeField] private int currentHealth = 100;

// Các script khác có thể ĐỌC nhưng KHÔNG THỂ SỬA trực tiếp currentHealth
public int CurrentHealth => currentHealth; 
```