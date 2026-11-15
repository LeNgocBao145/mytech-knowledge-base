**Calico** (hay Project Calico) là một giải pháp mạng (**[[CNI]] - Container Network Interface**) và bảo mật mạng (Network Security) mã nguồn mở rất phổ biến trong Kubernetes.

Nếu như **Flannel** chỉ tập trung làm nhiệm vụ duy nhất là "nối mạng" đơn giản giữa các Pod, thì **Calico** được thiết kế mạnh mẽ hơn rất nhiều, cung cấp cả **khả năng định tuyến mạng hiệu năng cao** lẫn **chính sách bảo mật (Security Policy)** chi tiết.

### 1. Calico giải quyết bài toán gì?

Calico đảm nhận 2 vai trò chính trong cụm Kubernetes:

1. **Quản lý mạng (Networking - L3 Routing):**
    
    - Cấp phát địa chỉ IP cho từng Pod.
        
    - Định tuyến gói tin giữa các Pod trên các Worker Node khác nhau mà **không nhất thiết phải dùng mạng ảo (Overlay network như VXLAN)**.
        
    - Calico sử dụng giao thức **BGP (Border Gateway Protocol)** — giao thức tiêu chuẩn đang chạy trên toàn bộ mạng Internet — giúp các Node trao đổi bảng đường đi trực tiếp với nhau hoặc kết nối thẳng với hạ tầng mạng vật lý của Data Center.
        
2. **Bảo mật mạng (Network Security / Firewall):**
    
    - Cung cấp cơ chế **Network Policy** cực kỳ linh hoạt.
        
    - Cho phép bạn tạo các quy tắc tường lửa: Pod nào được phép kết nối với Pod nào, Pod nào bị chặn, namespace nào được ra Internet,... giúp triển khai mô hình bảo mật **Zero Trust**.
        

### 2. So sánh Calico với Flannel

|**Tiêu chí**|**Flannel**|**Calico**|
|---|---|---|
|**Mục tiêu chính**|Mạng đơn giản, dễ cài đặt.|Mạng hiệu năng cao + Bảo mật chi tiết.|
|**Bảo mật (Network Policy)**|❌ **Không hỗ trợ**. Tất cả Pod mặc định thấy nhau.|✅ **Hỗ trợ rất mạnh** (L3, L4 và cả L7 nếu dùng eBPF/Envoy).|
|**Cơ chế truyền gói tin**|Dùng **Overlay** (VXLAN) — đóng gói gói tin nên tốn một chút chi phí CPU/Latency.|Hỗ trợ cả **Non-overlay (BGP)** (truyền trực tiếp, hiệu năng tối đa) lẫn Overlay (VXLAN/IPIP).|
|**Độ phức tạp**|Rất nhẹ, cấu hình đơn giản.|Phức tạp hơn, cần hiểu biết cơ bản về IP routing/BGP nếu cấu hình nâng cao.|
|**Phù hợp cho**|Cụm Dev/Test, cụm nhỏ, không yêu cầu bảo mật giữa các Pod.|Cụm Production, hệ thống Microservices doanh nghiệp lớn.|

### 3. Ví dụ thực tế về tính năng Bảo mật của Calico

Trong một cụm Kubernetes dùng Calico, bạn có thể dễ dàng khai báo một file YAML để chặn ứng dụng **Frontend** truy cập trực tiếp vào **Database**:

YAML

```
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: block-frontend-to-db
  namespace: production
spec:
  selector: app == 'database'
  ingress:
  - action: Deny
    source:
      selector: app == 'frontend'
```

_Nếu dùng Flannel, bạn sẽ không thể thực thi được quy tắc bảo mật này._

### Tóm lại:

Calico là một [[CNI]] "chuẩn công nghiệp" (Production-ready). Khi bạn bắt đầu đưa Kubernetes vào môi trường thực tế (Production) và cần **bảo mật phân vùng mạng giữa các Microservices** hoặc **tối ưu tốc độ mạng**, Calico là một trong những lựa chọn hàng đầu thay thế cho Flannel.