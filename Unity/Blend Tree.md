**Blend Tree** trong Unity là một dạng State đặc biệt trong Animator Controller, cho phép **pha trộn (blend)** hai hoặc nhiều file Animation Clip lại với nhau một cách mượt mà dựa trên các tham số số thực (Float Parameters).

Thay vì chuyển đổi ngắt đoạn từ animation này sang animation khác, Blend Tree tính toán trọng số (weight) của từng clip để tạo ra một chuyển động trung gian tự nhiên.

### Sự khác biệt giữa State Transition và Blend Tree

|**Tiêu chí**|**State Transition (Nối dây thông thường)**|**Blend Tree**|
|---|---|---|
|**Cơ chế**|Chuyển đổi từ State A sang State B khi thỏa mãn điều kiện (True/False).|Hòa trộn các clip **cùng một lúc** theo tỷ lệ phần trăm dựa trên biến số.|
|**Trải nghiệm**|Animation chạy hết clip này rồi nhảy sang clip khác.|Chuyển động biến thiên mượt mà theo vận tốc/hướng di chuyển.|
|**Số lượng Clip**|Mỗi State chỉ chứa 1 Animation Clip.|1 Blend Tree Node chứa được hàng chục Animation Clip.|
|**Ứng dụng**|Bấm phím để Bắn, Nhảy, Bị thương, Chết...|Đi bộ $\rightarrow$ Chạy nhanh theo tốc độ, Xoay 8 hướng theo cần Analog.|

### Các loại Blend Tree phổ biến

1. **1D Blend Tree (1 Tham số):**
    
    - Sử dụng 1 biến float duy nhất (ví dụ: `Speed`).
        
    - _Ví dụ:_ Khi `Speed = 0` $\rightarrow$ Chạy clip `Idle`. Khi `Speed = 2` $\rightarrow$ Trộn $50\%$ `Walk` + $50\%$ `Run`. Khi `Speed = 5` $\rightarrow$ Chạy $100\%$ `Run`.
        
2. **2D Blend Tree (2 Tham số):**
    
    - Sử dụng 2 biến float song song (ví dụ: `MoveX` và `MoveY` hoặc `LastMoveX` và `LastMoveY`).
        
    - Đây chính là giải pháp tối ưu cho **game 2D Top-Down / Isometric 8 hướng**:
        
        - **2D Simple Directional:** Dùng cho các hướng phân biệt rõ ràng (Up, Down, Left, Right, 4 góc chéo).
            
        - **2D Freeform Directional / Cartesian:** Dùng khi các hướng có thể vừa di chuyển vừa kết hợp với gốc tọa độ $(0, 0)$.
            
3. **Direct Blend Tree:**
    
    - Cho phép điều khiển trực tiếp trọng số của từng clip riêng biệt thông qua các tham số tương ứng. Thường dùng cho biểu cảm khuôn mặt (mắt nhắm/mở, miệng cười/méo) hoặc cử chỉ bàn tay.
        

### Lợi ích lớn nhất trong thiết kế Game

- **Giữ Animator Controller sạch vẽ:** Gom tất cả 8 hướng `Idle` (hoặc 8 hướng `Run`) vào đúng **1 ô State duy nhất** trên sơ đồ Animator, giúp Animator gọn gàng và không bị rối dây.
    
- **Tối ưu code C#:** Bạn không cần viết hàng loạt câu lệnh `if-else` phức tạp để bật/tắt từng clip. Trong C#, bạn chỉ cần truyền giá trị hướng di chuyển vào Animator:

```
// Chỉ cần 2 dòng code này để điều khiển toàn bộ Blend Tree 8 hướng
animator.SetFloat("MoveX", inputDirection.x);
animator.SetFloat("MoveY", inputDirection.y);
```

**Blend Tree** ra đời để giải quyết 2 vấn đề lớn nhất trong việc làm hoạt ảnh game: **Pha trộn chuyển động mượt mà theo tham số** và **Giữ cho sơ đồ Animator Controller gọn gàng**.

### 1. Blend Tree giải quyết cụ thể vấn đề gì?

- **Pha trộn (Interpolate) các Animation cùng loại:** Thay vì chỉ bật/tắt một clip cố định, Blend Tree tự động tính toán tỷ lệ % để hòa trộn nhiều clip cùng lúc dựa vào các biến số (`Float Parameters`).
    
    - _Ví dụ Tốc độ:_ Chuyển mượt từ `Đứng yên` $\rightarrow$ `Đi bộ` $\rightarrow$ `Chạy nhanh` khi bạn đẩy nhẹ hoặc đẩy hết hành trình cần Analog.
        
    - _Ví dụ Hướng (2D Top-Down/8 hướng):_ Tự động chọn đúng clip `Look_Up`, `Look_Down`, `Look_Left`, `Look_Right` hoặc các góc chéo dựa vào tọa độ `Vector2(X, Y)`.
        
- **Rút gọn sơ đồ Animator State Machine:** Blend Tree nén toàn bộ các clip thuộc cùng một bộ hành động (ví dụ: 8 clip Idle của 8 hướng) vào **đúng 1 State Node duy nhất**.
    
- **Đơn giản hóa Logic trong C#:** Bạn không cần viết code để chọn và phát từng tên file clip. Trong C#, bạn chỉ cần truyền dữ liệu đầu vào (ví dụ: vận tốc hay hướng di chuyển) cho Animator, Blend Tree sẽ tự lo phần còn lại:
    
    C#
    
    ```
    animator.SetFloat("MoveX", input.x);
    animator.SetFloat("MoveY", input.y);
    ```
    

### 2. Nếu KHÔNG sử dụng Blend Tree thì sao?

Nếu không dùng Blend Tree, bạn vẫn có thể làm game, nhưng sẽ phải đối mặt với các hạn chế và phiền phức sau:

#### ① Sơ đồ Animator biến thành "mạng nhện"

- Để làm nhân vật 8 hướng (Idle + Run), bạn phải tạo **16 State lẻ** (`Idle_Up`, `Idle_Down`, `Run_Up`, `Run_Down`...) trên Animator Canvas.
    
- Bạn phải chăng hàng chục đường dây nối (`Transitions`) chéo qua lại giữa các State này kèm hàng loạt điều kiện so sánh (`MoveX > 0.1`, `MoveY < -0.1`...), khiến Animator cực kỳ rối mắt, khó quản lý và rất dễ sót lỗi logic.
    

#### ② Chuyển động bị giật cục, thiếu tự nhiên

- Không có cơ chế pha trộn trọng số, nhân vật chỉ có thể phát clip A hoặc clip B. Khi chuyển từ đi bộ sang chạy, hình ảnh sẽ bị nhảy phắt sang clip mới ngay lập tức chứ không có sự tăng tốc dần dần.
    

#### ③ Code C# phình to và tốn công bảo trì

- Bạn sẽ phải tự viết hàng loạt câu lệnh `if-else` hoặc `switch-case` trong C# để kiểm tra hướng/tốc độ, sau đó gọi tên clip bằng chuỗi dạng `animator.Play("Knight_Run_UpRight")`.
    
- Mỗi lần đổi tên clip animation hoặc thêm hướng mới trong Unity, bạn bắt buộc phải quay lại sửa code C#.
    

|**Tiêu chí**|**Dùng Blend Tree**|**KHÔNG dùng Blend Tree**|
|---|---|---|
|**Số State trong Animator**|1 State duy nhất cho mỗi bộ hành động|Hàng chục State lẻ cho từng góc/tốc độ|
|**Đường nối Transition**|Không cần dây nối giữa các hướng|Hàng chục dây nối chéo phức tạp|
|**Độ mượt chuyển động**|Tự động hòa trộn mượt mà theo tham số|Chuyển clip đột ngột, dễ bị giật frame|
|**Bảo trì Code C#**|Rất ít code (chỉ cần truyền `SetFloat`)|Phải viết `if-else` check hướng và phát clip bằng tên|