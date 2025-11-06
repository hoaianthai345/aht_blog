Linear regression là một thuật toán supervised, ở đó quan hệ giữa đầu vào và đầu ra được mô tả bởi một hàm tuyến tính. Thuật toán này còn được gọi là _linear fitting_ hoặc _linear least square_.
# 1. Giới thiệu
Xét bài toán ước lượng giá của một căn nhà:
- rộng $x₁$ (m²),
- $x₂$ phòng ngủ 
- Cách trung tâm thành phố $x₃$ (km). 

Giả sử ta đã thu thập được số liệu từ 1000 căn nhà trong thành phố đó, liệu rằng khi có một căn nhà mới với các thông số về diện tích $x₁$, số phòng ngủ $x₂$ và khoảng cách tới trung tâm $x₃$, chúng ta có thể dự đoán được giá $y$ của căn nhà đó không? Nếu có thì hàm dự đoán $y = f(x)$ sẽ có dạng như thế nào. 
Ở đây, vector đặc trưng $\mathbf{x} = [x₁, x₂, x₃]ᵀ$ là một vector cột chứa thông tin đầu vào, đầu ra y là một số vô hướng.

Một cách trực quan, ta có thể thấy rằng: 
(i) diện tích nhà càng lớn thì giá nhà càng cao; 
(ii) số lượng phòng ngủ càng lớn thì giá nhà càng cao; 
\(iii) càng xa trung tâm thì giá nhà càng giảm. 

Dựa trên quan sát này, ta có thể mô hình quan hệ giữa đầu ra và đầu vào bằng một hàm tuyến tính đơn giản:
$$
y \approx \hat{y} = f(x) = w_1x_1 + w_2x_2 + w_3x_3 = x^T w
$$
trong đó $w = [w₁, w₂, w₃]ᵀ$ là vector hệ số (hoặc trong số–_weight vector_) ta cần đi tìm. Đây cũng chính là tham số mô hình của bài toán. 
Mối quan hệ $y \approx f(x)$ là một _linear regression_ xuất phát từ dữ liệu.

Bài toán trên đây là bài toán dự đoán giá trị của đầu ra dựa trên vector đặc trưng đầu vào (input feature vector). Ngoài ra, giá trị của đầu ra có thể nhận rất nhiều giá trị thực dụng khác nhau. Vì vậy, đây là một bài toán regression. Mối quan hệ $\hat{y} = x^T w$ là một mối quan hệ tuyến tính. Tên gọi _linear regression_ xuất phát từ đây.

**Chú ý:**
1. $y$ là giá trị thực của đầu ra (_ground truth_), trong khi $\hat{y}$ là giá trị đầu ra dự đoán (_predicted output_) của mô hình linear regression. Nhìn chung, $y$ và $\hat{y}$ là hai giá trị khác nhau do có sai số mô hình, tuy nhiên, chúng ta mong muốn rằng sự khác nhau này rất nhỏ.
2. _Linear_ hay _tuyến tính_ hiểu một cách đơn giản là **thẳng, phẳng**. 
	1. Trong không gian hai chiều, một hàm số được gọi là tuyến tính nếu đồ thị của nó có dạng một **đường thẳng**. 
	2. Trong không gian ba chiều, một hàm số được gọi là tuyến tính nếu đồ thị của nó có dạng một **mặt phẳng**. 
	3. Trong không gian nhiều hơn ba chiều, khái niệm mặt phẳng không còn phù hợp nữa, thay vào đó, một khái niệm khác ra đời được gọi là **siêu mặt phẳng (hyperplane)**. 
	4. Các hàm số tuyến tính là các hàm đơn giản nhất, vì chúng thuận tiện trong việc hình dung và tính toán.
# 2. Xây dựng bài toán tối ưu hàm mất mát
## 2.1. Sai số dự đoán
Sau khi đã xây dựng được mô hình dự đoán đầu ra như (7.1), ta cần tìm một phép đánh giá phù hợp với bài toán. Với bài toán regression nói chung, ta mong muốn rằng sự sai khác $e$ giữa giá trị thực $y$ và giá trị dự đoán $\hat{y}$ là nhỏ nhất. Nói cách khác, chúng ta muốn giá trị sau đây càng nhỏ càng tốt:
$$
\frac{1}{2} e^2 = \frac{1}{2}(y - \hat{y})^2 = \frac{1}{2}(y - x^T w)^2
$$
Ở đây ta lấy bình phương vì $e = y - \hat{y}$ có thể là số âm.
Việc sai số là nhỏ nhất có thể được mô tả bằng cách lấy trị tuyệt đối $|e| = |y - \hat{y}|$, tuy nhiên cách làm này ít được sử dụng vì hàm trị tuyệt đối không khả vi tại mọi điểm, không thuận tiện cho việc tối ưu sau này.
Hệ số $\tfrac{1}{2}$ sẽ bị triệt tiêu sau này khi lấy đạo hàm của $e$ theo tham số mô hình $w$.
## 2.2. Hàm mất mát
Điều tương tự xảy ra với tất cả các cặp (_input, output_) $(x_i, y_i), i = 1, 2, \ldots, N$ với $N$ là số lượng dữ liệu quan sát được. Điều chúng ta mong muốn–trung bình sai số là nhỏ nhất–tương đương với việc tìm $w$ để hàm số sau đạt giá trị nhỏ nhất:
$$
\mathcal{L}(w) = \frac{1}{2N} \sum_{i=1}^N (y_i - x_i^T w)^2
$$
Hàm số $\mathcal{L}(w)$ chính là hàm mất mát của linear regression với tham số mô hình $\theta = w$. Chúng ta luôn mong muốn rằng sự mất mát là nhỏ nhất, điều này có thể đạt được bằng cách tối thiểu hàm mất mát theo $w$:
$$
w^* = \underset{w}{\arg\min} \, \mathcal{L}(w) \tag{7.4}
$$

>  `argmin` là _argument of the minimum_.
>  Khi viết $w^* = \arg\min_w \mathcal{L}(w),$ nghĩa là ta tìm giá trị của biến số $w$ sao cho hàm $\mathcal{L}(w)$ đạt giá trị nhỏ nhất.
>  
>  **min**: trả về giá trị nhỏ nhất của hàm số.
>  **argmin**: trả về vị trí (giá trị biến số x) mà tại đó hàm đạt giá trị nhỏ nhất
>  Tương tự với **argmax**

$w^*$ là nghiệm cần tìm, đôi khi dấu * được lượt bỏ đi và nghiệm có thể được viết gọn lại thành $w = \arg\min_w \mathcal{L}(w)$.

> **Trung bình sai số**  
> Việc lấy trung bình (hệ số $\tfrac{1}{N}$) hay lấy tổng trong hàm mất mát, về mặt toán học, không ảnh hưởng tới nghiệm của bài toán. 
> Trong machine learning, các hàm mất mát thường có chứa hệ số tính trung bình theo từng điểm dữ liệu trong tập huấn luyện. Khi tính giá trị của hàm mất mát trên tập kiểm tra, ta cũng thường lấy trung bình lỗi của dữ liệu. 
> Việc lấy trung bình này quan trọng về thống kê nhằm đảm bảo rằng mọi tập dữ liệu có thể thay đổi. Việc tính toán mất mát trên từng điểm dữ liệu sẽ hữu ích cho bước trong việc đánh giá chất lượng mô hình sau này. Ngoài ra, việc lấy trung bình cũng giúp tránh hiện tượng tràn số khi số lượng điểm dữ liệu quá nhiều.

Ta thấy rằng hàm số này có thể được viết gọn lại dưới dạng ma trận, vector, và norm như dưới đây:
$$
\mathcal{L}(w) = \frac{1}{2N} \sum_{i=1}^N (y_i - x_i^T w)^2
= \frac{1}{2N} \left\| 
\begin{bmatrix} 
y_1 \\ y_2 \\ \vdots \\ y_N 
\end{bmatrix} 
- 
\begin{bmatrix} 
x_1^T \\ x_2^T \\ \vdots \\ x_N^T 
\end{bmatrix} w 
\right\|^2_2
= \frac{1}{2N} \| y - X^T w \|_2^2 
$$
Với $y = [y_1, y_2, \ldots, y_N]^T, \; X = [x_1, x_2, \ldots, x_N]$. Như vậy $\mathcal{L}(w)$ là một hàm số liên quan đến bình phương của $\ell_2$ norm.

> [!Note]
> **Norm**: là một hàm đo “độ lớn” hoặc “chiều dài” của một vector. 
> Kí hiệu: $\|x\|$ 
> Các loại **norm**:
> - L 2 norm (Euclidean norm): $\|x\|_2 = \sqrt{x_1^2 + x_2^2 + \cdots + x_n^2}$
> - L 1 norm (Manhattan norm): $\|x\|_1 = |x_1| + |x_2| + \cdots + |x_n|$
> - L∞ norm (Max norm): $\|x\|_\infty = \max_i |x_i|$
> Đọc lại ở bài KNN: [[M03W2.3_Warmup k-Nearest Neighbors (KNN)#Bước 2 Tính khoảng cách|Các loại khoảng cách]]

## 2.3. Nghiệm cho bài toán Linear Regression
Nhận thấy rằng hàm mất mát $\mathcal{L}(w)$ có đạo hàm tại mọi $w$. Vậy việc tìm giá trị tối ưu của $w$ có thể được thực hiện thông qua việc giải phương trình đạo hàm của $\mathcal{L}(w)$ theo $w$ bằng không. May mắn, đạo hàm của hàm mất mát của linear regression rất đơn giản:
$$
\frac{\nabla \mathcal{L}(w)}{\nabla w} = \frac{1}{N} X (X^T w - y)
$$
Giải phương trình đạo hàm bằng không:
$$
\frac{\nabla \mathcal{L}(w)}{\nabla w} = 0 \iff X X^T w = X y
$$
- Nếu ma trận $XX^T$ **khả nghịch**, phương trình trên có nghiệm duy nhất $w = (XX^T)^{-1} Xy$
- Nếu ma trận $XX^T$ **không khả nghịch**, phương trình trên có nghiệm hoặc vô số nghiệm. Lúc này, một nghiệm đặc biệt của phương trình có thể được xác định nhờ vào _giả nghịch đảo_ (pseudo-inverse) của ma trận. 
	- Người ta chứng minh được rằng ma trận $X$, luôn tồn tại duy nhất một giá trị $w$ có $\ell_2$ norm nhỏ nhất giúp tối thiểu $\|X^T w - y\|_F^2$. Cụ thể, $w = (XX^T)^{\dagger} Xy,$ 
		- trong đó $(XX^T)^{\dagger}$ là giả nghịch đảo của $XX^T$. Giả nghịch đảo của một ma trận $A$ luôn luôn tồn tại, thậm chí cả khi ma trận đó không vuông. Khi $A$ là vuông và khả nghịch thì giả nghịch đảo chính là nghịch đảo
Đọc lại: [[1. Ôn tập đại số tuyến tính#4.2. Ma trận nghịch đảo|Ma trận nghịch đảo]]
Nghiệm của bài toán tối ưu là:
$$
w = (XX^T)^{\dagger} Xy
$$ Hàm số tính giả nghịch đảo của một ma trận bất kỳ có sẵn trong thư viện **numpy**.



