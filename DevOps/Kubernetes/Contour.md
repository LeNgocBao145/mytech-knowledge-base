**Project Contour** là một **Ingress Controller** mã nguồn mở dành cho Kubernetes, sử dụng **Envoy** làm reverse proxy và load balancer để quản lý cũng như điều hướng lưu lượng truy cập (L7) từ bên ngoài vào trong cluster.

Contour thuộc quản lý của tổ chức CNCF (Cloud Native Computing Foundation), được thiết kế nhằm giải quyết các hạn chế của Ingress mặc định trong Kubernetes.

![[Pasted image 20261014190607.png]]
### Kiến trúc hai phần chính của Contour

- **Control Plane (Contour):** Theo dõi Kubernetes API Server để lắng nghe các thay đổi về cấu hình Ingress/HTTPProxy, sau đó chuyển đổi cấu hình này thành định dạng hiểu được và truyền sang Envoy qua giao thức xDS gRPC API.
    
- **Data Plane (Envoy):** Đảm nhận nhiệm vụ xử lý thực tế lưu lượng truy cập mạng (High-performance Proxy), điều hướng dữ liệu trực tiếp tới các Pod backend.
    

### Các tính năng nổi bật của Contour

1. **Custom Resource Definition (`HTTPProxy`):**
    
    - Thay thế cho `Ingress` resource truyền thống vốn khó mở rộng.
        
    - Hỗ trợ chia quyền quản lý cấu hình theo từng Namespace (Multi-tenant) một cách an toàn.
        
    - Cho phép cấu hình nâng cao như **Weight-based Load Balancing** (chia tỉ lệ traffic cho Canary deployment), **Header Rewrite**, **Rate Limiting**, **Circuit Breaking**, và **Health Checking**.
        
2. **Dynamic Updates (Không cần Reload):**
    
    - Các Ingress Controller cũ (như NGINX) thường phải reload lại tiến trình khi cấu hình thay đổi, có thể làm gián đoạn kết nối ngắn hạn. Contour cập nhật cấu hình trực tiếp vào Envoy qua gRPC API theo thời gian thực mà không làm ngắt kết nối.
        
3. **Tích hợp sẵn TLS/SSL & Cert-Manager:**
    
    - Dễ dàng tự động hóa việc cấp phát và gia hạn chứng chỉ HTTPS.
        

### Ví dụ cấu hình `HTTPProxy` của Contour

```
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: my-app-proxy
  namespace: default
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
    - conditions:
        - prefix: /
      services:
        - name: service-v1
          port: 80
          weight: 90 # 90% traffic vào v1
        - name: service-v2
          port: 80
          weight: 10 # 10% traffic vào v2 (Canary release)
```