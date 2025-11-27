---
draft: true
---

Các chủ đề:
- Text classification (Ứng dụng trong ....)
- Sentiment Analysis (Ứng dụng trong....)
- NER/IE/RE
- POS tagging
- Topic modelling

Yêu cầu:
- Mô tả rõ quy trình scrawl data (Tiếng Việt)
- Khuyến khích gán nhãn thủ công -> Dùng Label Studio: Mô tả cách thức gán nhãn, không dùng dataset bên ngoài.
- Sử dụng 3 mô hình conventional ML (đa dạng chủng loại mô hình)
- Sử dụng 1 deep learning
- Phân tích đánh giá so sánh
- Ứng dụng cụ thể (Gradio)

Có thể dùng thêm EasyNTM để hỗ trợ dịch thuật, so sánh hai phiên bản để đánh giá.

Báo cáo:
- 40-50 trang
***
# Giới thiệu về các bài toán

| Bài toán                | Loại học          | Đầu ra             | Ứng dụng chính                | Độ phức tạp     |
| ----------------------- | ----------------- | ------------------ | ----------------------------- | --------------- |
| **Text Classification** | Supervised        | Nhãn phân loại     | Phân loại tin, spam, nội dung | Trung bình      |
| **Sentiment Analysis**  | Supervised        | Nhãn cảm xúc       | Đánh giá cảm xúc, marketing   | Trung bình      |
| **NER / IE / RE**       | Supervised / Semi | Thực thể & quan hệ | Knowledge extraction, QA      | Cao             |
| **POS Tagging**         | Supervised        | Loại từ            | Phân tích cú pháp             | Thấp–Trung bình |
| **Topic Modeling**      | Unsupervised      | Cụm chủ đề         | Phân tích văn bản lớn         | Trung bình      |
|                         |                   |                    |                               |                 |

## 2. Text Classification

### 2.1. Bài toán là gì

Text Classification là bài toán gán **một hoặc nhiều nhãn** cho văn bản:

* Spam / Not Spam
    
* Hỏi – Phản hồi – Than phiền (ticket customer service)
    
* Chủ đề bài viết
    

### 2.2. Quy trình thực hiện

1. **Xác định mục tiêu**  
    Ví dụ: phân loại tin nhắn người dùng thành 5 nhóm ý định.
    
2. **Chuẩn bị dữ liệu**
    
    * Thu thập văn bản.
        
    * Thiết kế nhãn.
        
    * Annotate bằng Label Studio hoặc tập public.
        
3. **Tiền xử lý**
    
    * Chuẩn hóa (lowercase, bỏ ký tự thừa).
        
    * Tokenization (BPE, SentencePiece).
        
4. **Huấn luyện mô hình**
    
    * Mô hình baseline: Logistic Regression, SVM.
        
    * Mô hình hiện đại: BERT/PhoBERT, DistilBERT.
        
5. **Đánh giá**
    
    * Accuracy.
        
    * Macro-F 1 cho dữ liệu lệch.
        
6. **Triển khai**
    
    * Model API (FastAPI).
        
    * Integration vào ứng dụng.
        

* * *

## 3. Sentiment Analysis

### 3.1. Bài toán là gì

Phân tích cảm xúc của văn bản:

* Positive / Negative / Neutral
    
* Emotion detection: vui, buồn, giận, lo lắng…
    

### 3.2. Quy trình thực hiện

Tương tự classification nhưng lưu ý:

* Dữ liệu cảm xúc cần chất lượng tốt, tránh nhiễu.
    
* Cân nhắc label ambiguity (ví dụ: sarcastic content khó gán nhãn).
    

Workflow:

* Thu thập review hoặc social posts.
    
* Annotate bằng Label Studio (template Choices).
    
* Huấn luyện bằng mô hình pretrained (PhoBERT / BERT multilingual).
    
* Đánh giá bằng F 1 và confusion matrix để tránh bias.
    

* * *

## 4. NER / IE / RE (Entity – Information Extraction – Relation Extraction)

### 4.1. Bài toán là gì

* **NER**: phát hiện thực thể (PERSON, ORG, LOCATION…).
    
* **IE**: trích xuất thuộc tính từ văn bản (ví dụ: giá, sản phẩm, ngày).
    
* **RE**: trích xuất quan hệ giữa các thực thể.
    

Đây là bài toán nền tảng để xây dựng:

* Hệ thống hỏi đáp.
    
* Tìm kiếm thông minh.
    
* Knowledge graph.
    

### 4.2. Quy trình thực hiện

1. **Thiết kế schema**
    * Danh sách thực thể.
    * Danh sách quan hệ (nếu có).
        
2. **Annotate**  
    Với Label Studio dùng:
    
    ```xml
    <Labels name="ner" toName="text">
      <Label value="PERSON"/>
      <Label value="ORG"/>
    </Labels>
    ```
    
3. **Tiền xử lý**
    
    * Chuyển dữ liệu sang BIO hoặc IOB 2.
        
4. **Huấn luyện**
    
    * Mô hình CRF (baseline).
        
    * BERT + token classification head.
        
5. **Relation Extraction**
    
    * Mô hình sequence-to-sequence (T 5).
        
    * Hoặc mô hình pairwise classification.
        
6. **Đánh giá**
    
    * Precision / Recall / F 1 cho từng entity-type.
        
    * Micro/macro F 1.

``

* * *

## 5. POS Tagging

### 5.1. Bài toán

Gán nhãn loại từ cho từng token: danh từ, động từ, tính từ…  
Dùng để:

* Parsing câu.
    
* Nghiên cứu ngôn ngữ.
    
* Hỗ trợ mô hình ngôn ngữ truyền thống.
    

### 5.2. Quy trình

1. Chuẩn hóa token.
    
2. Annotate POS bằng Label Studio (dạng sequence labeling).
    
3. Dùng mô hình BiLSTM + CRF hoặc BERT.
    
4. Đánh giá accuracy hoặc F 1.
    

* * *

## 6. Topic Modeling

### 6.1. Bài toán

Tự động nhóm văn bản có nội dung tương đồng **không cần nhãn**:

* Phân tích comment hàng nghìn dòng.
    
* Tóm tắt insight thị trường.
    

### 6.2. Quy trình

1. Thu thập văn bản.
    
2. Tách từ → TF-IDF hoặc bag-of-words.
    
3. Áp dụng LDA, NMF hoặc BERTopic.
    
4. Đánh giá bằng coherence score.
    
5. Gán label cho từng topic thủ công hoặc semi-automatic.
    

* * *

## 7. Hướng dẫn chung (workflow áp dụng mọi bài toán NLP)

### 7.1. Bước 1: Xác định mục tiêu và loại bài toán

Đặt câu hỏi:

* Dữ liệu có nhãn không?
    
* Cần supervised hay unsupervised?
    
* Output là gì (label, entity, cluster…)?
    

### 7.2. Bước 2: Thu thập và chuẩn bị dữ liệu

* Crawl dữ liệu hoặc lấy từ DB.
    
* Làm sạch dữ liệu.
    
* Loại bỏ trùng lặp.
    
* Tách train/val/test.
    

### 7.3. Bước 3: Annotate bằng Label Studio

* Tạo project.
    
* Chọn template phù hợp.
    
* Định nghĩa nhãn rõ ràng.
    
* Đảm bảo quality bằng review/consensus.
    

### 7.4. Bước 4: Tiền xử lý dữ liệu

* Tokenization.
    
* Tách câu.
    
* Chuyển format đúng nhu cầu mô hình (BIO, JSONL…).
    

### 7.5. Bước 5: Training baseline

* Logistic Regression / SVM / Naive Bayes.
    
* BiLSTM hoặc CNN.
    
* BERT/PhoBERT fine-tune.
    

### 7.6. Bước 6: Đánh giá

Chọn metric tùy bài toán:

* Accuracy, F 1, Precision/Recall (Classification).
    
* Micro/Macro F 1 (NER).
    
* Coherence (Topic Modeling).
    

### 7.7. Bước 7: Deploy

* Đóng mô hình thành API.
    
* Tích hợp vào ứng dụng.
    
* Kết nối ngược lại với Label Studio để auto-suggest.