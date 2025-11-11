# Track 2
# 1. Mô tả đề và nội quy
![[image-184.png]]

**Track 2**
![[image-185.png]]
![[image-186.png]]
![[image-187.png]]

Dữ liệu mẫu:
![[image-188.png]]
![[image-189.png]]
![[image-190.png]]

# 2. Kinh nghiệm các cuộc thi
## Xếp hạng
- 50 - 60%: Baseline, kéo model open source về chạy là được
- 60 - 70%: Augmentation, Ensemble learning
- 70 - 80%: Engineering expert -> Trình độ vjp pro
-  90%: Không thể vì thường các cuộc thi thiết kế outlier gây nhiễu

> [!Note] Thảo luận về outlier
> Thường thì trong lý thuyết khi gặp outlier thì phải xóa để tránh nhiễu mô hình nhưng trong thực tế đây là những giá trị mô hình phải nhận biết được.
> 
> Nếu chúng ta bỏ outlier ra thì mô hình không thể dự đoán được những điểm đó.

Ở đây có rất nhiều hướng tiếp cận để cải thiện kết quả, tùy thuộc vào kinh nghiệm và phong cách để xử lý:
- Có thể là chỉ cần finetune mô hình (nhưng chọn đúng model tốt)
- Phân chia thành các thành phần nhỏ để cải thiện -> Phức tạp
# 3. Pipeline
Mô hình VLM
Từ keyframe chia thành 2 nhánh:
- Trên ảnh
- Trên text
Dùng model Vintern đã được traning rất nhiều trên dữ liệu Việt Nam.

Về lý thuyết nên lấy hết video để train chứ không train trên keyframe bất kì.
![[image-181.png]]

Phân tích đúng việc chọn Model để train:
- Lấy danh sách các model
- Dùng pre-train để đánh giá trên tập cuộc thi
- Dựa vào kết quả, chọn model tốt nhất để train.

Cách set up data để train cho model reasoning về luật giao thông
![[image-182.png]]

![[image-183.png]]

Đây là bài phụ thuộc vào cách chúng ta setup data để model reasoning, khi bạn setup tốt thì model sẽ học được tốt và output sẽ tăng.
-> Sử dụng format chúng ta có thể điều khiển và cân bằng tốt để setup.

Dùng google ai studio để generate data reasoning.

Unsloth:

### Hướng tiếp cận ensemble learning
Lấy 5 model 1.5 B, lấy kết quả cuối cùng của 5 model và chọn ra model tốt nhất.
Hướng ensemble bên trong mô hình cần phải đầu tư thời gian học và nghiên cứu nhiều hơn.


Thực hiện trên LoRA 5000 4 B là ổn
SST zalo data và 
RL chỉ train trên zalo data


## Các hướng nên tránh
Bài toán này không nên đi theo hướng xử lý ảnh vì performance sẽ tăng không nhiều, các model hiện tại đã rất tốt ở xử lý ảnh.

Không nên dùng RAG vì đã sử dụng các model 7 B (tức ra rất tốt rồi)

## Các hướng nên thực hiện
- Ensemble learning
- Reinformance learning
## Các keyword cần tìm hiểu
- SST
- RL

# Phần cứng
Dùng trên google colab T 4 là ngang với cấu hình của btc đề xuất rồi.
