# 1. Graphs
**Đồ thị (graph)** là một cách biểu diễn các mối quan hệ tồn tại giữa các cặp đối tượng. Nói cách khác, một đồ thị là một tập hợp các đối tượng gọi là **đỉnh (vertices)**, cùng với một tập hợp các kết nối theo cặp giữa chúng, gọi là **cạnh (edges)**. Đồ thị có nhiều ứng dụng trong nhiều lĩnh vực khác nhau, bao gồm lập bản đồ, vận tải, mạng máy tính và kỹ thuật điện.

Một đồ thị $G$ đơn giản là một tập $V$ các **đỉnh (vertices)** và một tập $E$ các **cặp đỉnh** từ $V$, gọi là **các cạnh (edges)**. Như vậy, đồ thị là một cách biểu diễn các kết nối hay mối quan hệ giữa các cặp đối tượng thuộc một tập hợp $V$.  

Các cạnh trong đồ thị có thể là **hướng (directed)** hoặc **vô hướng (undirected)**.
* Một cạnh $(u, v)$ được gọi là **có hướng** từ $u$ đến $v$ nếu cặp $(u, v)$ có thứ tự (nghĩa là $u$ đứng trước $v$).
* Một cạnh $(u, v)$ được gọi là **vô hướng** nếu cặp $(u, v)$ không có thứ tự, thường ký hiệu bằng $\{u, v\}$.

Ví dụ đồ thị có hướng:
![[image-101.png]]

* **Phân loại đồ thị:**
    * Nếu **mọi cạnh đều vô hướng**, đồ thị được gọi là **undirected graph**.
    * Nếu **mọi cạnh đều có hướng**, đồ thị được gọi là **directed graph (digraph)**.
    * Nếu có cả **cạnh hướng và vô hướng**, gọi là **mixed graph**.
    * Đồ thị vô hướng hoặc hỗn hợp có thể chuyển thành đồ thị có hướng bằng cách thay mỗi cạnh vô hướng $(u, v)$ bằng hai cạnh có hướng $(u, v)$ và $(v, u)$.

* **Các khái niệm cơ bản:**
    * Hai đỉnh nối bằng một cạnh gọi là **end vertices** (hoặc **endpoints**).
    * Nếu cạnh có hướng: đầu xuất phát gọi là **origin**, đầu kết thúc gọi là **destination**.
    * Hai đỉnh $u$ và $v$ **adjacent** nếu có cạnh nối giữa chúng.
    * Một cạnh **incident** với một đỉnh nếu đỉnh đó là một trong hai đầu mút của cạnh.
    * **Outgoing edges**: các cạnh có hướng đi ra từ một đỉnh.
    * **Incoming edges**: các cạnh có hướng đi vào một đỉnh.
    * **Degree (bậc)** của đỉnh $v$: tổng số cạnh nối với $v$, ký hiệu $deg(v)$.
    * **In-degree** và **out-degree**: lần lượt là số cạnh đi vào và đi ra từ $v$, ký hiệu $indeg(v)$ và $outdeg(v)$.

