**VXLAN (Virtual Extensible LAN)** là một giao thức đóng gói mạng (network encapsulation protocol) thuộc nhóm công nghệ **Overlay Network** (RFC 7348). VXLAN cho phép mở rộng mạng Lớp 2 (Layer 2 - Ethernet) truyền đi trên nền hạ tầng mạng Lớp 3 (Layer 3 - IP network) bằng cách đóng gói khung tin Ethernet vào bên trong gói tin UDP.

### Vấn đề VXLAN giải quyết

VXLAN ra đời nhằm khắc phục các giới hạn của mạng VLAN truyền thống trong các trung tâm dữ liệu (Data Center) và môi trường điện toán đám mây:

1. **Vượt qua giới hạn 4.094 VLAN:** VLAN truyền thống sử dụng **VLAN ID 12-bit**, chỉ tạo tối đa $2^{12} = 4.094$ mạng ảo — con số quá nhỏ cho các nhà cung cấp Cloud hoặc Data Center đa người dùng (Multi-tenant). VXLAN sử dụng **VNI (VXLAN Network Identifier) 24-bit**, cho phép tạo tối đa $2^{24} \approx 16,7 \text{ triệu}$ mạng ảo riêng biệt.
    
2. **Mở rộng kết nối L2 qua hạ tầng L3 (L2 extension over L3):** Cho phép các máy ảo (VM) hoặc Container thuộc cùng một phân đoạn L2 có thể giao tiếp trực tiếp với nhau ngay cả khi nằm ở các rack, hạ tầng hoặc Data Center khác nhau mà không cần đổi địa chỉ IP (hỗ trợ Live Migration / vMotion qua L3).
    
3. **Loại bỏ hạn chế của Spanning Tree Protocol (STP):** Mạng Lớp 2 truyền thống phụ thuộc vào STP để chống lặp (loop), dẫn đến việc phải khóa nhiều đường truyền dự phòng. VXLAN chạy trên nền mạng Lớp 3 (Underlay), tận dụng được các thuật toán định tuyến L3 như **ECMP (Equal-Cost Multi-Path)** để khai thác tối đa băng thông tất cả các đường truyền.
    

### Thành phần và Cơ chế hoạt động

Cấu trúc mạng VXLAN chia làm 2 lớp:

- **Underlay Network:** Hạ tầng mạng vật lý Lớp 3 (Switches, Routers) chịu trách nhiệm định tuyến các gói tin IP giữa các nút mạng.
    
- **Overlay Network:** Mạng ảo Lớp 2 được thiết lập đè lên trên mạng Underlay.
    

#### Các thành phần chính:

- **VTEP (VXLAN Tunnel Endpoint):** Là điểm đầu/cuối của đường hầm VXLAN, chịu trách nhiệm đóng gói (encapsulate) traffic L2 từ VM/Container thành gói UDP trước khi gửi đi, và giải đóng gói (decapsulate) khi nhận dữ liệu. VTEP có thể là thiết bị phần cứng (Switch) hoặc phần mềm (như vSwitch trong ESXi, Open vSwitch trong Linux, K8s CNI plugin).
    
- **VNI (VXLAN Network Identifier):** Mã định danh 24-bit dùng để phân biệt các mạng ảo L2 với nhau (tương tự như VLAN ID).
    

#### Cấu trúc gói tin VXLAN (MAC-in-UDP):

Gói tin gốc Lớp 2 được bọc thêm các header trước khi truyền đi trên hạ tầng IP:

$$\text{[ Outer MAC ] } \rightarrow \text{ [ Outer IP ] } \rightarrow \text{ [ UDP Header (Port 4789) ] } \rightarrow \text{ [ VXLAN Header (VNI) ] } \rightarrow \text{ [ Inner Ethernet Frame ]}$$

- **Outer IP:** Chứa IP nguồn/đích của 2 thiết bị VTEP.
    
- **UDP Header:** Sử dụng **UDP Port 4789** (chuẩn IANA).
    
- **VXLAN Header:** Chứa thông tin 24-bit VNI.
    
- **Inner Ethernet Frame:** Gói tin Lớp 2 gốc của ứng dụng/VM.
    

### Ứng dụng thực tế

- **Kubernetes Networking (CNI):** Các plugin như **Calico**, **Cilium**, **Flannel** thường dùng VXLAN mode để tạo overlay network kết nối các Pod chạy trên các Worker Node thuộc các subnet L3 khác nhau.
    
- **Software-Defined Networking (SDN):** Sử dụng trong VMware NSX, Cisco ACI, OpenStack Neutron để khởi tạo và quản lý mạng ảo linh hoạt theo nhu cầu của ứng dụng.
    
- **Multi-tenant Cloud:** Cách ly lưu lượng mạng giữa các khách hàng khác nhau trong cùng một hạ tầng trung tâm dữ liệu vật lý.