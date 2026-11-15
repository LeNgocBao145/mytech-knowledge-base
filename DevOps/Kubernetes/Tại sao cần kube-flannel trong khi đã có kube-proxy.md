Dù cả hai đều liên quan đến mạng (Networking), **`kube-flannel` ([[CNI]] Plugin)** và **`kube-proxy`** giải quyết **hai bài toán hoàn toàn khác nhau** ở các lớp mạng khác nhau.

#### Sự khác biệt cốt lõi:

| **Tiêu chí**      | **kube-flannel ([[CNI]] Plugin)**                                                                    | **kube-proxy**                                                                   |
| ----------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Vai trò chính** | Mạng giữa các Pod (**Pod-to-Pod Networking**)                                                        | Định tuyến cho Service (**Service Networking**)                                  |
| **Mục tiêu**      | Cung cấp địa chỉ IP cho Pod và tạo mạng phẳng (Flat Overlay Network) kết nối tất cả các Worker Node. | Điều hướng lưu lượng từ IP ảo của Service (ClusterIP) đến đúng các Pod phía sau. |
| **Phạm vi**       | Lớp mạng vật lý/ảo giữa các Node (L3 Overlay Network).                                               | Bảng điều hướng `iptables` / `ipvs` trên từng Node.                              |

#### Chi tiết bài toán từng thành phần giải quyết:

#### **Tại sao có `kube-proxy` là CHƯA ĐỦ?**

1. **`kube-proxy` chỉ quản lý IP ảo của Service:** `kube-proxy` theo dõi các đối tượng Service và Endpoints. Nó tạo ra các quy tắc `iptables` hoặc `IPVS` trên từng Node để khi có traffic gửi tới **Service IP** (ví dụ: `10.96.0.10`), nó sẽ chuyển tiếp (NAT) traffic đó đến **Pod IP** tương ứng.
    
2. **Hạn chế:** `kube-proxy` **không hề biết** làm cách nào để gói tin đi từ Node A sang Node B nếu Pod chạy trên Node A muốn nói chuyện trực tiếp với Pod chạy trên Node B.
    

#### **`kube-flannel` làm nhiệm vụ gì?**

1. **Gán địa chỉ IP cho Pod:** Mỗi khi một Pod mới được tạo, [[CNI]] như Flannel sẽ cấp cho Pod đó một địa chỉ IP duy nhất trong toàn cụm.
    
2. **Kết nối mạng giữa các Node:** Flannel thiết lập một mạng ảo (Overlay Network - thường dùng VXLAN). Khi Pod ở **Node A** gửi dữ liệu trực tiếp tới Pod ở **Node B**:
    
    - Flannel đóng gói gói tin của Pod vào trong gói tin mạng thực của Node.
        
    - Gửi gói tin sang Node B qua hạ tầng mạng vật lý.
        
    - Flannel ở Node B mở gói tin ra và chuyển tiếp chính xác vào Pod đích.