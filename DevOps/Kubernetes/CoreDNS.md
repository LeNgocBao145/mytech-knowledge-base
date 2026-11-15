**CoreDNS** là một DNS server (hệ thống phân giải tên miền) linh hoạt, có khả năng mở rộng cao và được viết bằng ngôn ngữ Go. Trong Kubernetes (k8s), CoreDNS đóng vai trò là **DNS server mặc định của cụm (cluster)** từ phiên bản 1.13 (thay thế cho Kube-DNS).

Nó chịu trách nhiệm tự động phát hiện các Service, Pod trong cụm và cung cấp cơ chế phân giải tên miền (DNS resolution) giúp các ứng dụng giao tiếp với nhau dễ dàng.

### CoreDNS hoạt động như thế nào trong Kubernetes?

1. **Service Discovery (Tự động phát hiện dịch vụ):**
    
    - Khi bạn tạo một Service mới trong Kubernetes, Kubernetes sẽ gán cho nó một tên miền nội bộ theo định dạng:
        
        `<ten-service>.<namespace>.svc.cluster.local`
        
    - CoreDNS lắng nghe các sự kiện từ Kubernetes API Server. Khi có Service hoặc Pod mới được tạo, sửa, hoặc xóa, CoreDNS lập tức cập nhật bản đồ tên miền tương ứng.
        
2. **Cách Pod truy cập dịch vụ:**
    
    - Thay vì phải nhớ địa chỉ IP động của các Pod/Service, một ứng dụng trong Pod chỉ cần gọi tên Service (ví dụ: `http://backend-service`).
        
    - Yêu cầu DNS sẽ được gửi đến CoreDNS, CoreDNS tra cứu và trả về địa chỉ IP ClusterIP tương ứng của Service đó.
        

### Kiến trúc chính: Plugin-based

Sức mạnh lớn nhất của CoreDNS nằm ở kiến trúc **mọi thứ là Plugin**. Quá trình xử lý một truy vấn DNS đi qua một chuỗi chuỗi (chain) các plugin:

- **`kubernetes` plugin:** Đọc dữ liệu từ Kubernetes API để phân giải các tên miền `.cluster.local`.
    
- **`forward` / `proxy` plugin:** Nối tiếp các truy vấn tên miền bên ngoài (ví dụ: `google.com`) đến các DNS server ngoài cụm (như `8.8.8.8` hoặc DNS của nhà cung cấp cloud).
    
- **`cache` plugin:** Lưu tạm kết quả truy vấn DNS để tăng tốc độ phản hồi và giảm tải cho API Server.
    
- **`prometheus` plugin:** Xuất các thông số đo lường (metrics) giúp giám sát hiệu năng của DNS server qua Prometheus.
    

### Xem thông tin và Cấu hình CoreDNS

CoreDNS chạy dưới dạng một **Deployment** nằm trong namespace `kube-system`:

Bash

```
# Kiểm tra trạng thái Pod CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Kiểm tra Service của CoreDNS
kubectl get svc -n kube-system -l k8s-app=kube-dns
```

Cấu hình của CoreDNS trong Kubernetes được lưu trong một ConfigMap tên là `coredns` thuộc namespace `kube-system`:

Bash

```
# Xem file cấu hình CoreDNS (Corefile)
kubectl get configmap coredns -n kube-system -o yaml
```

Cấu hình cơ bản (Corefile) thường có dạng:

Plaintext

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### Tại sao CoreDNS lại thay thế Kube-DNS?

- **Hiệu năng và Tài nguyên:** Tốn ít bộ nhớ (RAM) và CPU hơn.
    
- **Ít container hơn:** Kube-DNS dùng 3 container độc lập cho mỗi Pod, trong khi CoreDNS chỉ dùng 1 container duy nhất.
    
- **Dễ mở rộng:** Có thể thêm hoặc tùy biến plugin cực kỳ đơn giản mà không cần biên dịch lại toàn bộ hệ thống.