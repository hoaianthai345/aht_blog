# 1. Gradient Boost For Regression
**Cho bài toán sau:**
Bảng dữ liệu minh họa cách áp dụng **Gradient Boosting** trong bài toán **hồi quy (regression)**. Ở đây, ta có **tập input** gồm các đặc trưng: _Height, Favorite Color, Gender_ và **output** là _Weight_.

|Height|Favorite Color|Gender|Weight|
|---|---|---|---|
|1.6|Blue|Male|88|
|1.6|Green|Female|76|
|1.5|Blue|Female|56|
|1.8|Red|Male|73|
|1.5|Green|Male|77|
|1.4|Blue|Female|57|
Trong bài toán này, mô hình sẽ **dự đoán cân nặng (Weight)** dựa trên chiều cao, màu sắc yêu thích và giới tính. Các đặc trưng này có thể thuộc dạng số (Height), dạng phân loại (Favorite Color, Gender), do đó cần **mã hóa (encoding)** trước khi huấn luyện.

Gradient Boost sẽ kết hợp nhiều **cây quyết định hồi quy nhỏ** (weak learners) thành một mô hình mạnh, liên tục giảm sai số bằng cách học trên phần **residuals** (phần chênh lệch giữa dự đoán và giá trị thật).
***
## Step 1: Build first tree
Ở bước đầu tiên của **Gradient Boost Regression**, mô hình sẽ khởi tạo bằng cách tính **giá trị trung bình (average)** của biến mục tiêu (target variable – ở đây là _Weight_).

**Tính trung bình**

Tổng cân nặng: $88 + 76 + 56 + 73 + 77 + 57 = 427$

Số mẫu: $n = 6$

Trung bình: $\frac{427}{6} = 71.17$

Kết quả này chính là **nút gốc (root node)** của cây đầu tiên trong Gradient Boosting.
***
**Vì sao lại lấy trung bình?**

Trong hồi quy, mô hình ban đầu chưa có bất kỳ đặc trưng nào để học, nên cách tốt nhất để khởi tạo dự đoán là dùng **giá trị trung bình** của toàn bộ tập dữ liệu. Trung bình sẽ giúp **giảm tổng sai số bình phương (MSE)** so với nếu chọn ngẫu nhiên một giá trị nào khác.

***
## Step 2 – Xây dựng cây thứ hai

Sau khi khởi tạo bằng giá trị trung bình **71.17** ở cây đầu tiên, Gradient Boosting sẽ bắt đầu cải thiện dự đoán bằng cách xây dựng **cây thứ hai**.

### Nguyên tắc thực hiện

1. **Tính residuals (sai số còn lại):**  
    Residual = Giá trị thực – Giá trị dự đoán ban đầu.  
    Với dự đoán ban đầu = 71.17 cho tất cả các mẫu:

    |Height|Favorite Color|Gender|Weight|Residual = Weight – 71.17|
    |---|---|---|---|---|
    |1.6|Blue|Male|88|+16.83|
    |1.6|Green|Female|76|+4.83|
    |1.5|Blue|Female|56|-15.17|
    |1.8|Red|Male|73|+1.83|
    |1.5|Green|Male|77|+5.83|
    |1.4|Blue|Female|57|-14.17|

2. **Huấn luyện cây thứ hai** trên các residuals này.
    
    - Thay vì dự đoán trực tiếp _Weight_, cây sẽ cố gắng dự đoán _residual_.
        
    - Điều này giúp mô hình học được **những gì trung bình chưa thể giải thích**.
        
3. **Kết hợp dự đoán:**
    
    - Dự đoán cuối cùng sau bước này = Dự đoán ban đầu + Learning rate × Dự đoán từ cây thứ hai.
        

### Ý nghĩa trực quan

- Cây đầu tiên chỉ “đoán bừa thông minh” bằng trung bình.
    
- Cây thứ hai bắt đầu học các **mối quan hệ phức tạp hơn** từ đặc trưng (Height, Favorite Color, Gender) để sửa lỗi cho cây đầu tiên.


