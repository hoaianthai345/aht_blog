# I. 9 NHÓM CÂU HỎI CHUẨN CỦA VIDEOQA ZALO 2025

(được tổng hợp từ toàn bộ list bạn gửi)

## **1. Câu hỏi ĐẾM (Counting)**

Ví dụ:

* Có bao nhiêu biển báo?
    
* Bao nhiêu biển cảnh báo nguy hiểm?
    
* Có bao nhiêu biển cấm xuất hiện?
    

→ Bắt buộc dùng **toàn bộ video**, không thể dùng 1 keyframe.

* * *

## **2. Câu hỏi NHẬN DIỆN BIỂN BÁO (Sign Classification)**

Ví dụ:

* Biển báo tam giác có ý nghĩa gì?
    
* Biển cấm trong video là gì?
    
* Biển bên phải là gì?
    

→ Cần detect biển, crop biển, OCR nội dung, lookup bảng quy chuẩn.

* * *

## **3. Câu hỏi HƯỚNG – ĐƯỜNG NÀO (Navigation/Direction)**

Ví dụ:

* Nếu đi chếch phải thì vào đường nào?
    
* Hướng đi cầu Tân Thuận?
    
* Đi thẳng sẽ đến đâu?
    

→ Cần biển chỉ dẫn → OCR → hiểu hướng → so match với lựa chọn.

* * *

## **4. Câu hỏi ĐÈN TÍN HIỆU (Traffic Light State)**

Ví dụ:

* Video có đèn đỏ không?
    
* Phía trước có đèn giao thông không?
    

→ Dùng detector riêng cho traffic light.

* * *

## **5. Câu hỏi VẠCH KẺ – LÀN ĐƯỜNG (Lane/Vạch)**

Ví dụ:

* Xe có đang đi đúng làn?
    
* Vạch vàng gấp khúc ý gì?
    

→ Không cần Vintern. Cần lane-segmentation.

* * *

## **6. Câu hỏi TỐC ĐỘ – BIỂN TỐC ĐỘ (Speed Limit)**

Ví dụ:

* Tốc độ tối đa là bao nhiêu?
    
* Biển 70 km/h đúng không?
    

→ OCR + detect biển hình tròn màu trắng viền đỏ.

* * *

## **7. Câu hỏi CẤM / HẠN CHẾ (Restrictions)**

Ví dụ:

* Cấm xe nào?
    
* Xe tải bị cấm trong khung giờ nào?
    
* Cấm quay đầu?
    

→ YOLO + OCR text + rule-based.

* * *

## **8. Câu hỏi TÌNH HUỐNG GIAO THÔNG (Scenario Logic)**

Ví dụ:

* Xe máy có vi phạm không?
    
* Xe có được chuyển làn không?
    

→ Multi-frame + rule-based logic (không dùng Vintern).

* * *

## **9. Câu hỏi KHOẢNG CÁCH – THỜI ĐIỂM (Distance/Time)**

Ví dụ:

* Theo biển báo, cầu Phú Mỹ cách bao nhiêu km?
    
* Biển phụ 200 m đúng không?
    

→ OCR nội dung biển phụ.

* * *

# II. VẤN ĐỀ: “Trích keyframe rồi hỏi Vintern” **KHÔNG THỂ hoạt động**

Lý do:

## **1. 80% câu hỏi yêu cầu xử lý nhiều frame → không được phép chỉ nhìn 1 frame**

Đếm biển, nhận dạng biển, hiểu hướng… đều cần **video-level reasoning**.

## **2. Vintern không hiểu layout biển chỉ dẫn Việt Nam**

* Không hiểu cấu trúc biển xanh (2–3 hàng chữ, mũi tên + khoảng cách).
    
* Không đọc được text tiếng Việt trên biển nhỏ.
    

## **3. Keyframe selection thường bỏ mất biển báo quan trọng**

Rất nhiều biển chỉ xuất hiện **0.2–0.4 s**, YOLO miss → coi như thua.

## **4. Multi-class sign detection không thể rely vào CLIP**

CLIP match câu hỏi ↔ frame không chính xác khi biển nhỏ (far object).

* * *

# III. GIẢI PHÁP MỚI — CHUẨN 2025, KHÔNG KEYFRAME

Đây là kiến trúc chính xác để đạt top 1 leaderboard:

* * *

# **A. TRÍCH XUẤT TOÀN BỘ BIỂN BÁO → “SIGN TRACKING”**

(đây là thay thế keyframe)

## **Bước 1: Quét 100% frame bằng YOLO (traffic-sign model)**

* Kích hoạt `stream=True`
    
* Mỗi detection → lưu:
    
    * class
        
    * bounding box
        
    * frame index
        
    * crop ảnh
        

## **Bước 2: Tracking biển báo (SORT/ByteTrack)**

→ Gom 1 biển xuất hiện nhiều frame thành 1 object duy nhất.

### Output:

```json
{
  "sign_id": 12,
  "frames": [100, 101, 102, ...],
  "class": "warning",
  "crop": "image",
  "ocr_text": "200m"
}
```

→ Bạn đã có **danh sách biển cuối cùng** từ video.

* * *

# **B. OCR BIỂN CHỈ DẪN (BLUE SIGN OCR PIPELINE)**

Biển chỉ dẫn màu xanh là phần khó nhất → phải làm pipeline riêng:

## **Step OCR dành riêng cho biển xanh**

1. Color mask (HSV) để lấy vùng xanh
    
2. YOLO detect biển xanh
    
3. Crop
    
4. Enhancement (CLAHE + Sharpen)
    
5. VietOCR / Rosetta / EasyOCR model optimize cho biển chữ trắng
    

* * *

# **C. XỬ LÝ CÂU HỎI → KHÔNG DÙNG VINTERN**

Bạn không cần LLM reasoning.

Tất cả câu hỏi đã có **pattern cố định** → build rule-based parser:

## **Intent classifier 9 lớp**

* COUNT_SIGNS
    
* LIST_SIGNS
    
* CHECK_SIGN_EXISTS
    
* GET_DIRECTION
    
* GET_DISTANCE
    
* LIGHT_STATE
    
* LANE_RULE
    
* SPEED_RULE
    
* RESTRICTION_RULE
    

→ HuggingFace DistilBERT 40 MB là đủ.

* * *

# **D. HỆ LUẬT (RULE ENGINE) CHO TỪNG DẠNG**

### Ví dụ:

**Q: “Trong video có bao nhiêu biển cảnh báo nguy hiểm?”**

→ rule:

```python
count(sign.class == "warning")
```

**Q: “Theo biển báo trong video, cầu Phú Mỹ cách bao nhiêu km?”**

→ rule:

```python
read signs where class == "guide_blue" and text contains "Phú Mỹ"
extract numbers (regex)
```

**Q: “Hướng đi chếch phải là vào đường nào?”**

→ rule:

* detect biển chỉ dẫn
    
* match arrow direction (“↗”)
    
* OCR text tương ứng
    

* * *

# IV. 10 GIÂY INFERENCE — PIPELINE TỐI ƯU NHẤT

Để đáp ứng giới hạn 10 s:

## 1. YOLO 11 n-traffic-sign (custom, 15 ms/frame)

→ detect 100% frame.

## 2. Tracking (ByteTrack) — cực nhanh.

## 3. OCR chỉ khi có biển cần đọc.

## 4. Answering dùng LLM

https://huggingface.co/datasets/5CD-AI/Viet-OCR-VQA/tree/main

* * *

# V. CHỐT LẠI – HƯỚNG LÀM CHÍNH XÁC NHẤT

**Thay keyframe → chuyển sang SIGN TRACKING + OCR + RULE ENGINE.**

Bạn sẽ lấy được:

* đầy đủ biển
    
* đầy đủ text
    
* đầy đủ group (cấm / cảnh báo / chỉ dẫn / tốc độ)
    
* có thể đếm
    
* có thể suy luận hướng
    
* có thể trả lời 100% câu hỏi dạng A/B/C/D