---
Author: HoàiAn Thái
Date: Invalid date
Status: Not started
Tag:
  - Book
---
> [!important] **Table of Content**
> 
> ---
> 
> - [[#Tham khảo]]
> - [[#3. Hệ trục tọa độ]]
>     - [[#a. Hệ trục tọa độ đường cong (curved axes)]]
>         - [[#Polar coordinate system: hệ tọa độ cực]]
>         - [[#Dữ liệu bản đồ (Geospatial data, maps)]]
> - [[#4. Color scales]]
>     - [[#4.1. Thể hiện sự phân nhóm dữ liệu (định danh)]]
>     - [[#4.2. Color to Represent Data Values]]
>     - [[#4.3. Dùng để nhấn mạnh (highlight)]]
> - [[#5. Các loại biểu đồ]]
>     - [[#5.1. Amounts]]
>     - [[#5.2. Distributions]]
>     - [[#5.3. Proportions]]
>     - [[#5.4. X–Y Relationships]]
>     - [[#5.5 Geospatial data]]
>     - [[#5.6 Uncertainty]]
> 
>   

> [!important]
> 
> # Tham khảo
> 
> Fundamentals of Data Visualization: [https://clauswilke.com/dataviz/](https://clauswilke.com/dataviz/)
> 
> [[M03W1.6_Advanced Data Visualization]]
> 
> [[Directory of visualizations]]
# 3. Hệ trục tọa độ
## a. Hệ trục tọa độ đường cong (curved axes)
### Polar coordinate system: hệ tọa độ cực
Hệ tọa độ cực là một cách biểu diễn vị trí của điểm trong mặt phẳng thông qua **khoảng cách** đến gốc tọa độ và **góc** so với trục chuẩn. Thay vì dùng cặp tọa độ Descartes (x, y), ta dùng cặp tọa độ cực (r, θ).
- **r**: khoảng cách từ điểm đến gốc tọa độ (r ≥ 0).
- **θ**: góc hợp bởi đoạn nối từ gốc đến điểm với trục Ox, đo bằng radian hoặc độ.
![[image 69.png|image 69.png]]
![[image 1 33.png|image 1 33.png]]
**Công thức chuyển đổi:**
Từ cực → Descartes:
```Python
x = r * cos(θ)
y = r * sin(θ)
```
Từ Descartes → cực:
```Python
r = √(x² + y²)
θ = atan2(y, x)
```
→ Tung độ là bán kính
→ Hoành độ là vị trí tương đối khi chuyển thành góc
**Khi nào ta dùng tọa độ cực?**
- Dữ liệu có tính chu kì:
    - Dữ liệu dạng chu kỳ (thời gian theo ngày, theo mùa, hướng gió, pha sóng) thường được biểu diễn tốt hơn trong hệ cực.
    - Ví dụ: so sánh tốc độ gió theo hướng → vẽ **wind rose chart** (hoa hồng gió).
### Dữ liệu bản đồ (Geospatial data, maps)
**Tham khảo:** GIS là gì
![[image 2 31.png|image 2 31.png]]
# 4. **Color scales**
## 4.1. Thể hiện sự phân nhóm dữ liệu (định danh)
- Chỉ phục vụ cho phân biệt → Biến độc lập nominal (biến định danh)
- Không có tính thứ tự, mức độ đậm nhạt, không có liên hệ
Ví dụ:
- Giới tính: {Nam, Nữ, Khác}.
- Màu sắc: {Đỏ, Xanh, Vàng}.
- Mã lớp học: {A1, A2, A3}.
![[image 3 29.png|image 3 29.png]]
Khi xử lý biến định danh, ta thường mã hóa bằng **One-hot encoding** hoặc **Label encoding** để sử dụng trong mô hình thống kê/machine learning.
## 4.2. Color to Represent Data Values
  
## 4.3. Dùng để nhấn mạnh (highlight)
  
# 5. Các loại biểu đồ
Ở đây tác giả chia theo chức năng và là overview qua các loại biểu đồ.
Để đọc kĩ hơn:
[[Directory of visualizations]]

> [!important] **Vấn đề ở cách tiếp cận này là**
> 
> - Một biểu đồ có thể có nhiều chức năng (và tùy loại chức năng quan trọng ít hay nhiều)
>     
>     → Từ đây dẫn đến một vấn đề là chúng ta trong trường hợp có thể dùng lại biểu đồ rất nhiều lần.
>     
> 
> Đề xuất phân nhóm theo thầy:
> 
> - Dạng biểu đồ thống kê (statistical graph):
>     
>     - Dot plot
>     - Line graph
>     - Radar chart
>     - Area graph
>     - Stacked area graph: Trực quan được tổng số. (có cả chênh lệch giá trị
>         
>         ![[image 4 24.png|image 4 24.png]]
>         
>     - Stream graph: Nhấn mạnh chênh lệch
>         
>         ![[image 5 22.png|image 5 22.png]]
>         
>     - Bar chart
>     - Radial bar chart
>         
>         ![[image 6 21.png|image 6 21.png]]
>         
>     - Radial column chart (là đổi trục x và y từ bar chart)
>         
>         ![[image 7 16.png|image 7 16.png]]
>         
>     - Spiral plot (time series spiral) → Dùng khi biến độc lập có nhiều giá trị và biến thời gian có tính chu kỳ.
>         
>         ![[image 8 16.png|image 8 16.png]]
>         
>     - Group bar plot: So sánh giữa từng nhóm con của nhóm.
>     - Stack bar plot: So sánh giá trị tổng thể của nhóm
>     - Histogram: thể hiện tần số, không được có khoảng trắng giữa các thanh để thể hiện được Cumulative Distribution Function (CDF)
>     - Heatmap
>     - Word cloud
>     - Quantile dot plot: Dùng cho nhị phân
>     - Stack histogram: Khác so với Stack bar vì để tính diện tích CDF
>         
>           
>         
>     - Span chart: Thể hiện min/max của từng đối tượng
>         
>         ![[image 9 15.png|image 9 15.png]]
>         
>     - Error bar: Biểu diễn độ nhiễu
>         
>         ![[image 10 14.png|image 10 14.png]]
>         
>         ![[image 11 14.png|image 11 14.png]]
>         
>     - Graded error bar:
>     - Candlestick chart:
>         
>           
>         
>     
>       
>     
> - Dạng sơ đồ (diagram)
> - Dạng khác
  
## 5.1. Amounts
Để trực quan hóa **giá trị số theo danh mục**, cách phổ biến nhất là dùng **biểu đồ cột** (dọc hoặc ngang). Thay vì cột, ta cũng có thể dùng **dấu chấm** tại vị trí đầu cột.
![[image 12 13.png|image 12 13.png]]
Khi có **nhiều nhóm danh mục**, ta có thể sử dụng **bar chart dạng nhóm** (grouped) hoặc **dạng chồng** (stacked). Ngoài ra, có thể ánh xạ danh mục lên trục x và y, rồi thể hiện giá trị bằng màu sắc trong **heatmap**.
![[image 13 12.png|image 13 12.png]]

> [!important] Khi cần so sánh nhiều nhóm, grouped bar chart dễ đọc hơn stacked bar chart.
## 5.2. Distributions
Để trực quan hóa **phân phối dữ liệu**, các lựa chọn bao gồm:
- **Histogram** và **density plot**: cách trực quan nhất để mô tả phân phối. Tuy nhiên, cả hai đều phụ thuộc vào các tham số tùy ý (như độ rộng bin, bandwidth) nên đôi khi có thể gây sai lệch.
- **Cumulative density** và **q-q plot**: luôn trung thực nhưng khó diễn giải.
    
    ![[image 14 12.png|image 14 12.png]]
    
- **Boxplot, violin, strip chart, sina plot**: phù hợp để so sánh nhiều phân phối cùng lúc hoặc xem xu hướng tổng thể.
- **Stacked histogram, overlapping density**: để so sánh số ít phân phối (stacked histogram khó đọc).
- **Ridgeline plot**: thay thế violin plot, hữu ích khi số lượng phân phối lớn hoặc thay đổi theo thời gian.
    
    ![[image 15 12.png|image 15 12.png]]
    
  
## 5.3. Proportions
Để trực quan hóa **tỉ lệ thành phần**, ta dùng:
![[image 16 10.png|image 16 10.png]]
- **Pie chart**: nhấn mạnh rằng các phần cộng thành một tổng thể, thích hợp khi muốn làm nổi bật phân số đơn giản.
- **Side-by-side bar chart**: dễ so sánh từng phần tử hơn pie chart.
- **Stacked bar chart**: không thích hợp cho một tập tỉ lệ đơn lẻ, nhưng hữu ích khi so sánh nhiều điều kiện.
Khi trực quan hóa **nhiều tập tỉ lệ** hoặc sự thay đổi tỉ lệ giữa các điều kiện:
![[image 17 10.png|image 17 10.png]]
- Pie chart kém hiệu quả về không gian và dễ làm mất đi mối quan hệ giữa các thành phần.
- Grouped bar chart tốt khi số điều kiện vừa phải.
- Stacked bar chart hiệu quả khi số điều kiện lớn.
- **Stacked densities**: phù hợp khi tỉ lệ thay đổi theo biến liên tục.
Với **nhiều biến phân nhóm**, ta có các lựa chọn khác:
![[image 18 10.png|image 18 10.png]]
- **Mosaic plot**: giả định rằng mọi cấp độ của biến này đều có thể kết hợp với mọi cấp độ của biến khác.
- **Treemap**: không cần giả định này, phù hợp ngay cả khi các nhóm hoàn toàn tách biệt.
- **Parallel sets**: hiệu quả hơn cả mosaic và treemap khi có trên hai biến phân nhóm.
## 5.4. X–Y Relationships
Để thể hiện mối quan hệ giữa hai hoặc nhiều biến số định lượng:
![[image 19 10.png|image 19 10.png]]
- **Scatterplot**: cơ bản nhất, hiển thị một biến định lượng theo biến còn lại.
- **Bubble chart**: giống scatterplot nhưng thêm biến thứ ba bằng kích thước điểm.
- **Paired Scatterplot: v**ới dữ liệu **paired data** (hai biến đo bằng cùng đơn vị), thường thêm đường **x = y** để dễ so sánh.
- **Slope graph**: nối cặp dữ liệu bằng đường thẳng.
Với dữ liệu dày đặc gây **overplotting**:
![[image 20 9.png|image 20 9.png]]
- **Contour lines:**
- **2D bins:**
- **Hex bins:**
- **Correlogram:** thể hiện hệ số tương quan thay vì dữ liệu gốc khi biểu diễn nhiều hơn hai biến
Khi trục x là **thời gian** hoặc biến tăng đơn điệu (ví dụ liều điều trị)
![[image 21 9.png|image 21 9.png]]
- **Line graph:** phổ biến khi muốn biểu thị trend.
- **Connected scatterplot:** Nếu có chuỗi thời gian của **hai biến phản hồi**, đầu tiên vẽ scatterplot, sau đó nối các điểm liền kề theo thời gian.
- **Smooth line:** để biểu diễn xu hướng tổng quát trong tập dữ liệu lớn.
## **5.5 Geospatial data**
- **Map**
    - Hình thức phổ biến nhất để hiển thị dữ liệu địa lý.
    - Lấy tọa độ trên quả địa cầu và chiếu lên mặt phẳng 2D.
    - Hình dạng và khoảng cách được biểu diễn gần đúng.
- **Choropleth**
    - Các vùng được tô màu dựa trên giá trị dữ liệu.
    - Ví dụ: bản đồ nhiệt dân số, thu nhập theo khu vực.
- **Cartogram**
    - Biến dạng kích thước vùng theo một đại lượng khác (ví dụ: dân số).
    - Có thể đơn giản hóa vùng thành các ô vuông.
    - Dùng để nhấn mạnh tầm quan trọng theo quy mô dữ liệu thay vì diện tích thực.
![[image 22 7.png|image 22 7.png]]

> [!important] Dùng **choropleth** khi muốn giữ tính trực quan địa lý; dùng **cartogram** khi muốn nhấn mạnh tỉ trọng dữ liệu (như dân số) thay vì diện tích địa lý.
## **5.6 Uncertainty**
- **Error bars**
    - Thể hiện khoảng giá trị có khả năng xảy ra quanh một ước lượng hoặc phép đo.
    - Kéo dài theo chiều ngang hoặc dọc từ điểm tham chiếu (dot hoặc bar).
    - **Graded error bars**: nhiều error bar chồng lên nhau với độ dày khác nhau, biểu thị các mức độ tin cậy khác nhau.
![[image 23 7.png|image 23 7.png]]
- **Phân phối xác suất / hậu nghiệm**
    - Thay vì error bar, có thể trực quan hóa toàn bộ **confidence distribution** hoặc **posterior distribution**.
    - **Confidence strips**: trực quan rõ ràng nhưng khó đọc chính xác.
    - **Eyes** và **half-eyes**: kết hợp error bar với violin/ridgeline, vừa thể hiện khoảng tin cậy, vừa mô tả toàn bộ phân phối bất định.
    - **Quantile dot plot**: thay thế cho violin/ridgeline bằng cách biểu diễn phân phối dưới dạng các đơn vị rời rạc, dễ đọc hơn nhưng kém chính xác hơn.
![[image 24 7.png|image 24 7.png]]
- **Confidence bands (cho line graph)**
    - Tương tự error bar nhưng áp dụng cho đường xu hướng.
    - Biểu diễn dải giá trị mà đường có thể đi qua ở một mức tin cậy.
    - Có thể dùng **graded confidence bands** để hiển thị nhiều mức tin cậy cùng lúc.
    - Ngoài ra, có thể trực quan hóa bằng **nhiều fitted draws** thay cho confidence bands.
![[image 25 7.png|image 25 7.png]]