# 1. Levenshtein Distance
Đọc bài: [[Levenshtein Distance]]


# 2. N-gram

**Ngram** là một khái niệm trong xử lý ngôn ngữ tự nhiên (NLP) và thống kê văn bản, dùng để mô tả các chuỗi con liên tiếp trong một tập hợp văn bản. Ngram có thể là một từ, cụm từ hoặc bất kỳ chuỗi ký tự nào có độ dài cụ thể.
### Các loại Ngram:
- **Unigram (1-gram)**: Là các đơn vị (từ hoặc ký tự) đơn lẻ trong văn bản. Ví dụ: đối với câu "I love AI", các unigram là "I", "love", "AI".
- **Bigram (2-gram)**: Là các cặp liên tiếp của từ. Ví dụ: đối với câu "I love AI", các bigram là "I love" và "love AI".
- **Trigram (3-gram)**: Là các bộ ba từ liên tiếp trong văn bản. Ví dụ: đối với câu "I love AI", trigram sẽ là "I love AI".
- **N-gram (N-gram)**: Tổng quát cho bất kỳ chuỗi con dài N từ liên tiếp nào. Ví dụ: với N = 4, các 4-gram của câu "I love AI" sẽ là "I love AI".
    

### Ứng dụng của Ngram:
1. **Mô hình ngôn ngữ**: Ngram là cơ sở cho nhiều mô hình ngôn ngữ trong NLP, chẳng hạn như mô hình dự đoán từ tiếp theo hoặc phân loại văn bản.
2. **Dự đoán từ**: Ngram giúp dự đoán từ tiếp theo trong một câu dựa trên các từ trước đó. Ví dụ: trong một trình soạn thảo văn bản, hệ thống có thể dự đoán từ tiếp theo khi người dùng bắt đầu gõ một từ.
3. **Tìm kiếm văn bản**: Ngram có thể được sử dụng trong các công cụ tìm kiếm để cải thiện độ chính xác trong việc tìm kiếm các cụm từ liên quan.
4. **Phân loại văn bản**: Ngram có thể giúp trong việc phân loại văn bản theo các chủ đề khác nhau, vì mỗi chủ đề có thể có các Ngram đặc trưng riêng.
    

### Các ưu và nhược điểm:
- **Ưu điểm**:
    - Dễ hiểu và dễ triển khai.
    - Có thể áp dụng cho nhiều loại dữ liệu ngôn ngữ khác nhau.
    - Hiệu quả trong việc phát hiện các mẫu ngữ nghĩa trong văn bản.
        
- **Nhược điểm**:
    - Ngram thường không hiểu được ngữ cảnh rộng (dài hạn) trong văn bản.
    - Đối với các n-gram có giá trị N cao, số lượng các tổ hợp có thể trở nên rất lớn, dẫn đến vấn đề "chết dữ liệu" (data sparsity).
    - Các mô hình Ngram đơn giản có thể không tốt trong việc xử lý các câu phức tạp với ngữ nghĩa không rõ ràng.

### Cách tính:
**Ví dụ:** Giả sử bạn muốn tính xác suất rằng chuỗi từ "I love AI programming" xuất hiện trong văn bản. Để tính xác suất này:
Ở đây sử dụng định lý Bayes: [[M02W2.4_Basic Probability|Bayes]]
$$
P(\text{"I love AI programming"}) = P(\text{"I"}) \cdot P(\text{"love"}|\text{"I"}) \cdot P(\text{"AI"}|\text{"I love"}) \cdot P(\text{"programming"}|\text{"I love AI"})
$$

* $P(\text{"I"})$: Xác suất từ "I" xuất hiện.
* $P(\text{"love"}|\text{"I"})$: Xác suất từ "love" xuất hiện ngay sau "I".
* $P(\text{"AI"}|\text{"I love"})$: Xác suất từ "AI" xuất hiện ngay sau "I love".
* $P(\text{"programming"}|\text{"I love AI"})$: Xác suất từ "programming" xuất hiện ngay sau "I love AI".
    

### Một số điều cần lưu ý:
* **Tính toán xác suất dựa trên dữ liệu**: Để tính toán các xác suất này, bạn cần có một bộ dữ liệu lớn (corpus) và thực hiện đếm tần suất của các từ và các cặp từ (hoặc nhóm từ) trong văn bản.
* **Mô hình Markov**: Đây là một mô hình dựa trên khả năng xảy ra của một từ dựa trên số lượng từ trước đó, như trong ví dụ trên, bạn chỉ cần xem xét các từ trước đó để tính xác suất của từ tiếp theo.
Với các mô hình lớn, có thể áp dụng các kỹ thuật như **smoothing** (làm mượt) để giải quyết vấn đề tần suất thấp hoặc zero-frequency khi một từ hoặc cụm từ không xuất hiện trong dữ liệu huấn luyện.

## Giả định Markov (Markov Assumption)
**Giả định Markov**, hay còn gọi là **Giả định chuỗi Markov** (Markov Assumption), là một nguyên lý quan trọng trong lý thuyết xác suất và lý thuyết quá trình ngẫu nhiên. Giả định này nói rằng xác suất của trạng thái hiện tại chỉ phụ thuộc vào trạng thái ngay trước đó, mà không phụ thuộc vào các trạng thái trong quá khứ xa hơn.

**Cụ thể hơn, Giả định Markov có thể được mô tả như sau:**
* Trong một chuỗi các sự kiện, xác suất của sự kiện tiếp theo chỉ phụ thuộc vào sự kiện hiện tại và không phụ thuộc vào bất kỳ sự kiện nào xảy ra trước đó.
Nói cách khác, trong một quá trình Markov, **tương lai chỉ phụ thuộc vào hiện tại, không phụ thuộc vào quá khứ**.

**Mô tả chính thức:**

$$P(X_{n+1} | X_1, X_2, \dots, X_n) = P(X_{n+1} | X_n)$$

Trong đó:
* $X_n$ là trạng thái tại thời điểm $n$.
* $P(X_{n+1} | X_1, X_2, \dots, X_n)$ là xác suất của trạng thái $X_{n+1}$ xảy ra dựa trên chuỗi trạng thái trước đó (từ $X_1$ đến $X_n$).
* **Giả định Markov** nói rằng xác suất của $X_{n+1}$ chỉ phụ thuộc vào trạng thái hiện tại $X_n$, mà không cần biết các trạng thái trước đó (tức là không cần biết $X_1, X_2, \dots, X_{n-1}$).
    

**Ví dụ trong ngữ cảnh văn bản (NLP):**
Trong ngữ cảnh của ngữ nghĩa và chuỗi văn bản, giả định Markov được áp dụng trong các mô hình ngôn ngữ như **Mô hình Markov ẩn** (Hidden Markov Model - HMM). Mô hình này giả định rằng xác suất xuất hiện của mỗi từ trong chuỗi chỉ phụ thuộc vào từ trước đó (hoặc một số từ gần đó).
Ví dụ:
* Giả sử bạn có câu "I love AI programming" và bạn muốn tính xác suất của chuỗi từ này trong một mô hình Markov, giả định Markov nói rằng:
    $$P(\text{"I love AI programming"}) = P(\text{"I"}) \cdot P(\text{"love"}|\text{"I"}) \cdot P(\text{"AI"}|\text{"love"}) \cdot P(\text{"programming}|\text{"AI"})$$
    
    Ở đây, xác suất của mỗi từ chỉ phụ thuộc vào từ liền trước nó.
    

### Đặc điểm của giả định Markov:
1. **Không nhớ quá khứ**: Quá trình chỉ phụ thuộc vào trạng thái hiện tại, tức là quá trình này "không nhớ" các trạng thái trước đó.
2. **Chuỗi Markov**: Một chuỗi các sự kiện tuân theo giả định Markov được gọi là chuỗi Markov. Các quá trình này có thể được mô tả bằng một ma trận xác suất chuyển tiếp.
3. **Các ứng dụng**: Giả định Markov được ứng dụng rộng rãi trong các lĩnh vực như mô hình hóa hệ thống động, dự đoán chuỗi thời gian, nhận dạng giọng nói, phân tích văn bản, và các hệ thống học máy như **HMM**, **Markov Decision Process (MDP)**, **Reinforcement Learning**, ... X

### Một ví dụ khác: Mô hình Markov trong chuyển động ngẫu nhiên

Trong một mô hình chuyển động ngẫu nhiên (random walk), giả định Markov cho phép tính xác suất của việc di chuyển từ một điểm đến điểm tiếp theo chỉ dựa trên vị trí hiện tại mà không cần biết các bước đi trước đó.
X 

# Homework

1/ Thiết kế lại Levenshtein Distance và bổ sung backtrace

2/ 
Lấy cơ sở dữ liệu văn bản: Corpus: wiki (Việt Nam)

Bóc tách từ với các cấp độ bigram, trigram

Cho câu: "Việt Nam là đức nước nằm ở khu vực Đông Nam Á"
Xác định từ sai chính tả và sửa lại -> Liệt kê 5 từ phù hợp sau đó xác định từ đúng nhất để sửa
	- Dùng cả hai phương pháp Damuran Levenshtein và NGram