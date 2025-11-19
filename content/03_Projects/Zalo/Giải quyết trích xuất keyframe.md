# 1. Hiểu bản chất câu hỏi dạng “theo thời gian”

Ví dụ:

* “Biển báo thứ hai là gì?”
    
* “Xe nào đến giao lộ trước?”
    
* “Có bao nhiêu xe đi qua trước khi đèn chuyển đỏ?”
    
* “Sau khi xe tải quẹo, biển báo nào xuất hiện?”
    

Đây là **aggregate-temporal questions**.  
VIntern không có sẵn, nhưng mình có thể _ép_ quy trình xử lý.

* * *

# 2. Giải pháp tổng thể: “Pipeline Temporal Intelligence”

**Video → Frame Sampling → Sort Frame → Per-frame Inference → Temporal Logic Layer → Output**

Cái quan trọng nhất là layer cuối: **temporal logic**, mình phải code tay.

* * *

# 3. Chiến lược chính để xử lý câu hỏi theo thời gian

## A. Multi-frame Captioning → Sequence Extraction

Cho VIntern caption từng frame:

```
Frame 1: “Có biển báo cấm bóp còi bên phải”
Frame 2: “Biển báo rẽ trái xuất hiện”
Frame 3: “Biển báo tốc độ tối đa 40”
...
```

Sau đó phân tích chuỗi caption:

→ biển báo thứ 2 = biển báo trong frame số 2  
→ xe nào đến trước = xe xuất hiện ở frame index nhỏ hơn  
→ đèn chuyển đỏ = phát hiện trong caption frame tương ứng

Đây là mô hình:

```
caption_seq = [model(img1), model(img2), ..., model(imgN)]
apply rule-based logic theo thứ tự index
```

Ưu điểm: đơn giản, hiệu quả, không cần training.

* * *

## B. Timeline Object Tracking (rất cần cho biển báo thứ 2)

Dùng YOLOv 8 n hoặc YOLOv 8 s để detect:

* biển báo
    
* loại biển báo
    
* vị trí (bbox)
    

Sau đó chạy **tracking theo thời gian**:

```
track = SORT hoặc ByteTrack
```

Ta thu được:

```
time 1: sign = “cấm vượt”
time 2: sign = “rẽ trái”
time 3: sign = “tốc độ 40”
```

Câu hỏi: “Biển báo thứ hai là gì?”  
→ chính là track[1].

Đây là cách **đơn giản nhưng mạnh nhất** vì biển báo rõ, track dễ.

* * *

## C. Multi-frame QA + Voting theo Frame Index

Bạn có thể đưa vào VIntern câu hỏi dạng:

> “Trong frame này, biển báo thứ hai là gì? Nếu không có, trả lời: none.”

Sau đó:

* chạy VIntern cho từng frame
    
* gom kết quả theo thứ tự
    
* chọn frame thứ 2 có biển báo hợp lệ nhất
    

Sơ đồ:

```
frames = sorted by timestamp
answers = vintern(frame[i])
Pick answer at i=2 (hoặc frame có confidence cao nhất)
```

Dùng nhất khi câu hỏi yêu cầu extract thông tin đơn giản.

* * *

## D. LLM-style Reasoning trên Caption Sequence

Bạn có thể hỏi VIntern như cách dùng LLaVA:

```
“Dưới đây là mô tả liên tiếp theo thời gian:
1. <caption frame 1>
2. <caption frame 2>
3. <caption frame 3>
…
Dựa trên chuỗi thời gian trên, trả lời câu hỏi: ‘Biển báo thứ hai là gì?’”
```

Nghĩa là:

* VIntern nhìn từng ảnh → caption
    
* Bạn gộp caption lại -> tạo temporal sequence
    
* Cho VIntern reasoning lại dựa trên văn bản
    

Cực kỳ hiệu quả cho câu hỏi dạng diễn dịch.

* * *

# 4. Chọn giải pháp phù hợp theo loại câu hỏi

| Loại câu hỏi | Cách làm tốt nhất |
| --- | --- |
| Biển báo thứ 2, xe thứ 3 | A + B (Caption + Tracking) |
| Xe nào đến giao lộ trước | Caption sequence + compare positions theo thời gian |
| Sự kiện “sau khi” | Caption sequence → find event boundary |
| Đếm số sự kiện | Caption sequence → scan for pattern |
| Biển báo thay đổi theo thời gian | Tracking |

VIntern không giúp bạn điều này.  
**Giải pháp là mình tự làm temporal logic layer.**

* * *

# 5. Pipeline hoàn chỉnh (khuyến nghị cao nhất)

```
1. Uniform sampling (16–24 frame)
2. YOLO detect biển báo từng frame → class + time
3. Track biển báo + đọc thứ tự
4. Đồng thời: Caption từng frame bằng VIntern
5. Tạo sequence timeline
6. Dựa trên dạng câu hỏi → suy luận theo rule-based logic
7. Nếu câu hỏi dạng phân loại A/B/C → trả về label tương ứng
```

Đây là cách các đội mạnh sẽ làm.  
Tối ưu accuracy cực tốt mà không phải fine-tune.

* * *

# 6. Tóm gọn – trả lời câu hỏi của bạn

### “Vậy thiết kế làm sao để model hiểu time series?”

Không dùng VIntern để hiểu time series.  
**Bạn phải tự tạo “temporal layer”:**

* Caption sequence
    
* Object tracking
    
* Frame indexing
    
* Rule-based temporal reasoning
    
* Aggregation logic
    

**Model chỉ xử lý ảnh,  
Temporal là do pipeline bạn xử lý, không phải model.**