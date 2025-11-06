---
Author: HoàiAn Thái
Date: Invalid date
Status: Not started
Tag:
  - School
---
> [!important] **Table of Content**
> 
> ---
> 
> - [[#Tham khảo]]
> - [[#1. Giới thiệu]]
> - [[#2. LU and QR decomposition]]
>     - [[#2.1. LÚ decomposition]]
>         - [[#LU trong giải hệ phương trình tuyến tính]]
>         - [[#Điều kiện áp dụng phân rã LU]]
>         - [[#Dùng ma trận hoán vị (LU decomposition with pivoting)]]
>         - [[#Code Python scipy.linalg.lu()]]
>         - [[#lu_factor()]]
>     - [[#2.2. QR Decomposition]]
>         - [[#Thuật toán Gram-Schmidt]]
>         - [[#Giải hệ phương trình A.X = B bằng phân rã Q.R]]
> - [[#3. Eigen decomposition (Phân rã giá trị riêng)]]
>     - [[#3.1. Trị riêng]]
>         - [[#Định nghĩa]]
>         - [[#Phương trình thuần nhất (Homogeneous Linear System)]]
>         - [[#Đa thức đặc trưng (Characteristic Polynomial)]]
>         - [[#Phương trình đặc trưng]]
>     - [[#3.2. Vector riêng]]
>     - [[#3.3. Chéo hóa ma trận (diagonalization)]]
>     - [[#3.x. Phân rã Cholesky (đọc thêm)]]
>         - [[#Thuật toán Cholesky]]
> - [[#4. Singular Value Decomposition]]
> 
>   

> [!important]
> 
> # Tham khảo
> 
> [[PCA và Kỹ Thuật Giảm Chiều Dữ Liệu]]
> 
> [[M01W1.6_Linear Algebra and Applications]]
# 1. Giới thiệu
**Matrix Factorization (phân rã ma trận)** là một kỹ thuật trong toán học và machine learning nhằm tách một ma trận phức tạp thành tích của các ma trận đơn giản hơn.
Mục tiêu là **biểu diễn dữ liệu ở dạng tiềm ẩn (latent factors)** để dễ dàng phân tích, giảm chiều, hoặc dự đoán.
# 2. LU and QR decomposition
## 2.1. LÚ decomposition
### LU trong giải hệ phương trình tuyến tính
**LU decomposition** (phân rã LU) là một kỹ thuật trong **đại số tuyến tính** để giải hệ phương trình tuyến tính, tính ma trận nghịch đảo, và xác định định thức.
Ý tưởng: Phân tích thừa số thành nhân tử
$$A = LU$$
  
**Ví dụ: Giải hệ phương trình sau**
$$A = \begin{bmatrix}  
1 & 2 & 4 \\  
3 & 8 & 14 \\  
2 & 6 & 13  
\end{bmatrix},  
B= \begin{bmatrix}  
3 \\ 13\\4  
\end{bmatrix}$$
**Ta phân rã LU như sau:**
**Bước 1:** Ta ấn định trước của ma trận $L$ và $U$ là:
$$L = \begin{bmatrix}  
1 & 0 & 0 \\  
l_{21} & 1 & 0 \\  
l_{31} & l_{32} & 1  
\end{bmatrix}$$
$$U = \begin{bmatrix}  
u_{11} & u_{12} & u_{13} \\  
0 & u_{22} & u_{23} \\  
0 & 0 & u_{33}  
\end{bmatrix}$$
Sau đó ta thấy:
$$\begin{bmatrix}  
1 & 2 & 4 \\  
3 & 8 & 14 \\  
2 & 6 & 13  
\end{bmatrix}  
=  
\begin{bmatrix}  
1 & 0 & 0 \\  
l_{21} & 1 & 0 \\  
l_{31} & l_{32} & 1  
\end{bmatrix}  
\times  
\begin{bmatrix}  
u_{11} & u_{12} & u_{13} \\  
0 & u_{22} & u_{23} \\  
0 & 0 & u_{33}  
\end{bmatrix}  
=  
\begin{bmatrix}  
u_{11} & u_{12} & u_{13} \\  
l_{21}.u_{11} & l_{21}.u_{12}+u_{22} & l_{21}.u_{13}+u_{23} \\  
l_{31}.u_{11} & l_{31}.u_{12}+l_{32}.u_{22} & l_{31}.u_{11}+l_{32}.u_{22}+u_{33}  
\end{bmatrix}$$
Vậy ta có thể phân tích được $L$ và $U$ như sau:
$$L=\begin{bmatrix}1&0&0\\3&1&0\\2&1&1\end{bmatrix},U=\begin{bmatrix}1&2&4\\0&2&2\\0&0&3\end{bmatrix}$$
(Kiểm tra: $LU=A$, $\det A=1\cdot2\cdot3=6\neq0$ nên phân rã hợp lệ.)
  
Thay $A = LU$ vào phương trình ta có:
$$LU\mathbf{x} = \mathbf{b}$$
Phân tích, ta có:
$$L\mathbf{y} = \mathbf{b} \\ U\mathbf{x} = \mathbf{y}$$
Ta chỉ cần giải $U\mathbf{x} = \mathbf{y}$ là sẽ tìm được nghiệm
Vậy đầu tiên ta giải $L\mathbf{y} = \mathbf{b}$:
**Ví dụ 2:**
Phân rã: $A=\begin{bmatrix}0 & 1 \\1 & 2\end{bmatrix}$
→ Không phân rã được. Từ đây ta thấy được vấn đề của phương pháp này.
---
### Điều kiện áp dụng phân rã LU
- Leading principal submatrices (Các **ma trận con chính phía trước)**:
    - Ma trận con chính phía trước bậc $k$ được lấy từ các hàng và cột đầu tiên của $A$:
        
        $$A_k = A[1:k,\, 1:k]$$
        
    - Điều kiện:
        
        $$\det(A_k) \neq 0 \quad \forall k = 1,2,\dots,n.$$
        
        Điều này đảm bảo quá trình khử Gauss không gặp trường hợp pivot bằng 0.
        
- $A$ khả nghịch:
    
    $$\det(A) \neq 0$$
    
### Dùng ma trận hoán vị (**LU decomposition with pivoting)**
Nếu điều kiện trên **không thỏa** (có pivot = 0 ở đường chéo), thì phải dùng **LU decomposition with pivoting**:
$$PA = LU$$
trong đó $P$ là ma trận hoán vị để đổi chỗ các hàng, giúp tránh pivot = 0 hoặc giá trị quá nhỏ (tăng tính ổn định số).
**Ví dụ:**
- Ma trận $\begin{bmatrix}0&1\\1&2\end{bmatrix}$ không thỏa vì pivot đầu tiên = 0 ⇒ cần **pivoting**.
  
**Giải hệ phương trình** $A.X = B$ **áp dụng LU decomposition with pivoting**
$$A = P^{-1}LU \Rightarrow A\mathbf{x} = LU\mathbf{x} = P\mathbf{b}$$
  
---
**Nhắc lại về ma trận hoán vị (Permutation Matrix):**
**Ma trận hoán vị** là một loại ma trận vuông đặc biệt dùng để **hoán đổi (swap) hàng hoặc cột** của một ma trận khác.
- Ma trận hoán vị kích thước $n \times n$ được tạo ra bằng cách **hoán đổi các hàng của ma trận đơn vị** $I_n$**.**
- Trong mỗi hàng và mỗi cột, chỉ có **một phần tử bằng 1**, các phần tử còn lại bằng 0.
Ví dụ:
$$P = \begin{bmatrix}0 & 1 & 0 \\1 & 0 & 0 \\0 & 0 & 1\end{bmatrix}$$
là ma trận hoán vị 3×3, dùng để đổi chỗ hàng 1 và hàng 2.
Ở đây sử dụng các phép biến đổi sơ cấp trên dòng của ma trận
---
**Xác định** $P$
…
  
**Bài tập:** Phân rã ma trận $A=\begin{bmatrix}0&1&4\\1&2&2\\0&1&3\end{bmatrix}.$
Vì $a_{11}=0$ (pivot đầu tiên bằng 0), phân rã **LU không pivot** sẽ thất bại. Ta dùng **partial pivoting**:
$$⁍$$
Cách làm:
- **Hoán vị hàng 1↔2** để đưa phần tử khác 0 lên vị trí (1,1), đó là $P$.
- Phân tích $PA$ thành $L$ và $U$
Kiểm tra:
$$PA=\begin{bmatrix}1&2&2\\0&1&4\\0&1&3\end{bmatrix},LU=\begin{bmatrix}1&2&2\\0&1&4\\0&1&3\end{bmatrix}\;\Rightarrow\;PA=LU.$$
---
### Code Python `scipy.linalg.lu()`
```Python
import numpy as np
from scipy.linalg import lu
A = np.array([[0, 1, 4],
              [1, 2, 2],
              [0, 1, 3]], dtype=float)
P, L, U = lu(A)
print("P =\n", P)
print("L =\n", L)
print("U =\n", U)
print("PA =\n", P @ A)
print("LU =\n", L @ U)
```
```Python
Ouput:
P =
 [[0. 1. 0.]
 [1. 0. 0.]
 [0. 0. 1.]]
L =
 [[1. 0. 0.]
 [0. 1. 0.]
 [0. 1. 1.]]
U =
 [[ 1.  2.  2.]
 [ 0.  1.  4.]
 [ 0.  0. -1.]]
PA =
 [[1. 2. 2.]
 [0. 1. 4.]
 [0. 1. 3.]]
LU =
 [[1. 2. 2.]
 [0. 1. 4.]
 [0. 1. 3.]]
```
**Chú ý:** Về mặt lý thuyết phân rã LU chỉ dùng trên ma trận vuông, nhưng thư viện scipy cho phép ta có thể phân rã ma trận hình chữ nhật:
```Python
## Tạo ma trận HÌNH CHỮ NHẬT
A = np.array([[1, 2,  4, 3],
              [3, 8, 14, 0],
              [2, 6, 13, 2]])
## Phân tích A thành các thành phần P, L, U
P, L, U = lu(A)
print('Ma trận HOÁN VỊ P', P.shape, ':\n', P)
print('Ma trận L', L.shape, ':\n', L)
print('Ma trận U', U.shape, ':\n', U)
print('Tái tạo A từ P, L, U (kiểm chứng phép phân rã):\n', (P @ L @ U).astype(int))
```
```Python
Output:
Ma trận HOÁN VỊ P (3, 3) :
 [[0. 0. 1.]
 [1. 0. 0.]
 [0. 1. 0.]]
Ma trận L (3, 3) :
 [[ 1.          0.          0.        ]
 [ 0.66666667  1.          0.        ]
 [ 0.33333333 -1.          1.        ]]
Ma trận U (3, 4) :
 [[ 3.          8.         14.          0.        ]
 [ 0.          0.66666667  3.66666667  2.        ]
 [ 0.          0.          3.          5.        ]]
Tái tạo A từ P, L, U (kiểm chứng phép phân rã):
 [[ 1  2  4  2]
 [ 3  8 14  0]
 [ 2  6 13  2]]
```
### `lu_factor()`
```Python
import numpy as np
from scipy.linalg import lu_factor, lu_solve
A = np.array([[0, 1, 4],
              [1, 2, 2],
              [0, 1, 3]], dtype=float)
b = np.array([3, 13, 4])
# Phân rã LU có pivoting
lu, piv = lu_factor(A)
print("LU matrix:\n", lu)
print("Pivot indices:\n", piv)
# Giải hệ A x = b
x = lu_solve((lu, piv), b)
print("Nghiệm x =", x)
```
```Python
LU matrix:
 [[ 1.  2.  2.]
 [ 0.  1.  4.]
 [ 0.  1. -1.]]
Pivot indices:
 [1 1 2]
Nghiệm x = [ 1.  7. -1.]
```
- `lu`: là ma trận kết hợp, trong đó:
    - phần **tam giác trên** chứa các phần tử của $U$
    - phần **tam giác dưới**, không bao gồm đường chéo, chứa các hệ số của $L$.
    - đường chéo của $L$ luôn bằng 1, nên không lưu trong `lu`.
- `piv`: là vector chỉ ra các hoán đổi hàng đã được thực hiện.
    
    Ví dụ: nếu `piv = [1, 1, 2]`, tức là:
    
    - bước 1: đổi hàng 0 ↔ 1,
    - bước 2: không đổi (hàng 1 ↔ 1),
    - bước 3: hàng 2 ↔ 2 (không đổi).

> [!important]
> 
> - `lu_factor()` chỉ cần **tính một lần** cho nhiều vector $b$ khác nhau (hữu ích nếu giải nhiều hệ có cùng ma trận A).
> - Nhập $L$ và $U$ thành một ma trận $LU$ và chuyển $P$ thành vector → Giảm được độ phức tạp của bài toán.
> - Thường dùng kết hợp với `lu_solve()` để tăng hiệu quả và tránh sai số khi nghịch đảo ma trận.
## 2.2. QR Decomposition
**QR decomposition** (phân rã QR) là kỹ thuật phân tích một ma trận $A \in \mathbb{R}^{m \times n}$ (với $m \geq n$) thành tích của 2 ma trận:
$$A = QR$$
Trong đó:
- $Q \in \mathbb{R}^{m \times m}$: **ma trận trực chuẩn** (orthogonal matrix), tức $Q^T Q = I$
- $R \in \mathbb{R}^{m \times n}$: **ma trận tam giác trên** (upper triangular)
Nếu $A$ là ma trận vuông, thì $Q$ và $R$ cũng vuông, và:
$$Q^T = Q^{-1}$$
  
  
### Thuật toán Gram-Schmidt
**Gram-Schmidt orthogonalization** là thuật toán biến một tập các vector tuyến tính độc lập thành một tập các **vector trực giao** (hoặc trực chuẩn nếu được chuẩn hóa).
Ứng dụng phổ biến trong việc **phân rã QR c**ủa một ma trận.
**Đầu vào - mục tiêu:**
Cho một tập vector $n$ độc lập:
$$\mathbf{a}_1, \mathbf{a}_2, \dots, \mathbf{a}_n \in \mathbb{R}^m$$
Mục tiêu: tạo ra các vector trực chuẩn $\mathbf{q}_1, \dots, \mathbf{q}_n$ sao cho:
- $\langle \mathbf{q}_i, \mathbf{q}_j \rangle = 0$ nếu $i \neq j$ (trực giao)
- $\|\mathbf{q}_i\| = 1$ (nếu chuẩn hóa)
Thuật toán:
```Python
Input: A = [a1, a2, ..., an] ∈ ℝ^{m×n} với các cột tuyến tính độc lập
Output: Q = [q1, q2, ..., qn] trực chuẩn; R ∈ ℝ^{n×n} tam giác trên
for j = 1 to n:
    v_j ← a_j
    for i = 1 to j-1:
        r_ij ← q_iᵀ · a_j
        v_j ← v_j - r_ij · q_i
    r_jj ← ‖v_j‖
    q_j ← v_j / r_jj
```
**Ví dụ:**
$$\mathbf{a}_1 = \begin{bmatrix}1\\1\\0\end{bmatrix},\quad\mathbf{a}_2 = \begin{bmatrix}1\\0\\1\end{bmatrix}$$
- $\mathbf{q}_1 = \dfrac{\mathbf{a}_1}{\|\mathbf{a}_1\|} = \dfrac{1}{\sqrt{2}}\begin{bmatrix}1\\1\\0\end{bmatrix}$
- Chiếu $a_2$ lên $q_1$:
$$\text{proj}_{q_1}(a_2) = (\mathbf{q}_1^T \mathbf{a}_2) \mathbf{q}_1 = \dfrac{1}{\sqrt{2}} \cdot 1 \cdot \mathbf{q}_1 = \dfrac{1}{2} \begin{bmatrix}1\\1\\0\end{bmatrix}$$
  
### Giải hệ phương trình A.X = B bằng phân rã Q.R
  
# 3. Eigen decomposition (Phân rã giá trị riêng)
**Eigen decomposition** là quá trình phân rã một ma trận vuông $A \in \mathbb{R}^{n \times n}$ thành các thành phần liên quan đến **eigenvectors** (vector riêng) và **eigenvalues** (giá trị riêng).
## 3.1. Trị riêng
### Định nghĩa
Cho ma trận: $A \in \mathbb{R}^{n \times n}$
Ta muốn tìm các **giá trị riêng (eigenvalues)** $\lambda$ sao cho:
$$A \mathbf{v} = \lambda \mathbf{v},\quad \text{với } \mathbf{v} \neq \mathbf{0}$$
Chuyển vế:
$$(A - \lambda I)\mathbf{v} = 0$$
  
Phương trình trên có nghiệm **không tầm thường** (vector $\mathbf{v} \neq \mathbf{0}$) khi và chỉ khi:
$$\det(A - \lambda I) = 0$$
### Phương trình thuần nhất (Homogeneous Linear System)
  
### Đa thức đặc trưng (Characteristic Polynomial)
Gọi:
$$p_A(\lambda) = \det(A - \lambda I)$$
Thì $p_A(\lambda)$ là
  
  
  
Ví dụ:
$$A = \begin{bmatrix}4 & 1\\2 & 3\end{bmatrix}$$
Tìm $\lambda$ bằng cách giải như sau:
- Xét ma trận $\lambda I$: $\begin{bmatrix} \lambda & 0\\0 & \lambda\end{bmatrix}$
- Ta có $A - \lambda I$: $\begin{bmatrix}4 - \lambda & 1\\2 & 3 - \lambda\end{bmatrix}$
- Sau đó cho định thức bằng 0
$$\det(A - \lambda I) = 0\Rightarrow\left|\begin{array}{cc}4 - \lambda & 1\\2 & 3 - \lambda\end{array}\right| = 0  
\\  
(4 - \lambda)(3 - \lambda) - 2 = \lambda^2 - 7\lambda + 10 = 0 \Rightarrow \lambda = 5, 2$$
  
### Phương trình đặc trưng
## 3.2. Vector riêng
  
## 3.3. Chéo hóa ma trận (diagonalization)
  
**Ứng dụng:**
Nhân lũy thừa ma trận đối với những Ma trận lớn → Chuỗi Markov
  
## 3.x. Phân rã Cholesky (đọc thêm)
**Cholesky decomposition** là một phương pháp đặc biệt để phân rã một **ma trận đối xứng xác định dương** thành tích của một ma trận tam giác dưới và chuyển vị của nó:
$$A = LL^T$$
Trong đó:
- $A \in \mathbb{R}^{n \times n}$:
- $L \in \mathbb{R}^{n \times n}$
- $L^T$: chuyển vị của $L$, là tam giác trên.
  
**Ưu điểm:**
- **Nhanh gấp đôi** so với LU decomposition (do chỉ cần phân nửa ma trận)
- **Ổn định số** cao hơn nhiều thuật toán khác
- Dùng trong: hệ phương trình tuyến tính, Gaussian Processes, Optimization, Monte Carlo,...
**Nhược điểm:**
- Cần thỏa điều kiện $A$ vuông, đối xứng.
- $A$ phải là ma trận xác định dương → các trị riêng dương.
### Thuật toán Cholesky
  
**Ví dụ**
  
  
# 4. Singular Value Decomposition