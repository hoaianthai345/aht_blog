---
Author: HoàiAn Thái
Date: Invalid date
Status: Not started
Tag:
  - NLP
  - Reading
  - WarmUp
---
# Mô tả
TF-IDF (term frequency–inverse document frequency) là một kỹ thuật phổ biến  
trong lĩnh vực xử lý ngôn ngữ tự nhiên và khai thác văn bản. Mục đích của TF-  
IDF là để đánh giá tầm quan trọng của một từ trong một tài liệu so với toàn bộ  
tập hợp các tài liệu (corpus).
# Term Frequency (TF)
Đo lường tần suất xuất hiện của một từ trong một tài liệu. Công thức tính:
$$TF(t, d) = \frac{\text{số lần xuất hiện của từ t trong tài liệu d} }{\text{tổng số từ trong tài liệu d } }$$
# **Inverse Document Frequency (IDF)**
  
$$IDF(t, D) = \log \left( \frac{|D|}{\text{số tài liệu chứa từ t} + 1 } \right)$$
Ghi chú:
- $∣D∣$: tổng số tài liệu trong tập
- Thêm +1 vào mẫu số để tránh chia cho 0
# TF- IDF
Là tích của TF và IDF, giúp xác định tầm quan trọng của một từ trong một tài  
liệu cụ thể, đồng thời giảm thiểu ảnh hưởng của các từ phổ biến nhưng ít mang  
ý nghĩa
$$TF\text{-}IDF(t, d, D) = TF(t, d) \times IDF(t, D)$$
TF-IDF giúp phân biệt các từ quan trọng đối với nội dung của một tài liệu so với  
các từ xuất hiện thường xuyên trong nhiều tài liệu khác nhau nhưng ít mang giá  
trị thông tin.
# Câu hỏi:
**1. Tại sao TF-IDF phải nhân cả TF và IDF, thay vì chỉ dùng một trong hai?**
→ TF đo mức độ **phổ biến trong 1 tài liệu**, IDF đo mức độ **đặc trưng trên toàn bộ tập tài liệu**. Nếu chỉ dùng TF, các từ như “là”, “của”, “và” sẽ được đánh giá cao dù không mang ý nghĩa gì. Nếu chỉ dùng IDF, bạn không biết từ đó có thực sự quan trọng trong **tài liệu đang xét** hay không.
**2. Nếu một từ xuất hiện trong mọi tài liệu thì IDF của nó sẽ bằng bao nhiêu?**
→ Nếu số tài liệu chứ từ $t = |D|$, thì:
$$IDF(t, D) = \log \left( \frac{|D|}{1 + \text{số tài liệu chứa từ } t} \right) \approx \log(1) = 0$$
→ TF-IDF = 0 → nghĩa là **không có giá trị phân biệt**, nên bị loại bỏ.
# Thực hành:
Viết chương trình Python để tính toán giá trị TF-IDF của các từ trong một  
List gồm các câu: ["Tôi thích học AI", "AI là trí tuệ nhân tạo", "AGI là siêu trí tuệ  
nhân tạo"] và sử dụng thư viện numpy để hỗ trợ trong việc tính toán các ma trận.  
```Python
import numpy as np
import math
# Bước 1: Tạo tập tài liệu mẫu
documents = ["Tôi thích học AI", "AI là trí tuệ nhân tạo", 
						 "AGI là siêu trí tuệ nhân tạo"]
# Bước 2: Tiền xử lý - tách từ và tính tần số
def compute_tf (doc) :
		## Your code here ##
	
# Bước 3: Tính toán IDF
def compute_idf (docs):
		## Your code here ##
		
# Bước 4: Tính toán TF-IDF
def compute_tf_idf (tf, idf):
		## Your code here ##
# In kết quả
## Your code here ##

>> Output:
Tài liệu 1:
		Tôi: 0.1014
		thích: 0.1014
		học: 0.1014
		AI: 0.0000
Tài liệu 2:
		AI: 0.0000
		là: 0.0000
		trí: 0.0000
		tuệ: 0.0000
		nhân: 0.0000
		tạo: 0.0000
Tài liệu 3:
		AGI: 0.0579
		là: 0.0000
		siêu: 0.0579
		trí: 0.0000
		tuệ: 0.0000
		nhân: 0.0000
		tạo: 0.0000
```