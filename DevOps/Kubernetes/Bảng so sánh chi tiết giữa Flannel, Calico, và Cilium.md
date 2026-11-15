Dưới đây là bảng so sánh cực kỳ chi tiết giữa **Flannel**, **[[Calico]]**, và **[[Cilium]]** — ba giải pháp [[CNI]] (Container Network Interface) phổ biến nhất trong hệ sinh thái Kubernetes hiện nay.

## 1. Bảng so sánh tổng quan & Chi tiết kỹ thuật

|**Tiêu chí**|**Flannel**|**[[Calico]]**|**[[Cilium]]**|
|---|---|---|---|
|**Thế hệ [[CNI]]**|Thế hệ đầu (Legacy / Basic)|Thế hệ thứ hai (Production Standard)|Thế hệ thứ ba ([[eBPF]]-native / Next-Gen)|
|**Công nghệ xử lý cốt lõi**|Linux Bridge, TUN/TAP, VXLAN|Linux Routing, BGP, `iptables` / `IPVS`|**[[eBPF]] (Extended Berkeley Packet Filter)**|
|**Kiến trúc mạng (Data Plane)**|Overlay (mặc định VXLAN, UDP, Host-GW)|Cả Non-Overlay (BGP) và Overlay (VXLAN, IP-in-IP)|Cả Non-Overlay (Direct Routing) và Overlay (VXLAN, Geneve)|
|**Bảo mật (Network Policy)**|❌ **Không hỗ trợ**|✅ **Rất mạnh (L3/L4)**: IP, Port, Protocol, Namespace, Service Account|✅ **Cực mạnh (L3/L4/L7)**: HTTP Method, Path, gRPC, Kafka, FQDN|
|**Thay thế `kube-proxy`**|❌ Không|⚠️ Có (qua [[eBPF]] Data Plane ở bản mới)|✅ **Có (Kube-proxy Replacement bằng [[eBPF]])**|
|**Hiệu năng & Latency**|Trung bình (Tốn CPU do đóng gói VXLAN)|Cao (Rất nhanh ở chế độ BGP Native Routing)|**Cực cao** (Bypass TCP/IP stack, tối ưu độ trễ thấp nhất)|
|**Khả năng mở rộng (Scale)**|Nhỏ - Trung bình (~ Khuyên dùng $< 500$ nodes)|Rất lớn (Hàng nghìn Nodes với BGP Route Reflectors)|Rất lớn (Hàng nghìn Nodes, xử lý $O(1)$ Service Lookup)|
|**Tích hợp Service Mesh**|❌ Không|⚠️ Cần Sidecar (Envoy/Istio)|✅ **Service Mesh không Sidecar (Sidecarless)**|
|**Khả năng quan sát (Observability)**|❌ Rất hạn chế (Chỉ có log cơ bản)|⚠️ Trung bình ([[Calico]] Felix metrics, Prometheus)|✅ **Cực mạnh (Hubble UI/CLI)**: Flow log, L7 metrics, Tracing|
|**Yêu cầu Kernel Linux**|Thấp (Mọi Kernel Linux cũ)|Trung bình ($\ge 3.10$ cho iptables, $\ge 5.3$ cho [[eBPF]])|Cao ($\ge 5.4$ khuyến nghị để dùng đầy đủ tính năng [[eBPF]])|
|**Độ phức tạp vận hành**|Rất thấp (Cài là chạy, gần như zero-config)|Trung bình - Cao (Cần kiến thức BGP/Network nếu dùng Native)|Trung bình - Cao (Cần hiểu biết về [[eBPF]]/Kernel khi troubleshoot)|

## 2. Phân tích chuyên sâu từng [[CNI]]

### 🤖 1. Flannel: Sự lựa chọn tối giản

- **Cơ chế hoạt động:** Flannel gán cho mỗi Node một dải IP (Subnet) lấy từ một CIDR chung. Dữ liệu giữa các Pod ở khác Node được đóng gói (encapsulated) qua giao thức VXLAN hoặc UDP để truyền đi.
    
- **Ưu điểm:**
    
    - **Cực kỳ nhẹ và đơn giản:** Cấu hình rất ít, gần như không gây lỗi xung đột hạ tầng.
        
    - **Độ tin cậy cao cho mục đích cơ bản:** Bất kỳ hệ điều hành Linux nào cũng hỗ trợ tốt.
        
- **Nhược điểm:**
    
    - **Không có Network Policy:** Mọi Pod đều có thể tự do giao tiếp với tất cả các Pod khác trong cụm (vi phạm nguyên tắc bảo mật Zero-Trust).
        
    - **Hiệu năng bị giới hạn:** Do việc đóng/mở gói tin (Encap/Decap) ở tầng Overlay tốn thêm thời gian CPU.
        
- **Kịch bản sử dụng phù hợp:** Các cụm Kubernetes thử nghiệm (Lab, Dev/Staging), cụm IoT/Edge nhẹ, các hệ thống nhỏ không có yêu cầu khắt khe về phân vùng bảo mật.
    

### 🛡️ 2. [[Calico]]: Chuẩn mực cho môi trường Production truyền thống

- **Cơ chế hoạt động:** [[Calico]] biến mỗi Worker Node thành một Router (BGP Speaker). Thay vì đóng gói gói tin thành Overlay, [[Calico]] định tuyến trực tiếp các IP của Pod trên hạ tầng mạng vật lý (dùng giao thức BGP). Nếu hạ tầng không hỗ trợ BGP, [[Calico]] có thể chuyển sang dùng VXLAN hoặc IP-in-IP.
    
- **Ưu điểm:**
    
    - **Hiệu năng mạng vượt trội (ở chế độ BGP):** Do không tốn chi phí đóng gói gói tin Overlay, dữ liệu đi thẳng qua card mạng vật lý với độ trễ thấp nhất.
        
    - **Hệ thống Network Policy cực kỳ mạnh mẽ:** Cho phép định nghĩa quy tắc tường lửa chi tiết theo Label, Namespace, CIDR, Port, Service Account.
        
    - **Hỗ trợ Multi-Cloud & Hybrid-Cloud:** Có thể chạy nhất quán trên AWS, Azure, On-Premise, Bare-Metal.
        
- **Nhược điểm:**
    
    - Bảng `iptables` phình to khi số lượng Pod/Service tăng lên hàng nghìn, gây ảnh hưởng đến CPU của Node và làm chậm tốc độ cập nhật rule.
        
    - Cấu hình BGP yêu cầu đội ngũ vận hành có kiến thức tốt về Mạng (Networking/Routing).
        
- **Kịch bản sử dụng phù hợp:** Các hệ thống Enterprise Production quy mô vừa và lớn, hạ tầng On-premise/Data Center có hạ tầng mạng hỗ trợ BGP, các hệ thống đòi hỏi tuân thủ tiêu chuẩn bảo mật khắt khe ở Tầng 3/4.
    

### ⚡ 3. [[Cilium]]: Công nghệ thế hệ mới dẫn đầu xu hướng

- **Cơ chế hoạt động:** [[Cilium]] sử dụng **[[eBPF]]** để can thiệp trực tiếp vào Linux Kernel. Nó thay thế hoàn toàn bảng `iptables` bằng các bảng hash $O(1)$ trong Kernel bộ nhớ. Nó cũng có thể xử lý trực tiếp gói tin tại tầng driver card mạng (XDP).
    
- **Ưu điểm:**
    
    - **Hiệu năng đỉnh cao & Loại bỏ `kube-proxy`:** Định tuyến và Load Balancing trực tiếp trong Kernel, giảm thiểu tối đa Overhead và CPU Spike khi số lượng Service phình to.
        
    - **Bảo mật Tầng 7 (L7 Security):** Kiểm soát chi tiết đến mức HTTP Method (`GET`, `POST`), URL Path, gRPC call hay Kafka Topic.
        
    - **Giám sát trực quan với Hubble:** Cung cấp khả năng quan sát dòng chảy dữ liệu (Traffic Flow), phát hiện lỗi kết nối, đo Latency theo thời gian thực mà không cần cài thêm Agent ngoài.
        
    - **Service Mesh không Sidecar (Sidecarless):** Triển khai mTLS, Tracing, L7 Traffic Management ngay tại Kernel mà không cần tốn tài nguyên RAM/CPU để chèn Envoy Sidecar vào từng Pod.
        
- **Nhược điểm:**
    
    - Yêu cầu Kernel Linux mới ($\ge 5.4$).
        
    - Độ phức tạp khi troubleshoot sâu trong Kernel nếu chương trình [[eBPF]] gặp sự cố bất ngờ.
        
- **Kịch bản sử dụng phù hợp:** Cụm Kubernetes quy mô siêu lớn (Large-Scale / High-Density), ứng dụng đòi hỏi độ trễ cực thấp (Fintech, Real-time Streaming), các hệ thống Microservices phức tạp muốn tối ưu chi phí tài nguyên bằng Service Mesh không Sidecar.
    

## 3. Tổng kết: Khi nào nên chọn [[CNI]] nào?

```
                 [Nhu cầu của bạn là gì?]
                            │
         ┌──────────────────┼──────────────────┐
         ▼                  ▼                  ▼
   [Đơn giản nhất]    [Chuẩn Enterprise]   [Hiệu năng cao nhất]
   [Cụm Dev / Lab]    [L3/L4 Security]     [L7 Security + eBPF]
         │                  │                  │
         ▼                  ▼                  ▼
    🟢 FLANNEL          🟡 CALICO          🔵 CILIUM
```

- **Chọn FLANNEL nếu:** Bạn mới bắt đầu học Kubernetes, cần dựng cụm đơn giản nhanh chóng, hoặc chạy trên các thiết bị tài nguyên hạn chế (như Raspberry Pi / K3s) mà không lo về bảo mật Network Policy.
    
- **Chọn [[CALICO]] nếu:** Bạn dựng cụm Production trên hạ tầng sẵn có, cần giải pháp bảo mật L3/L4 chuẩn mực, ổn định, đã được kiểm chứng qua nhiều năm và hạ tầng mạng đã quen thuộc với BGP / `iptables`.
    
- **Chọn [[CILIUM]] nếu:** Bạn muốn tối ưu hóa tài nguyên cho tương lai, cần quan sát mạng chuyên sâu (Observability), muốn triển khai Service Mesh tiết kiệm tài nguyên, hoặc hệ thống chạy hàng nghìn Microservices với tần suất giao tiếp cực cao.