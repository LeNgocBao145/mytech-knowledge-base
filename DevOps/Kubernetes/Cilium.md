**Cilium** là một giải pháp [[CNI]] (Container Network Interface) thế hệ mới dành cho Kubernetes, nổi tiếng nhờ việc sử dụng công nghệ cốt lõi **eBPF (Extended Berkeley Packet Filter)** tích hợp trực tiếp trong Kernel của Linux.

Nếu như **Flannel** đại diện cho thế hệ [[CNI]] đơn giản, **[[Calico]]** đại diện cho thế hệ [[CNI]] mạnh về định tuyến/tường lửa bằng `iptables`, thì **Cilium** được xem là [[CNI]] hiện đại nhất hiện nay, thay đổi hoàn toàn cách xử lý Mạng (Networking), Bảo mật (Security) và Giám sát (Observability) trong Kubernetes.

### 1. Cilium giải quyết bài toán gì và điểm đặc biệt là gì?

Điểm khác biệt lớn nhất của Cilium nằm ở **eBPF**:

- **Xử lý mạng cực nhanh ở tầng Kernel:** Thay vì phải đẩy gói tin qua các bảng quy tắc `iptables` hoặc `IPVS` cồng kềnh (vốn trở nên rất chậm khi cụm có hàng nghìn Service/Pod), Cilium chèn các chương trình eBPF nhỏ trực tiếp vào Linux Kernel. Gói tin được xử lý và định tuyến ngay tại điểm nhận mạng mà không tốn chi phí chuyển đổi giữa Kernel và User-space.
    
- **Thay thế hoàn toàn `kube-proxy`:** Cilium có chế độ _kube-proxy replacement_. Nó loại bỏ hoàn toàn `kube-proxy` và `iptables`, giúp việc Load Balancing cho các Kubernetes Service đạt hiệu năng cao vượt trội và tốn ít CPU hơn.
    
- **Bảo mật & Giám sát sâu ở Tầng 7 (L7 Aware):** Cilium không chỉ hiểu địa chỉ IP hay Port (Tầng 3/4), mà còn hiểu được các giao thức ứng dụng như HTTP, gRPC, Kafka,... Bạn có thể viết chính sách bảo mật kiểu: _"Pod Frontend chỉ được gửi phương thức `GET /api/v1/users` tới Pod Backend, chặn mọi lệnh `POST` hoặc `DELETE`"_.
    

### 2. Bảng so sánh: Flannel vs [[Calico]] vs Cilium

|**Tiêu chí**|**Flannel**|**[[Calico]]**|**Cilium**|
|---|---|---|---|
|**Công nghệ cốt lõi**|VXLAN / Overlay|BGP, IPAM, `iptables`|**eBPF (Linux Kernel)**|
|**Bảo mật (Network Policy)**|❌ Không hỗ trợ|✅ L3/L4 (`iptables`)|✅ **L3/L4 và L7** (HTTP, gRPC, Kafka...)|
|**Thay thế kube-proxy**|❌ Không|❌ Không|✅ **Có** (eBPF Service Load Balancing)|
|**Khả năng quan sát (Observability)**|❌ Cơ bản|⚠️ Cần công cụ ngoài|✅ **Cực mạnh** (Tích hợp sẵn Hubble)|
|**Tích hợp Service Mesh**|❌ Không|⚠️ Hỗ trợ hạn chế|✅ **Service Mesh không cần Sidecar**|
|**Độ phức tạp**|Rất nhẹ, đơn giản nhất|Trung bình|Yêu cầu Kernel Linux mới (>= 5.4)|

### 3. Hệ sinh thái đi kèm của Cilium

Cilium không chỉ dừng lại ở vai trò một [[CNI]] thông thường, nó đã phát triển thành một hệ sinh thái mạnh mẽ:

1. **Hubble:** Công cụ quan sát mạng (Observability) tích hợp sẵn. Cho phép bạn xem realtime luồng traffic giữa các Pod, biết chính xác request nào bị block, latency bao nhiêu, lỗi HTTP 5xx xuất hiện ở đâu thông qua giao diện đồ họa (UI) hoặc CLI.
    
2. **Cilium Service Mesh (Sidecarless):** Giúp triển khai các tính năng của Service Mesh (mTLS, Canary deployment, Tracing...) **mà không cần cài đặt Envoy Sidecar container vào từng Pod**. Điều này giúp tiết kiệm đáng kể RAM và CPU cho cụm Kubernetes.
    
3. **Tetragon:** Công cụ bảo mật thời gian thực (Security Observability & Runtime Enforcement) phát hiện các hành vi bất thường của tiến trình chạy bên trong container.
    

### Tóm lại:

- **Nên chọn Flannel khi:** Cần dựng cụm Lab/Dev cực kỳ nhẹ và đơn giản.
    
- **Nên chọn [[Calico]] khi:** Cần hạ tầng mạng Production ổn định, dùng giao thức BGP truyền thống, hạ tầng chạy Linux Kernel cũ.
    
- **Nên chọn Cilium khi:** Cần hiệu năng mạng tối đa, cụm Kubernetes quy mô lớn (hàng nghìn Pod), cần khả năng quan sát mạng chuyên sâu (Hubble) hoặc muốn dùng Service Mesh mà không tốn tài nguyên cho Sidecar.