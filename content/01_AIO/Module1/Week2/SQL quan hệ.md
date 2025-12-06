---
Author: HoàiAn Thái
Date: Invalid date
Day/Week:
  - Module 1
Status: Not started
Tag:
  - AIO
---
# 1. **Entity Relationship Diagram**
### Phụ thuộc hàm (Functional Dependency)
- **Định nghĩa:** Trong một quan hệ $R$, một thuộc tính (hoặc tập thuộc tính) $X$ được gọi là **xác định** một thuộc tính (hoặc tập thuộc tính) $Y$ nếu và chỉ nếu với **mọi bộ dữ liệu** có cùng giá trị ở $X$, thì các giá trị ở $Y$ cũng giống nhau.
- Ký hiệu:
$$X \to Y$$
nghĩa là: $X$**phụ thuộc hàm** xác định $Y$
**Ví dụ:**
Trong bảng **SINHVIEN(MSSV, HoTen, NgaySinh, Lop)**:
- $MSSV \to HoTen, NgaySinh, Lop$
    
    (MSSV duy nhất nên xác định được các thông tin khác).
    
### Phụ thuộc đa trị (Multivalued Dependency – MVD)
**Định nghĩa:**
Trong một quan hệ $R$, một thuộc tính (hoặc tập thuộc tính) $X$ **đa trị xác định** $Y$ nếu với mỗi giá trị của
$X$, ta có một **tập nhiều giá trị độc lập** của $Y$, và các giá trị này **không phụ thuộc** vào các thuộc tính khác.
$$X \twoheadrightarrow Y$$
**Ví dụ:**
Trong quan hệ **SINHVIEN(MSSV, MonHoc, SoThich)**:
- Một sinh viên có thể đăng ký nhiều môn học.
- Một sinh viên cũng có thể có nhiều sở thích.
- Mối quan hệ:
    
    $MSSV \twoheadrightarrow MonHoc \quad \text{và} \quad MSSV \twoheadrightarrow SoThich$

# 2. **Database Normalization**
Chuẩn hóa Database ^836244
