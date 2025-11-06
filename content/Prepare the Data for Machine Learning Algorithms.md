# Clean the Data

> **Scikit-Learn Design**
> API của Scikit-Learn được thiết kế cực kỳ tốt. Đây là **các nguyên tắc thiết kế chính**:
> 
> **Tính nhất quán:**
> Tất cả các đối tượng chia sẻ một [[M01W1.7 Coding Methodology#5. SOLID Principles|Interface]] nhất quán và đơn giản:
> 	**Estimators (Bộ ước lượng)**
> 		Bất kỳ đối tượng nào có thể ước lượng một số tham số dựa trên tập dữ liệu đều được gọi là _estimator_ (ví dụ, `SimpleImputer` là một estimator). Việc ước lượng được thực hiện bởi phương thức `fit()`, nhận một tập dữ liệu làm tham số, hoặc hai tập (cho học có giám sát – tập thứ hai là nhãn).  
> 		
> 		Note: Estimators hay `fit()` có thể được hiểu là mô hình đang học và cập nhật trọng số ($w$ weight) từ dataset.
> 	
> 	**Transformers (Bộ biến đổi)**  
> 		Một số estimators (như `SimpleImputer`) có thể biến đổi tập dữ liệu; chúng được gọi là _transformers_. 
> 		API cũng rất đơn giản: phép biến đổi được thực hiện bằng phương thức `transform()` với tập dữ liệu cần biến đổi làm tham số. Nó trả về tập dữ liệu đã biến đổi. Quá trình biến đổi này thường dựa trên các tham số đã học được (như `SimpleImputer`).
> 		
> 		Note: Transformes hay `transform()` có thể được hiểu là mô hình tính toán giá trị mục tiêu ($\hat{y}$ ) từ dataset.
> 		Ngoài ra, chúng có phương thức `fit_transform()` = học (`fit`) + biến đổi (`transform`) trong một bước.
> 		
> 	**Predictors (Bộ dự đoán)**  
> 		 Một số estimators, khi nhận tập dữ liệu, có thể đưa ra dự đoán; chúng được gọi là _predictors_. Ví dụ, `LinearRegression` là một predictor: cho GDP bình quân đầu người, nó dự đoán mức độ hài lòng với cuộc sống.  
> 		
> 		cPredictor có phương thức `predict()` nhận tập dữ liệu mới và trả về tập dự đoán tương ứng. Nó cũng có phương thức `score()` để đo chất lượng dự đoán, khi cho một tập kiểm tra (và nhãn đi kèm, với thuật toán học có giám sát).
> 
> **Kiểm tra (Inspection)**  
> Tất cả hyperparameters của estimator có thể truy cập trực tiếp qua biến instance công khai (ví dụ `imputer.strategy`), và tất cả tham số đã học được có thể truy cập qua biến instance công khai có hậu tố gạch dưới `_` (ví dụ `imputer.statistics_`).
> 
> **Không tạo quá nhiều lớp (Nonproliferation of classes)**  
> Tập dữ liệu được biểu diễn dưới dạng mảng NumPy hoặc ma trận thưa SciPy, thay vì tạo lớp riêng. Hyperparameters chỉ là chuỗi hoặc số Python bình thường.
> 
> **Tái sử dụng thành phần (Composition)**  
> Các khối xây dựng sẵn được tái sử dụng càng nhiều càng tốt. Ví dụ: có thể dễ dàng tạo một estimator `Pipeline` từ một chuỗi transformers tùy ý, sau đó là một estimator cuối cùng.
> 
> **Giá trị mặc định hợp lý (Sensible defaults)**  
> Scikit-Learn cung cấp các giá trị mặc định hợp lý cho hầu hết tham số, giúp dễ dàng nhanh chóng xây dựng một hệ thống cơ bản hoạt động.

---


