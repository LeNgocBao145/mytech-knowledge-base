**CNI (Container Network Interface)** trong Kubernetes là một **chuẩn giao diện (spec/API)** quy định cách các plugin mạng kết nối và thiết lập mạng cho các Pod trong cluster.

Nói một cách ngắn gọn: Kubernetes không tự xây dựng cơ chế kết nối mạng chi tiết cho các Pod mà chỉ đưa ra tiêu chuẩn CNI. Sau đó, các công cụ mạng bên thứ ba (như Calico, Flannel, Cilium) sẽ cài đặt theo chuẩn CNI này để thực hiện việc cấp phát IP và định tuyến dữ liệu.

### Vai trò chính của CNI Plugin

1. **Cấp phát IP:** Tự động gán cho mỗi Pod một địa chỉ IP duy nhất trong toàn bộ cluster.
    
2. **Quản lý vòng đời mạng:** Tự động tạo giao diện mạng (veth pair) khi Pod được khởi tạo và thu hồi IP/xóa giao diện mạng khi Pod bị xóa.
    
3. **Đảm bảo kết nối Flat Network:** Cho phép tất cả các Pod trên các Node khác nhau có thể giao tiếp trực tiếp với nhau bằng IP mà không cần NAT (Network Address Translation).
    
4. **Bảo mật (Network Policy):** Một số CNI cao cấp cung cấp tính năng tường lửa (Network Policy) để chặn hoặc cho phép luồng truy cập giữa các Pod.
    

### Các CNI Plugin phổ biến hiện nay

|**CNI Plugin**|**Đặc điểm nổi bật**|**Trường hợp sử dụng**|
|---|---|---|
|**Flannel**|Đơn giản, nhẹ, dễ cài đặt (chỉ định tuyến L3).|Cluster nhỏ, môi trường Dev/Test.|
|**Calico**|Hiệu năng cao, hỗ trợ Network Policy mạnh mẽ.|Production, hệ thống cần bảo mật tốt.|
|**Cilium**|Dựa trên công nghệ eBPF, hiệu năng vượt trội, hỗ trợ quan sát (Observability) và L7 Security.|Production hiện đại, hệ thống quy mô lớn.|
|**AWS VPC CNI / Azure CNI**|Tích hợp trực tiếp IP của Pod vào hạ tầng mạng của Cloud Provider.|Kubernetes quản lý trên Cloud (EKS, AKS).|

## Cho tôi luồng di chuyển của gói tin từ pod này sang pod khi khác node để chứng minh công dụng của CNI lấy ví dụ Flannel

### Kịch bản giả định

- **Node 1 (`192.168.1.10`)**: Chứa **Pod A** (`10.244.1.5`)
    
- **Node 2 (`192.168.1.20`)**: Chứa **Pod B** (`10.244.2.8`)
    
- **CNI Plugin**: Flannel (chạy chế độ **VXLAN** mặc định)
    

### Luồng di chuyển của gói tin (Packet Flow)

```
[ Pod A: 10.244.1.5 ]
       │ (veth)
       ▼
[ Bridge: cni0 ] 
       │ (Kernel Routing Table)
       ▼
[ Interface: flannel.1 (VTEP) ] ── (Đóng gói Outer UDP Header: 192.168.1.10 -> 192.168.1.20)
       │
       ▼ (Mạng vật lý / Underlay Network)
[ Physical Switch / Router ]
       │
       ▼
[ Interface: eth0 (Node 2) ]
       │ (UDP Port 4789)
       ▼
[ Interface: flannel.1 (VTEP) ] ── (Giải đóng gói, lấy gói tin gốc: 10.244.1.5 -> 10.244.2.8)
       │
       ▼
[ Bridge: cni0 ]
       │ (veth)
       ▼
[ Pod B: 10.244.2.8 ]
```

#### Bước 1: Khởi tạo gói tin từ Pod A

Pod A muốn gửi dữ liệu tới Pod B. Nó tạo một gói tin IP chuẩn:

- **Src IP:** `10.244.1.5` | **Dst IP:** `10.244.2.8`
    
- Gói tin đi qua giao diện mạng ảo `eth0` bên trong Pod, chạy qua cặp **veth pair** để ra ngoài namespace của Node 1 và đi vào bridge `cni0`.
    

#### Bước 2: Tra cứu bảng định tuyến tại Node 1

Kernel của Node 1 nhận gói tin tại `cni0` và tra cứu bảng định tuyến (Routing Table) của Host:

- Bảng định tuyến ghi nhận: Các gói tin đến dải `10.244.2.0/24` phải chuyển sang interface `flannel.1` (VTEP - VXLAN Tunnel Endpoint).
    

#### Bước 3: Đóng gói Overlay (VXLAN Encapsulation) tại Node 1

Interface `flannel.1` lấy gói tin gốc (Inner Packet) và đóng gói thêm một lớp vỏ bên ngoài (Outer Header):

- **Inner Header (Gốc):** `[10.244.1.5] -> [10.244.2.8]`
    
- **VXLAN Header:** VNI = 1 (mã định danh mạng ảo)
    
- **Outer UDP Header:** Dst Port = `4789`
    
- **Outer IP Header:** `[192.168.1.10] -> [192.168.1.20]`
    

> _Để biết IP của Node 2 (`192.168.1.20`), `flannel.1` tra cứu bảng FDB (Forwarding Database) và ARP table nội bộ do tiến trình `flanneld` duy trì._

#### Bước 4: Truyền qua hạ tầng mạng vật lý

Gói tin lúc này là một gói UDP thông thường. Card mạng vật lý `eth0` của Node 1 gửi gói tin đi. Các Switch/Router vật lý trên mạng nội bộ chỉ làm nhiệm vụ chuyển tiếp gói IP từ `192.168.1.10` đến `192.168.1.20` mà hoàn toàn không cần biết sự tồn tại của dải IP Pod `10.244.x.x`.

#### Bước 5: Giải đóng gói (Decapsulation) tại Node 2

Card mạng `eth0` của Node 2 nhận gói tin UDP tại Port 4789 và chuyển cho interface `flannel.1` xử lý:

- `flannel.1` trên Node 2 gỡ bỏ lớp vỏ Outer IP, Outer UDP và VXLAN Header.
    
- Thu lại gói tin IP gốc: `[Src: 10.244.1.5] -> [Dst: 10.244.2.8]`.
    

#### Bước 6: Chuyển tiếp vào Pod B

- Kernel Node 2 nhận gói tin gốc, kiểm tra bảng định tuyến local và đẩy gói tin vào bridge `cni0`.
    
- Bridge `cni0` chuyển gói tin qua cặp `veth` tương ứng để vào namespace của Pod B.
    
- Pod B nhận gói tin và thấy IP nguồn chính xác là `10.244.1.5` (không bị NAT).
    

### Công dụng của CNI được chứng minh qua ví dụ

Qua luồng di chuyển trên, CNI (Flannel) đóng vai trò giải quyết 4 bài toán lớn mà Linux mặc định không tự xử lý được trong môi trường Kubernetes:

1. **IPAM (IP Address Management - Cấp phát IP không trùng lặp):** Flannel tự động chia nhỏ dải mạng chung của Cluster (`10.244.0.0/16`) thành các subnet riêng cho từng Node (Node 1 giữ `10.244.1.0/24`, Node 2 giữ `10.244.2.0/24`). Nếu không có CNI, các Pod trên các Node khác nhau sẽ bị trùng IP.
    
2. **Khởi tạo và quản lý hạ tầng mạng ảo:** CNI tự động tạo ra các thiết bị mạng ảo trên từng Node: cặp `veth` nối Pod ra Host, bridge `cni0`, và thiết bị đường hầm `flannel.1`.
    
3. **Đồng bộ hóa Control Plane (Routing/FDB/ARP Tables):** Daemon `flanneld` liên tục lắng nghe Kubernetes API Server. Khi Node 2 xuất hiện hoặc Pod B được tạo, `flanneld` trên Node 1 tự động cập nhật bảng định tuyến và bảng FDB. Nếu không có CNI, Kernel Node 1 sẽ không biết phải gửi gói tin cho `10.244.2.8` đi đâu.
    
4. **Đảm bảo nguyên tắc "Flat Network" của Kubernetes:** CNI giúp Pod A giao tiếp trực tiếp với Pod B bằng IP thực của nó xuyên qua môi trường mạng L3 mà **không cần thông qua SNAT/DNAT**, giúp ứng dụng giữ nguyên được danh tính (Source IP) của client khi xử lý request.

## Flannel có mode khác ngoài VXLAN ?

**Có**, ngoài VXLAN, Flannel hỗ trợ nhiều chế độ backend khác để tối ưu theo hiệu năng, hạ tầng mạng hoặc yêu cầu bảo mật.

### 1. `host-gw` (Host Gateway)

- **Loại:** Direct Routing (Underlay — **Không** đóng gói gói tin).
    
- **Cơ chế:** Flannel không dùng tunnel mà tự động thêm đường dẫn (route) trực tiếp vào bảng định tuyến kernel của từng Node. Khi Pod A gửi gói tin tới Pod B, Node 1 chỉ cần chuyển tiếp gói tin IP gốc tới IP của Node 2 làm Next Hop.
    
- **Ưu điểm:** **Hiệu năng cao nhất** (tương đương tốc độ phần cứng/wire-speed), độ trễ (latency) thấp nhất và tốn ít CPU do loại bỏ bước bọc/gỡ VXLAN header.
    
- **Điều kiện:** **Tất cả các Worker Node bắt buộc phải nằm trên cùng một Layer 2 Subnet/VLAN** (có thể thấy MAC Address của nhau qua ARP).
    

### 2. `wireguard`

- **Loại:** Encrypted Overlay (Mạng phủ có mã hóa).
    
- **Cơ chế:** Tận dụng mô-đun WireGuard trong Linux Kernel để đóng gói và **mã hóa (encryption)** toàn bộ lưu lượng giao tiếp giữa các Pod ở khác Node.
    
- **Ưu điểm:** Đảm bảo an toàn tuyệt đối cho lưu lượng nội bộ (Zero Trust), cấu hình đơn giản hơn so với IPsec.
    
- **Nhược điểm:** Tốn thêm tài nguyên CPU cho việc mã hóa và giải mã dữ liệu trên mỗi gói tin.
    

### 3. Tích hợp Cloud Provider (`aws-vpc`, `gce`, `aliyun`)

- **Loại:** Native Cloud Routing.
    
- **Cơ chế:** Daemon Flannel giao tiếp trực tiếp với API của Cloud Provider (AWS, GCP, Alibaba Cloud) để tự động cập nhật dải IP của Pod (Pod CIDR) vào Bảng định tuyến (VPC Route Table) của hạ tầng Cloud.
    
- **Ưu điểm:** Không cần Overlay network, gói tin đi trực tiếp qua hạ tầng mạng gốc của nhà cung cấp Cloud.
    

### 4. `alloc` (Allocation Only)

- **Loại:** Chỉ quản lý IP.
    
- **Cơ chế:** Flannel chỉ thực hiện nhiệm vụ chia nhỏ dải IP (Subnet) cho từng Node lưu vào etcd, **hoàn toàn không can thiệp vào quá trình chuyển tiếp dữ liệu**.
    
- **Ứng dụng:** Dùng khi bạn đã có một cơ chế định tuyến bên ngoài (ví dụ: Router vật lý chạy BGP hoặc phần mềm SDN khác) đảm nhận việc chuyển tiếp gói tin giữa các Node.
    

### 5. `udp` (Legacy / Không còn khuyến nghị)

- **Loại:** User-space Overlay.
    
- **Cơ chế:** Sử dụng thiết bị mạng ảo TAP/TUN (`/dev/net/tun`) để bọc gói tin UDP ở tầng **User-space**.
    
- **Nhược điểm:** Hiệu năng rất chậm do phải chuyển đổi ngữ cảnh (Context Switch) liên tục giữa Kernel space và User space cho mỗi gói tin. Hiện tại chế độ này đã lỗi thời và chỉ dùng cho mục đích debug hoặc trên các hệ điều hành rất cũ không hỗ trợ VXLAN.
    

### Bảng so sánh nhanh các Mode phổ biến

|Tiêu chí|`vxlan` (Mặc định)|`host-gw`|`wireguard`|
|---|---|---|---|
|**Loại mạng**|Overlay (L3)|Underlay (L2)|Encrypted Overlay|
|**Bọc gói tin**|Có (VXLAN/UDP)|**Không**|Có (WireGuard)|
|**Hiệu năng**|Khá|**Rất cao**|Khá (Tốn CPU mã hóa)|
|**Mã hóa dữ liệu**|Không|Không|**Có**|
|**Yêu cầu hạ tầng**|Chạy được trên mọi hạ tầng L3|Cùng 1 L2 Subnet|Kernel hỗ trợ WireGuard|