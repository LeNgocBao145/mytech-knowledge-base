## Transform
Trong Unity, **`Transform`** là một Component quan trọng nhất của mọi GameObject. Tất cả mọi GameObject trong Unity khi được tạo ra (từ Empty GameObject, Camera, Light cho đến các mô hình UI/3D) **bắt buộc phải có sẵn component Transform** và không thể xóa bỏ component này.

Component này lưu trữ và quản lý tất cả các thông tin liên quan đến **Vị trí**, **Góc xoay**, **Kích thước** và **Mối quan hệ Cha - Con (Parent - Child)** của GameObject đó trong không gian.

### 1. 3 Thuộc tính cơ bản của Transform

Trên bảng **Inspector**, Transform hiển thị 3 nhóm thông số chính:

- **Position (Vị trí):** Lưu trữ tọa độ của GameObject trong không gian ($X, Y, Z$).
    
    - Với game 2D: Thường thao tác trên trục $X$ (ngang) và $Y$ (dọc).
        
    - Với game 3D: Dùng cả $X, Y, Z$ (ngang, cao, sâu).
        
- **Rotation (Góc xoay):** Độ xoay của GameObject theo các trục $X, Y, Z$ (tính bằng độ - Degrees). Trong code, Unity lưu góc xoay dưới dạng `Quaternion`, nhưng hiển thị ra Inspector dưới dạng góc `Euler` để dễ chỉnh sửa.
    
- **Scale (Kích thước / Tỉ lệ):** Độ thu phóng của GameObject theo 3 chiều $X, Y, Z$ so với kích thước gốc (mặc định là $(1, 1, 1)$).
    

> **Lưu ý về Tọa độ Tuyệt đối và Tương đối:**
> 
> - Nếu GameObject nằm ở cấp ngoài cùng (không thuộc con của ai): Thông số Position/Rotation/Scale thể hiện vị trí trong **World Space** (Không gian thế giới game).
>     
> - Nếu GameObject là **con (child)** của một GameObject khác: Thông số thể hiện vị trí trong **Local Space** (Không gian tương đối so với GameObject cha).
>     

### 2. Quản lý Mối quan hệ Cha - Con (Hierarchy / Parent-Child)

Ngoài việc định vị, `Transform` trong C# còn đóng vai trò là một **danh sách chứa (container)** giúp quản lý các đối tượng con.

- **`transform.parent`**: Lấy hoặc gán GameObject cha.
    
- **`transform.childCount`**: Trả về số lượng đối tượng con trực tiếp nằm bên trong.
    
- **`transform.GetChild(index)`**: Lấy ra Transform của đối tượng con ở vị trí chỉ định (`index`).
    

**Đặc tính kế thừa:** Khi GameObject cha di chuyển, xoay hoặc thay đổi kích thước, toàn bộ các GameObject con nằm bên trong nó sẽ di chuyển, xoay và thu phóng theo.

### 3. Các hàm và thuộc tính C# phổ biến thường dùng

Khi viết script C#, bạn có thể truy cập thẳng vào `transform` của GameObject đó mà không cần gọi `GetComponent<Transform>()`:


```
// 1. Di chuyển đối tượng sang phải theo thời gian
transform.Translate(Vector3.right * speed * Time.deltaTime);

// 2. Xoay đối tượng quanh trục Z (cho game 2D)
transform.Rotate(0, 0, turnSpeed * Time.deltaTime);

// 3. Đổi vị trí tới tọa độ mới
transform.position = new Vector3(0, 5, 0); // World position

// 4. Đặt làm con của một Panel/Object khác
transform.SetParent(inventoryPanel.transform);

// 5. Quay mặt về phía đối tượng mục tiêu (Target)
transform.LookAt(targetTransform);
```

## `RectTransform`
**`RectTransform`** là một component đặc thù trong Unity dùng để quản lý vị trí, kích thước và cách căn chỉnh của tất cả các phần tử giao diện người dùng (**UI Element**) như `Image`, `Text`, `Button`, `Panel`,...

Nó kế thừa trực tiếp từ component `Transform` thông thường nhưng được thiết kế riêng cho không gian 2D/UI.

### Các thuộc tính cốt lõi của RectTransform

Khác với `Transform` 3D thông thường chỉ có _Position_, _Rotation_, và _Scale_, `RectTransform` bổ sung thêm 3 khái niệm quan trọng để UI tự động co giãn (Responsive) trên nhiều độ phân giải màn hình khác nhau:

#### 1. Anchors (Điểm neo)

- **Khái niệm:** Là điểm cố định (hoặc vùng cố định) trên **[[Canvas]]/Parent** mà UI sẽ bám theo khi màn hình thay đổi kích thước.
    
- **Biểu tượng:** Gồm 4 hình tam giác nhỏ chụm lại thành hình con thoi trong Scene view.
    
- **Cấu trúc:** Gồm `Min` ($X, Y$) và `Max` ($X, Y$) tính theo tỉ lệ từ $0$ đến $1$ của Parent:
    
    - Nếu 4 điểm neo chụm lại 1 điểm: UI giữ nguyên kích thước cố định (`Width`/`Height`) và di chuyển theo điểm neo đó.
        
    - Nếu 4 điểm neo kéo giãn ra (Stretch): UI sẽ tự động co giãn kích thước theo lề của Parent.
        

#### 2. Pivot (Điểm tâm / Điểm xoay)

- **Khái niệm:** Là điểm mốc nằm trên chính phần tử UI đó (tính từ $(0,0)$ ở góc dưới-trái đến $(1,1)$ ở góc trên-phải).
    
- **Vai trò:**
    
    - Là tâm điểm khi xoay (`Rotation`) hoặc thu phóng (`Scale`).
        
    - Tọa độ `anchoredPosition` chính là khoảng cách từ **Anchor** đến **Pivot** này.
        
    - Mặc định Pivot nằm ở chính giữa UI: $(0.5, 0.5)$.
        

#### 3. Anchored Position ($Pos X, Pos Y$) & Size ($Width, Height$)

- **`anchoredPosition`:** Tọa độ vị trí của Pivot so với Anchor Reference Point.
    
- **`sizeDelta` ($Width, Height$):** Kích thước chiều rộng và chiều cao của UI (khi Anchor không ở chế độ Stretch).
    

### Sự khác biệt giữa Transform và RectTransform

|**Đặc điểm**|**Transform (3D / Non-UI)**|**RectTransform (UI)**|
|---|---|---|
|**Không gian áp dụng**|Dùng cho GameObject 3D / 2D Sprite thông thường|Dành riêng cho các phần tử UI trong [[Canvas]]|
|**Đơn vị vị trí**|Tọa độ World Space ($X, Y, Z$)|Tọa độ Pixel / Screen Space dựa theo Anchor|
|**Khả năng co giãn**|Thay đổi kích thước bằng `Scale` ($X, Y, Z$)|Thay đổi kích thước bằng `Width`, `Height` hoặc `Anchors Stretch`|
|**Điểm neo (Anchors)**|Không có|Có (giúp UI responsive trên nhiều màn hình)|

### Một số thao tác RectTransform thường dùng trong C#

```
// Lấy component RectTransform
RectTransform rect = GetComponent<RectTransform>();

// 1. Chỉnh vị trí tương đối so với Anchor (Pos X, Pos Y)
rect.anchoredPosition = new Vector2(100, -50);

// 2. Chỉnh chiều rộng và chiều cao
rect.sizeDelta = new Vector2(200, 200);

// 3. Đưa UI về chính giữa Parent
rect.anchoredPosition = Vector2.zero;

// 4. Đặt Pivot về chính giữa góc trên-trái
rect.pivot = new Vector2(0, 1);
```

## Canvas Group

GameObject phải có component Canvas Group thì mới dragable ?

**Không bắt buộc.** Một GameObject UI **không cần** phải có component `Canvas Group` thì mới có thể kéo thả (Drag and Drop) được.

Để làm một UI GameObject có thể Drag & Drop trong Unity, điều kiện **bắt buộc** chỉ bao gồm:

1. **Giao diện nhận tương tác chuột:**
    
    - Tệp/UI Element phải chứa ít nhất một Component thuộc dạng **Graphic** có bật `Raycast Target` (như `Image`, `RawImage`, hoặc `Text`).
        
    - [[Canvas]] chứa UI đó phải có component **Graphic Raycaster**, và Scene phải có **EventSystem**.
        
2. **Cài đặt các Interface kéo thả của EventSystem:**
    
    - Script C# gắn trên GameObject đó chỉ cần `using UnityEngine.EventSystems;` và kế thừa các interface:
        
        - `IBeginDragHandler` (khi bắt đầu kéo)
            
        - `IDragHandler` (trong khi đang kéo)
            
        - `IEndDragHandler` (khi thả chuột)
            

### Vậy tại sao trong các hướng dẫn làm Drag & Drop người ta hay thêm `Canvas Group`?

Dù không bắt buộc để kéo thả, `Canvas Group` lại là một công cụ cực kỳ hữu ích để xử lý **2 vấn đề logic phổ biến** khi làm hệ thống Inventory/Drag & Drop:

#### 1. Xử lý đâm xuyên Raycast (`blocksRaycasts = false`)

- **Vấn đề:** Khi bạn đang kéo một icon item đi theo con trỏ chuột, chiếc icon đó nằm ngay bên dưới con trỏ chuột. Nếu bạn thả item vào một ô `Slot` mới, sự kiện `OnDrop` của ô `Slot` sẽ **không được kích hoạt** vì con trỏ chuột bị chính cái icon đang kéo che mất (Raycast trúng icon chứ không trúng ô Slot).
    
- **Giải pháp với Canvas Group:** Khi bắt đầu kéo (`OnBeginDrag`), bạn đặt `canvasGroup.blocksRaycasts = false`. Lúc này chuột sẽ "xuyên qua" icon item và nhận diện chính xác ô `Slot` bên dưới. Khi thả chuột (`OnEndDrag`), bạn bật lại `canvasGroup.blocksRaycasts = true`.
    

#### 2. Tạo hiệu ứng mờ nhạt khi kéo (`alpha`)

- Giúp chỉnh nhanh thuộc tính `canvasGroup.alpha = 0.6f` khi đang kéo để tạo cảm giác item đang được nhấc lên.
    

### Ví dụ code Drag & Drop cơ bản **không cần** và **có dùng** Canvas Group

```
using UnityEngine;
using UnityEngine.EventSystems;

public class DragItem : MonoBehaviour, IBeginDragHandler, IDragHandler, IEndDragHandler
{
    private RectTransform rectTransform;
    private CanvasGroup canvasGroup; // Không bắt buộc, nhưng khuyên dùng

    private void Awake()
    {
        rectTransform = GetComponent<RectTransform>();
        canvasGroup = GetComponent<CanvasGroup>(); // Lấy nếu có
    }

    public void OnBeginDrag(PointerEventData eventData)
    {
        if (canvasGroup != null)
        {
            canvasGroup.blocksRaycasts = false; // Chuột xuyên qua item để nhận diện ô Slot thả vào
            canvasGroup.alpha = 0.6f;          // Làm mờ nhẹ khi kéo
        }
    }

    public void OnDrag(PointerEventData eventData)
    {
        // Di chuyển UI theo con trỏ chuột
        rectTransform.anchoredPosition += eventData.delta;
    }

    public void OnEndDrag(PointerEventData eventData)
    {
        if (canvasGroup != null)
        {
            canvasGroup.blocksRaycasts = true;  // Bật lại raycast khi thả xong
            canvasGroup.alpha = 1f;            // Đặt lại độ trong suốt ban đầu
        }
    }
}
```

`blocksRaycasts` trong **Canvas Group** không phải là thuật toán `Physics.Raycast` bắn tia vật lý 3D/2D, mà nó quản lý hệ thống **Graphic Raycast** (tia chiếu UI 2D) của Unity UI (UGUI).

### Graphic Raycast hoạt động như thế nào?

Khi bạn nhấp chuột hoặc chạm vào màn hình, **EventSystem** kết hợp với **Graphic Raycaster** trên [[Canvas]] sẽ phát một tia từ vị trí con trỏ vuông góc vào màn hình để kiểm tra xem con trỏ đang đè lên phần tử UI nào (`Image`, `Button`, `Text`).

1. **Raycast Target = true (trên từng phần tử UI):** Cho phép phần tử UI đó đăng ký nhận tia Graphic Raycast.
    
2. **`blocksRaycasts` (trên Canvas Group):** Bật/tắt khả năng "hấp thụ" hoặc "chặn" tia Graphic Raycast cho **toàn bộ Canvas Group** và tất cả các phần tử UI con nằm bên trong nó.
    

### Thuộc tính `blocksRaycasts` quyết định điều gì?

|**Giá trị blocksRaycasts**|**Hành vi của UI**|**Ứng dụng thực tế**|
|---|---|---|
|**`true` (Mặc định)**|UI **chặn** tia Raycast. Các nút bấm nhận tương tác chuột bình thường. Các phần tử UI phía sau sẽ **bị che khuất** và không bấm được.|Menu chính, Nút bấm, Ô chứa đồ Inventory chuẩn.|
|**`false`**|UI trở nên **vô hình trước tia Raycast** (tia đâm xuyên qua). Toàn bộ nhóm UI đó không nhận click/drag và không che các phần tử bên dưới.|Khi đang kéo thả vật phẩm (Drag & Drop), làm mờ màn hình loading nhưng cho click xuyên qua.|

### Tại sao lại cần `blocksRaycasts` khi làm Drag & Drop?

Khi bạn nhấc một icon vật phẩm (`Item`) và kéo đi, icon đó luôn bám theo con trỏ chuột.

- **Nếu `blocksRaycasts = true`:** Icon nằm ngay dưới con trỏ chuột và hứng toàn bộ tia Raycast. Khi bạn nhả chuột trên một ô `Slot`, event `OnDrop` của ô `Slot` **không thể kích hoạt** vì tia Raycast bị chiếc icon đang kéo chặn lại trước khi chạm tới `Slot`.
    
- **Nếu `blocksRaycasts = false` (khi đang kéo):** Con trỏ chuột sẽ "nhìn xuyên" qua icon `Item` đang kéo và chiếu thẳng xuống ô `Slot` nằm bên dưới, giúp kích hoạt hàm `OnDrop` trên `Slot` thành công.