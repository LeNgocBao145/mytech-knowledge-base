**Limits** nằm trong cơ chế quản lý tài nguyên (Resource Management) cho container, thường đi đôi với **Requests**:

```
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

- **Requests (Mức tối thiểu cam kết):** Số lượng tài nguyên (CPU/RAM) mà K8s **đảm bảo cấp** cho Pod. K8s dựa vào chỉ số này để quyết định xếp Pod vào Node nào (Scheduling).
    
- **Limits (Mức tối đa cho phép):** Ranh giới tài nguyên **tối đa** mà một Pod được phép sử dụng.
    
    - **Nếu vượt quá CPU Limit:** Container sẽ bị bóp hiệu năng (CPU Throttling) nhưng không bị kill.
        
    - **Nếu vượt quá Memory Limit:** Container sẽ bị hệ điều hành kill lập tức do hết bộ nhớ (Lỗi **OOMKilled** - Out Of Memory Killed).