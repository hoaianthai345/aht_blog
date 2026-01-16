# 1. Bài toán Speech Recognition

**Challenges**
Những vấn đề khó trong bài toán này là:
- Unfamiliar data type: Dữ liệu âm thanh sẽ lạ
- Complex features extraction process: Cần các kĩ thuật khó để xử lý kiểu dữ liệu âm thanh

**Overview ASR pipeline**
![[image-411.png]]
**Preprocessing:**
**DNN:** sử dụng các kiến trúc mạng RNN (dùng để xử lý chuỗi) để feature extract âm thanh. Sau đó 


# 2. Audio preprocessing
**Overview audio preprocessing pipeline**
![[image-413.png]]


## Audio signals

**Waves**
![[image-414.png]]
Signals âm thanh sẽ tuân theo hàm lượng giác.
Có hai giá trị chính chúng ta cần quan tâm:
- Amplitude: là giá trị A trong hàm sóng $y$ ở trên
- Cycle: là một lần dao động hoàn chỉnh của sóng.
- Period: chu kỳ, là thời gian để hoàn thiện một cycle

**Frequency & Period**
![[image-415.png]]

**Audio signal**
Khi ta kết hợp hai dạng sóng lại thì có được hình như sau:
![[image-416.png]]
Vậy sóng âm thanh là dạng sóng được kết hợp từ nhiều dạng sóng có tần số khác nhau.
Ví dụ đây là hình sóng âm thanh khi phát âm chữ `[iy]`:
![[image-417.png]]

## Analog-to-digital conversion
Cách để thu một tính hiệu sóng âm thanh (Analog) thành dạng Digital để cho máy tính có thể phân tích:
![[image-418.png]]

![[image-419.png]]
Để thực hiện thì chúng ta cần đi qua hai bước là **Sampling** và **Quantization**
![[image-420.png]]
### Sampling
Sampling được thực hiện là tại một thời điểm t, chúng ta sẽ lấy giá trị Amplitude ra -> Từ đó chúng ta sẽ rời rạc hóa được sóng liên tục.
![[image-421.png]]
Sampling rate (sampling frequency): là tần số lấy mẫu, số lượng mẫu dữ liệu lấy theo giây. Tần số lấy mẫu càng cao thì sóng dữ liệu càng mịn.

Đây là ví dụ trực quan thể hiện khi ta lấy mẫu theo các sampling rate khác nhau thì dạng sóng sẽ thay đổi như nào. Ở đây ta thấy với Sampling rate = 100, 200, 500 thì là đủ tốt và càng tăng thì không có thay đổi nhiều nên ta có thể chọn 100 là tối ưu.
![[image-422.png]]

Dưới đây là đoạn code minh họa
![[image-423.png]]
**Nyquist Theorem**
Khi ta thu sóng âm thanh, nếu muốn thu được hết dạng sóng của âm thanh thì ta cần phải lấy $F_{sampling} ≥ 2 F_{signal}$ 
![[image-424.png]]
### Quantization
Là chuyển đổi từ dạng liên tục thành rời rạc (lượng tử hóa)
![[image-425.png]]
**Lưu ý:** Thực tế thì hai bước Sampling và Quantization ta không cần thực hiện trong việc xử lý vì file dữ liệu đã được xử lý rồi.

## Wavefile format
![[image-426.png]]
Thư viện librosa hỗ trợ xử lý dữ liệu âm thanh.
Hoặc là ta dùng `from scipy.io import wavfile`
![[image-427.png]]

**Default Resampling**
![[image-428.png]]