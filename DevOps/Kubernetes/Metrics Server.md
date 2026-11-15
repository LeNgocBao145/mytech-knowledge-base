**Metrics Server** trong Kubernetes là một dịch vụ mở rộng (add-on) thu thập các thông số tài nguyên như **CPU** và **RAM** từ toàn bộ Pod và Node trong cluster.

Nó đóng vai trò là "mắt thần" theo dõi lượng tài nguyên đang được tiêu thụ theo thời gian thực và cung cấp dữ liệu này cho Kubernetes API Server.

### 1. Metrics Server dùng để làm gì?

- **Tự động co giãn (Autoscaling):** Đây là nhiệm vụ quan trọng nhất. Các thành phần như **HPA** (Horizontal Pod Autoscaler - tự động tăng/giảm số lượng Pod) và **VPA** (Vertical Pod Autoscaler - tự động điều chỉnh CPU/RAM của Pod) phụ thuộc hoàn toàn vào Metrics Server để đưa ra quyết định co giãn.
    
- **Giám sát nhanh qua CLI:** Cho phép quản trị viên kiểm tra mức độ sử dụng tài nguyên trực tiếp bằng lệnh:
    
    - `kubectl top nodes`: Xem CPU/RAM tiêu thụ của từng máy chủ.
        
    - `kubectl top pods`: Xem CPU/RAM tiêu thụ của từng Pod.
        

### 2. Nguyên lý hoạt động

1. Trên mỗi Node trong cluster có một agent gọi là **kubelet**, bên trong kubelet tích hợp sẵn bộ thu thập dữ liệu **cAdvisor**.
    
2. **Metrics Server** định kỳ truy vấn (scrape) dữ liệu CPU/RAM từ kubelet trên từng Node.
    
3. Metrics Server lưu dữ liệu này vào **bộ nhớ tạm (in-memory)** và cung cấp lại qua **Metrics API** (`metrics.k8s.io`).
    

### 3. Những lưu ý quan trọng (Những gì Metrics Server KHÔNG làm)

- **Không phải hệ thống Monitoring/Alerting hoàn chỉnh:** Metrics Server **không lưu trữ dữ liệu lịch sử** (chỉ giữ dữ liệu thời gian thực trong RAM) và không có giao diện đồ họa (UI) hay cơ chế gửi cảnh báo. Nếu cần xem biểu đồ lịch sử hoặc cảnh báo qua Slack/Email, bạn cần dùng giải pháp như **Prometheus + Grafana**.
    
- **Chỉ thu thập CPU và RAM:** Metrics Server không thu thập các chỉ số nâng cao như lưu lượng mạng (Network I/O), đọc/ghi đĩa (Disk I/O) hay các chỉ số tùy chỉnh của ứng dụng (Custom Metrics).