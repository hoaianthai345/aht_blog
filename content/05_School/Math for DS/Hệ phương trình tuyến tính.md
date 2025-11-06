## 1. Định nghĩa hệ phương trình tuyến tính

Một **hệ phương trình tuyến tính** (Linear System of Equations) có dạng tổng quát:
$$\begin{cases}  
a_{11}x_1 + a_{12}x_2 + \dots + a_{1n}x_n = b_1 \\  
a_{21}x_1 + a_{22}x_2 + \dots + a_{2n}x_n = b_2 \\  
\vdots \\  
a_{m1}x_1 + a_{m2}x_2 + \dots + a_{mn}x_n = b_m  
\end{cases}$$
Trong đó:
* $a_{ij}$ là các hệ số (coefficients),
* $x_1, x_2, \dots, x_n$ là ẩn số,
* $b_i$ là hằng số (kết quả vế phải).

Viết gọn dưới dạng **ma trận**:
$$A \mathbf{x} = \mathbf{b}$$
Với:
* $A \in \mathbb{R}^{m \times n}$ (ma trận hệ số),
* $\mathbf{x} = (x_1, x_2, \dots, x_n)^T$,
* $\mathbf{b} = (b_1, b_2, \dots, b_m)^T$.
* * *

## 2. Các cách giải hệ phương trình tuyến tính

### (a) **Phương pháp thế (Substitution Method)**

* Thường áp dụng với hệ ít phương trình (2 hoặc 3 ẩn).
* Quy trình:
    1. Giải một phương trình để biểu diễn một ẩn theo các ẩn khác.
    2. Thay thế vào các phương trình còn lại.
    3. Tiếp tục cho đến khi tìm được tất cả nghiệm.
Ví dụ:
$$\begin{cases}  
x + y = 5 \\  
2x - y = 1  
\end{cases}$$
→ Từ (1): $y = 5 - x$.  
→ Thế vào (2): $2x - (5-x) = 1 \Rightarrow 3x - 5 = 1 \Rightarrow x=2 \Rightarrow y=3$.
* * *
### (b) **Phương pháp cộng (Elimination Method / Gaussian Elimination)**
* Kết hợp các phương trình để khử dần các ẩn.
* Chính là cơ sở của **phép khử Gauss (Gaussian elimination)**:
    * Dùng phép biến đổi sơ cấp để đưa ma trận về dạng bậc thang.
    * Giải ngược lại từ dưới lên.
Ví dụ:
$$\begin{cases}  
x + y = 5 \\  
2x - y = 1  
\end{cases}$$
Nhân (1) với 2: $2x + 2y = 10$.  
Trừ cho (2): $3y = 9 \Rightarrow y=3, x=2$.
* * *
### (c) **Phương pháp ma trận – Cramer's Rule** (áp dụng cho hệ vuông, $n$ phương trình – $n$ ẩn)
Nếu $\det(A) \neq 0$:
$$x_i = \frac{\det(A_i)}{\det(A)}, \quad i=1,\dots,n$$
Trong đó $A_i$ là ma trận $A$ nhưng thay cột thứ $i$ bằng vector $\mathbf{b}$.  
(Nhược: chỉ áp dụng được khi số phương trình = số ẩn và định thức khác 0.)
* * *
### (d) **Phương pháp ma trận – Nghịch đảo**
Nếu $A$ là ma trận vuông và khả nghịch ($\det(A) \neq 0$):
$$\mathbf{x} = A^{-1} \mathbf{b}$$

* Tính nghịch đảo $A^{-1}$, sau đó nhân với $\mathbf{b}$.
* Ưu điểm: gọn, công thức đẹp.
* Nhược điểm: tính nghịch đảo tốn kém khi hệ lớn (O ($n^3$)).
* * *

### (e) **Phương pháp ma trận – Giả nghịch đảo (Moore–Penrose Pseudoinverse)**

* Dùng khi $A$ **không vuông** hoặc **không khả nghịch**.
    
* Công thức nghiệm bình phương tối thiểu (Least Squares):
    

$$\mathbf{x} = (A^T A)^{-1} A^T \mathbf{b}$$

* Đây chính là công thức trong **Linear Regression**.
    

* * *

### (f) **Phương pháp số (Numerical Methods)**

Khi hệ quá lớn, ta dùng phương pháp lặp:

* **Jacobi Method**
    
* **Gauss–Seidel Method**
    
* **Gradient Descent** (trong ML)
    

Những cách này cho nghiệm gần đúng, nhưng nhanh và phù hợp cho hệ kích thước lớn.

* * *

## 3. Tóm lại

* Hệ nhỏ (2–3 phương trình): dùng **thế** hoặc **cộng đại số**.
    
* Hệ vừa (ma trận vuông, định thức ≠ 0): dùng **Cramer** hoặc **nghịch đảo**.
    
* Hệ lớn, không vuông: dùng **Least Squares (pseudoinverse)**.
    
* Hệ rất lớn: dùng **phương pháp số** (Jacobi, Gauss–Seidel, Gradient Descent).
    
