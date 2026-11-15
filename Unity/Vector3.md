Trong Unity, `Vector3` là một struct đại diện cho **véc-tơ 3 chiều** (chứa $x, y, z$). Nó được sử dụng rộng rãi để biểu diễn vị trí (Position), hướng (Direction), vận tốc (Velocity), lực (Force) hoặc tỷ lệ (Scale).

Dưới đây là tổng hợp chi tiết các thuộc tính, hàm tĩnh (static methods), hàm thực thể (instance methods) cùng tham số phổ biến nhất của `Vector3`.

## 1. Các thuộc tính tĩnh dựng sẵn (Static Properties)

Cung cấp các véc-tơ hướng/vị trí chuẩn nhanh chóng mà không cần khởi tạo mới.

- **`Vector3.zero`**: $(0, 0, 0)$
    
- **`Vector3.one`**: $(1, 1, 1)$
    
- **`Vector3.up`**: $(0, 1, 0)$ – Hướng lên trên.
    
- **`Vector3.down`**: $(0, -1, 0)$ – Hướng xuống dưới.
    
- **`Vector3.left`**: $(-1, 0, 0)$ – Hướng sang trái.
    
- **`Vector3.right`**: $(1, 0, 0)$ – Hướng sang phải.
    
- **`Vector3.forward`**: $(0, 0, 1)$ – Hướng tiến về phía trước (trục Z 3D).
    
- **`Vector3.back`**: $(0, 0, -1)$ – Hướng lùi về phía sau.
    
- **`Vector3.positiveInfinity`**: $(\infty, \infty, \infty)$
    
- **`Vector3.negativeInfinity`**: $(-\infty, -\infty, -\infty)$
    

## 2. Các thuộc tính thực thể (Instance Properties)

- **`x`, `y`, `z`** (`float`): Giá trị tọa độ trên từng trục.
    
- **`magnitude`** (`float`): Độ dài (độ lớn) thực tế của véc-tơ, tính bằng công thức $\sqrt{x^2 + y^2 + z^2}$.
    
- **`sqrMagnitude`** (`float`): Bình phương độ dài ($x^2 + y^2 + z^2$).
    
    - _Ứng dụng:_ Dùng để so sánh khoảng cách giữa 2 object nhanh hơn `magnitude` hoặc `Vector3.Distance` vì không phải tính căn bậc hai (tiết kiệm hiệu năng).
        
- **`normalized`** (`Vector3`): Trả về véc-tơ đơn vị (có cùng hướng nhưng độ dài = $1$).
    

## 3. Các hàm toán học & Thao tác Véc-tơ (Static Methods)

### A. Nội suy & Di chuyển

- **`Vector3.Lerp(Vector3 a, Vector3 b, float t)`**
    
    - **Giải thích:** Nội suy tuyến tính (Linear Interpolation) từ véc-tơ `a` đến `b` theo tỷ lệ `t` (kẹp trong khoảng $[0, 1]$).
        
    - **Tham số:** `a` (điểm bắt đầu), `b` (điểm kết thúc), `t` (tỷ lệ $0 \le t \le 1$).
        
- **`Vector3.LerpUnclamped(Vector3 a, Vector3 b, float t)`**
    
    - Tương tự `Lerp` nhưng không giới hạn `t` trong $[0, 1]$ (có thể bứt phá ra ngoài khoảng $a \rightarrow b$).
        
- **`Vector3.Slerp(Vector3 a, Vector3 b, float t)`**
    
    - **Giải thích:** Nội suy theo đường cong hình cầu (Spherical Linear Interpolation). Thường dùng cho hướng quay hoặc di chuyển theo hình vòng cung.
        
- **`Vector3.MoveTowards(Vector3 current, Vector3 target, float maxDistanceDelta)`**
    
    - **Giải thích:** Di chuyển vị trí `current` về phía `target` với tốc độ tối đa không quá `maxDistanceDelta` trong frame đó.
        
    - **Tham số:** `current` (vị trí hiện tại), `target` (vị trí đích), `maxDistanceDelta` (khoảng cách tối đa di chuyển).
        
- **`Vector3.SmoothDamp(Vector3 current, Vector3 target, ref Vector3 currentVelocity, float smoothTime, float maxSpeed = Mathf.Infinity, float deltaTime = Time.deltaTime)`**
    
    - **Giải thích:** Di chuyển mượt mà về đích giống như có lực lò xo/giảm xóc (thường dùng làm Camera đi theo người chơi).
        
    - **Tham số:**
        
        - `current`: Vị trí hiện tại.
            
        - `target`: Vị trí đích.
            
        - `currentVelocity`: Biến tham chiếu (`ref`) lưu vận tốc hiện tại (tự động cập nhật qua mỗi frame).
            
        - `smoothTime`: Thời gian xấp xỉ để đạt tới đích.
            

### B. Phép toán Đại số Véc-tơ

- **`Vector3.Dot(Vector3 lhs, Vector3 rhs)`**
    
    - **Giải thích:** Tích vô hướng (Dot Product).
        
    - Trả về $>0$ nếu 2 véc-tơ cùng hướng, $=0$ nếu vuông góc, $<0$ nếu ngược hướng. Thường dùng kiểm tra kẻ địch có nằm trong tầm mắt hay không.
        
- **`Vector3.Cross(Vector3 lhs, Vector3 rhs)`**
    
    - **Giải thích:** Tích có hướng (Cross Product). Trả về một véc-tơ **vuông góc với cả 2 véc-tơ đầu vào**.
        
- **`Vector3.Distance(Vector3 a, Vector3 b)`**
    
    - **Giải thích:** Tính khoảng cách thực tế giữa 2 điểm `a` và `b` (tương đương `(a - b).magnitude`).
        
- **`Vector3.Angle(Vector3 from, Vector3 to)`**
    
    - **Giải thích:** Trả về góc bé nhất (tính bằng Độ - Degree) giữa 2 véc-tơ.
        
- **`Vector3.SignedAngle(Vector3 from, Vector3 to, Vector3 axis)`**
    
    - **Giải thích:** Trả về góc giữa 2 véc-tơ có dấu âm/dương dựa theo trục xoay `axis`.
        
- **`Vector3.ClampMagnitude(Vector3 vector, float maxLength)`**
    
    - **Giải thích:** Giới hạn độ dài của `vector` không vượt quá `maxLength`. Thường dùng để giới hạn tốc độ di chuyển nhân vật.
        

### C. Phản xạ & Chiếu (Projection)

- **`Vector3.Reflect(Vector3 inDirection, Vector3 inNormal)`**
    
    - **Giải thích:** Tính véc-tơ phản xạ từ hướng `inDirection` đập vào mặt phẳng có pháp tuyến `inNormal`. Dùng làm đạn nảy, tia laser phản xạ.
        
- **`Vector3.Project(Vector3 vector, Vector3 onNormal)`**
    
    - **Giải thích:** Chiếu một véc-tơ lên một véc-tơ hướng khác.
        
- **`Vector3.ProjectOnPlane(Vector3 vector, Vector3 planeNormal)`**
    
    - **Giải thích:** Chiếu một véc-tơ lên một mặt phẳng được xác định bởi pháp tuyến `planeNormal`. Thường dùng để tính hướng di chuyển song song với mặt đất dốc.
        

## 4. Các hàm thực thể (Instance Methods)

- **`Normalize()`**: Chuẩn hóa véc-tơ hiện tại về độ dài = 1 (thay đổi trực tiếp chính véc-tơ đó, khác với `normalized` tạo ra bản sao mới).
    
- **`Set(float newX, float newY, float newZ)`**: Gán lại giá trị 3 trục $X, Y, Z$ cho véc-tơ hiện tại.
    
- **`ToString(string format)`**: Chuyển véc-tơ thành chuỗi ký tự hiển thị (ví dụ: `ToString("F2")` lấy 2 chữ số thập phân).
    

## 5. Các toán tử (Operators)

`Vector3` hỗ trợ các phép toán tiêu chuẩn:

- `+`, `-`: Cộng/Trừ từng tọa độ tương ứng của 2 véc-tơ.
    
- `*`, `/`: Nhân/Chia véc-tơ với một số `float` (kéo dài hoặc thu ngắn độ dài).
    
- `==`, `!=`: So sánh 2 véc-tơ có bằng nhau (hoặc xấp xỉ) hay không.