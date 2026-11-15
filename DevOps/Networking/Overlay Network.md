**Overlay Network (mạng phủ / mạng ảo)** là một mạng máy tính logic được khởi tạo và chạy đè lên trên một hạ tầng mạng vật lý có sẵn (được gọi là **Underlay Network**).

Trong Overlay Network, các nút mạng (Nodes) kết nối với nhau bằng các liên kết ảo (virtual links). Dữ liệu truyền giữa các nút mạng ảo này vẫn được đóng gói và vận chuyển qua các router/switch thực tế của mạng Underlay bên dưới, nhưng các ứng dụng trên Overlay chỉ thấy sơ đồ mạng ảo của chính nó.

### Mô hình So sánh: Overlay vs. Underlay

|**Đặc tính**|**Underlay Network (Mạng nền)**|**Overlay Network (Mạng phủ)**|
|---|---|---|
|**Bản chất**|Hạ tầng vật lý (Switches, Routers, Cáp mạng, BGP, OSPF).|Mạng logic/phần mềm (Virtual interfaces, Tunnels, Software-Defined).|
|**Nhiệm vụ**|Đảm bảo kết nối IP cơ bản, định tuyến gói tin vật lý giữa các điểm cuối.|Cung cấp tính năng mạng nâng cao: cô lập dữ liệu, bảo mật, ảo hóa L2/L3.|
|**Cơ chế**|Xử lý phần cứng (Hardware forwarding).|Đóng gói dữ liệu (Encapsulation: VXLAN, GRE, WireGuard, IPsec...).|
|**Ví dụ**|Mạng LAN văn phòng, mạng Internet của ISP, hạ tầng Switch L3 Data Center.|Kubernetes Pod Network, Tailscale/VPN, VXLAN, mạng Tor.|

### Cơ chế hoạt động chính: Đóng gói (Encapsulation)

Quá trình truyền dữ liệu trong Overlay Network dựa trên cơ chế **Tunneling (Tạo đường hầm)**:

1. **Đóng gói (Encapsulate):** Khi một gói tin logic được gửi từ nguồn (ví dụ: Container A), thiết bị đầu cuối của Overlay (VTEP, VPN Gateway, hoặc CNI Agent) sẽ bọc gói tin đó vào bên trong một gói tin IP/UDP chuẩn của mạng Underlay.
    
2. **Vận chuyển:** Mạng vật lý (Underlay) chỉ nhìn thấy header ngoài và chuyển tiếp gói tin như một lưu lượng IP bình thường qua các Router/Switch.
    
3. **Giải đóng gói (Decapsulate):** Khi đến đích (Node chứa Container B), điểm cuối Overlay gỡ bỏ header bên ngoài và chuyển gói tin gốc đến đúng ứng dụng nhận.
    

### Vai trò và Ứng dụng thực tế

- **Kubernetes & Container Networking (CNI):** Các plugin như **Flannel, Calico, Cilium** tạo một Overlay Network (dùng VXLAN hoặc IP-in-IP) để toàn bộ Pod trên nhiều Worker Node khác nhau có thể giao tiếp trực tiếp qua IP nội bộ mà không cần quan tâm đến sơ đồ mạng vật lý bên dưới.
    
- **Trung tâm dữ liệu & Cloud (SDN):** Công nghệ như VXLAN hay NVGRE giúp các nhà cung cấp đám mây (AWS, GCP, VMware NSX) tạo ra hàng triệu mạng VPC/VLAN ảo độc lập cho từng khách hàng trên cùng một hạ tầng phần cứng dùng chung.
    
- **Mạng riêng ảo (VPN / Mesh Network):** WireGuard, Tailscale, OpenVPN tạo các mạng ảo bảo mật nối liền các máy tính ở nhiều vị trí địa lý khác nhau qua môi trường Internet public.
    
- **Mạng ngang hàng (P2P) & Bảo mật:** Mạng BitTorrent (để định vị file) hay Tor (mạng định tuyến ẩn danh Onion Routing) đều hoạt động theo mô hình Overlay Network trên nền Internet.