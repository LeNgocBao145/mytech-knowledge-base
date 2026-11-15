**Shader** là một chương trình máy tính nhỏ được thực thi trực tiếp trên card màn hình (**GPU** - Graphics Processing Unit). Nhiệm vụ chính của Shader là tính toán và quyết định cách các đối tượng 3D hoặc 2D được hiển thị lên màn hình — bao gồm màu sắc, độ sáng, độ bóng, độ phản chiếu, độ trong suốt hay các hiệu ứng hình ảnh đặc biệt.

### Shader giải quyết vấn đề gì?

Trước khi có Shader, card đồ họa sử dụng **Fixed-Function Pipeline** (luồng dựng hình cố định), lập trình viên chỉ có thể bật/tắt các thiết lập ánh sáng có sẵn. Shader ra đời cho phép lập trình viên **mặt đối mặt điều khiển GPU** để tạo ra bất kỳ hiệu ứng thị giác nào mong muốn.

1. **Tính toán màu sắc và ánh sáng (Lighting & Shading):** Đột phá từ khối màu phẳng thành các bề mặt có độ nhám, kim loại, phản chiếu ánh sáng môi trường (PBR - Physically Based Rendering).
    
2. **Tạo chất liệu bề mặt (Materials):** Biến một lưới polyme phẳng thành bề mặt gỗ, nước, đá, dung nham hoặc tóc nhân vật.
    
3. **Biến dạng hình học (Geometry Deformation):** Làm cho cành cây lay chuyển theo gió, sóng biển nhấp nhô hay tạo hiệu ứng nhân vật hóa thành cát mà không cần thay đổi file 3D gốc.
    
4. **Hiệu ứng hậu kỳ (Post-Processing):** Xử lý toàn bộ khung hình sau khi vẽ xong (như hiệu ứng làm mờ Bloom, trắng đen, làm nhòe Motion Blur, hay ống kính camera cũ).
    

### Các loại Shader phổ biến trong Game Development

Trong quy trình dựng hình (Render Pipeline), Shader được chia thành nhiều loại đảm nhận các công đoạn khác nhau:

|**Loại Shader**|**Vị trí xử lý**|**Chức năng chính**|
|---|---|---|
|**Vertex Shader**|Tọa độ đỉnh (Vertices)|Biến đổi vị trí $X, Y, Z$ của các đỉnh 3D, xử lý biến dạng mô hình (sóng nước, cờ bay).|
|**Fragment / Pixel Shader**|Từng pixel trên màn hình|Tính toán màu sắc cuối cùng của từng pixel dựa trên ánh sáng, texture, bóng đổ.|
|**Compute Shader**|Dữ liệu chung trên GPU|Chạy các thuật toán song song không liên quan đến đồ họa (giả lập vật lý hạt, AI, Raytracing).|

### Shader trong Unity

Trong Unity, bạn tương tác với Shader qua 2 cách chính:

1. **Shader Graph (Visual Scripting):** Công cụ kéo-nối các node trực quan (phổ biến trong URP/HDRP), giúp thiết kế shader nhanh chóng mà không cần viết code.
    
2. **HLSL / Cg / ShaderLab (Code):** Viết mã mã hóa trực tiếp bằng ngôn ngữ HLSL để tùy biến các hiệu ứng phức tạp hoặc tối ưu sâu cho phần cứng.