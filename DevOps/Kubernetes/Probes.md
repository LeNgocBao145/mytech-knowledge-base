**Probes** (Cơ chế kiểm tra sức khỏe) là cách K8s theo dõi trạng thái của các container bên trong Pod để tự động xử lý khi có lỗi. K8s cung cấp 3 loại Probe chính:

- **Liveness Probe:** Kiểm tra xem container còn sống hay không. Nếu kiểm tra thất bại, K8s sẽ **kill container và khởi động lại** (restart) nó theo chính sách `restartPolicy`.
    
    - _Mục đích:_ Xử lý các lỗi treo app, deadlock (app vẫn chạy nhưng không phản hồi).
        
- **Readiness Probe:** Kiểm tra xem container đã **sẵn sàng nhận traffic** chưa. Nếu thất bại, K8s sẽ **tắt đường truyền traffic** (gỡ Pod khỏi Endpoints/Service) tới Pod đó, nhưng **không kill Pod**.
    
    - _Mục đích:_ Đảm bảo người dùng không kết nối vào một Pod đang khởi động hoặc đang bị quá tải.
        
- **Startup Probe:** Kiểm tra xem ứng dụng đã khởi động hoàn tất chưa. Trong khi Startup Probe đang chạy, Liveness và Readiness Probe sẽ bị tạm dừng.
    
    - _Mục đích:_ Dành cho các ứng dụng cũ hoặc ứng dụng nặng tốn nhiều thời gian để boot up, tránh việc Liveness Probe kill nhầm app khi nó chưa kịp khởi động xong.
        

**Các phương thức kiểm tra:**

- `httpGet`: Gửi HTTP Request (ví dụ `/healthz`) đến container, phản hồi $200-399$ là thành công.
    
- `tcpSocket`: Kiểm tra xem một cổng TCP cụ thể có mở hay không.
    
- `exec`: Chạy một lệnh shell bên trong container, trả về mã $0$ là thành công.