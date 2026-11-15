**Falco** là một công cụ phân tích hành vi (behavioral analysis) và phát hiện mối đe dọa theo thời gian thực (real-time threat detection) mã nguồn mở, được thiết kế chuyên dụng cho môi trường **Linux, Container, Docker và Kubernetes**.

Nó ban đầu được phát triển bởi Sysdig và hiện tại là một dự án đã tốt nghiệp (Graduated Project) thuộc tổ chức **CNCF** (Cloud Native Computing Foundation).

### Cách Falco hoạt động (Cơ chế bên dưới)

Trong khi các hệ thống giám sát khác soi traffic mạng (như Wireshark) hoặc phân tích log file tĩnh, Falco hoạt động bằng cách **lắng nghe trực tiếp các Lệnh gọi Hệ thống (System Calls - syscalls)** ở cấp độ **Linux Kernel**:

1. **Gắn hook vào Linux Kernel:** Falco sử dụng công nghệ **eBPF** (hoặc kernel module) để giám sát mọi luồng lệnh gọi hệ thống (mở file, tạo process, thiết lập kết nối mạng, thay đổi quyền root...).
    
2. **Khớp luật hành vi (Rule Engine):** Falco so sánh các syscalls này với bộ quy tắc hành vi (Ruleset) được định nghĩa trước.
    
3. **Phát ra Cảnh báo (Alerting):** Nếu một tiến trình thực hiện hành vi bất thường/đáng ngờ, Falco sẽ ngay lập tức bắn cảnh báo (qua Slack, Datadog, Webhook, PagerDuty...) hoặc kích hoạt phản ứng tự động (như cô lập hoặc xóa Pod bị nhiễm).
    

Plaintext

```
[ Linux Kernel (Syscalls) ] ──(eBPF / Kernel Module)──> [ Falco Rules Engine ] ──> [ Alerts / Actions ]
```

### Mấy ví dụ thực tế về hành vi Falco phát hiện được

Falco giúp trả lời câu hỏi: _"Có ai hoặc tiến trình nào đang làm điều bất thường trong container của tôi không?"_

- **Chạy Shell bất hợp pháp:** Một ai đó `exec` vào container production và mở shell (`bash` hoặc `sh`).
    
- **Sửa đổi thư mục nhạy cảm:** Tiến trình trong container cố gắng ghi/sửa đổi các file hệ thống như `/etc/shadow`, `/etc/passwd` hoặc cài lại package (`apt-get`).
    
- **Hành vi bất thường của Process:** Một web server (`nginx` hoặc `node`) đột nhiên sinh ra một tiến trình lạ như `netcat`, `curl` để tải script đào tiền ảo.
    
- **Đọc file cấu hình bảo mật:** Container ứng dụng đọc SSH keys, Kubelet credentials hoặc TLS certificates ngoài thẩm quyền.
    

### So sánh nhanh: Falco vs Envoy / Istio / CNI

|**Công cụ**|**Vị trí giám sát**|**Mục đích chính**|
|---|---|---|
|**Falco**|**Linux Kernel (Syscalls)**|Phát hiện xâm nhập & phân tích hành vi bất thường (Runtime Security).|
|**Envoy / Istio**|**Layer 7 (HTTP/gRPC Traffic)**|Routing, mTLS, Load Balancing giữa các service (Service Mesh).|
|**Cilium / Calico**|**Layer 3 / Layer 4 (Network Packets)**|Phân chia phân vùng mạng (Network Policy) & Mã hóa traffic.|
