# 1. Label Studio là gì?
[https://labelstud.io/](https://labelstud.io/)
Label Studio là một nền tảng mã nguồn mở cho phép tạo, quản lý và vận hành quy trình gán nhãn (annotation) dữ liệu đa phương thức. Công cụ này được thiết kế để phục vụ các pipeline AI/ML, đặc biệt trong các tác vụ cần dữ liệu có cấu trúc như computer vision, NLP, audio processing, video, time-series.

Cốt lõi của Label Studio là cho phép xây dựng **giao diện gán nhãn linh hoạt bằng XML**, kết hợp với một API mạnh để mở rộng thành các workflow tự động, semi-automatic hoặc kết hợp mô hình học máy.

Điểm mạnh quan trọng:

- Hỗ trợ nhiều loại dữ liệu: ảnh, text, audio, video, HTML, PDF, multi-modality.
    
- UI annotation gần như tùy biến toàn bộ.
    
- Kết nối với mô hình máy học (ML backend).
    
- Dễ tích hợp vào MLOps pipeline.
    
- Có giao diện quản trị project, người dùng, phân công task.
# 2. Sử dụng Label Studio
Cài đặt:
```python
# Install the package
# into python virtual environment
pip install -U label-studio
# Launch it!
label-studio
```

Sau đó được một giao diện như vầy:
![[image-195.png]]
Nếu chưa Sign up thì đăng ký và nhập Email, mật khẩu rất đơn giản.
Đây là giao diện khi đăng nhập:
![[image-196.png]]

Sau đó tạo project mới và import data bằng file json hoặc file csv.
Tùy theo task chúng ta thực hiện mà sẽ setting để bắt đầu label:
![[image-197.png]]

Ví dụ trên đây là set up cho task NER

Lúc này khi về lại Project và select từng mẫu chúng ta có thể label được với giao diện như sau:
![[image-198.png]]






## **Bài tập về nhà 13-11**
Dùng label studio gán thẻ 
Ngoài ra, nên cài thêm cái này (thư viện dịch Thuật trực tiếp en-vi cho Python):
https://pypi.org/project/optimumEasyNMT/?Utm_source=chatgpt.com#:~:text=Installation%20for%20Python
Và nên cài bộ từ điển 5 GB (cuộn cuối trang, có Tên thư viện):
https://pypi.org/project/optimumEasyNMT/?Utm_source=chatgpt.com

- Một nhóm NER
- Một nhóm POS

