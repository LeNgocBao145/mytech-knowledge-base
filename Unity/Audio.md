
Trong hệ thống Unity Audio, cấu trúc âm thanh được chia thành 3 nhóm chính: **Assets (Tài nguyên)**, **Components (Thành phần)** và **GameObjects đóng vai trò cụ thể**.

### 1. Các thành phần trong Unity Audio

#### A. Assets (Tài nguyên lưu trong Project)

- **`AudioClip`:** File dữ liệu âm thanh gốc (`.wav`, `.mp3`, `.ogg`). Chứa sóng âm thực tế để phát.
    
- **`AudioMixer`:** Asset quản lý, phân luồng và trộn nhiều kênh âm thanh cùng lúc (giống như một bàn Mixer ngoài đời).
    
- **`AudioMixerGroup`:** Các tuyến kênh con bên trong `AudioMixer` (ví dụ: kênh Master $\rightarrow$ phân nhánh thành kênh BGM, SFX, Voice).
    
- **`AudioMixerSnapshot`:** Bản lưu trạng thái tham số âm thanh (ví dụ: tạo snapshot làm mờ tiếng khi mở Pause Menu hoặc khi nhân vật lặn xuống nước).
    

#### B. Components (Gắn lên các GameObject)

- **`AudioListener` (Tai nghe):** Thu nhận tất cả âm thanh phát ra trong môi trường 3D/2D và gửi đến loa thiết bị người dùng. Mỗi Scene thông thường **chỉ được phép có 1 `AudioListener` hoạt động** (mặc định được gắn sẵn ở `Main Camera`).
    
- **`AudioSource` (Loa phát):** Thành phần chịu trách nhiệm phát `AudioClip` vào không gian game. Quản lý volume, pitch, loop, 3D Spatial Blend (khoảng cách phát âm thanh 3D) và định hướng đầu ra tới `AudioMixerGroup`.
    
- **Audio Filters (Bộ lọc âm thanh DSP):** `AudioLowPassFilter` (lọc tiếng trong/đục), `AudioEchoFilter` (tạo tiếng vang vọng), `AudioDistortionFilter` (tạo tiếng biến dạng/méo âm).
    
- **`AudioReverbZone`:** Định nghĩa một vùng không gian có hiệu ứng vang (như hang động, đường hầm, căn phòng trống).
    

#### C. GameObjects liên quan

- **Audio Listener Object:** Thường là **`Main Camera`** hoặc **`Player`** (nơi đại diện cho góc nhìn/tai nghe của người chơi).
    
- **Audio Emitter Objects:** Bất kỳ GameObject nào trong Scene có gắn `AudioSource` (ví dụ: cây súng, kẻ địch, xe cộ, chiếc rương).
    

### 2. Để play Sound Effect (SFX) có cần AudioSource và AudioClip không?

**Chính xác, bạn bắt buộc phải có cả 2 thành phần này.**

- **`AudioClip`:** Là **dữ liệu âm thanh** (Nội dung: _Phát tiếng gì?_ - ví dụ: `sfx_explosion.wav`).
    
- **`AudioSource`:** Là **bộ phát/loa** (Cơ chế: _Phát như thế nào? Ở vị trí nào? Âm lượng bao nhiêu?_).
    

> **Mô phỏng thực tế:** `AudioClip` giống như **file nhạc MP3**, còn `AudioSource` là **chiếc loa**. Không có loa thì file nhạc không thể vang lên, còn không có file nhạc thì chiếc loa không có gì để phát.

#### 2 cách phát SFX phổ biến trong C#:

1. **Gán qua `AudioSource` có sẵn (Khuyên dùng cho SFX liên tục/tái sử dụng):**
    
    
    ```
    [SerializeField] private AudioSource audioSource;
    [SerializeField] private AudioClip sfxClip;
    
    public void PlaySFX()
    {
        // PlayOneShot giúp phát đè nhiều âm thanh cùng lúc mà không ngắt tiếng trước đó
        audioSource.PlayOneShot(sfxClip); 
    }
    ```
    
2. **Phát âm thanh 3D tại một vị trí bất kỳ (Phù hợp cho hiệu ứng nổ/tiêu hủy):**
    
    
    ```
    [SerializeField] private AudioClip explosionClip;
    
    public void OnExplode(Vector3 position)
    {
        // Tự tạo 1 GameObject tạm thời chứa AudioSource tại position, phát xong tự Destroy
        AudioSource.PlayClipAtPoint(explosionClip, position);
    }
    ```

## Audio Source

Thuộc tính **`Pitch`** của `AudioSource` trong Unity dùng để điều chỉnh **độ cao (tần số/cao độ)** và **tốc độ phát** của âm thanh cùng một lúc.

- **Giá trị mặc định (`pitch = 1.0`):** Phát âm thanh ở tốc độ và cao độ gốc.
    
- **`pitch > 1.0` (ví dụ `1.2` đến `2.0`):** Âm thanh phát **nhanh hơn** và giọng **cao/bổng hơn** (giống tiếng nhân vật hít khí Helium).
    
- **`0 < pitch < 1.0` (ví dụ `0.5` đến `0.8`):** Âm thanh phát **chậm hơn** và giọng **trầm/thấp hơn**.
    
- **`pitch = 0`:** Tạm ngưng âm thanh ngay lập tức mà không làm mất vị trí đang phát.
    
- **`pitch < 0` (ví dụ `-1.0`):** Phát **ngược** âm thanh từ cuối về đầu.
    

### Các ứng dụng thực tế phổ biến trong làm Game

#### 1. Biến đổi SFX để tránh lặp âm chán ngắt (Audio Variation)

Khi nhân vật bước đi hoặc bắn súng liên tục, nếu chỉ dùng 1 file âm thanh phát lặp đi lặp lại sẽ tạo cảm giác máy móc. Thay đổi nhẹ `pitch` mỗi lần phát giúp âm thanh tự nhiên hơn mà không cần nạp thêm nhiều file audio rác:

C#

```
public void PlayFootstep()
{
    // Thay đổi pitch ngẫu nhiên từ 0.9 đến 1.1
    audioSource.pitch = Random.Range(0.9f, 1.1f);
    audioSource.PlayOneShot(footstepClip);
}
```

#### 2. Hiệu ứng quay chậm (Slow-Motion) & Dừng thời gian

Khi game kích hoạt trạng thái Bullet-time hoặc Slow-motion, bạn chỉ cần hạ `pitch` của không gian/nhạc nền xuống thấp (ví dụ `0.4f`), âm thanh sẽ tự động trầm uốn và kéo dài ra đúng với cảm giác thời gian bị ngưng trệ.

#### 3. Động cơ phương tiện (Engine RPM)

Trong game đua xe, tiếng động cơ xe máy/ô tô tăng dần theo tốc độ di chuyển. Thay vì tạo nhiều file tiếng động cơ ở các dải tốc độ khác nhau, người ta map trực tiếp tốc độ xe vào `pitch`:

C#

```
// Khi xe chạy càng nhanh, pitch càng cao
audioSource.pitch = Mathf.Clamp(currentSpeed / maxSpeed, 0.8f, 2.5f);
```

#### 4. Kích thích cảm xúc & Áp lực thời gian

- **Power-up / Combo:** Mỗi khi ăn chuỗi item liên tiếp, tăng dần `pitch` (1.0 $\rightarrow$ 1.1 $\rightarrow$ 1.2...) tạo cảm giác phấn khích.
    
- **Sắp hết giờ:** Tăng `pitch` của BGM ở 10 giây cuối để đẩy nhịp tim người chơi lên cao.