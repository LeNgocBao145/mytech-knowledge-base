**Sprite Atlas** trong Unity là một loại Asset dùng để **gom/gói (pack) nhiều hình ảnh Sprite lẻ lại thành một bức ảnh lớn duy nhất** (gọi là tấm Atlas) ở background khi game chạy.

### Sprite Atlas giải quyết vấn đề gì?

#### 1. Giảm số lượng Draw Call (Vấn đề lớn nhất về hiệu năng)

- **Khi KHÔNG có Sprite Atlas:** Nếu màn hình game của bạn đang hiển thị 20 loại quái vật, 10 cây cối và 5 vật phẩm (mỗi cái là một file `.png` riêng), CPU sẽ phải gửi 35 lệnh vẽ riêng biệt (**35 Draw Calls**) sang GPU. Càng nhiều Draw Call, game càng bị giật/lag và sụt FPS (đặc biệt trên Mobile).
    
- **Khi CÓ Sprite Atlas:** Bạn kéo toàn bộ ảnh quái vật, cây cối, vật phẩm đó vào 1 Sprite Atlas. Unity sẽ gộp tất cả thành 1 bức ảnh lớn duy nhất. Kết quả: GPU chỉ cần **1 Draw Call** duy nhất để vẽ toàn bộ 35 đối tượng đó lên màn hình.
    

#### 2. Đơn giản hóa quy trình làm việc (Workflow)

- **Trước đây (Không dùng Atlas):** Bạn phải tự gộp các hình ảnh lẻ bằng phần mềm bên ngoài (như Photoshop, Aseprite hay TexturePacker) thành một file Sprite Sheet lớn trước khi đưa vào Unity. Mỗi lần muốn sửa 1 nhân vật hay thêm 1 trang bị, bạn phải xuất lại cả file Sprite Sheet đó.
    
- **Hiện tại (Dùng Sprite Atlas):** Bạn cứ lưu các file `.png` lẻ từng đồ vật, từng quái vật trong các thư mục dự án cho dễ quản lý. Unity sẽ tự động gộp chúng lại thành Atlas khi bạn Play game hoặc Build file game.
    

#### 3. Tối ưu hóa bộ nhớ GPU (VRAM)

- GPU xử lý cực kỳ tốt các file ảnh có kích thước theo **lũy thừa của 2** (ví dụ: $128 \times 128$, $512 \times 512$, $2048 \times 2048$).
    
- Nếu bạn có nhiều ảnh lẻ kích thước tự do (như $35 \times 40\text{px}$, $18 \times 22\text{px}$), GPU sẽ xử lý rất kém hiệu quả. Sprite Atlas sẽ tự động sắp xếp các ảnh nhỏ này vào một tấm khung chuẩn (ví dụ $1024 \times 1024\text{px}$) để GPU nạp và giải nén tối ưu nhất.
    

### Lưu ý quan trọng khi cấu hình Sprite Atlas cho Pixel Art:

Khi tạo Sprite Atlas cho game Pixel Art, bạn cần chú ý các thông số trong **Inspector** của file Atlas:

1. **Allow Rotation:** **Tắt đi (Uncheck)**. Nếu bật, Unity có thể xoay ngang/dọc các Sprite Pixel Art để nhồi nhét cho vừa khung, khiến đường nét pixel bị lệch trục.
    
2. **Tight Packing:** **Tắt đi (Uncheck)**. Tùy chọn này bó sát viền nét vẽ, dễ gây lỗi làm văng/mất nét đồ họa Pixel.
    
3. **Padding:** Đặt khoảng **2 đến 4 px**. Khoảng trống này ngăn hiện tượng "Bleeding" (lem màu của Sprite bên cạnh vào viền nhân vật khi camera di chuyển).
    
4. **Filter Mode:** Chọn **`Point (no filter)`** (để giữ nét sắc mịn cho Pixel Art).

## Filter Mode

**Filter Mode** của **Sprite Atlas** (hoặc Texture nói chung) quyết định cách engine xử lý và hiển thị các pixel khi hình ảnh được phóng to (upscale) hoặc thu nhỏ (downscale) so with kích thước gốc.

Khi bạn chọn **Filter Mode = Point (no filter)**, Unity sẽ sử dụng thuật toán **Nearest Neighbor** (láng giềng gần nhất).

### Cách Point Filter hoạt động

- **Không làm mịn (No Interpolation):** Khi phóng to sprite, Unity không hòa trộn (blend) màu giữa các pixel lân cận. Thay vào đó, nó chọn chính xác màu của pixel gần nhất trong file ảnh gốc.
    
- **Hình ảnh sắc nét, rõ viền:** Các cạnh của pixel giữ nguyên dạng hình vuông hoàn hảo, không bị mờ hay nhòe.
    

|**Chế độ (Filter Mode)**|**Đặc điểm hiển thị**|**Trải nghiệm thị giác**|
|---|---|---|
|**Point (no filter)**|Giữ nguyên từng pixel gốc, cạnh vuông vức|Sắc nét, dạng khối pixel (pixelated)|
|**Bilinear / Trilinear**|Nội suy và làm mịn màu giữa các pixel|Mượt mà, nhưng dễ bị mờ (blurry) khi phóng to|

### Khi nào nên dùng Point Filter?

1. **Game Pixel Art 2D:** Đây là thiết lập bắt buộc đối với đồ họa Pixel Art. Nếu chọn Bilinear, hình ảnh pixel sẽ bị mờ nhòe và mất đi độ chi tiết đặc trưng.
    
2. **UI hoặc Font chữ dạng Pixel:** Giúp các icon, font chữ vuông vức hiển thị rõ ràng, không bị lem viền trên các màn hình độ phân giải khác nhau.
    
3. **Texture dữ liệu (Data Textures):** Các texture chứa dữ liệu chính xác (như tilemap index, mask, heightmap) cần giá trị pixel nguyên bản mà không bị hòa trộn.
    

### Lưu ý quan trọng khi dùng trong Sprite Atlas

- **Padding giữa các Sprite:** Khi đóng gói (pack) các sprite vào Atlas với Point Filter, hãy đảm bảo thiết lập **Padding** (thường từ 2–4 px) để tránh hiện tượng dính viền màu (bleed) từ sprite bên cạnh khi camera di chuyển hoặc thay đổi tỉ lệ màn hình.
    
- **Compression (Nén ảnh):** Ngay cả khi chọn Point Filter, nếu bạn đặt **Compression** ở mức quá cao (như DXT5/ETC2 chất lượng thấp), hình ảnh vẫn có thể bị vỡ hoặc đổi màu do thuật toán nén. Nên kết hợp Point Filter với định dạng nén chất lượng cao hoặc Uncompressed nếu cần độ chính xác tuyệt đối.