**Ingress Controller** trong Kubernetes là một ứng dụng (Reverse Proxy) chạy bên trong cluster, có nhiệm vụ tiếp nhận các truy cập từ bên ngoài (Internet/Intranet) và điều hướng truy cập đó đến đúng Service/Pod bên trong theo các quy tắc được định nghĩa.

Để dễ hình dung:

- **Ingress (Resource):** Là "bản thiết kế" hay tập hợp các quy tắc định tuyến (Routing Rules) do bạn khai báo bằng YAML (ví dụ: truy cập `api.example.com` thì vào Service A, `app.example.com` thì vào Service B).
    
- **Ingress Controller:** Là "người thực thi" (như NGINX, Traefik, HAProxy, Envoy). Nó đọc các quy tắc từ Ingress Resource và trực tiếp xử lý, điều hướng lưu lượng truy cập thực tế.
![[Pasted image 20261014190439.png]]
### Chức năng chính của Ingress Controller

1. **Định tuyến L7 (HTTP/HTTPS):** Điều hướng dựa trên tên miền (Domain/Host) hoặc đường dẫn (Path).
    
    - Ví dụ Path-based: `[example.com/api](https://example.com/api)` -> Service API, `[example.com/web](https://example.com/web)` -> Service Web.
        
    - Ví dụ Host-based: `api.domain.com` -> Service API, `app.domain.com` -> Service Web.
        
2. **Quản lý SSL/TLS Termination:** Giải mã traffic HTTPS ngay tại Ingress Controller trước khi chuyển tiếp HTTP nội bộ tới các Pod, giúp giảm tải việc xử lý chứng chỉ cho ứng dụng.
    
3. **Cân bằng tải (Load Balancing):** Tự động phân phối truy cập đến các Pod đang chạy.
    
4. **Tiết kiệm chi phí:** Thay vì tạo mỗi ứng dụng 1 Load Balancer của đám mây (NodePort/LoadBalancer Service), bạn chỉ cần **1 Load Balancer duy nhất** cho Ingress Controller để định tuyến cho hàng chục dịch vụ bên trong.
    
### Ví dụ cấu hình một Ingress Resource

Sau khi đã cài đặt một Ingress Controller (ví dụ NGINX Ingress Controller), bạn định nghĩa quy tắc điều hướng bằng file YAML như sau:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  ingressClassName: nginx # Khai báo Controller xử lý
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Các Ingress Controller phổ biến

| **Ingress Controller**       | **Đặc điểm chính**                                        |
| ---------------------------- | --------------------------------------------------------- |
| **NGINX Ingress Controller** | Phổ biến nhất, dễ dùng, cộng đồng hỗ trợ lớn.             |
| **Traefik**                  | Tự động phát hiện dịch vụ, giao diện Dashboard trực quan. |
| **HAProxy Ingress**          | Tối ưu hiệu năng cao, chịu tải lớn.                       |
| **[[Contour]] / Istio Ingress**  | Thường dùng trong kiến trúc Service Mesh phức tạp.        |