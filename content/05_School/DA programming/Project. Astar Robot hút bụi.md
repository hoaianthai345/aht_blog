
# Lý thuyết
A* (A-star) là thuật toán tìm **đường đi ngắn nhất** trong một **đồ thị có trọng số dương**, ví dụ như bản đồ, mê cung, hay lưới ô vuông (grid).  
Nó là sự kết hợp giữa hai hướng tiếp cận:

* **Dijkstra** (tìm đường ngắn nhất tuyệt đối)
    
* **Greedy Best-First Search** (ưu tiên hướng đến đích nhanh hơn)

**Công thức đánh giá:**
Mỗi node $n$ có giá trị: $f(n) = g(n) + h(n)$
Trong đó:
* $g(n)$: Chi phí thực tế từ điểm bắt đầu → node hiện tại
* $h(n)$: Ước lượng chi phí từ node hiện tại → đích (Heuristic)
* $f(n)$: Tổng chi phí ước lượng (mục tiêu là tìm node có f nhỏ nhất)
    
### Heuristic
* **Manhattan distance**:
    $$h(n) = |x_{goal} - x_n| + |y_{goal} - y_n|$$
    
    → dùng khi chỉ được đi 4 hướng (trái, phải, lên, xuống)
    
* **Euclidean distance**:
    $$h(n) = \sqrt{(x_{goal} - x_n)^2 + (y_{goal} - y_n)^2}$$
    
    → dùng khi được đi chéo



Nếu heuristic **h** tốt, A* sẽ **rất nhanh và chính xác**.
# Xây dựng dự án

**`get_neighbors()`**
![[image-100.png]]