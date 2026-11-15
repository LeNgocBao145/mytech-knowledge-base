**Raycast** (chiếu tia) là một kỹ thuật trong Unity (và các game engine nói chung) cho phép bạn **bắn một tia vô hình (Ray)** từ một vị trí gốc (Origin) đi theo một hướng xác định (Direction) với một khoảng cách nhất định (Distance) để kiểm tra xem tia đó có va chạm với đối tượng nào hay không.

Nếu tia gặp một đối tượng có gắn **Collider**, engine sẽ trả về kết quả va chạm cùng các thông tin chi tiết (như vị trí va chạm, khoảng cách, đối tượng bị bắn trúng, hướng bề mặt,...).

### Các ứng dụng phổ biến của Raycast

1. **Bắn súng trong game (Hitscan Weapons):** Khi người chơi bấm bắn, một Raycast được phát ra từ nòng súng. Nếu va chạm với kẻ địch, xử lý trừ máu ngay lập tức mà không cần tạo viên đạn vật lý di chuyển.
    
2. **Kiểm tra mặt đất (Ground Check):** Bắn một tia ngắn từ chân nhân vật thẳng xuống dưới. Nếu va chạm với bề mặt ground, nhân vật được tính là đang đứng trên mặt đất (để cho phép nhảy).
    
3. **Tương tác vật phẩm / Nhặt đồ:** Bắn tia từ tâm màn hình hoặc con trỏ chuột ra không gian game để xem người chơi đang chỉ vào cửa, vật phẩm hay NPC nào.
    
4. **Tầm nhìn kẻ địch (Line of Sight):** Bắn tia từ quái vật đến người chơi để kiểm tra xem có bức tường hay vật cản nào che mất tầm nhìn của quái vật hay không.
    

### Phân loại Raycast trong Unity

Unity cung cấp các hàm Raycast cho cả không gian 2D và 3D:

#### 1. Physics.Raycast (Game 3D)

Dùng để bắn tia trong môi trường 3D với các Collider 3D (BoxCollider, MeshCollider,...).

C#

```
RaycastHit hit;
// Bắn 1 tia từ vị trí transform.position, theo hướng phía trước (forward), tối đa 10m
if (Physics.Raycast(transform.position, transform.forward, out hit, 10f))
{
    Debug.Log("Bắn trúng đối tượng: " + hit.collider.name);
    Debug.Log("Vị trí va chạm: " + hit.point);
}
```

#### 2. Physics2D.Raycast (Game 2D)

Dùng để bắn tia trong môi trường 2D với các Collider 2D (BoxCollider2D, PolygonCollider2D,...).

```
// Bắn 1 tia từ vị trí nhân vật xuống phía dưới 1 mét
RaycastHit2D hit = Physics2D.Raycast(transform.position, Vector2.down, 1f);

if (hit.collider != null)
{
    Debug.Log("Chân đang chạm đất: " + hit.collider.name);
}
```

### Các thành phần chính của một lệnh Raycast

- **Origin (Điểm bắt đầu):** Tọa độ bắt đầu phát tia ($X, Y, Z$).
    
- **Direction (Hướng):** Hướng di chuyển của tia (được chuẩn hóa dưới dạng `Vector3` hoặc `Vector2`, ví dụ `Vector3.forward`, `Vector2.down`).
    
- **Max Distance (Khoảng cách tối đa):** Độ dài tối đa của tia (nếu để mặc định sẽ là vô hạn).
    
- **LayerMask (Lớp lọc đối tượng):** Cho phép chỉ định tia Raycast chỉ va chạm với các Layer cụ thể (ví dụ: chỉ bắt va chạm với Layer `Ground` hoặc `Enemy` và bỏ qua Layer `UI` hay `Player`).

