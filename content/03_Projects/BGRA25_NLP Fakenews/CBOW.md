# Mục tiêu 
Dự đoán từ còn thiếu dựa vào ngữ cảnh.

![[image-108.png]]

# Implement from scratch
Note: Các hàm tiện ích ở đây sẽ sử dụng lại từ bài [[Skip-gram]].
```python
import numpy as np
import random
import re
from collections import Counter

def tokenize(text):
    """Tokenize the input text."""
    return re.findall(r'\w+', text.lower())

def build_vocab(tokens):
    """Build a vocabulary from the tokens."""
    vocab = Counter(tokens)
    vocab_size = len(vocab)
    word2id = {word: i for i, (word, _) in enumerate(vocab.items())}
    id2word = {i: word for word, i in word2id.items()}
    return vocab, word2id, id2word

def generate_context_pairs(tokens, window_size=2):
    """Generate context pairs for CBOW: (context, target)."""
    context_pairs = []
    for i in range(window_size, len(tokens) - window_size):
        context = tokens[i - window_size:i] + tokens[i + 1:i + window_size + 1]
        target = tokens[i]
        context_pairs.append((context, target))
    return context_pairs

```

### Define the CBOW Model
```python
class CBOW:
    def __init__(self, vocab_size, embed_size, lr=0.025):
        self.vocab_size = vocab_size
        self.embed_size = embed_size
        self.lr = lr
        
        # Initialize weight matrices
        self.W_in = np.random.randn(vocab_size, embed_size) * 0.01  # Input to hidden layer
        self.W_out = np.random.randn(embed_size, vocab_size) * 0.01  # Hidden to output layer
        
    def forward(self, context):
        """Forward pass of the CBOW model."""
        # Step 1: Average the embeddings of context words
        context_vectors = self.W_in[context]
        h = np.mean(context_vectors, axis=0)  # Average the context word vectors
        
        # Step 2: Calculate scores for each word in the vocabulary
        u = np.dot(h, self.W_out)  # Dot product to get scores
        
        # Step 3: Apply softmax to get probabilities for each word in the vocab
        exp_u = np.exp(u - np.max(u))  # Stability trick to prevent overflow
        probs = exp_u / np.sum(exp_u)
        
        return probs, h
    
    def backpropagate(self, context, target, probs, h):
        """Backward pass to update the weights."""
        # Step 1: Calculate the error (target - prediction)
        target_vector = np.zeros(self.vocab_size)
        target_vector[target] = 1  # One-hot encoding of the target word
        
        error = probs - target_vector
        
        # Step 2: Gradients for the output layer
        grad_W_out = np.outer(h, error)  # Gradient for W_out
        
        # Step 3: Gradients for the hidden layer
        grad_h = np.dot(self.W_out, error)
        
        # Step 4: Gradients for the input layer
        grad_W_in = np.zeros_like(self.W_in)
        for i in range(len(context)):
            grad_W_in[context[i]] += grad_h / len(context)  # Average gradient for context words
        
        # Step 5: Update the weights
        self.W_in -= self.lr * grad_W_in
        self.W_out -= self.lr * grad_W_out

```

#### Các tham số của class
```python
def __init__(self, vocab_size, embed_size, lr=0.025):
        self.vocab_size = vocab_size
        self.embed_size = embed_size
        self.lr = lr
        
        # Initialize weight matrices
        self.W_in = np.random.randn(vocab_size, embed_size) * 0.01  # Input to hidden layer
        self.W_out = np.random.randn(embed_size, vocab_size) * 0.01  # Hidden to output layer
```

- Khởi tạo ngẫu nhiên hai ma trận input và output có shape là vocab_size x embed_size
***
#### Forward
```python
def forward(self, context):
        """Forward pass of the CBOW model."""
        # Step 1: Average the embeddings of context words
        context_vectors = self.W_in[context]
        h = np.mean(context_vectors, axis=0)  # Average the context word vectors
        
        # Step 2: Calculate scores for each word in the vocabulary
        u = np.dot(h, self.W_out)  # Dot product to get scores
        
        # Step 3: Apply softmax to get probabilities for each word in the vocab
        exp_u = np.exp(u - np.max(u))  # Stability trick to prevent overflow
        probs = exp_u / np.sum(exp_u)
        
        return probs, h
```

Ví dụ ta có câu:
* **"I love learning"**
* **Tokens**: `['i', 'love', 'learning']`

**Bước 1: Tạo từ điển ánh xạ các từ tới chỉ số của chúng:**
* `word2id = {'i': 0, 'love': 1, 'learning': 2}`
* `id2word = {0: 'i', 1: 'love', 2: 'learning'}`

**Bước 2: Xác định content và target**
Với **window size = 1**, từ trung tâm sẽ có một từ trước và một từ sau là ngữ cảnh của nó.
* **Target word** (từ trung tâm) là "love".
* **Context words** (từ ngữ cảnh) là từ trước và sau "love", tức là "i", "learning".

Với **CBOW**, ta sẽ cố gắng dự đoán từ **target word** ("love") từ các **context words** ("i", "learning").

**Bước 3: Khởi tạo các embedding và trọng số**
Chúng ta khởi tạo ngẫu nhiên các ma trận trọng số (embedding matrices).
* **Ma trận W_in (Input-to-Hidden)**: Là ma trận ánh xạ từ các từ trung tâm tới không gian embedding.
* **Ma trận W_out (Hidden-to-Output)**: Là ma trận ánh xạ từ không gian embedding tới không gian output, nơi ta dự đoán xác suất của các từ trong từ vựng.

Giả sử **embedding dimension = 2**, và chúng ta khởi tạo trọng số ngẫu nhiên:
* $W_{\text{in}}$ (với 3 từ trong từ vựng, mỗi từ có 2 chiều embedding):
    $$W_{\text{in}} =  
    \begin{bmatrix}  
    0.1 & 0.3 \\  
    0.4 & 0.2 \\  
    0.6 & 0.9  
    \end{bmatrix}$$
    * **i**: [0.1, 0.3]
    * **love**: [0.4, 0.2]
    * **learning**: [0.6, 0.9]
        
* $W_{\text{out}}$ (với 2 chiều embedding và 3 từ trong từ vựng):
    $$W_{\text{out}} =  
    \begin{bmatrix}  
    0.2 & 0.1 & 0.3 \\  
    0.4 & 0.5 & 0.6  
    \end{bmatrix}$$
    * **Output vector của "i"**: [0.2, 0.1]
    * **Output vector của "love"**: [0.4, 0.5]
    * **Output vector của "learning"**: [0.3, 0.6]
        

**Bước 4: Tính toán forward pass (dự đoán)**
* **Context words**: "i" và "learning".
* **Context word vectors**:
    * **"i"** (context): [0.1, 0.3]
    * **"learning"** (context): [0.6, 0.9]
        
1. **Average embedding của context words**:  
    Chúng ta sẽ lấy trung bình của các vector embedding của các từ ngữ cảnh (context):$$h = \frac{1}{2} \left( [0.1, 0.3] + [0.6, 0.9] \right) = \frac{1}{2} [0.7, 1.2] = [0.35, 0.6]$$
    **Vector trung gian (hidden layer)** $h = [0.35, 0.6]$.

2. **Tính output scores**:  
    Để dự đoán từ trung tâm, chúng ta tính điểm $u$ cho mỗi từ trong từ vựng bằng cách nhân vector trung gian $h$ với ma trận $W_{\text{out}}$.
    $$u = h \cdot W_{\text{out}}$$
    
    Tính toán cho mỗi từ trong từ vựng:$$u_{\text{"i"}} = [0.35, 0.6] \cdot [0.2, 0.4] = 0.35 \times 0.2 + 0.6 \times 0.4 = 0.07 + 0.24 = 0.31$$$$u_{\text{"love"}} = [0.35, 0.6] \cdot [0.1, 0.5] = 0.35 \times 0.1 + 0.6 \times 0.5 = 0.035 + 0.3 = 0.335$$ $$u_{\text{"learning"}} = [0.35, 0.6] \cdot [0.3, 0.6] = 0.35 \times 0.3 + 0.6 \times 0.6 = 0.105 + 0.36 = 0.465$$
    
    **Điểm số $u$ cho mỗi từ**:
    * **"i"**: 0.31
    * **"love"**: 0.335
    * **"learning"**: 0.465
        
3. **Áp dụng softmax để tính xác suất cho mỗi từ**:  
    Chúng ta sử dụng hàm softmax để chuyển các điểm số $u$ thành xác suất. Công thức softmax là:
    $$\sigma(u) = \frac{e^{u_i}}{\sum_{i=1}^{|V|} e^{u_i}}$$
    
    Tính toán:$$\text{exp}(u_{\text{"i"}}) = e^{0.31} \approx 1.363, \quad \text{exp}(u_{\text{"love"}}) = e^{0.335} \approx 1.398, \quad \text{exp}(u_{\text{"learning"}}) = e^{0.465} \approx 1.592$$
    Tổng $e^u$:
    $$\sum_{i=1}^{3} e^{u_i} = 1.363 + 1.398 + 1.592 = 4.353$$
    
    Xác suất softmax cho mỗi từ:
    $$P(\text{"i"}) = \frac{1.363}{4.353} \approx 0.313, \quad P(\text{"love"}) = \frac{1.398}{4.353} \approx 0.321, \quad P(\text{"learning"}) = \frac{1.592}{4.353} \approx 0.365$$

***
#### Backpropagation
**1. Tính toán lỗi (Error)**
Lỗi trong mô hình được tính là **sự khác biệt giữa xác suất dự đoán** (probs) và **xác suất thực tế** (one-hot encoding của target).
* **`target_vector`** là một **mảng one-hot** cho từ trung tâm (target). Vì trong ví dụ của chúng ta, từ "love" là từ trung tâm, chúng ta sẽ có:
* $$\text{target\_vector} = [0, 1, 0]$$
* **`probs`** là xác suất mà mô hình dự đoán cho mỗi từ trong từ vựng, mà chúng ta đã tính được từ **forward pass**:
    $$\text{probs} = [0.313, 0.321, 0.365]$$
* **`error`** là **sự khác biệt** giữa xác suất dự đoán và target (lỗi dự đoán):$$\text{error} = \text{probs} - \text{target\_vector} = [0.313, 0.321, 0.365] - [0, 1, 0] = [0.313, -0.679, 0.365]$$
    Điều này có nghĩa là mô hình đã dự đoán một xác suất khá cao cho từ "love", nhưng xác suất cho từ "love" thực tế phải là 1 (một-hot encoding). Vì vậy, ta có lỗi âm cho "love" và lỗi dương cho các từ khác.
    
**2. Gradient cho output layer**
Tiếp theo, chúng ta cần tính gradient cho **ma trận $W_{\text{out}}$** (từ hidden layer đến output layer). **Gradient cho output** được tính bằng cách sử dụng **tích ngoài (outer product)** của vector ẩn $h$ và **lỗi** (error).
* **`grad_W_out = np.outer(h, error)`**: Tích ngoài giữa vector ẩn $h$ và lỗi. Cách tính này cho phép tính được sự thay đổi cần thiết cho mỗi trọng số của ma trận $W_{\text{out}}$.
    

Ví dụ:
* $h = [0.35, 0.6]$ là vector trung gian (hidden layer).
* **`error = [0.313, -0.679, 0.365]`** là lỗi.

Tính toán **tích ngoài**:
$$\text{grad\_W\_out} =  
\begin{bmatrix}  
0.35 & 0.6  
\end{bmatrix}^T  
\cdot  
\begin{bmatrix}  
0.313 \\  
-0.679 \\  
0.365  
\end{bmatrix}  
=  
\begin{bmatrix}  
0.35 \times 0.313 & 0.35 \times -0.679 & 0.35 \times 0.365 \\  
0.6 \times 0.313 & 0.6 \times -0.679 & 0.6 \times 0.365  
\end{bmatrix}$$ $$\text{grad\_W\_out} =  
\begin{bmatrix}  
0.10955 & -0.23765 & 0.12775 \\  
0.1878 & -0.4074 & 0.219  
\end{bmatrix}$$

* * *

**3. Gradient cho hidden layer (grad_h)**
Sau khi tính toán gradient cho $W_{\text{out}}$, chúng ta tiếp tục tính **gradient cho hidden layer**. Gradient này cho biết mức độ ảnh hưởng của mỗi hidden layer (vector ẩn $h$) đối với lỗi:
* **`grad_h = np.dot(self.W_out, error)`**: Tính tích vô hướng giữa ma trận $W_{\text{out}}$ và lỗi $error$.
* Điều này giúp xác định độ thay đổi của các vector **context** trong không gian ẩn.
    
Tính toán:
$$\text{grad\_h} =  
\begin{bmatrix}  
0.2 & 0.1 \\  
0.4 & 0.5 \\  
0.3 & 0.6  
\end{bmatrix}  
\cdot  
\begin{bmatrix}  
0.313 \\  
-0.679 \\  
0.365  
\end{bmatrix}  
=  
\begin{bmatrix}  
0.2 \times 0.313 + 0.1 \times -0.679 + 0.4 \times 0.365 \\  
0.4 \times 0.313 + 0.5 \times -0.679 + 0.3 \times 0.365 \\  
0.3 \times 0.313 + 0.6 \times -0.679 + 0.5 \times 0.365  
\end{bmatrix}$$ $$\text{grad\_h} =  
\begin{bmatrix}  
0.0626 + (-0.0679) + 0.146 = 0.1407 \\  
0.1252 + (-0.3395) + 0.1095 = -0.1048 \\  
0.0939 + (-0.4074) + 0.1825 = -0.131  
\end{bmatrix}$$

Kết quả là:
$$\text{grad\_h} = [0.1407, -0.1048, -0.131]$$

**4. Gradient cho input layer (grad_W_in)**

Cuối cùng, chúng ta tính **gradient cho ma trận $W_{\text{in}}$**, tức là những thay đổi cần thiết cho các vector context trong quá trình học. Gradient này được tính bằng cách tính tổng của các gradient ảnh hưởng từ các từ context.
* **`grad_W_in[context[i]] += grad_h / len(context)`**: Vì trong CBOW, chúng ta sử dụng **average context vectors**, nên gradient cho từng vector context sẽ được tính trung bình.
    

Tính gradient cho từng từ trong context:
* **context = ["i", "learning"]** (với chỉ số từ là `0` và `2`).
* **grad_h = [0.1407, -0.1048, -0.131]**

Cập nhật gradient cho **W_in**:

$$\text{grad\_W\_in}[0] += \frac{1}{2} [0.1407, -0.1048, -0.131]$$ $$\text{grad\_W\_in}[2] += \frac{1}{2} [0.1407, -0.1048, -0.131]$$

Kết quả sẽ là:
$$\text{grad\_W\_in} =  
\begin{bmatrix}  
0.07035 & -0.0524 & -0.0655 \\  
0 & 0 & 0 \\  
0.07035 & -0.0524 & -0.0655  
\end{bmatrix}$$

* * *

**5. Cập nhật trọng số**
Cuối cùng, chúng ta sử dụng **gradient descent** để cập nhật các trọng số:
$$W_{\text{in}} \leftarrow W_{\text{in}} - \eta \times \text{grad\_W\_in}$$ $$W_{\text{out}} \leftarrow W_{\text{out}} - \eta \times \text{grad\_W\_out}$$

Trong đó:
* $\eta$ là learning rate (giả sử là 0.025).
* 
