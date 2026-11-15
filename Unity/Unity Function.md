## Event Function

Trong Unity, các hàm như `Awake`, `Start`, `Update`... được gọi là **Event Functions** (hoặc Unity Lifecycle Messages).

Một điểm quan trọng mà newbie rất dễ nhầm lẫn: **Bạn KHÔNG dùng từ khóa `override`** đối với các hàm này (vì chúng không phải là `virtual` method từ class cha `MonoBehaviour`). Bạn chỉ cần viết đúng tên hàm (ví dụ: `private void Awake()`), Unity sẽ tự động tìm và gọi chúng thông qua cơ chế Reflection/Native Call theo đúng vòng đời.

Dưới đây là tổng hợp các hàm quan trọng nhất trong `MonoBehaviour`, được chia theo từng giai đoạn vòng đời để bạn dễ theo dõi:

## 1. Giai đoạn Khởi tạo (Initialization)

- **`Awake()`**
    
    - **Vai trò:** Chạy **đầu tiên và duy nhất 1 lần** khi Script/GameObject được nạp vào Scene (ngay cả khi Script đang bị Disable). Dùng để khởi tạo các biến nội bộ, cài đặt Singleton, hoặc thiết lập các thành phần không phụ thuộc vào script khác.
        
    - **Tham số:** Không có (`void Awake()`).
        
- **`OnEnable()`**
    
    - **Vai trò:** Chạy mỗi khi GameObject hoặc Script chuyển từ trạng thái **Disable ($\rightarrow$) Enable** (được bật lên). Dùng để đăng ký các Event, Reset lại trạng thái khi Object xuất hiện lại (ví dụ: lấy từ Object Pool).
        
    - **Tham số:** Không có (`void OnEnable()`).
        
- **`Start()`**
    
    - **Vai trò:** Chạy **1 lần duy nhất** ngay trước frame đầu tiên mà script bắt đầu hoạt động (chỉ chạy khi script được Enable). Dùng để truy xuất hoặc giao tiếp dữ liệu với các script/component khác vì lúc này tất cả `Awake()` của mọi object đã chạy xong.
        
    - **Tham số:** Không có (`void Start()`).
        

## 2. Giai đoạn Cập nhật từng Frame (Game Loop)

- **`Update()`**
    
    - **Vai trò:** Được gọi **mỗi frame 1 lần**. Đây là nơi xử lý logic game chính: đọc Input từ người chơi (bàn phím, chuột), tính toán thời gian cooldown, làm hiệu ứng UI đơn giản.
        
    - **Tham số:** Không có (`void Update()`).
        
- **`FixedUpdate()`**
    
    - **Vai trò:** Được gọi theo **chu kỳ thời gian cố định** (mặc định 0.02s / 50 lần/giây), không phụ thuộc vào FPS của máy. **Bắt buộc dùng cho mọi xử lý Vật lý (Physics)** như tác động lực (`AddForce`), thay đổi vận tốc (`velocity`).
        
    - **Tham số:** Không có (`void FixedUpdate()`).
        
- **`LateUpdate()`**
    
    - **Vai trò:** Được gọi **mỗi frame**, nhưng luôn chạy **sau khi tất cả hàm `Update()` trên mọi script đã xử lý xong**. Thường dùng để làm Camera di chuyển đi theo nhân vật (đảm bảo nhân vật đã di chuyển xong trong `Update()` thì Camera mới tính toán vị trí theo sau).
        
    - **Tham số:** Không có (`void LateUpdate()`).
        

## 3. Giai đoạn Va chạm Vật lý 2D (Physics 2D)

_(Nếu làm game 3D, bạn chỉ cần bỏ chữ `2D` ở cuối tên hàm, ví dụ: `OnCollisionEnter(Collision collision)`)._

### A. Nhóm Trigger (Xuyên qua nhau, chỉ bắt sự kiện - Is Trigger = Checked)

- **`OnTriggerEnter2D(Collider2D collider)`**
    
    - **Vai trò:** Chạy đúng **1 frame đầu tiên** khi collider của object này bắt đầu chạm/đi vào vùng Trigger của object khác. Dùng để nhặt item, vào vùng nhặt tiền, bước vào portal.
        
    - **Tham số:** `Collider2D collider` – Đại diện cho Collider của **object kia** (object bị va chạm).
        
- **`OnTriggerStay2D(Collider2D collider)`**
    
    - **Vai trò:** Chạy **liên tục mỗi frame** chừng nào 2 object vẫn còn nằm đè lên nhau. Dùng làm vùng chịu sát thương theo thời gian (vùng độc, lửa đốt).
        
    - **Tham số:** `Collider2D collider` – Collider của object kia.
        
- **`OnTriggerExit2D(Collider2D collider)`**
    
    - **Vai trò:** Chạy đúng **1 frame** khi object vừa rời khỏi vùng Trigger. Dùng để tắt popup thông báo khi chơi rời khỏi NPC, thoát khỏi vùng an toàn.
        
    - **Tham số:** `Collider2D collider` – Collider của object vừa rời đi.
        

### B. Nhóm Collision (Cản trở nhau, va chạm cứng - Is Trigger = Unchecked)

- **`OnCollisionEnter2D(Collision2D collision)`**
    
    - **Vai trò:** Chạy khi 2 object va chạm vật lý nảy ra/cản lại nhau. Dùng để tính sát thương khi đạn đập vào tường, nhân vật tiếp đất.
        
    - **Tham số:** `Collision2D collision` – Chứa chi tiết thông tin va chạm (điểm va chạm `contacts`, lực va chạm, collider của đối phương...).
        
- **`OnCollisionStay2D(Collision2D collision)`**
    
    - **Vai trò:** Chạy liên tục khi 2 object đang tiếp xúc/đẩy nhau (như nhân vật đang đứng trên mặt đất).
        
- **`OnCollisionExit2D(Collision2D collision)`**
    
    - **Vai trò:** Chạy khi 2 object không còn chạm vào nhau nữa (như nhân vật bắt đầu nhảy lên khỏi mặt đất).
        

## 4. Giai đoạn Tắt & Hủy Object (Decommissioning)

- **`OnDisable()`**
    
    - **Vai trò:** Chạy khi GameObject hoặc Script bị **tắt (Disable)**. Dùng để hủy đăng ký Event (để tránh rò rỉ bộ nhớ), tạm dừng thanh audio.
        
    - **Tham số:** Không có (`void OnDisable()`).
        
- **`OnDestroy()`**
    
    - **Vai trò:** Chạy khi GameObject bị **xóa khỏi Scene** (`Destroy(gameObject)`) hoặc khi chuyển sang Scene mới. Dùng để dọn dẹp tài nguyên tĩnh (clean up static references/events).
        
    - **Tham số:** Không có (`void OnDestroy()`).
        

## 5. Giai đoạn Render & UI Support

- **`OnDrawGizmos()` / `OnDrawGizmosSelected()`**
    
    - **Vai trò:** Dùng để vẽ các hình học hỗ trợ debugger (vòng tròn tầm đánh, ô kiểm tra mặt đất, đường di chuyển) trên cửa sổ **Scene** (không hiển thị khi build game thật).
        
    - **Tham số:** Không có.
        

### 💡 Thứ tự chạy tóm tắt bạn cần nhớ nhất:

$$\text{Awake} \rightarrow \text{OnEnable} \rightarrow \text{Start} \rightarrow [\text{FixedUpdate} \rightarrow \text{Update} \rightarrow \text{LateUpdate}] \rightarrow \text{OnDisable} \rightarrow \text{OnDestroy}$$

### `Instantiate(slotPrefab, inventoryPanel.transform)`

Hàm `Instantiate` dùng để nhân bản (tạo ra một bản sao mới) của một đối tượng có sẵn (Prefab hoặc GameObject) vào Scene.

- **Hàm:** `Instantiate(Object original, Transform parent)`
    
- **Các tham số:**
    
    - **`slotPrefab`** (_Tham số 1 - `original`_): Đối tượng mẫu (Prefab) của ô chứa đồ mà bạn muốn tạo ra.
        
    - **`inventoryPanel.transform`** (_Tham số 2 - `parent`_): Xác định **Transform của cha** (Parent) chứa đối tượng mới được tạo. Ở đây, ô chứa đồ mới sẽ được đặt làm con của `inventoryPanel` (giao diện khung hành trang), giúp UI tự động sắp xếp vị trí ô này theo quy tắc layout của Panel.
### `.GetComponent<Slot>()`

Hàm `GetComponent` dùng để tìm và lấy ra một [[Component]] (Script hoặc thuộc tính) gắn trên GameObject vừa được tạo.

- **Hàm:** `GetComponent<T>()`
    
- **Tham số kiểu dữ liệu `<Slot>`**: Đây là tham số kiểu hằng đẳng (Generic Type). Nó báo cho Unity biết bạn muốn lấy script/[[component]] tên là **`Slot`** đang gắn trên chiếc `slotPrefab` vừa khởi tạo, sau đó gán vào biến `slot` để quản lý.

## `GetComponentInParent<T>()`

Hàm `GetComponentInParent<T>()` trong Unity dùng để **tìm kiếm và lấy ra Component kiểu `T` nằm trên chính GameObject đó hoặc ngược lên các GameObject cha (Parent, Grandparent,...) trong cây Hierarchy**.

### Cách thức hoạt động của `GetComponentInParent<T>()`

Hàm sẽ tìm kiếm theo thứ tự từ dưới lên trên:

1. **Kiểm tra chính GameObject hiện tại:** Nếu chính GameObject đó có chứa component `T`, hàm sẽ trả về component đó ngay lập tức.
    
2. **Duyệt ngược lên các cấp cha (Parent):** Nếu không thấy ở GameObject hiện tại, nó sẽ di chuyển lên GameObject cha trực tiếp, rồi lên ông/bà (Grandparent) cho đến khi tìm thấy component `T` đầu tiên hoặc chạm đến Root (cấp cao nhất).
    
3. **Trả về `null`:** Nếu duyệt đến hết cấp cha trên cùng mà vẫn không tìm thấy component `T`, hàm sẽ trả về `null`.
    

### Tại sao hàm này cực kỳ hữu ích khi làm UI và Drag & Drop?

Khi bạn thiết lập một ô `Slot` trong UI Inventory, ô `Slot` thường có cấu trúc cây gồm nhiều phần tử con:

Plaintext

```
[Slot] (Chứa Script Slot)
  └── [Image_Background]
  └── [Image_Icon] (Nơi con trỏ chuột chạm vào)
  └── [Text_Quantity]
```

Khi người chơi nhả chuột thả Item vào ô `Slot`, điểm con trỏ chuột (`eventData.pointerEnter`) có thể chạm trực tiếp vào hình ảnh `Image_Icon` hoặc chữ `Text_Quantity` con ở bên trong chứ không chạm trực tiếp vào khung ngoài `Slot`.

- **Nếu dùng `GetComponent<Slot>()`:** Unity chỉ tìm trên chính `Image_Icon` hay `Text_Quantity`. Do các GameObject con này không có script `Slot`, hàm sẽ trả về `null` (gây ra lỗi không nhận ô Slot).
    
- **Nếu dùng `GetComponentInParent<Slot>()`:** Unity tìm trên `Image_Icon` không thấy, nó sẽ tự động **leo ngược lên GameObject cha (`Slot`)** và tìm thấy script `Slot` một cách chính xác.
    

### Các biến thể phổ biến của hàm

1. **`GetComponentInParent<T>()`**: Trả về component `T` đầu tiên tìm thấy trên chính nó hoặc các cấp cha.
    
2. **`GetComponentsInParent<T>()`**: Trả về một mảng chứa **tất cả** các component `T` tìm thấy từ chính nó ngược lên hết các cấp cha.
    
3. **`GetComponentInParent<T>(bool includeInactive)`**: Bổ sung tham số `bool`. Nếu truyền `true`, hàm sẽ tìm kiếm cả trên các GameObject cha đang bị ẩn (`SetActive(false)`).

## `StartCoroutine` 

- **`StartCoroutine(...)`** là phương thức dùng để **kích hoạt và bắt đầu chạy** một Coroutine. Khi bạn gọi `StartCoroutine(FadedOutAndDestroy(popup, popupDuration))`, Unity sẽ đăng ký Coroutine này vào tiến trình xử lý hàng khung hình.

## **`DontDestroyOnLoad`**

**`DontDestroyOnLoad`** là một phương thức trong Unity (`Object.DontDestroyOnLoad`) dùng để **giữ cho một GameObject không bị tiêu hủy** khi bạn chuyển đổi từ Scene này sang Scene khác.

### 1. Cơ chế hoạt động

- **Mặc định trong Unity:** Khi bạn gọi `SceneManager.LoadScene()`, Unity sẽ tự động dọn dẹp và tiêu hủy (`Destroy`) toàn bộ các GameObject ở Scene hiện tại để giải phóng bộ nhớ trước khi nạp Scene mới.
    
- **Khi dùng `DontDestroyOnLoad(gameObject)`:** Unity sẽ chuyển GameObject đó (cùng tất cả Script và GameObject con đính kèm) sang một vùng quản lý đặc biệt có tên là **`DontDestroyOnLoad`** trong bảng Hierarchy. Nó sẽ tồn tại xuyên suốt suốt chu kỳ sống của ứng dụng cho đến khi game đóng hoặc bạn chủ động gọi `Destroy()`.
    

### 2. Các trường hợp hay sử dụng

`DontDestroyOnLoad` thường được dùng cho các hệ thống quản lý toàn cục (Global Managers) áp dụng mô hình Singleton:

- **`AudioManager` / `BGM`:** Giúp nhạc nền tiếp tục phát mượt mà không bị ngắt quãng hay giật khi đổi Scene.
    
- **`GameManager` / `DataManager`:** Lưu trữ thông tin người chơi (điểm số, inventory, cài đặt âm thanh/đồ họa) xuyên suốt các màn chơi.
    
- **`NetworkManager`:** Giữ kết nối Server/Client ổn định trong các game Multiplayer.
    

### 3. Vấn đề hay gặp & Cách xử lý (Duplicate Instances)

Lỗi phổ biến nhất khi dùng `DontDestroyOnLoad` là **trùng lặp đối tượng**: Nếu bạn chuyển sang Scene B rồi quay lại Scene A (nơi đặt GameObject ban đầu), Unity sẽ khởi tạo thêm một GameObject mới. Kết quả là bạn sẽ có 2 `AudioManager` chạy song song gây đè âm thanh.

**Giải pháp (Kết hợp với mẫu thiết kế Singleton):**

C#

```
using UnityEngine;

public class AudioManager : MonoBehaviour
{
    public static AudioManager Instance { get; private set; }

    private void Awake()
    {
        // Kiểm tra nếu đã có Instance tồn tại
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject); // Xóa bản sao mới tạo để tránh trùng lặp
            return;
        }

        // Nếu chưa có, gán Instance và giữ lại GameObject này
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }
}
```

### ⚠️ Lưu ý quan trọng

1. **Chỉ áp dụng trực tiếp cho Root GameObject:** Đối tượng truyền vào `DontDestroyOnLoad` phải là GameObject cấp cao nhất trong Hierarchy (không nằm trong GameObject cha nào). Nếu truyền một Child GameObject, Unity sẽ tự động tách nó khỏi cha hoặc đưa toàn bộ Root GameObject cha vào `DontDestroyOnLoad`.
    
2. **Quản lý bộ nhớ:** Đừng đưa quá nhiều dữ liệu rác vào các đối tượng `DontDestroyOnLoad`, vì chúng sẽ không bao giờ được giải phóng tự động cho đến khi thoát game.