**Containers (bao gồm cả Minikube và Pod bên trong) không có Kernel riêng mà dùng chung Kernel với Linux Host (WSL2)**, trong khi AppArmor lại là một mô-đun bảo mật nằm ở tầng **Kernel**.

Dưới đây là cơ chế chi tiết giải thích tại sao cách làm này hoạt động:

### 1. Tất cả đều dùng chung 1 Linux Kernel của WSL2

Khi bạn chạy Minikube bằng driver Docker:

- WSL2 chạy một Linux Kernel thật trên máy tính của bạn.
    
- Minikube chỉ là một **Docker Container** chạy trên WSL2.
    
- Các Pod trong Kubernetes lại là các **Container con** chạy bên trong Minikube (Container-in-Container).
    

Dù có bao nhiêu lớp cô lập (Namespaces, cgroups), **tất cả các tiến trình (processes) của Pod cuối cùng vẫn do chính Linux Kernel của WSL2 quản lý và điều phối execution**.

### 2. AppArmor hoạt động ở tầng Kernel (Kernel-space)

AppArmor là một **Linux Security Module (LSM)**.

- Khi bạn cài `apparmor-utils` và chạy lệnh `apparmor_parser` ở ngoài WSL2, lệnh này sẽ biên dịch profile thành dạng bytecode và **nạp trực tiếp vào bộ nhớ của WSL2 Kernel**.
    
- Vì profile đã nằm trong Kernel, toàn bộ các tiến trình đang chạy trên Kernel đó (dù là tiến trình trên WSL2 hay tiến trình bên trong Pod của Minikube) đều có thể truy xuất và áp dụng được profile này.
    

### 3. Cách Container Runtime kết nối Profile với Pod

Quy trình từ lúc nạp Profile đến khi Pod bị chặn diễn ra như sau:

1. **Nạp Profile (ở WSL2):** Bạn chạy `apparmor_parser` $\rightarrow$ WSL2 Kernel ghi nhận trong bộ nhớ: _"Ta đã có một luật bảo mật tên là `k8s-deny-ping`"_.
    
2. **Khai báo Pod:** Bạn apply file YAML với khai báo `localhostProfile: k8s-deny-ping`.
    
3. **Thực thi:**
    
    - Kubernetes truyền yêu cầu này xuống Container Runtime (`runc` / `containerd`).
        
    - Khi khởi tạo tiến trình cho container `network-call`, `runc` gọi một System Call bảo Kernel: _"Hãy gắn AppArmor profile `k8s-deny-ping` cho tiến trình này"_.
        
    - Kernel kiểm tra danh sách profile trong bộ nhớ $\rightarrow$ Thấy `k8s-deny-ping` tồn tại $\rightarrow$ Áp đặt các luật (như cấm `net_raw`) lên tiến trình của Pod.

## Vậy nếu là trên k8s cluster thì mình ssh vào master node cài ?

Không chỉ riêng Master Node, mà quan trọng nhất là bạn bắt buộc phải nạp AppArmor Profile vào Kernel của **tất cả các Worker Node** (nơi Pod trực tiếp khởi tạo và chạy).

Nếu bạn chỉ cài Profile trên Master Node mà Pod lại được Kubernetes Scheduler xếp lịch chạy trên Worker Node 1, Kernel của Worker Node 1 sẽ không tìm thấy Profile và Pod sẽ bị hủy với lỗi `AppArmor profile not present` hoặc `ContainerCannotRun`.

Trong môi trường Kubernetes Cluster thực tế (Production), người ta không SSH thủ công vào từng Node để cài. Thay vào đó, có 3 cách triển khai tiêu chuẩn sau:

### Cách 1: Dùng Kubernetes DaemonSet (Tự động hóa dễ nhất)

Bạn tạo một `DaemonSet` để Kubernetes tự động chạy 1 Pod đặc quyền (`privileged: true`) lên **mọi Node trong cluster** (bao gồm cả Worker Nodes và Master Nodes). Pod này sẽ copy file profile và thực thi `apparmor_parser` nạp trực tiếp vào Kernel của từng Node.

Ví dụ file `apparmor-daemonset.yaml`:

YAML

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: apparmor-loader
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: apparmor-loader
  template:
    metadata:
      labels:
        name: apparmor-loader
    spec:
      # Cho phép DaemonSet chạy trên cả Master Node nếu cần
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: apparmor-loader
        image: debian:bookworm-slim
        command: ["/bin/sh", "-c"]
        args:
        - |
          apt-get update && apt-get install -y apparmor
          
          # 1. Ghi file Profile
          cat << 'EOF' > /etc/apparmor.d/k8s-deny-ping
          #include <tunables/global>
          profile k8s-deny-ping flags=(attach_disconnected,mediate_deleted) {
            #include <abstractions/base>
            deny network raw,
            deny capability net_raw,
          }
          EOF
          
          # 2. Nạp Profile vào Kernel của Host Node
          apparmor_parser -r -W /etc/apparmor.d/k8s-deny-ping
          echo "AppArmor profiles loaded successfully on node!"
          
          # Giữ container chạy ngầm
          sleep infinity
        securityContext:
          privileged: true
        volumeMounts:
        - name: sys
          mountPath: /sys
        - name: apparmord
          mountPath: /etc/apparmor.d
      volumes:
      - name: sys
        hostPath:
          path: /sys
      - name: apparmord
        hostPath:
          path: /etc/apparmor.d
```

### Cách 2: Dùng Security Profiles Operator (Chuẩn Enterprise / Cloud-Native)

Kubernetes có dự án chính thức tên là **Security Profiles Operator (SPO)** chuyên dùng để quản lý AppArmor, Seccomp, và SELinux trong cluster thông qua Custom Resources (CRDs).

1. Cài đặt Operator vào Cluster:
    
    
    ```
    kubectl apply -f https://github.com/kubernetes-sigs/security-profiles-operator/releases/download/v0.8.2/security-profiles-operator.yaml
    ```
    
2. Khai báo AppArmor Profile bằng file YAML như một tài nguyên K8s bình thường:
    
    
    ```
    apiVersion: security-profiles-operator.x-k8s.io/v1alpha1
    kind: AppArmorProfile
    metadata:
      name: k8s-deny-ping
      namespace: default
    spec:
      profile: |
        #include <tunables/global>
        profile k8s-deny-ping flags=(attach_disconnected,mediate_deleted) {
          #include <abstractions/base>
          deny network raw,
          deny capability net_raw,
        }
    ```
    
3. Operator sẽ tự động đồng bộ (sync) và nạp Profile này xuống Kernel của toàn bộ các Node trong Cluster mà bạn không cần đụng đến SSH hay DaemonSet thủ công.
    

### Cách 3: Nạp sẵn từ bước Khởi tạo Node (Ansible / Cloud-Init / Packer)

Nếu bạn quản lý hạ tầng bằng các công cụ như Ansible, Terraform hoặc Cloud-Init:

- Bạn thêm bước đẩy file `/etc/apparmor.d/k8s-deny-ping` và chạy `apparmor_parser -r -W` vào playbook/script khởi tạo Worker Node.
    
- Mỗi khi một Worker Node mới được scale-out (thêm vào cluster), nó đã có sẵn AppArmor Profile trong Kernel ngay trước khi gia nhập (join) vào Kubernetes Cluster.
    

### Tóm lại

- **Không nên** SSH vào từng máy để nạp thủ công nếu cluster có nhiều node.
    
- **Master Node** chỉ quản lý API server, **Worker Node** mới là nơi chạy Pod.
    
- Dùng **DaemonSet** (dành cho dự án nhỏ/vừa) hoặc **Security Profiles Operator** (dành cho dự án lớn) để phân phối Profile tự động đến mọi Worker Node.

## Trong yaml của Cách 1: Dùng Kubernetes DaemonSet sao bạn lại mount 2 path đó và trong script tôi đâu thấy có dùng tới /sys đâu mà lại mount

Lý do bắt buộc phải mount 2 thư mục này liên quan trực tiếp đến cơ chế giao tiếp giữa công cụ `apparmor_parser` và **Kernel của Host (Worker Node)**.

### 1. Tại sao cần mount `/sys` (Dù trong script không gõ chữ `/sys`)?

Mặc dù trong câu lệnh shell bạn không gõ trực tiếp chữ `/sys`, nhưng **công cụ `apparmor_parser` chạy ngầm bên dưới bắt buộc phải ghi dữ liệu vào `/sys`**.

- **Giao diện SecurityFS của Kernel:** Khi AppArmor hoạt động, Linux Kernel sẽ mở một hệ thống file ảo gọi là SecurityFS tại đường dẫn **`/sys/kernel/security/apparmor/`**.
    
- **Cách `apparmor_parser` nạp profile:** Khi bạn chạy `apparmor_parser -r /etc/apparmor.d/k8s-deny-ping`, công cụ này thực hiện 2 bước:
    
    1. Biên dịch file cấu hình dạng văn bản thành dạng nhị phân (bytecode).
        
    2. Ghi nhị phân đó trực tiếp vào file ảo **`/sys/kernel/security/apparmor/.load`** (hoặc `.replace`) để nạp vào RAM của Host Kernel.
        

> **Nếu KHÔNG mount `/sys` của Host vào Container:** Container sẽ nhìn thấy một thư mục `/sys` ảo bị cô lập hoàn toàn. `apparmor_parser` không thể tìm thấy giao diện SecurityFS của Host Kernel và sẽ thất bại ngay lập tức với lỗi: `AppArmor filesystem not mounted` hoặc `Can't talk to AppArmor kernel interface`.

### 2. Tại sao cần mount `/etc/apparmor.d`?

Thư mục `/etc/apparmor.d` trên Host là nơi lưu trữ các file cấu hình AppArmor dạng văn bản.

- **Ghi file xuống ổ đĩa của Host:** Lệnh `cat << 'EOF' > /etc/apparmor.d/k8s-deny-ping` bên trong container sẽ ghi trực tiếp file cấu hình xuống ổ đĩa vật lý của Worker Node.
    
- **Tự động nạp lại khi Node Reboot:** Nếu Worker Node bị khởi động lại (RAM bị xóa sạch), dịch vụ `apparmor.service` trên Node OS sẽ tự động quét thư mục `/etc/apparmor.d/` này để nạp lại toàn bộ profile mà không làm mất cấu hình.
    

**Tóm lại:**

- Mount **`/etc/apparmor.d`** → Để lưu file cấu hình xuống **ổ đĩa của Host Node** (Lưu trữ vĩnh viễn).
    
- Mount **`/sys`** → Cho phép `apparmor_parser` mở "cánh cổng" gửi bytecode trực tiếp vào **RAM của Host Kernel** (Kích hoạt tức thì).

## Cách 1 chỉ cần cài thôi mà sao lại để nó chạy sleep infinite chi để mà tốn tài nguyên vậy?

Thắc mắc của bạn rất chính xác về mặt tư duy tối ưu hệ thống. Việc để một container chạy lệnh `sleep infinity` nhìn qua có vẻ thừa thãi, nhưng đây là **mẫu thiết kế bắt buộc (design pattern)** do cơ chế quản lý vòng đời (Lifecycle) của Kubernetes DaemonSet quy định.

Dưới đây là 3 lý do kỹ thuật giải thích tại sao người ta lại làm vậy:

### 1. Quy tắc của DaemonSet: `restartPolicy` luôn là `Always`

- Trong Kubernetes, một `DaemonSet` yêu cầu `spec.template.spec.restartPolicy` bắt buộc phải là `Always` (không thể đổi thành `OnFailure` hay `Never` như đối với K8s `Job`).
    
- Nếu script nạp AppArmor chạy xong và **thoát (`exit 0`)**, Kubernetes sẽ hiểu rằng container đã bị dừng và ngay lập tức **khởi động lại container đó liên tục**.
    
- Việc container bị restart liên tục (CrashLoopBackOff hoặc Completed -> Restarting) sẽ làm **tốn CPU/RAM và rác log** hơn rất nhiều so với việc để nó nằm ngủ với `sleep`.
    

### 2. Tự động nạp lại khi Node reboot & Báo cáo trạng thái (Health Check)

- **Khi Node khởi động lại:** AppArmor Profile lưu trong bộ nhớ RAM của Kernel sẽ bị mất. Việc giữ DaemonSet luôn ở trạng thái `Running` đảm bảo rằng mỗi khi Node reboot, DaemonSet sẽ tự động chạy lại và nạp lại Profile vào Kernel.
    
- **Đồng bộ trạng thái:** Kubernetes dùng số lượng Pod `Running` của DaemonSet để biết được bao nhiêu Node trong cluster đã được bảo vệ/nạp profile thành công (ví dụ: `DESIRED: 5, CURRENT: 5, READY: 5`).
    

### 3. Tiến trình `sleep` thực tế tốn bao nhiêu tài nguyên?

Tiến trình `sleep` trong Linux đi vào trạng thái **Deep Sleep (S - Interruptible Sleep)**:

- **CPU:** Tốn **0%**. CPU không hề tốn bất kỳ chu kỳ xử lý nào cho lệnh này.
    
- **RAM:** Chỉ tốn khoảng **1MB - 3MB** cho tiến trình shell cơ bản.
    

### 💡 Giải pháp tối ưu hơn nếu bạn không muốn tốn RAM

Nếu bạn vẫn muốn tiết kiệm tối đa RAM hoặc muốn làm chuẩn thiết kế Kubernetes hơn, có 2 cách tối ưu sau:

#### Cách A: Dùng Init Container + Pause Container (Tối ưu RAM xuống vài KB)

Tách phần cài đặt vào `initContainers`. Sau khi nạp xong, container chính chỉ dùng image `registry.k8s.io/pause` (đây là image chính thức của K8s, không làm gì cả và chỉ tốn vài KB RAM):

YAML

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: apparmor-loader
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: apparmor-loader
  template:
    metadata:
      labels:
        name: apparmor-loader
    spec:
      # Bước 1: Init Container chạy 1 lần duy nhất để nạp Profile
      initContainers:
      - name: install-profile
        image: debian:bookworm-slim
        command: ["/bin/sh", "-c"]
        args:
        - |
          apt-get update && apt-get install -y apparmor
          cat << 'EOF' > /etc/apparmor.d/k8s-deny-ping
          #include <tunables/global>
          profile k8s-deny-ping flags=(attach_disconnected,mediate_deleted) {
            #include <abstractions/base>
            deny network raw,
            deny capability net_raw,
          }
          EOF
          apparmor_parser -r -W /etc/apparmor.d/k8s-deny-ping
        securityContext:
          privileged: true
        volumeMounts:
        - name: sys
          mountPath: /sys
        - name: apparmord
          mountPath: /etc/apparmor.d

      # Bước 2: Container chính giữ DaemonSet ở trạng thái Running (gần như 0 RAM)
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
        resources:
          limits:
            cpu: 1m
            memory: 2Mi
          requests:
            cpu: 1m
            memory: 2Mi

      volumes:
      - name: sys
        hostPath:
          path: /sys
      - name: apparmord
        hostPath:
          path: /etc/apparmor.d
```

#### Cách B: Nạp thẳng bằng Systemd Service trên Node OS (Không dùng K8s Pod)

Nếu bạn quản lý Node bằng Ansible/Cloud-init, hãy tạo 1 file Systemd Service trên hệ điều hành của Node. Service dạng `Type=oneshot` sẽ chạy đúng 1 lần khi Node khởi động rồi tự kết thúc hoàn toàn mà không để lại tiến trình nào chạy ngầm.

