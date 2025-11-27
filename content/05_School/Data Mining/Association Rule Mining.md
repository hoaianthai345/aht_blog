# Lift
**Lift** đo mức độ **liên kết thực sự** giữa 2 tập mục A và B trong luật:

$$A \Rightarrow B$$

Lift trả lời câu hỏi:

> “A và B xuất hiện cùng nhau nhiều hơn hay ít hơn so với _ngẫu nhiên_?”

* * *

##### **2. Công thức Lift**

$$\text{Lift}(A \Rightarrow B) = \frac{P(A \cap B)}{P(A)\times P(B)}$$

Trong mining thường dùng support:

$$\text{Lift}(A \Rightarrow B) =  
\frac{support(A \cup B)}{support(A) \cdot support(B)}$$

* * *

##### **3. Cách diễn giải Lift**

###### **Lift > 1**

A và B **có quan hệ tích cực** (xuất hiện cùng nhau nhiều hơn mong đợi nếu độc lập).

→ Mối quan hệ mạnh  
→ Rất quan trọng trong bán kèm (cross-selling)

**Ví dụ:**  
Lift = 2.0 → A & B xuất hiện **gấp 2 lần** so với ngẫu nhiên.

* * *

###### **Lift = 1**

A và B **độc lập thống kê**.  
Không có ảnh hưởng lẫn nhau.

**Ví dụ**:  
Lift = 1 → Dù A xuất hiện hay không, xác suất mua B không đổi.

* * *

###### **Lift < 1**

A và B **có quan hệ tiêu cực** (khi có A thì ít khả năng có B).

**Ví dụ**:  
Lift = 0.5 → xác suất xuất hiện cùng nhau **chỉ bằng 50%** so với ngẫu nhiên.



# Bài tập
Một cửa hàng bán lẻ muốn phân tích dữ liệu giao dịch của mình để hiểu rõ hơn về hành vi mua sắm của khách hàng. Dữ liệu giao dịch được liệt kê như sau:
![[image-201.png|315x330]]

## 1. Apriori
1. Tìm các tập mục thường xuyên (Frequent Itemsets): Sử dụng thuật toán Apriori để tìm các tập mục thường xuyên với ngưỡng hỗ trợ tối thiểu (min_support) là 50%. Liệt kê các tập mục 1 phần tử, 2 phần tử và 3 phần tử thường xuyên.
2.	Sinh các luật kết hợp (Association Rules): Từ các tập mục thường xuyên đã tìm được, sinh các luật kết hợp với ngưỡng độ tin cậy tối thiểu (min_confidence) là 70%. Liệt kê các luật kết hợp có thể suy ra từ các tập mục thường xuyên.
3.	Phân tích kết quả:
- Mục hàng hóa nào xuất hiện nhiều nhất trong các giao dịch?
- Có những luật kết hợp nào quan trọng có thể áp dụng để cải thiện doanh số bán hàng?

### Bài làm:
Câu 1:
Đến số lần suất hiện:
```Python
from apyori import apriori
frequent_1 = []
frequent_2 = []
frequent_3 = []

results = apriori(itemsets, min_support=0.01, min_confidence=0.001)

for r in results:
    itemset = list(r.items)
    support = r.support
    
    if len(itemset) == 1:
        frequent_1.append((itemset, support))

    elif len(itemset) == 2:
        frequent_2.append((itemset, support))

    elif len(itemset) == 3:
        frequent_3.append((itemset, support))

# In kết quả
print("Frequent 1-itemsets (≥50%):")
for items, sup in frequent_1:
    print(f"  {items} — count = {sup*6} — support = {sup:.3f}")

print("\nFrequent 2-itemsets (≥50%):")
for items, sup in frequent_2:
    print(f"  {items} — support = {sup:.3f}")

print("\nFrequent 3-itemsets (≥50%):")
for items, sup in frequent_3:
    print(f"  {items} — support = {sup:.3f}")
```
Output:
```
Frequent 1-itemsets (≥50%):
  ['Bia'] — count = 3.0 — support = 0.500
  ['Bánh quy'] — count = 2.0 — support = 0.333
  ['Chuối'] — count = 4.0 — support = 0.667
  ['Sữa'] — count = 5.0 — support = 0.833
  ['Táo'] — count = 3.0 — support = 0.500
  ['Tã giấy'] — count = 3.0 — support = 0.500

Frequent 2-itemsets (≥50%):
  ['Bánh quy', 'Bia'] — support = 0.167
  ['Chuối', 'Bia'] — support = 0.167
  ['Sữa', 'Bia'] — support = 0.333
  ['Táo', 'Bia'] — support = 0.167
  ['Tã giấy', 'Bia'] — support = 0.500
  ['Chuối', 'Bánh quy'] — support = 0.167
  ['Sữa', 'Bánh quy'] — support = 0.167
  ['Bánh quy', 'Tã giấy'] — support = 0.167
  ['Chuối', 'Sữa'] — support = 0.667
  ['Táo', 'Chuối'] — support = 0.333
  ['Chuối', 'Tã giấy'] — support = 0.167
  ['Táo', 'Sữa'] — support = 0.500
  ['Sữa', 'Tã giấy'] — support = 0.333
  ['Táo', 'Tã giấy'] — support = 0.167

Frequent 3-itemsets (≥50%):
  ['Tã giấy', 'Bánh quy', 'Bia'] — support = 0.167
  ['Sữa', 'Chuối', 'Bia'] — support = 0.167
  ['Tã giấy', 'Chuối', 'Bia'] — support = 0.167
  ['Táo', 'Sữa', 'Bia'] — support = 0.167
  ['Tã giấy', 'Sữa', 'Bia'] — support = 0.333
  ['Tã giấy', 'Táo', 'Bia'] — support = 0.167
  ['Chuối', 'Bánh quy', 'Sữa'] — support = 0.167
  ['Táo', 'Chuối', 'Sữa'] — support = 0.333
  ['Tã giấy', 'Chuối', 'Sữa'] — support = 0.167
  ['Táo', 'Sữa', 'Tã giấy'] — support = 0.167
```

Sau đó ta có thể sửa lại khoản để lấy min_support=0.01

***



## 2. FP tree

```


```