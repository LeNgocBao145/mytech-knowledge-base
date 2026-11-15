
Dòng lệnh này dùng để **chuyển tạm thời item lên cấp cao nhất (root Canvas) trong cây Hierarchy khi bắt đầu kéo**.

Có 2 lý do kỹ thuật quan trọng bắt buộc phải làm việc này:

#### A. Tránh bị che khuất bởi các ô Slot khác (Thứ tự Render / Draw Order)

Trong Unity UI, các phần tử nằm **dưới** trong Hierarchy sẽ được vẽ **đè lên** các phần tử nằm phía trên.

- Khi item nằm bên trong ô `Slot 1` (ví dụ ở đầu danh sách), nếu bạn kéo item đó qua `Slot 2`, `Slot 3` nằm phía sau trong Hierarchy, các ô `Slot` đó sẽ **vẽ đè lên làm mất hút item** bạn đang kéo.
    
- Khi đổi parent sang `transform.root` (Canvas chính), item sẽ nằm ở cấp ngoài cùng hoặc dưới cùng của Canvas, đảm bảo **luôn nổi lên trên cùng màn hình** trong suốt quá trình kéo.
    

#### B. Tránh bị ảnh hưởng bởi Layout Group hoặc Mask của Slot cha

Các ô `Slot` trong Inventory thường nằm bên trong `Grid Layout Group` hoặc `Horizontal / Vertical Layout Group`.

- Nếu không đổi Parent, khi bạn di chuyển item bằng `transform.position = eventData.position`, component Layout của `Slot` hoặc `Inventory Panel` sẽ liên tục ép vị trí của item quay trở lại vị trí cũ trong lưới, làm item bị giật hoặc không thể di chuyển tự do theo con trỏ chuột.