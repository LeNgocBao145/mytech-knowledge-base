**Software Bill of Materials (SBOM)** — dịch nôm na là **Danh mục Thành phần Phần mềm** — là một bản kê chi tiết, toàn diện về tất cả các thành phần, thư viện (libraries), module mã nguồn mở (open-source), phụ thuộc (dependencies) và công cụ được sử dụng để tạo nên một ứng dụng phần mềm.

Nếu so sánh phần mềm với một sản phẩm đóng gói (như thực phẩm hay thuốc), SBOM chính là **bảng thành phần & dinh dưỡng** in trên bao bì.

### 1. Tại sao SBOM lại quan trọng?

Trong phát triển phần mềm hiện đại, các lập trình viên hiếm khi viết mọi thứ từ đầu. Khoảng **70%–90%** mã nguồn của một ứng dụng modern được xây dựng từ các thư viện mã nguồn mở có sẵn. SBOM giúp trả lời 3 câu hỏi cốt lõi về bảo mật:

- **Minh bạch (Transparency):** Biết chính xác phần mềm của bạn (hoặc phần mềm bạn mua) đang chứa những gì bên trong.
    
- **Quản lý lỗ hổng bảo mật (Vulnerability Management):** Khi một lỗ hổng mới bùng nổ (ví dụ: thảm họa _Log4j_ năm 2021), SBOM giúp doanh nghiệp quét và biết ngay lập tức hệ thống nào của mình đang dùng thư viện bị lỗi để vá kịp thời, thay vì mất nhiều tuần rà soát thủ công.
    
- **Tcompliance & Giấy phép (License Compliance):** Kiểm tra xem các thư viện mã nguồn mở đang dùng có vi phạm bản quyền hoặc xung đột giấy phép kinh doanh (như GPL, MIT, Apache) hay không.
    

### 2. SBOM chứa những thông tin gì?

Theo tiêu chuẩn tối thiểu của cơ quan quản lý (như NTIA - Bộ Thương mại Mỹ), một SBOM tiêu chuẩn cần ghi nhận các thông tin chính:

1. **Supplier Name:** Tên nhà cung cấp / tác giả của thành phần.
    
2. **Component Name:** Tên thư viện / phần mềm phụ thuộc.
    
3. **Version:** Phiên bản cụ thể đang sử dụng.
    
4. **Other Identifiers:** Mã định danh duy nhất (ví dụ: CPE, PURL).
    
5. **Dependency Relationship:** Mối quan hệ phụ thuộc (thư viện A gọi thư viện B, thư viện B gọi C).
    
6. **Author of SBOM Data:** Người / công cụ tạo ra bản SBOM này.
    
7. **Timestamp:** Thời gian tạo danh mục.
    

### 3. Các định chuẩn SBOM phổ biến

Để máy tính và các công cụ tự động có thể đọc và phân tích SBOM, dữ liệu được xuất ra dưới các định dạng chuẩn hóa:

|**Định dạng**|**Mô tả chính**|**Tổ chức quản lý**|
|---|---|---|
|**SPDX** _(Software Package Data Exchange)_|Định dạng chuẩn quốc tế (ISO/IEC 5962:2021), phổ biến cho quản lý bản quyền & bảo mật.|Linux Foundation|
|**CycloneDX**|Thiết kế riêng cho an ninh mạng, DevSecOps và phân tích chuỗi cung ứng phần mềm.|OWASP Foundation|
|**SWID Tag** _(Software Identification)_|Thẻ định danh phần mềm tự động phát hiện các tệp đã cài đặt trên thiết bị.|ISO/IEC 19770-2|

### 4. SBOM hoạt động như thế nào trong DevSecOps?

Trong quy trình tự động hóa (CI/CD Pipeline):

1. **Tạo SBOM tự động:** Khi lập trình viên build code, các công cụ (như _Syft_, _Trivy_, _Cdxgen_) sẽ tự động quét file phụ thuộc (như `package.json`, `pom.xml`, `requirements.txt`) để xuất ra file SBOM (JSON hoặc XML).
    
2. **Kiểm tra lỗ hổng:** File SBOM được đối chiếu tự động với cơ sở dữ liệu lỗ hổng toàn cầu (NVD, CVE database).
    
3. **Cảnh báo & Chặn:** Nếu phát hiện thư viện chứa mã độc hoặc lỗ hổng nghiêm trọng, hệ thống CI/CD sẽ chặn quá trình triển khai (deploy) cho đến khi được vá.

## SBOM quét layer trong image hay binary?

**SBOM có thể quét cả Layer trong Container Image lẫn File Binary**, nhưng **cách thức quét, độ sâu và thông tin thu được ở hai đối tượng này hoàn toàn khác nhau**.

Dưới đây là chi tiết so sánh cách SBOM xử lý với từng loại:

### 1. Quét Container Image (Layer-by-Layer)

Container Image (như Docker Image) thực chất là **tập hợp của nhiều tầng (layers) nén lên nhau**. Khi tạo SBOM cho Image, các công cụ quét (như _Syft_, _Trivy_, _Grype_) sẽ soi vào từng layer:

- **Cách thức:** Công cụ giải nén từng Layer từ bottom (Base OS Image như Alpine, Ubuntu) lên top (Layer chứa application code).
    
- **Thông tin thu thập được:**
    
    - **OS Packages:** Các gói phần mềm hệ thống do trình quản lý gói (Package Manager) cài vào ở từng layer (ví dụ: `apt`, `apk`, `yum/dnf`).
        
    - **Language Packages:** Các thư viện ứng dụng được cài ở từng layer (ví dụ: `node_modules` từ `npm`, `site-packages` từ `pip`, các file `.jar`).
        
- **Điểm mạnh:** Giúp bạn phát hiện chính xác **layer nào** đã đưa lỗ hổng hoặc thư viện độc hại vào Image để sửa lại `Dockerfile` tương ứng.
    

### 2. Quét Binary (Compiled File)

Với các tệp thực thi đã được biên dịch thành mã máy (như `.exe`, file ELF trên Linux, hoặc binary biên dịch từ Go, Rust, C/C++):

- **Cách thức:** Công cụ không xem cấu trúc layer (vì binary không có layer). Thay vào đó, nó quét theo cơ chế **Static Analysis / Signature Matching**:
    
    - **Phân tích Metadata & Strings:** Tìm các chuỗi thông tin về phiên bản, bản quyền nhúng bên trong file.
        
    - **Symbols & Dependency Graph:** Đối với các ngôn ngữ có tính năng nhúng dependency info vào binary (như Go hay Rust), công cụ đọc bảng biểu tượng (symbol table) hoặc thông tin build để trích xuất danh sách thư viện.
        
- **Thông tin thu thập được:** Danh sách các thư viện mã nguồn mở đã được đóng gói (statically linked) trực tiếp vào trong tệp binary đó.
    
- **Hạn chế:** Thách thức hơn rất nhiều so với quét Image. Nếu binary bị nén (pack), bị làm mờ code (obfuscate), hoặc biên dịch từ C/C++ mà không để lại symbol (stripped binary), việc tạo SBOM sẽ kém chính xác hoặc thiếu thông tin.
    

### Bảng so sánh tổng quan

|Đặc tính|Quét Layer trong Image|Quét File Binary|
|---|---|---|
|**Phạm vi nhận diện**|Toàn bộ Hệ điều hành (OS) + Thư viện ứng dụng|Chỉ các thư viện được đóng gói bên trong file binary|
|**Độ chính xác**|Rất cao (đọc trực tiếp file manifest & package manager)|Phụ thuộc vào ngôn ngữ & thông tin symbol còn lại|
|**Độ phức tạp**|Thấp (giải nén layer và đọc cấu trúc file system)|Cao (cần phân tích tĩnh, soi chuỗi bytes, đọc ELF/PE header)|
|**Công cụ phổ biến**|Syft, Trivy, Anchore Grype, Clair|Syft, Binwalk, Ghidra, cve-bin-tool|

> **Tóm lại:** Khi làm DevSecOps hiện đại, các công cụ SBOM thường **kết hợp cả hai**: Quét từng Layer của Image để lấy bức tranh toàn cảnh về OS/Môi trường chạy, đồng thời quét sâu vào các File Binary bên trong Layer đó để phát hiện các thư viện tĩnh.

