Phần này giới thiệu khái niệm **độ tương tự (similarity)** và **độ khác biệt (dissimilarity)** trong khai phá dữ liệu.

- Trong các ứng dụng như **phân cụm (clustering)**, **phát hiện ngoại lệ (outlier detection)** và **phân loại láng giềng gần nhất (nearest-neighbor classification)**, việc đo mức độ giống nhau giữa các đối tượng là rất quan trọng.
    
- Mục tiêu là **nhóm các đối tượng có đặc điểm tương đồng** (ví dụ: thu nhập, khu vực sống, độ tuổi của khách hàng), để tìm ra các **cụm (clusters)** — mỗi cụm gồm các đối tượng tương tự nhau, và khác biệt với đối tượng ở cụm khác.
    
- Phân tích tương tự cũng giúp **xác định ngoại lệ** — các đối tượng khác biệt rõ rệt so với phần lớn dữ liệu.
    
- Ngoài ra, độ tương tự còn được dùng trong **phân loại dựa trên láng giềng gần nhất (nearest-neighbor classification)**, khi cần gán nhãn cho một đối tượng mới dựa trên sự giống nhau với các đối tượng đã biết.
    
**Ví dụ minh họa:**

- **Hình 2.21**: Đám mây từ khóa (tag cloud) cho thấy các chủ đề phổ biến được gắn thẻ trên web, thể hiện mức độ tương tự giữa các chủ đề theo tần suất xuất hiện.
    ![[image-132.png]]
- **Hình 2.22**: Biểu đồ ảnh hưởng bệnh tật của người trên 20 tuổi (NHANES dataset) cho thấy mối quan hệ giữa các bệnh — những bệnh thường xuất hiện cùng nhau được xem là “tương tự” hơn.
	![[image-133.png]]

# Data Matrix vs. Dissimilarity Matrix
### **1. Data Matrix (ma trận dữ liệu)**

* Còn gọi là **object-by-attribute structure** hoặc **two-mode matrix**.
    ![[image-135.png]]

* Lưu trữ giá trị của các thuộc tính (attributes) cho từng đối tượng (objects).
    
* Có kích thước $n \times p$, trong đó:
    
    * $n$: số lượng đối tượng
        
    * $p$: số thuộc tính (hoặc đặc trưng – features)
        
* Mỗi hàng biểu diễn một **đối tượng** (object), mỗi cột biểu diễn một **thuộc tính**.
    
* Ký hiệu:
    
    $$\mathbf{x}_i = (x_{i1}, x_{i2}, \dots, x_{ip})$$
    
    là vector thuộc tính của đối tượng $i$.
    
* Ví dụ: tập dữ liệu về khách hàng với các đặc trưng như **tuổi, thu nhập, giới tính**.
    

* * *

### **2. Dissimilarity Matrix (ma trận độ khác biệt)**

* Còn gọi là **object-by-object structure** hoặc **one-mode matrix**.
    ![[image-134.png]]

* Lưu trữ giá trị đo **độ khác biệt (dissimilarity)** giữa từng cặp đối tượng $(i, j)$.
    
* Có kích thước $n \times n$.
    
* Phần tử $d(i, j)$ biểu diễn mức độ khác biệt giữa hai đối tượng:
    
    * $$d(i, j) \ge 0$$
        
    * $$d(i, i) = 0$$
        
    * $d(i, j) = d(j, i)$ (ma trận đối xứng)
        
* Nếu hai đối tượng giống nhau → $d(i, j) = 0$; càng khác nhau → giá trị càng lớn.
    
* Dùng trong nhiều thuật toán phân cụm và tìm kiếm tương tự.
    

* * *

### **3. Mối quan hệ giữa Similarity và Dissimilarity**

Hai khái niệm này **liên hệ nghịch đảo**:

$$sim(i, j) = 1 - d(i, j)$$

* $sim(i,j)$: độ tương tự giữa hai đối tượng
    
* $d(i,j)$: độ khác biệt giữa hai đối tượng
    
* $sim(i,j) = 1$ khi hai đối tượng giống hệt nhau; giảm dần khi chúng khác nhau.
    
# **Proximity Measures for Nominal Attributes**

### **1. Định nghĩa**

* **Nominal attribute** là thuộc tính có các giá trị (state) **phân loại** rời rạc, không có thứ tự.  
    → Ví dụ: `map_color` có thể có các giá trị `{red, yellow, green, pink, blue}`.
    
* Số lượng trạng thái của thuộc tính ký hiệu là $M$. Các giá trị có thể được biểu diễn bằng **chữ, ký hiệu, hoặc số nguyên** chỉ để xử lý dữ liệu — **không biểu thị thứ tự.**
    

* * *

### **2. Đo độ khác biệt (Dissimilarity)**

Giữa hai đối tượng $i$ và $j$ có $p$ thuộc tính, ta tính:

$$d(i,j) = \frac{p - m}{p}$$

* $m$: số lượng thuộc tính mà hai đối tượng **giống nhau (match)**
    
* $p$: tổng số thuộc tính mô tả đối tượng
    

Nói cách khác, **dissimilarity = tỷ lệ các thuộc tính khác nhau**.  
Nếu có trọng số cho từng thuộc tính, có thể điều chỉnh công thức để tăng ảnh hưởng của thuộc tính quan trọng hoặc có nhiều trạng thái hơn.

![[image-136.png]]
![[image-137.png]]
### **4. Độ tương tự (Similarity)**

Có thể tính tương tự từ công thức nghịch đảo:

$$sim(i,j) = 1 - d(i,j) = \frac{m}{p}$$

→ Là **tỷ lệ các thuộc tính giống nhau**.

* * *

### **5. Mã hóa thay thế (Alternative Encoding)**

Nominal attributes có thể chuyển đổi sang dạng **binary attribute** để tính khoảng cách:

* Với mỗi trạng thái (value) của thuộc tính danh nghĩa, tạo **một biến nhị phân**.
    
* Nếu đối tượng mang giá trị đó → đặt 1; ngược lại → 0.  
    Ví dụ:  
    `map_color` có 5 màu {red, yellow, green, pink, blue} → tạo 5 cột tương ứng.  
    Nếu đối tượng có màu `yellow`, thì vector = (0, 1, 0, 0, 0).
    

Sau khi mã hóa nhị phân, có thể áp dụng các công thức **dissimilarity** hoặc **similarity** đã được trình bày cho binary attributes ở phần tiếp theo (2.4.3).
# Proximity Measures for Binary Attributes
### **1. Khái niệm cơ bản**

* **Binary attribute** chỉ có hai trạng thái:
    
    * 1 → thuộc tính **hiện diện (present)**
        
    * 0 → thuộc tính **vắng mặt (absent)**  
        Ví dụ: thuộc tính _smoker_:
        
    * 1 = bệnh nhân có hút thuốc
        
    * 0 = bệnh nhân không hút thuốc
        

→ Không nên xem binary như giá trị số vì 0 và 1 không có tính liên tục.

* * *

### **2. Các loại binary attributes**

Có hai loại chính:

* **Symmetric binary attributes:**  
    Hai trạng thái 0 và 1 **quan trọng như nhau**.  
    → Dùng cho các đặc điểm như _giới tính (male/female)_.
    
* **Asymmetric binary attributes:**  
    Trạng thái 1 mang nhiều ý nghĩa hơn 0 (1 = “có”, 0 = “không”).  
    → Dùng cho các thuộc tính như _có bệnh, có triệu chứng, kết quả dương tính,…_
    

* * *

### **3. Bảng 2×2 Contingency Table**

| Object i | Object j = 1 | Object j = 0 | Tổng |
| --- | --- | --- | --- |
| **1** | q | r | q + r |
| **0** | s | t | s + t |
| **Tổng** | q + s | r + t | p (= q + r + s + t) |

Giải thích:

* $q$: số thuộc tính mà cả hai đều = 1
    
* $r$: thuộc tính = 1 ở $i$, nhưng = 0 ở $j$
    
* $s$: thuộc tính = 0 ở $i$, nhưng = 1 ở $j$
    
* $t$: thuộc tính mà cả hai đều = 0
    
* $p$: tổng số thuộc tính
    

* * *

### **4. Symmetric Binary Dissimilarity**

Với symmetric attributes (0 và 1 có giá trị tương đương):

$$d(i,j) = \frac{r + s}{q + r + s + t}$$

* Dissimilarity tăng khi số lượng thuộc tính khác nhau (r + s) tăng.
    

* * *

### **5. Asymmetric Binary Dissimilarity**

Với asymmetric attributes (0 không quan trọng, chỉ quan tâm đến 1):

$$d(i,j) = \frac{r + s}{q + r + s}$$

→ Bỏ qua các cặp (0,0) vì không mang thông tin.  
Đây là **Jaccard distance**.

* * *

### **6. Asymmetric Binary Similarity**

Từ đó, độ tương tự (similarity) được định nghĩa là:

$$sim(i,j) = 1 - d(i,j) = \frac{q}{q + r + s}$$

→ Gọi là **Jaccard coefficient**, rất phổ biến trong khai phá dữ liệu.

Ví dụ:
![[image-138.png]]
→ **Jack và Mary** có độ khác biệt thấp nhất → tương tự nhất (có thể mắc bệnh giống nhau).  
**Jim và Mary** khác biệt nhất → ít khả năng có cùng bệnh.

# Dissimilarity of Numeric Data: Minkowski Distance

### **2. Chuẩn hóa dữ liệu trước khi tính khoảng cách**

Trước khi tính toán, dữ liệu số thường được **chuẩn hóa** để tránh sự chênh lệch về đơn vị đo hoặc thang giá trị.  
Ví dụ: chiều cao có thể đo bằng cm hoặc inch, nếu không chuẩn hóa thì thuộc tính này sẽ ảnh hưởng quá lớn đến kết quả.

Các cách chuẩn hóa phổ biến:

* Thu nhỏ dữ liệu về khoảng $[-1,1]$ hoặc $[0,1]$.
    
* Nhằm **đảm bảo mỗi thuộc tính có trọng số tương đương** khi tính khoảng cách.
    

* * *

### **3. Euclidean Distance (L₂ norm)**

$$d(i,j) = \sqrt{(x_{i1}-x_{j1})^2 + (x_{i2}-x_{j2})^2 + \cdots + (x_{ip}-x_{jp})^2}$$

* Là khoảng cách “đường chim bay” giữa hai điểm trong không gian p-chiều.
    
* Dùng phổ biến nhất khi các thuộc tính độc lập và cùng thang đo.
    

* * *

### **4. Manhattan Distance (L₁ norm)**

$$d(i,j) = |x_{i1}-x_{j1}| + |x_{i2}-x_{j2}| + \cdots + |x_{ip}-x_{jp}|$$

* Còn gọi là **city block distance**, giống như đi theo lưới đường phố.
    
* Dùng tốt khi chỉ muốn tính tổng chênh lệch tuyệt đối, không quan tâm hướng.
    

* * *

### **5. Các tính chất của một “metric”**

Cả hai khoảng cách Euclidean và Manhattan đều thỏa mãn:

1. **Không âm:** $d(i,j) \ge 0$
    
2. **Phản xạ:** $d(i,i) = 0$
    
3. **Đối xứng:** $d(i,j) = d(j,i)$
    
4. **Bất đẳng thức tam giác:** $d(i,j) \le d(i,k) + d(k,j)$
    

* * *

### **6. Minkowski Distance (Lₚ norm) – Tổng quát hóa**

$$d(i,j) = \left( \sum_{f=1}^{p} |x_{if} - x_{jf}|^h \right)^{1/h}$$

với $h \ge 1$.

* $h=1 \Rightarrow$ **Manhattan distance**
    
* $h=2 \Rightarrow$ **Euclidean distance**
    
* $h\to\infty \Rightarrow$ **Supremum distance (Chebyshev)**
    

* * *

### **7. Supremum (Chebyshev) Distance**

$$d(i,j) = \lim_{h\to\infty}\left(\sum_{f=1}^{p} |x_{if}-x_{jf}|^h \right)^{1/h}  
= \max_{f}|x_{if}-x_{jf}|$$

* Lấy **độ chênh lệch lớn nhất** giữa hai đối tượng.
    
* Thường dùng khi chỉ quan tâm tới thuộc tính có sai khác lớn nhất.
    

* * *

### **8. Ví dụ minh họa**
Hai điểm $x_1=(1,2)$, $x_2=(3,5)$:
![[image-139.png]]


* * *

### **9. Weighted Euclidean Distance**

Nếu mỗi thuộc tính có **độ quan trọng khác nhau**, gán trọng số $w_f$:

$$d(i,j) = \sqrt{ w_1|x_{i1}-x_{j1}|^2 + w_2|x_{i2}-x_{j2}|^2 + \cdots + w_p|x_{ip}-x_{jp}|^2 }$$

→ Có thể áp dụng tương tự cho các loại khoảng cách khác.

# Proximity Measures for Ordinal Attributes
### **1. Đặc điểm của Ordinal Attributes**

* **Ordinal attribute** là thuộc tính **có thứ tự (ranking)** giữa các giá trị,  
    nhưng **không biết rõ mức chênh lệch** giữa các mức kế tiếp.  
    Ví dụ: _small < medium < large_ hoặc _fair < good < excellent_.
    
* Có thể thu được từ việc **rời rạc hóa dữ liệu số** (discretization),  
    Ví dụ: _nhiệt độ (°C)_ chia thành _cold, moderate, warm_.
    

* * *

### **2. Ký hiệu**

* Gọi $M_f$: số lượng thứ hạng có thể có của thuộc tính $f$.  
    Mỗi trạng thái được gán hạng $1, 2, \ldots, M_f$.
    

* * *

### **3. Các bước tính độ khác biệt cho thuộc tính Ordinal**

#### **Bước 1:**

Thay thế giá trị gốc $x_{if}$ của đối tượng $i$ bằng **thứ hạng tương ứng**:

$$r_{if} \in \{1, 2, \ldots, M_f\}$$

#### **Bước 2:**

Vì các thuộc tính ordinal có thể có số lượng thứ hạng khác nhau,  
nên cần **chuẩn hóa** về cùng khoảng $[0,1]$ để tránh chênh lệch trọng số.  
Công thức chuẩn hóa:

$$z_{if} = \frac{r_{if} - 1}{M_f - 1}$$

→ Sau chuẩn hóa, giá trị nhỏ nhất = 0, lớn nhất = 1.

#### **Bước 3:**

Tính **dissimilarity** giữa các đối tượng bằng bất kỳ khoảng cách cho dữ liệu số,  
Thường dùng **Euclidean distance** hoặc **Manhattan distance** (theo công thức mục 2.4.4),  
với giá trị đã chuẩn hóa $z_{if}$.

* * *

### **4. Ví dụ 2.21**

Bảng dữ liệu (rút từ Table 2.2):

| Object | test-2 (ordinal) |
| --- | --- |
| 1 | excellent |
| 2 | fair |
| 3 | good |
| 4 | excellent |
![[image-140.png]]
Giả sử $M_f = 3$ với thứ hạng:  
**fair = 1**, **good = 2**, **excellent = 3**

**Bước 1:** thay thế bằng rank → $r = [3, 1, 2, 3]$  
**Bước 2:** chuẩn hóa về [0,1]:

$$z = \frac{r - 1}{M_f - 1} = [1.0, 0.0, 0.5, 1.0]$$

**Bước 3:** dùng Euclidean distance → ma trận độ khác biệt:

$$\begin{bmatrix}  
0 & 1.0 & 0.5 & 0\\  
1.0 & 0 & 0.5 & 1.0\\  
0.5 & 0.5 & 0 & 0.5\\  
0 & 1.0 & 0.5 & 0  
\end{bmatrix}$$

→ Các cặp (1,2) và (2,4) khác biệt nhất ($d=1.0$),  
Vì “excellent” và “fair” nằm ở hai đầu dải giá trị.

* * *
# Dissimilarity for Attributes of Mixed Types
### **1. Vấn đề đặt ra**

Các mục trước (2.4.2–2.4.5) đã trình bày cách tính **độ khác biệt (dissimilarity)** cho từng loại thuộc tính **đơn lẻ**:

* Nominal
    
* Symmetric / Asymmetric binary
    
* Numeric
    
* Ordinal
    

Tuy nhiên, trong thực tế, nhiều **bộ dữ liệu thực** (như hồ sơ khách hàng, bệnh nhân, sản phẩm,...) thường chứa **thuộc tính hỗn hợp (mixed types)** — tức là cùng lúc bao gồm cả định tính và định lượng.

Ví dụ:

| ID | Gender | Age | Income | Satisfaction |
| --- | --- | --- | --- | --- |
| 1 | Male | 25 | 700 | High |
| 2 | Female | 35 | 400 | Low |

→ Gồm binary (Gender), numeric (Age, Income) và ordinal (Satisfaction).

* * *

### **2. Cách tiếp cận**

Có hai hướng chính:

1. **Phân tích riêng từng nhóm thuộc tính theo kiểu dữ liệu**,  
    rồi tổng hợp kết quả — tuy nhiên cách này hiếm khi cho kết quả nhất quán.
    
2. **Phương pháp tổng hợp (preferred approach):**  
    Kết hợp toàn bộ các thuộc tính trong một phân tích duy nhất,  
    Bằng cách **chuẩn hóa từng loại thuộc tính** để đưa về cùng **thang đo [0,1]**,  
    Sau đó tính **độ khác biệt trung bình có trọng số** giữa các đối tượng.
    

* * *

### **3. Công thức tổng quát**

Giả sử có $p$ thuộc tính (mixed type), độ khác biệt giữa hai đối tượng $i$ và $j$ được tính:

$$d(i,j) =  
\frac{\sum_{f=1}^{p} \delta_{ij}^{(f)} d_{ij}^{(f)}}  
{\sum_{f=1}^{p} \delta_{ij}^{(f)}}$$

Trong đó:

* $d_{ij}^{(f)}$: độ khác biệt của thuộc tính $f$ giữa hai đối tượng $i, j$.
    
* $\delta_{ij}^{(f)}$: chỉ số chỉ định có tính thuộc tính này hay không:
    
    * $\delta_{ij}^{(f)} = 0$ nếu dữ liệu thiếu (missing value),  
        hoặc $x_{if}=x_{jf}=0$ đối với **asymmetric binary**.
        
    * $\delta_{ij}^{(f)} = 1$ trong các trường hợp còn lại.
        

* * *

### **4. Cách tính $d_{ij}^{(f)}$ cho từng loại thuộc tính**

| Kiểu thuộc tính    | Công thức                                                                                | Giải thích                    |
| ------------------ | ---------------------------------------------------------------------------------------- | ----------------------------- |
| **Numeric**        | Distance                                                                                 | $x_{if} - x_{jf}$             |
| **Nominal/Binary** | $d_{ij}^{(f)} = 0$ nếu $x_{if}=x_{jf}$, ngược lại $=1$                                   | Giống nhau = 0, khác nhau = 1 |
| **Ordinal**        | Tính thứ hạng $r_{if}$, chuẩn hóa $z_{if} = \frac{r_{if}-1}{M_f-1}$, rồi xem như numeric | Dùng công thức như dữ liệu số |

* * *

### **5. Ví dụ 2.22 – Dissimilarity giữa thuộc tính hỗn hợp**

Dữ liệu (Table 2.2):

| Object | test-1 (nominal) | test-2 (ordinal) | test-3 (numeric) |
| --- | --- | --- | --- |
| 1 | code A | excellent | 45 |
| 2 | code B | fair | 22 |
| 3 | code C | good | 64 |
| 4 | code A | excellent | 28 |

**Cách làm:**

* test-1 (nominal): dùng công thức (2.11)
    
* test-2 (ordinal): chuẩn hóa như (2.21)
    
* test-3 (numeric): chuẩn hóa min–max (22–64)
    

Sau khi tính riêng cho từng loại, ghép lại bằng công thức (2.22).

Ví dụ:

$$d(3,1) = \frac{(1\times1.0)+(1\times0.50)+(1\times0.45)}{3} = 0.65$$

→ Dissimilarity matrix cuối cùng:

$$\begin{bmatrix}  
0 & 0.85 & 0.65 & 0.13\\  
0.85 & 0 & 0.83 & 0.71\\  
0.65 & 0.83 & 0 & 0.79\\  
0.13 & 0.71 & 0.79 & 0  
\end{bmatrix}$$

* * *

### **6. Kết luận từ ví dụ**

* **Objects 1 và 4**: nhỏ nhất $d=0.13$ → **giống nhau nhất** (cùng code A, cùng excellent).
    
* **Objects 1 và 2**: lớn nhất $d=0.85$ → **khác nhau nhất** (khác code, khác test-2, chênh nhiều ở test-3).

![[image-141.png]]