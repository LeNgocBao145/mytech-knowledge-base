**Canvas** trong Unity là một đối tượng (GameObject) chứa thành phần **Canvas [[Component]]**, đóng vai trò là "khung bạt" hoặc vùng không gian hiển thị cho tất cả các phần tử giao diện người dùng (UI - User Interface) như hình ảnh, văn bản, nút bấm, thanh máu, bảng menu,...

Trong Unity, **mọi phần tử UI bắt buộc phải là con (child) của một Canvas**. Nếu bạn tạo một UI Element (như `Button` hay `Text`) mà chưa có Canvas, Unity sẽ tự động tạo một Canvas và đưa phần tử đó vào bên trong.

### Canvas giải quyết vấn đề gì?

Trước khi có hệ thống **UGUI (Unity UI)** với Canvas làm trung tâm, việc xây dựng giao diện trong game gặp rất nhiều khó khăn. Canvas ra đời để giải quyết các bài toán lớn sau:

1. **Đồng bộ giao diện trên mọi độ phân giải màn hình (Responsive Design):**
    
    - **Vấn đề:** Màn hình điện thoại, PC, máy chơi game có vô số tỉ lệ ($16:9$, $18:9$, $4:3$) và độ phân giải ($HD$, $Full HD$, $4K$) khác nhau. Nếu vẽ UI cố định, giao diện sẽ bị lệch, tràn mép hoặc co rúm trên thiết bị khác.
        
    - **Giải pháp:** Canvas kết hợp với **Canvas Scaler** và hệ thống **Anchors / Pivots** giúp UI tự động căn chỉnh vị trí và co giãn kích thước linh hoạt theo độ phân giải thiết bị.
        
2. **Tách biệt không gian hiển thị (Screen vs. World Space):**
    
    - **Vấn đề:** Giao diện có 2 dạng chính: dạng đè cố định lên màn hình (như điểm số, nút bấm) và dạng nằm trực tiếp trong thế giới game (như thanh máu trên đầu quái vật, màn hình máy tính trong game 3D).
        
    - **Giải pháp:** Canvas cung cấp các **Render Mode** khác nhau để xử lý cả UI 2D cố định lẫn UI 3D trong không gian game một cách tự nhiên.
        
3. **Tối ưu hóa hiệu năng dựng hình (Draw Calls & Batching):**
    
    - **Vấn đề:** Nếu mỗi văn bản hay hình ảnh UI gửi một lệnh vẽ (Draw Call) riêng lên GPU, game sẽ bị giật lag nghiêm trọng khi có nhiều UI.
        
    - **Giải pháp:** Canvas tự động gom các UI phần tử có cùng Material/Texture lại thành một Mesh duy nhất (UI Batching) để gửi lên GPU trong 1 Draw Call duy nhất.
        

### Chi tiết 3 Chế độ Render (Render Modes) của Canvas

Trong [[component]] Canvas, **Render Mode** xác định cách UI được vẽ ra màn hình:

#### 1. Screen Space - Overlay (Mặc định)

- **Cơ chế:** Canvas được vẽ đè lên toàn bộ khung hình, phía trên tất cả mọi đối tượng 3D/2D và Camera trong game.
    
- **Đặc điểm:** Nếu màn hình thay đổi kích thước, Canvas tự động resize theo. Camera di chuyển hay xoay không ảnh hưởng tới Canvas này.
    
- **Ứng dụng:** Menu chính (Main Menu), HUD (thanh máu, điểm số, minimap), các bảng thông báo Popup.
    

#### 2. Screen Space - Camera

- **Cơ chế:** Canvas được đặt ở một khoảng cách cố định trước một Camera cụ thể.
    
- **Đặc điểm:** Bảng UI sẽ chịu ảnh hưởng bởi cài đặt của Camera đó (như Perspective/Orthographic, FOV, hiệu ứng Post-Processing). Nếu một đối tượng 3D nằm gần Camera hơn khoảng cách của Canvas, đối tượng đó sẽ che mất UI.
    
- **Ứng dụng:** UI có hiệu ứng 3D nhẹ, hiệu ứng chuyển cảnh cần tương tác với Camera, hoặc game có nhân vật/vật phẩm 3D hiển thị trên giao diện.
    

#### 3. World Space

- **Cơ chế:** Canvas hoạt động như một đối tượng 3D bình thường trong không gian game (có tọa độ $X, Y, Z$, có thể xoay, thu phóng).
    
- **Đặc điểm:** Kích thước UI phụ thuộc vào khoảng cách tới Camera. Nhân vật hoặc địa hình có thể đi đằng trước hoặc đằng sau Canvas này.
    
- **Ứng dụng:** Thanh máu/tên hiển thị trên đầu NPC, bảng thông số vật phẩm rơi trên mặt đất, bảng điều khiển VR, màn hình máy tính trong môi trường game 3D.
    

### Các [[Component]] quan trọng đi kèm Canvas

Để Canvas hoạt động hoàn chỉnh, Unity thường đi kèm các [[component]] phụ trợ:

- **Canvas Scaler:** Điều khiển tỉ lệ co giãn của UI theo các chế độ như _Constant Pixel Size_ (giữ nguyên kích thước pixel), _Scale With Screen Size_ (tự động co giãn theo độ phân giải tham chiếu - khuyên dùng), hoặc _Constant Physical Size_.
    
- **Graphic Raycaster:** Phát hiện các thao tác chạm/click chuột trên UI. Bất kỳ nút bấm hay phần tử UI nào muốn nhận tương tác chuột đều phải nằm trong Canvas có `Graphic Raycaster`.
    
- **EventSystem:** Quản lý các sự kiện đầu vào (mouse, touch, gamepad) và truyền tin tới các phần tử UI trong Canvas.