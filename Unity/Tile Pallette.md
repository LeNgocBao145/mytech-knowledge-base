Trong Unity, **Tile Palette** (Bảng ô vẽ) là một cửa sổ công cụ đóng vai trò như **"khay chứa cọ vẽ"**, nơi lưu trữ toàn bộ các ô gạch, tường, sàn hoặc vật thể (Tiles, Rule Tiles, Animated Tiles) để bạn tô vẽ trực tiếp lên bản đồ (**Tilemap**) trên cọ Scene.

### 1. Bản chất và Cách vận hành

Nếu hình dung **Tilemap** ngoài Scene là bức tranh canvas trắng, thì **Tile Palette** chính là khay đựng các miếng ghép Sprite 2D:

- **Không chứa GameObject riêng lẻ:** Tile Palette lưu trữ dữ liệu dạng Tile Asset nhẹ.
    
- **Tương tác theo dạng lưới (Grid):** Các ô trong Palette được chia theo các ô vuông bằng đúng kích thước grid (ví dụ $1 \times 1$ unit tương ứng với ảnh 16x16px).
    
- **Kết hợp cùng Tilemap Component:** Palette chọn cọ, còn Tilemap GameObject ngoài Scene nhận nét vẽ.
    

### 2. Các loại dữ liệu đưa được vào Tile Palette

1. **Default Tile:** Các bức ảnh Sprite $16 \times 16$ tĩnh thông thường (ghế, bàn, cây cối, gạch lót).
    
2. **Rule Tile (Tile tự động):** Các file logic tự động ghép viền (như file `Floor_RuleTile` bạn vừa cấu hình thành công). Khi chọn Rule Tile trên Palette và quét cọ, Unity tự tính toán các góc viền ở mép và lòng sàn ở giữa.
    
3. **Animated Tile:** Tile chạy animation lặp lại (như dòng nước chảy, ngọn lửa đốm, thảm cỏ đung đưa).
    
4. **Random Tile:** Tile tự động chọn ngẫu nhiên các biến thể hình ảnh khác nhau để tạo độ tự nhiên cho mặt đất.
    

### 3. Phím tắt công cụ vẽ trên Tile Palette

Khi mở bảng Tile Palette (`Window` $\rightarrow$ `2D` $\rightarrow$ `Tile Palette`), bạn sử dụng các phím tắt sau để thao tác vẽ nhanh trên Scene:

- **`S` (Select):** Chọn 1 ô tile trong Palette hoặc trên Scene.
    
- **`M` (Move):** Di chuyển các ô tile bên trong bảng Palette.
    
- **`B` (Paintbrush):** Cây cọ cơ bản — click hoặc giữ chuột để vẽ từng ô một.
    
- **`U` (Box Fill):** Kéo thả để tô một vùng hình chữ nhật lớn (cực kỳ nhanh khi lót sàn phòng rộng).
    
- **`G` (Flood Fill):** Cây xô sơn — tô phủ toàn bộ vùng trống liền kề.
    
- **`D` (Eraser):** Cục tẩy — xóa ô tile đã vẽ khỏi Scene.
    
- **`I` (Picker):** Cây hút mẫu — click vào 1 ô trên Scene để lấy nhanh mẫu tile đó lên cọ vẽ.
    

### 4. Tại sao bắt buộc dùng Tile Palette trong Game 2D?

- **Tối ưu hiệu năng kịch trần (Batching & Draw Calls):** Nếu bạn kéo 1.000 ô sàn 16x16 ra Scene dạng 1.000 GameObject riêng biệt, game sẽ bị giật lag nghiêm trọng do máy tính phải xử lý 1.000 lệnh vẽ (`Draw Calls`). Khi vẽ qua Tile Palette trên 1 Tilemap, Unity gộp toàn bộ 1.000 ô đó thành **1 Draw Call duy nhất**.
    
- **Tốc độ làm Game (Level Design):** Dùng Tile Palette giúp bạn "tô" một bản đồ game rộng lớn chỉ trong vài phút thay vì tốn hàng giờ đồng hồ căn chỉnh tọa độ từng ô vuông.