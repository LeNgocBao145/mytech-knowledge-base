**Flannel manifest** là một hoặc nhiều file cấu hình YAML chứa toàn bộ khai báo tài nguyên (resources) cần thiết để cài đặt và triển khai **Flannel** — một giải pháp mạng [[CNI]] (Container Network Interface) rất phổ biến và siêu nhẹ — vào trong cụm Kubernetes.

Nói một cách đơn giản, đó là "bản thiết kế" (blueprint) bằng mã lệnh để Kubernetes biết cách dựng hệ thống mạng Flannel lên cluster của bạn.

### 1. Trong Flannel manifest có chứa những gì?

Khi bạn dùng lệnh `kubectl apply -f kube-flannel.yml`, Kubernetes sẽ đọc file manifest này và tự động khởi tạo các thành phần cốt lõi:

- **DaemonSet (`kube-flannel-ds`):** Đảm bảo trên **mọi Node** (Master lẫn Worker) đều chạy 1 Pod Flannel (`flanneld`). Pod này chịu trách nhiệm quản lý dải IP cho các Pod trên Node đó và đóng gói/mở gói gói tin truyền giữa các Node (overlay network).
    
- **ConfigMap (`kube-flannel-cfg`):** Chứa cấu hình mạng chính, quan trọng nhất là:
    
    - **Network:** Dải IP chung cho toàn bộ Pod trong cluster (ví dụ: `10.244.0.0/16`).
        
    - **Backend Type:** Cơ chế đóng gói gói tin giữa các Node (mặc định thường là `vxlan`, hoặc `host-gw`, `ipsec`).
        
- **RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding):** Cấp quyền cho Flannel có thể truy cập Kubernetes API Server để đọc thông tin Node và cấu hình Pod CIDR.
    
- **CRD (Custom Resource Definitions):** Định nghĩa các tài nguyên tùy chỉnh riêng của Flannel (như `FlannelConfig` nếu dùng các bản Flannel mới).
    

### 2. File manifest này từ đâu ra và dùng khi nào?

- **Khi nào dùng:** Sau khi bạn vừa khởi tạo cụm Kubernetes (ví dụ bằng `kubeadm init`), các Node sẽ ở trạng thái `NotReady` vì chưa có [[CNI]] plugin để kết nối mạng giữa các Pod. Lúc này bạn cần apply Flannel manifest để kích hoạt mạng.
    
- **Lấy từ đâu:** Từ kho lưu trữ GitHub chính thức của dự án Flannel.
    

Lệnh cài đặt điển hình:

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 3. Ví dụ đoạn cấu hình quan trọng bên trong Manifest

Đoạn ConfigMap trong file manifest thường quy định dải IP mạng cho Pod:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
data:
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

> **Lưu ý quan trọng:** Dải IP trong `Network` (`10.244.0.0/16`) phải **trùng khớp** với dải `--pod-network-cidr` mà bạn đã khai báo khi khởi tạo cụm Kubernetes với `kubeadm init`. Nếu khác nhau, Pod trên các Node sẽ không thể giao tiếp được.