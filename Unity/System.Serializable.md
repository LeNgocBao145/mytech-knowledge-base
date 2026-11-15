Attribute **`[System.Serializable]`** (thuộc namespace `System` trong C#) dùng để đánh dấu một **`class`** hoặc **`struct`** có thể **tuần tự hóa (serialization)** — tức là cho phép chuyển đổi dữ liệu của đối tượng đó thành dạng luồng dữ liệu (bytes, JSON, XML...) để lưu trữ, truyền qua mạng, hoặc cho phép Unity đọc và quản lý.

Trong Unity, `[System.Serializable]` có **2 công dụng quan trọng nhất**:

### 1. Hiển thị Custom Class / Struct lên bảng Inspector

Mặc định, Unity chỉ hiển thị các kiểu dữ liệu cơ bản (`int`, `float`, `string`, `Vector3`, `GameObject`...) trên bảng Inspector. Nếu bạn tự định nghĩa một `class` hoặc `struct` riêng, Unity sẽ **không** hiển thị nó trừ khi bạn thêm `[System.Serializable]`.

#### Ví dụ:


```
using System;
using UnityEngine;

// Đánh dấu class này có thể Serialize
[System.Serializable]
public class StatInfo
{
    public string statName;
    public int baseValue;
    public Sprite icon;
}

public class PlayerController : MonoBehaviour
{
    // Nhờ có [System.Serializable] trên class StatInfo,
    // biến này và mảng bên dưới sẽ hiển thị đầy đủ trên Inspector để tùy chỉnh
    public StatInfo healthStat;
    public StatInfo[] allStats;
}
```

### 2. Hỗ trợ Lưu / Tải dữ liệu (Save / Load Game & JSON)

Khi muốn lưu dữ liệu game (như thông số người chơi, kho đồ) ra file hoặc dùng `JsonUtility` của Unity, các đối tượng được lưu bắt buộc phải có attribute này:


```
[System.Serializable]
public class PlayerSaveData
{
    public string playerName;
    public int level;
    public float health;
}

// Chuyển đối tượng thành chuỗi JSON để lưu file
PlayerSaveData data = new PlayerSaveData { playerName = "Hero", level = 10, health = 100f };
string json = JsonUtility.ToJson(data); // Bắt buộc PlayerSaveData phải có [System.Serializable]
```

### Những lưu ý quan trọng khi dùng trong Unity:

1. **Không cần gắn cho `MonoBehaviour` hoặc `ScriptableObject`:** Cả hai lớp này đã được Unity tự động tích hợp sẵn cơ chế serialization.
    
2. **Ẩn một biến cụ thể trong Serializable Class:** Nếu một class đã được đánh dấu `[System.Serializable]` nhưng bạn muốn ẩn 1 biến bên trong khỏi Inspector và không muốn lưu biến đó, hãy dùng `[System.NonSerialized]`:
    
    
    ```
    [System.Serializable]
    public class EnemyData
    {
        public string enemyName; // Hiển thị trên Inspector
    
        [System.NonSerialized] 
        public float currentHealthRuntime; // Không hiển thị, không bị lưu trữ
    }
    ```
    
3. **Điều kiện để biến bên trong được Serialize:** Các biến bên trong class/struct đó phải là dạng hợp lệ được Unity hỗ trợ (như kiểu dữ liệu nguyên thủy, `string`, `List<T>`, `Array`, các kiểu của Unity như `Vector3`, `Color`, hoặc tham chiếu tới `UnityEngine.Object`).