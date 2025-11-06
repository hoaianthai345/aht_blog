

```Python
# skipgram_numpy_from_scratch.py
# Author: you & ChatGPT — Minimal, readable, no deep deps.

import re
import math
import random
import numpy as np
from collections import Counter, defaultdict

# ----------------------------
# 1) Utils
# ----------------------------
def tokenize(text):
    """Tokenize rất đơn giản: chữ thường + tách theo \w+"""
    return re.findall(r"[a-zA-Z']+", text.lower())

def build_vocab(tokens, min_count=1):
    """Tạo vocab và lọc từ hiếm."""
    freq = Counter(tokens)
    vocab = [w for w, c in freq.items() if c >= min_count]
    vocab.sort()
    word2id = {w:i for i, w in enumerate(vocab)}
    id2word = {i:w for w,i in word2id.items()}
    counts = np.array([freq[w] for w in vocab], dtype=np.float64)
    return word2id, id2word, counts

def subsample_tokens(tokens, counts, word2id, t=1e-5):
    """Subsampling theo paper word2vec: keep prob ~ min(1, sqrt(t/f) + t/f).
       Cho corp nhỏ có thể tắt = None."""
    if t is None:
        return tokens
    total = sum(counts)
    freqs = {w: counts[word2id[w]] / total for w in word2id}
    kept = []
    for w in tokens:
        if w not in word2id: 
            continue
        f = freqs[w]
        p_keep = min(1.0, (math.sqrt(t/f) + t/f))
        if random.random() < p_keep:
            kept.append(w)
    return kept

def make_skipgram_pairs(token_ids, window_size=2):
    """Sinh cặp (center, context) với cửa sổ cố định."""
    pairs = []
    for i, c in enumerate(token_ids):
        left = max(0, i - window_size)
        right = min(len(token_ids), i + window_size + 1)
        for j in range(left, right):
            if j == i: 
                continue
            pairs.append((c, token_ids[j]))
    random.shuffle(pairs)
    return pairs

def make_unigram_table(counts, power=0.75):
    """Phân phối negative sampling ∝ count^power (không dùng alias table, đủ cho corp nhỏ)."""
    p = counts ** power
    p = p / p.sum()
    return p

def sigmoid(x):
    # clip để ổn định số học
    x = np.clip(x, -10, 10)
    return 1.0 / (1.0 + np.exp(-x))

def cosine_sim(a, b):
    a_norm = a / (np.linalg.norm(a) + 1e-9)
    b_norm = b / (np.linalg.norm(b) + 1e-9)
    return float(np.dot(a_norm, b_norm))

# ----------------------------
# 2) Skip-Gram NS Model
# ----------------------------
class SkipGramNS:
    def __init__(self, vocab_size, embed_dim=100, seed=42):
        rng = np.random.default_rng(seed)
        # Ma trận embedding input (V x D) và output (V x D)
        self.W_in = rng.normal(0, 0.01, size=(vocab_size, embed_dim)).astype(np.float64)
        self.W_out = rng.normal(0, 0.01, size=(vocab_size, embed_dim)).astype(np.float64)

    def train(
        self,
        pairs,
        neg_dist,
        neg_k=5,
        lr=0.025,
        epochs=2,
        anneal=True,
        report_every=10000
    ):
        """Huấn luyện SGD từng cặp với K negative."""
        V, D = self.W_in.shape
        step = 0
        for ep in range(1, epochs+1):
            random.shuffle(pairs)
            total_loss = 0.0

            for c_id, o_id in pairs:
                # 1) Sample negatives
                neg_ids = np.random.choice(V, size=neg_k, replace=True, p=neg_dist)
                # Tránh trùng positive context (không bắt buộc)
                neg_ids[neg_ids == o_id] = np.random.choice(V, p=neg_dist)

                v_c = self.W_in[c_id]               # (D,)
                u_o = self.W_out[o_id]              # (D,)
                u_neg = self.W_out[neg_ids]         # (K, D)

                # 2) Scores
                s_pos = np.dot(v_c, u_o)            # scalar
                s_neg = np.dot(u_neg, v_c)          # (K,)

                # 3) Sigmoid
                p_pos = sigmoid(s_pos)              # σ(v·u_o)
                p_neg = sigmoid(s_neg)              # σ(u_n·v)

                # 4) Loss: -log σ(pos) - Σ log σ(-neg)
                # log σ(-x) = log(1 - σ(x))
                loss = -np.log(p_pos + 1e-12) - np.sum(np.log(1.0 - p_neg + 1e-12))
                total_loss += loss

                # 5) Gradients
                # dL/dv = (σ(pos)-1)*u_o + Σ σ(neg)*u_n
                grad_v = (p_pos - 1.0) * u_o + np.sum(p_neg[:, None] * u_neg, axis=0)

                # dL/du_o = (σ(pos)-1)*v
                grad_uo = (p_pos - 1.0) * v_c

                # dL/du_n = σ(neg)*v  (cho từng negative)
                grad_un = (p_neg[:, None] * v_c[None, :])

                # 6) Update
                self.W_in[c_id]  -= lr * grad_v
                self.W_out[o_id] -= lr * grad_uo
                self.W_out[neg_ids] -= lr * grad_un

                # 7) LR anneal nhẹ theo bước (tùy chọn)
                step += 1
                if anneal and step % 100000 == 0:
                    lr = max(lr * 0.9, 0.0005)

                if report_every and step % report_every == 0:
                    avg = total_loss / report_every
                    print(f"[epoch {ep}] step={step} avg_loss={avg:.4f}")
                    total_loss = 0.0

    def most_similar(self, word, word2id, id2word, topk=10, use='in'):
        """Tìm từ gần nhất theo cosine trong không gian input (mặc định) hoặc output."""
        if word not in word2id:
            return []
        wid = word2id[word]
        vec = (self.W_in if use == 'in' else self.W_out)[wid]
        mat = self.W_in if use == 'in' else self.W_out

        sims = mat @ (vec / (np.linalg.norm(vec) + 1e-9))
        # loại chính nó
        sims[wid] = -1.0
        top_ids = np.argsort(-sims)[:topk]
        return [(id2word[i], float(sims[i])) for i in top_ids]


# ----------------------------
# 3) Demo ngắn
# ----------------------------
if __name__ == "__main__":
    random.seed(0)
    np.random.seed(0)

    # Một corpus toy; bạn thay bằng văn bản lớn hơn để học tốt hơn
    raw_text = """
    We are what we repeatedly do. Excellence, then, is not an act, but a habit.
    The only limit to our realization of tomorrow is our doubts of today.
    Not all those who wander are lost. To be, or not to be.
    """

    tokens = tokenize(raw_text)
    word2id, id2word, counts = build_vocab(tokens, min_count=1)

    # (Optional) subsample — với corpus nhỏ, nên tắt để khỏi mất dữ liệu
    tokens_kept = subsample_tokens(tokens, counts, word2id, t=None)

    token_ids = [word2id[w] for w in tokens_kept if w in word2id]
    pairs = make_skipgram_pairs(token_ids, window_size=2)

    neg_dist = make_unigram_table(counts, power=0.75)

    model = SkipGramNS(vocab_size=len(word2id), embed_dim=50, seed=42)
    model.train(
        pairs,
        neg_dist=neg_dist,
        neg_k=5,
        lr=0.05,
        epochs=5,
        anneal=True,
        report_every=2000
    )

    # Thử xem từ gần nhất
    query_words = ["not", "be", "habit", "limit", "today"]
    for qw in query_words:
        sims = model.most_similar(qw, word2id, id2word, topk=5)
        print(f"\nNearest to '{qw}':")
        for w, s in sims:
            print(f"  {w:>10s}  cos={s:.3f}")

```


#### Subsampling
Loại bỏ các tokens có tần suất xuất hiện nhiều nhưng không có quá nhiều ý nghĩa. (Loại bỏ stopword)
```Python
def subsample_tokens(tokens, counts, word2id, t=1e-5):
    """Subsampling theo paper word2vec: keep prob ~ min(1, sqrt(t/f) + t/f).
       Cho corp nhỏ có thể tắt = None."""
    if t is None:
        return tokens
    total = sum(counts)
    freqs = {w: counts[word2id[w]] / total for w in word2id}
    kept = []
    for w in tokens:
        if w not in word2id: 
            continue
        f = freqs[w]
        p_keep = min(1.0, (math.sqrt(t/f) + t/f))
        if random.random() < p_keep:
            kept.append(w)
    return kept
```

Ở đây hàm sẽ tính tần suất xuất hiện của từ và so với xác suất giữ lại bằng công thức sau:
$$P_{\text{keep}}(w) = \min\left(1, \sqrt{\frac{t}{f}} + \frac{t}{f}\right)$$
Trong đó:
-  $f$ là tần suất của từ
-  $t$ là tham số ngưỡng được cung cấp (có thể hiểu như một ngưỡng thấp cho tần suất).
- Nếu $\sqrt{\frac{t}{f}} + \frac{t}{f}$ lớn hơn 1 thì ta lấy 1.
Sau đó Hàm `random.random()` sinh ra một số ngẫu nhiên từ 0 đến 1. Nếu giá trị này nhỏ hơn $P_{\text{keep}}$ sẽ được giữ lại.

***
#### Lựa chọn cặp center
```Python
def make_skipgram_pairs(token_ids, window_size=2):
    """Sinh cặp (center, context) với cửa sổ cố định."""
    pairs = []
    for i, c in enumerate(token_ids):
        left = max(0, i - window_size)
        right = min(len(token_ids), i + window_size + 1)
        for j in range(left, right):
            if j == i: 
                continue
            pairs.append((c, token_ids[j]))
    random.shuffle(pairs)
    return pairs
```

Ví dụ:
![[image-105.png]]

***
#### Tham số cho class NSkip
```python
def __init__(self, vocab_size, embed_dim=100, seed=42):
        rng = np.random.default_rng(seed)
        # Ma trận embedding input (V x D) và output (V x D)
        self.W_in = rng.normal(0, 0.01, size=(vocab_size, embed_dim)).astype(np.float64)
        self.W_out = rng.normal(0, 0.01, size=(vocab_size, embed_dim)).astype(np.float64)
```
- Vocab_size: Số lượng từ vựng
- Embed_dim: số lượng chiều Embedding.

Khởi tạo ngẫu nhiên ma trận có kích thước Vocab_size x  Embed_dim:
$$W_{\text{in}}=
\begin{bmatrix}
0.10 & 0.00\\
\color{ #0b5 }{0.00} & \color{ #0b5 }{0.20}\\
-0.10 & 0.10\\
0.05 & -0.05
\end{bmatrix},\qquad
W_{\text{out}}=
\begin{bmatrix}
\color{ #0b5 }{0.00} & \color{ #0b5 }{0.10}\\
0.10 & -0.10\\
-0.10 & -0.10\\
0.00 & -0.20
\end{bmatrix}$$
- Hàng $i$ của $W_{\text{in}}$ là **vector “center”** của từ có id $i$.
- Hàng $i$ của $W_{\text{out}}$ là **vector “context”** của từ có id $i$.
- Ở đây dùng phân phối chuẩn $\mathcal N(0, 0.01^2)$ → **ngẫu nhiên nhỏ quanh 0** để:
	- phá vỡ tính đối xứng (mỗi từ bắt đầu khác nhau)
	- dot-product ban đầu $\approx 0$ -> Không làm **sigmoid bão hoà**.

***
#### Train function
**Tham số:**
```python
def train(
        self,
        pairs,
        neg_dist,
        neg_k=5,
        lr=0.025,
        epochs=2,
        anneal=True,
        report_every=10000
    ):
```
- `pairs`: danh sách các cặp **(center, context)** đã được tạo ra từ văn bản
-  `neg_dist` : phân phối lấy mẫu âm (negative sampling distribution) cho các từ âm (negative words)
- `neg_k`: Số lượng từ âm (negative samples) sẽ được lấy cho mỗi cặp (center, context) trong mỗi bước huấn luyện.
- `lr`: learning rate
- `anneal`: có **giảm dần learning rate** hay không.
- `report_every`: sau bao nhiêu bước huấn luyện mô hình sẽ in ra báo cáo tình trạng

**Giảm dần learning rate theo từng lần:**
```python
	step += 1
	if anneal and step % 100000 == 0:
		lr = max(lr * 0.9, 0.0005)
```
- Khi `anneal=True`, learning rate sẽ được giảm đi sau mỗi vài bước huấn luyện. Điều này giúp mô hình dần ổn định khi gần hội tụ, tránh việc học quá mạnh ở các vòng lặp sau cùng. Phương pháp **learning rate annealing** giúp tìm ra tối ưu toàn cầu thay vì tối ưu cục bộ.
- **Lý do**: Việc giảm dần learning rate giúp mô hình tránh bị quá điều chỉnh, đặc biệt là trong các bước cuối cùng khi mà mô hình đã học được nhiều từ điển ngữ nghĩa.

**Quy trình tính toán và cập nhật weight bằng Gradient:**
- Từ hai ma trận ta đã khởi tạo ban đầu (giống như khởi tạo trước weight ở linear regression), ta lấy ra các vector từ cần tính toán trong vòng lặp:
Ví dụ ta có Vocab (ID):
$\texttt{i}\to 0,\ \texttt{like}\to 1,\ \texttt{nlp}\to 2,\ \texttt{deep}\to 3$
- Chọn cặp train (center, context): $(\texttt{like}, \texttt{i}) = (c=1,\ o=0)$.  
- Negative đã sample: $\{n_1=\texttt{nlp}=2,\ n_2=\texttt{deep}=3\}$.  
- Siêu tham số: $D=2,\ K=2,\ \eta=\text{lr}=0.1$.
Khởi tạo (giả định để tính tay):
$$
W_{\text{in}}=
\begin{bmatrix}
0.10 & 0.00\\
\color{#0b5}{0.00} & \color{#0b5}{0.20}\\
-0.10 & 0.10\\
0.05 & -0.05
\end{bmatrix},\qquad
W_{\text{out}}=
\begin{bmatrix}
\color{#0b5}{0.00} & \color{#0b5}{0.10}\\
0.10 & -0.10\\
-0.10 & -0.10\\
0.00 & -0.20
\end{bmatrix}
$$
$$
\mathbf v_c = W_{\text{in}}[1] = \begin{bmatrix}0\\0.20\end{bmatrix},\quad
\mathbf u_o = W_{\text{out}}[0] = \begin{bmatrix}0\\0.10\end{bmatrix},\quad
\mathbf u_{n_1} = W_{\text{out}}[2] = \begin{bmatrix}-0.10\\-0.10\end{bmatrix},\quad
\mathbf u_{n_2} = W_{\text{out}}[3] = \begin{bmatrix}0\\-0.20\end{bmatrix}.
$$ Với: 
- $\mathbf v_c$ là vector của từ center
- $\mathbf u_o$ là vector của từ context
- $\mathbf u_{n1}$ là vector của từ negative

Tích tích vô hướng -> Thể hiện độ liên quan giữa các cặp từ:
$$\begin{aligned}
S_{\text{pos}} &= \mathbf v_c^\top \mathbf u_o
= 0\cdot 0 + 0.20\cdot 0.10 = \boxed{0.02},\\[4 pt]
S_{n_1} &= \mathbf u_{n_1}^\top \mathbf v_c
= (-0.10)\cdot 0 + (-0.10)\cdot 0.20
= \boxed{-0.02},\\[4 pt]
S_{n_2} &= \mathbf u_{n_2}^\top \mathbf v_c
= 0\cdot 0 + (-0.20)\cdot 0.20
= \boxed{-0.04}.
\end{aligned}$$
Sau đó ta dùng sigmoid $\sigma(x)=\frac{1}{1+e^{-x}}$ để chuẩn hóa:
$$
\begin{aligned}
\sigma(s_{\text{pos}}) &\approx \sigma(0.02) \approx \boxed{0.5050},\\
\sigma(s_{n_1}) &\approx \sigma(-0.02) \approx \boxed{0.4950},\\
\sigma(s_{n_2}) &\approx \sigma(-0.04) \approx \boxed{0.4900}.
\end{aligned}
$$
Tính loss:
$$
\mathcal{L} = - \log \sigma(\mathbf{v}_c^\top \mathbf{u}_o) - \sum_{k=1}^{K} \log \sigma(- \mathbf{v}_c^\top \mathbf{u}_{n_k})
$$
$$
\mathcal L
= -\log \sigma(s_{\text{pos}})
- \sum_{k=1}^{2} \log\!\bigl(1-\sigma(s_{n_k})\bigr)
\approx -\log(0.5050) - \log(0.5050) - \log(0.5100)
\approx \boxed{2.039}.
$$
- Đối với cặp (center, context), ta muốn **tích vô hướng** giữa vector center và vector context là **cao**. Điều này có nghĩa là ta muốn mô hình học sao cho khi từ center xuất hiện, từ context có khả năng xuất hiện cao.
- Mỗi negative sample $n_k$ là một từ không liên quan tới từ center. Ta muốn **đảm bảo rằng xác suất xuất hiện của từ âm là thấp**. -> Vì vậy lấy phủ định là $1-\sigma(s_{n_k})$ 

Gradient
$$\begin{aligned}  
\frac{\partial \mathcal L}{\partial \mathbf v_c}  
&= (\sigma(s_{\text{pos}})-1)\,\mathbf u_o  
+ \sum_{k=1}^2 \sigma(s_{n_k})\,\mathbf u_{n_k},\\[4pt]  
\frac{\partial \mathcal L}{\partial \mathbf u_o}  
&= (\sigma(s_{\text{pos}})-1)\,\mathbf v_c,\\[4pt]  
\frac{\partial \mathcal L}{\partial \mathbf u_{n_k}}  
&= \sigma(s_{n_k})\,\mathbf v_c.  
\end{aligned}$$

Thay số:
$$\sigma(s_{\text{pos}})-1 \approx 0.5050-1 = \boxed{-0.4950}.$$
* Thành phần cho $\partial \mathcal L/\partial \mathbf v_c$:

$$\underbrace{(-0.4950)\begin{bmatrix}0\\0.10\end{bmatrix}}_{\small [-0,\, -0.0495]}  
\ +\  
\underbrace{0.4950\begin{bmatrix}-0.10\\-0.10\end{bmatrix}}_{\small [-0.0495,\,-0.0495]}  
\ +\  
\underbrace{0.4900\begin{bmatrix}0\\-0.20\end{bmatrix}}_{\small [0,\,-0.0980]}  
= \boxed{\begin{bmatrix}-0.0495\\-0.1970\end{bmatrix}}.$$
* Gradient cho positive context:
$$\frac{\partial \mathcal L}{\partial \mathbf u_o}  
= (-0.4950)\begin{bmatrix}0\\0.20\end{bmatrix}  
= \boxed{\begin{bmatrix}0\\-0.0990\end{bmatrix}}.$$

* Gradient cho từng negative:
$$\frac{\partial \mathcal L}{\partial \mathbf u_{n_1}}  
= 0.4950\begin{bmatrix}0\\0.20\end{bmatrix}  
= \boxed{\begin{bmatrix}0\\0.0990\end{bmatrix}},\quad  
\frac{\partial \mathcal L}{\partial \mathbf u_{n_2}}  
= 0.4900\begin{bmatrix}0\\0.20\end{bmatrix}  
= \boxed{\begin{bmatrix}0\\0.0980\end{bmatrix}}.$$

Cập nhật tham số (SGD)
Với $\eta=0.1$: $\ \theta\leftarrow \theta-\eta\,\nabla_\theta$
* Center:
$$\mathbf v_c^{\text{new}}  
=  
\begin{bmatrix}0\\0.20\end{bmatrix}  
- 0.1\!\cdot\!  
\begin{bmatrix}-0.0495\\-0.1970\end{bmatrix}  
=  
\boxed{\begin{bmatrix}0.00495\\0.21970\end{bmatrix}}.$$

* Positive context:
$$\mathbf u_o^{\text{new}}  
=  
\begin{bmatrix}0\\0.10\end{bmatrix}  
- 0.1\!\cdot\!  
\begin{bmatrix}0\\-0.0990\end{bmatrix}  
=  
\boxed{\begin{bmatrix}0\\0.10990\end{bmatrix}}.$$

* Negative 1:
$$\mathbf u_{n_1}^{\text{new}}  
=  
\begin{bmatrix}-0.10\\-0.10\end{bmatrix}  
- 0.1\!\cdot\!  
\begin{bmatrix}0\\0.0990\end{bmatrix}  
=  
\boxed{\begin{bmatrix}-0.10\\-0.10990\end{bmatrix}}.$$

* Negative 2:
$$\mathbf u_{n_2}^{\text{new}}  
=  
\begin{bmatrix}0\\-0.20\end{bmatrix}  
- 0.1\!\cdot\!  
\begin{bmatrix}0\\0.0980\end{bmatrix}  
=  
\boxed{\begin{bmatrix}0\\-0.20980\end{bmatrix}}.$$

Kiểm tra “hướng” cập nhật (tăng pos, giảm neg)

Tính lại score sau update:
$$\begin{aligned}  
s_{\text{pos}}^{\text{new}}  
&= (\mathbf v_c^{\text{new}})^\top \mathbf u_o^{\text{new}}  
= 0.00495\cdot 0 + 0.21970\cdot 0.10990  
= \boxed{0.02414503}\ (>0.02),\\[4pt]  
s_{n_1}^{\text{new}}  
&= (\mathbf u_{n_1}^{\text{new}})^\top \mathbf v_c^{\text{new}}  
= (-0.10)\cdot 0.00495 + (-0.10990)\cdot 0.21970  
= \boxed{-0.02464003}\ (<-0.02),\\[4pt]  
s_{n_2}^{\text{new}}  
&= (\mathbf u_{n_2}^{\text{new}})^\top \mathbf v_c^{\text{new}}  
= 0\cdot 0.00495 + (-0.20980)\cdot 0.21970  
= \boxed{-0.04609306}\ (<-0.04).  
\end{aligned}$$

Kết quả đúng “trực giác”: score dương (pos) tăng, score âm (negatives) giảm — phù hợp mục tiêu tối ưu.

![[image-107.png]]
***
#### Most similar

```python
def most_similar(self, word, word2id, id2word, topk=10, use='in'):
        """Tìm từ gần nhất theo cosine trong không gian input (mặc định) hoặc output."""
        if word not in word2id:
            return []
        wid = word2id[word]
        vec = (self.W_in if use == 'in' else self.W_out)[wid]
        mat = self.W_in if use == 'in' else self.W_out

        sims = mat @ (vec / (np.linalg.norm(vec) + 1e-9))
        # loại chính nó
        sims[wid] = -1.0
        top_ids = np.argsort(-sims)[:topk]
        return [(id2word[i], float(sims[i])) for i in top_ids]
```

Sau khi có được các vector từ trong hai ma trận in và out thì ta có thể tìm các từ liên quan nhau bằng cosine similarity.
- $W_{\text{in}}$ Embedding của center word (từ trung tâm): Mỗi dòng là **embedding của một từ khi nó làm center word**.
- $W_{\text{out}}$ Embedding của context word (từ ngữ cảnh): Mỗi hàng trong ma trận này đại diện cho một **vector embedding** của một từ trong từ vựng khi từ đó đóng vai trò là **context word** (từ ngữ cảnh).
Cách dùng:
**Vector của từ center** được dùng để dự đoán các từ context
**Vector của từ context** được dùng để tối đa hóa xác suất dự đoán
