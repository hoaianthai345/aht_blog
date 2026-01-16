---
draft:
---


| Tên                                                                                      | Thời gian | Loại model       | Metrics | Note                                                | Link                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------- | --------- | ---------------- | ------- | --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ichigo Whisper                                                                           | 2024      | Speech tokenizer |         |                                                     |                                                                                                                                                                                                                                                       |
| VietASR                                                                                  | 2023      | Model            |         |                                                     | https://github.com/dangvansam/viet-asr.git                                                                                                                                                                                                            |
| ZipFormer 30 M RNNT                                                                      | 2025      | Model            |         |                                                     | https://huggingface.co/hynt/Zipformer-30M-RNNT-6000h?fbclid=IwY2xjawPUHtlleHRuA2FlbQIxMABicmlkETE3djhtYmw5YjhjVm9Ua2tsc3J0YwZhcHBfaWQQMjIyMDM5MTc4ODIwMDg5MgABHjg2oI-BOZl7rj2_pV-HvCEUvxUtG4DoP3M6RRiPoRSlcW38FpYtE-C-uy5O_aem_tJRZd_GWWg8_dp37vh_O6g |
| PhoWhisper                                                                               |           | Model            |         |                                                     | https://arxiv.org/pdf/2406.02555                                                                                                                                                                                                                      |
| ViASR: A Novel Benchmark Dataset and Methods for Vietnamese Automatic Speech Recognition | 2023      |                  |         |                                                     | https://aclanthology.org/2023.paclic-1.38.pdf                                                                                                                                                                                                         |
| Chunkformer ctc large vie                                                                |           |                  |         | Ở đây có link các dataset dùng để train nên rất tốt | https://huggingface.co/khanhld/chunkformer-ctc-large-vie                                                                                                                                                                                              |
| VLSP 2025 ASR-SER From Data Exploration to Model Training: A Strategic Approach          |           |                  |         |                                                     | https://aclanthology.org/2025.vlsp-1.5.pdf                                                                                                                                                                                                            |
|                                                                                          |           |                  |         |                                                     |                                                                                                                                                                                                                                                       |
|                                                                                          |           |                  |         |                                                     |                                                                                                                                                                                                                                                       |
## Ichigo whisper

[**Ichigo Whisper**](https://huggingface.co/Menlo/Ichigo-whisper-v0.1?fbclid=IwY2xjawPT5U1leHRuA2FlbQIxMABicmlkETFNNmk0RjFudmpOSkZEYXVNc3J0YwZhcHBfaWQQMjIyMDM5MTc4ODIwMDg5MgABHgBZTiRh3XEjcaratOReuX5QWXhQVOcz9C8te7j5SbFXRX5PilR6NOIsz5U3_aem_qrTj3zHnIqaMDUWmTmQZrA) là một _speech tokenizer_ mã nguồn mở, gọn nhẹ (22 triệu tham số) dành cho mô hình **Whisper-medium**, được thiết kế nhằm nâng cao hiệu năng trên các ngôn ngữ đa ngữ, đặc biệt là các ngôn ngữ tài nguyên thấp, trong khi vẫn giữ ảnh hưởng tối thiểu đến khả năng tiếng Anh gốc của mô hình.

Không giống các mô hình xuất ra embedding liên tục (_continuous embeddings_), Ichigo Whisper **nén tín hiệu giọng nói thành các token rời rạc (discrete tokens)**, giúp tương thích tốt hơn với các mô hình ngôn ngữ lớn (LLM) cho các tác vụ hiểu tiếng nói trực tiếp.

| Tiêu chí        | Whisper              | Speech Tokenizer    |
| --------------- | -------------------- | ------------------- |
| Output          | Văn bản (text)       | Chuỗi token rời rạc |
| Dạng biểu diễn  | Chữ (string)         | Integer token IDs   |
| Tính liên tục   | Không áp dụng        | Discrete            |
| Phù hợp cho LLM | Gián tiếp (qua text) | Trực tiếp           |

Speech tokenizer này được huấn luyện trên khoảng **~400 giờ dữ liệu tiếng Anh** và **~1000 giờ dữ liệu tiếng Việt**.

Ichigo Whisper là một thành phần cốt lõi trong hệ sinh thái **Ichigo v0.5**.

**Thông tin mô hình**
- **Đơn vị phát triển:** Homebrew Research
    
- **Kiến trúc mô hình:** WhisperVQ
    
- **Loại mô hình:** Bộ lượng tử hóa (Quantizer) cho Whisper
    
- **Ngôn ngữ hỗ trợ:** Tiếng Anh, Tiếng Việt
    
- **Giấy phép:** CC-BY-NC-SA-4.0

 **Thông số huấn luyện (Training Specs)**
Cấu hình phần cứng

|Thành phần|Chi tiết|
|---|---|
|GPU|8 × NVIDIA A6000|


Thời gian huấn luyện

|Giai đoạn|Thời lượng|
|---|---|
|Phase 1|75 giờ (50 epoch)|
|Phase 2|29 giờ (20 epoch)|
|**Tổng cộng**|**104 giờ**|



Phase 1: Có sử dụng KL Loss

|Tham số|Giá trị|
|---|---|
|Phương pháp khởi tạo|Embedding WhisperVQ-Large-v3 (7 ngôn ngữ), có nhân bản|
|Số epoch|50|
|Global Batch Size|336|
|Learning Rate|1e-3|
|Scheduler|Linear warm-up + Cosine decay|
|Optimizer|AdamW|
|Warmup Ratio|500|
|Weight Decay|0.001|
|Độ dài audio tối đa|30 giây (audio được padding)|



Phase 2: Không sử dụng KL Loss

|Tham số|Giá trị|
|---|---|
|Phương pháp khởi tạo|Checkpoint từ Phase 1|
|Số epoch|20|
|Global Batch Size|336|
|Learning Rate|1e-3|
|Scheduler|Linear warm-up + Cosine decay|
|Optimizer|AdamW|
|Warmup Ratio|500|
|Weight Decay|0.001|
|Độ dài audio tối đa|30 giây (audio được padding)|
**Đánh giá (Evaluation)**

Tiếng Việt

|Tên mô hình|Kích thước codebook|Dataset test|Số mẫu test|WER|
|---|---|---|---|---|
|IchigoWhisper|2561|viVoice|10,000|**11.68**|
|Whisper Medium|–|viVoice|10,000|18.30|
Tiếng Anh

|Tên mô hình|Kích thước codebook|Dataset test|Số mẫu test|WER|
|---|---|---|---|---|
|IchigoWhisper|2561|LibriTTS-R|4,689|**11.89**|
|Whisper Medium|–|LibriTTS-R|4,689|13.06|