**`nicolaka/netshoot`** là một Docker image chuyên dụng chứa bộ công cụ chẩn đoán và xử lý sự cố mạng (network troubleshooting) cực kỳ phổ biến trong môi trường **Kubernetes** và **Container/Docker**.

Nó được ví như một **"chiếc hộp công cụ mạng" (Network Swiss Army Knife)** cho các DevOps, SysAdmin, và Kubernetes Engineer.

### Các trường do/Mục đích sử dụng chính

1. **Khắc phục sự cố trong Container tối giản (Distroless / Scratch):**
    
    - Các container ứng dụng sản xuất (production) thường được đóng gói rất nhỏ (dùng Alpine, Distroless) và bị lược bỏ hoàn toàn các lệnh như `curl`, `ping`, `netstat`, `dig` để tối ưu dung lượng và tăng tính bảo mật.
        
    - Khi ứng dụng không kết nối được DB hoặc microservice khác, bạn không thể cài công cụ trực tiếp vào đó. Giải pháp là chạy `netshoot` dưới dạng một Ephemeral Container (container tạm thời) đính kèm vào network namespace của container lỗi để kiểm tra.
        
2. **Chẩn đoán mạng trong Kubernetes (Debug Pod):**
    
    - Dùng lệnh `kubectl debug` để đẩy một pod `netshoot` vào chung mạng với Pod bị lỗi nhằm soi traffic, kiểm tra DNS resolution, hoặc test cổng kết nối.
        
3. **Kiểm tra kết nối giữa các node và Service:**
    
    - Kiểm tra xem CNI (Flannel, Calico, Cilium), Ingress Controller, hoặc Service Mesh (Istio, Linkerd) có đang routing đúng traffic không.
        

### Các công cụ nổi bật được tích hợp sẵn bên trong

`netshoot` bao gồm hầu hết tất cả công cụ mạng mạnh mẽ nhất hiện nay:

|Nhóm công cụ|Công cụ tích hợp sẵn|
|---|---|
|**Kiểm tra kết nối & Cổng (Port)**|`curl`, `wget`, `ping`, `netcat` (`nc`), `socat`, `nmap`, `telnet`|
|**Phân tích DNS**|`dig`, `nslookup`, `host`, `drill`|
|**Soi & Bắt gói tin (Packet Capture)**|`tcpdump`, `tshark` (Wireshark CLI), `ngrep`|
|**Routing & Giao diện mạng**|`iproute2` (`ip`), `traceroute`, `mtr`, `iftop`, `iperf3`|
|**Công cụ chuyên dụng K8s/Cloud**|`calicoctl`, `etcdctl`, `openssl`, `ebpf` tools|

### Lệnh sử dụng thực tế phổ biến

- **Khởi chạy nhanh trên Docker để test:**
    
    Bash
    
    ```
    docker run -it --rm nicolaka/netshoot
    ```
    
- **Chạy đính kèm vào một Docker Container đang bị lỗi mạng:**
    
    Bash
    
    ```
    docker run -it --rm --net=container:<container_name_or_id> nicolaka/netshoot
    ```
    
- **Debug một Pod trong Kubernetes bằng `kubectl`:**
    
    Bash
    
    ```
    kubectl debug -it <pod-name> --image=nicolaka/netshoot --target=<container-name>
    ```

## nicolaka/netshoot sao chạy các lệnh curl, ... trong pod chạy app production hay vậy?

Lý do `nicolaka/netshoot` có thể chạy các lệnh như `curl`, `tcpdump`, `ping` để kiểm tra container ứng dụng production mà **không cần cài trực tiếp công cụ nào vào container ứng dụng** là nhờ vào cơ chế **Chia sẻ Namespace (Network & Process Namespace)** của Linux Kernel và Docker/Kubernetes.

### Nguyên lý cốt lõi: Linux Namespaces

Trong Linux, Container thực chất chỉ là một tiến trình (process) thông thường nhưng được cô lập bằng các **Namespaces**:

- **Network Namespace (netns):** Quản lý IP, bảng routing, các cổng network interface.
    
- **PID Namespace:** Quản lý các tiến trình (Process ID).
    
- **Mount Namespace:** Quản lý hệ thống file (Filesystem).
    

Các công cụ mạng như `curl`, `tcpdump`, `ping` hoạt động bằng cách tương tác với **Network Namespace** (địa chỉ IP và các cổng mạng), chứ chúng không phụ thuộc vào hệ thống file hay môi trường của container ứng dụng.

### 2 Cơ chế triển khai thực tế

#### 1. Cơ chế trong Kubernetes (Ephemeral Containers / Debug Pod)

Khi bạn dùng lệnh:

Bash

```
kubectl debug -it <pod-app> --image=nicolaka/netshoot --target=<container-app>
```

- Trong Kubernetes, tất cả các container trong cùng 1 Pod mặc định **dùng chung 1 Network Namespace** (thông qua `pause` container).
    
- Khi lệnh `kubectl debug` bơm `netshoot` vào Pod, container `netshoot` khởi chạy và gia nhập thẳng vào Network Namespace của Pod đó.
    
- `netshoot` mang sẵn binary của `curl`, `tcpdump`, `dig` trong hệ thống file của chính nó, nhưng khi thực thi lệnh, các gói tin lại đi qua đúng IP, interface và bảng routing của Pod production.
    

#### 2. Cơ chế trong Docker (`--net=container:...`)

Khi bạn chạy lệnh:

```
docker run -it --rm --net=container:<app_container_id> nicolaka/netshoot
```

- Tham số `--net=container:<id>` bảo Linux Kernel gán container `netshoot` vào **chính xác Network Namespace** của container ứng dụng.
    
- Container ứng dụng vẫn giữ nguyên tính chất siêu nhẹ (Distroless/Alpine không có `curl`), còn `netshoot` đóng vai trò là "chiếc hộp công cụ bên ngoài" được cắm chung vào cùng một đường ống mạng đó.
    

### Tóm tắt dễ hiểu

Container ứng dụng giống như một **căn phòng bị khóa cửa** và chỉ có ứng dụng chạy bên trong. `netshoot` không đi qua cửa chính để cài phần mềm vào phòng (không đụng vào Filesystem của app), mà nó **đấu nhờ đường dây điện và dây mạng** của căn phòng đó vào một hộp công cụ của riêng nó.

Nhờ vậy, bạn vừa chẩn đoán được mạng của ứng dụng production, vừa giữ cho container production siêu sạch và an toàn.