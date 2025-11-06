---
Author: HoàiAn Thái
Date: Invalid date
Status: Not started
Tag:
  - Research
---
> [!important] **Table of Content**
> 
> ---
> 
> - [[#Link]]
> - [[#I. Import thư viện và các tài nguyên]]
>     - [[#2. Pre-trained word embeddings]]
>         - [[#fastText embedding]]
>         - [[#word2vec]]
> - [[#II. Dataset]]
>     - [[#vi_preprocess()]]
>     - [[#1. Class Dataloader]]
>         - [[#load_data()]]
> - [[#III. BiLSTM Model]]
>     - [[#init()]]
>     - [[#build_model()]]
>     - [[#summary()]]
>     - [[#Hàm save() và load()]]
> - [[#IV. Trainer]]
>     - [[#_auto_build()]]
>     - [[#default_callbacks()]]
>     - [[#train()]]
>     - [[#2. Class BinaryMetricsCallback]]
> - [[#V. Evaluation]]
>     - [[#__init__]]
>     - [[#evaluate()]]
>     - [[#classification_report()]]
>     - [[#plot_history()]]
>     - [[#confusion_matrix()]]
>     - [[#predict_single()]]
>     - [[#plot_roc_auc]]
> - [[#Chạy chương trình]]
>     - [[#Config]]
> 
>   
  

> [!important]
> 
> # Link
> 
> - Colab:
> - Tham khảo: [https://github.com/kimkim00/VLSP_ReINTEL/blob/main/biLSTM.ipynb](https://github.com/kimkim00/VLSP_ReINTEL/blob/main/biLSTM.ipynb)
> 
> .
> 
> .
> 
> .
> 
> .
> 
> .
> 
> .
> 
> .
  
# I. Import thư viện và các tài nguyên
```Python
!pip install seaborn -q
!pip install scikit-learn -q
!pip install tensorflow -q
!pip install keras -q
!pip install -U gdown
```
  
**1.** `**seaborn**`
Thư viện trực quan hóa dữ liệu (data visualization), xây dựng trên nền `matplotlib`, cung cấp nhiều biểu đồ đẹp mắt, dễ tùy chỉnh.
2. `scikit-learn`
Thư viện máy học (machine learning) phổ biến, cung cấp đầy đủ thuật toán như: hồi quy (regression), phân loại (classification), phân cụm (clustering), giảm chiều dữ liệu (dimensionality reduction).
3. `tensorflow`
Thư viện mã nguồn mở của Google cho deep learning và AI, hỗ trợ xây dựng và huấn luyện mạng neuron phức tạp trên GPU.
4. `keras`
API cấp cao (high-level API) cho việc xây dựng, huấn luyện các mô hình deep learning, dễ dùng hơn so với việc code thuần TensorFlow.
5. `gdown`
Tiện ích tải file từ Google Drive về máy, rất hữu ích khi làm việc trên Colab hoặc khi cần chia sẻ bộ dữ liệu lớn.
---
## **2. Pre-trained word embeddings**
### **fastText embedding**
```Python
!wget https://dl.fbaipublicfiles.com/fasttext/vectors-crawl/cc.vi.300.vec.gz
```
  
`!wget`: **"Web get"** công cụ dòng lệnh (command-line utility) dùng để tải file từ Internet về máy tính. Ở đây tải về RAM google colab.
==**1 Giới thiệu fastText**==
**fastText embedding** là phương pháp biến đổi từ ngữ thành vector số, được phát triển bởi Facebook AI Research. Điểm đặc biệt của fastText so với các phương pháp embedding trước đó (như Word2Vec) là nó xem mỗi từ như một tập hợp các ký tự con (subword), giúp hiểu được nghĩa của từ ngay cả khi từ đó hiếm gặp hoặc là từ mới.
**2 Cách hoạt động**
Thay vì chỉ học vector cho từng từ hoàn chỉnh, fastText còn học vector cho các “n-gram ký tự” bên trong từ.
Khi gặp một từ mới (chưa từng thấy), fastText sẽ lấy trung bình các vector n-gram của từ đó để tạo ra embedding tương ứng.
**Ví dụ**:
Từ “học_sinh” có thể được tách thành các n-gram như: “họ”, “ọc”, “_s”, “si”, “inh”. Embedding của “học_sinh” sẽ là trung bình cộng của các vector này.
**3 Công thức toán**
Giả sử một từ www có tập các n-gram ký tự là GwG_wGw, embedding của từ www được tính bằng:
vw=1∣Gw∣∑g∈Gwzg\mathbf{v}_w = \frac{1}{|G_w|} \sum_{g \in G_w} \mathbf{z}_g
vw=∣Gw∣1g∈Gw∑zg
Trong đó:
vw\mathbf{v}_wvw: vector của từ www
zg\mathbf{z}_gzg: vector embedding của n-gram ggg
GwG_wGw: tập hợp các n-gram của từ www
---
### word2vec
  
==**word2vec là gì?**==
word2vec học vector cho từ dựa trên mối quan hệ ngữ cảnh: từ nào thường xuất hiện gần nhau trong văn bản sẽ có vector gần nhau. Có hai kiến trúc chính để huấn luyện:
- CBOW (Continuous Bag-of-Words): dự đoán một từ dựa trên các từ xung quanh (ngữ cảnh).
- Skip-gram: dự đoán các từ ngữ cảnh dựa trên một từ trung tâm.
**Ví dụ**: Nếu trong văn bản “tôi thích uống trà đá”, từ “uống” và “trà” xuất hiện gần nhau, word2vec sẽ học sao cho vector của “uống” gần với “trà”.
---
  
---
  
---
# II. Dataset
### `vi_preprocess()`
```Python
def vi_preprocess(text):
    # Lowercase
    text = text.lower()
    # Loại bỏ ký tự không phải chữ/số
    text = re.sub(r"[^\w\s]", " ", text)
    # Loại bỏ số, nếu muốn
    # text = re.sub(r"\d+", " ", text)
    # Chuẩn hóa khoảng trắng
    text = re.sub(r"\s+", " ", text).strip()
    return text
```
  
---
## 1. Class Dataloader
```Python
import yaml
import pandas as pd
import numpy as np
import re
from tensorflow.keras.preprocessing.text import Tokenizer
from tensorflow.keras.preprocessing.sequence import pad_sequences
class DataLoader:
    def __init__(self, config_path, mode='train', embedding_type=None):
        # Đọc file config
        with open(config_path, 'r', encoding='utf-8') as f:
            self.config = yaml.safe_load(f)
        self.mode = mode
        # Lấy đường dẫn file theo mode (train/test/val/warmup)
        paths = self.config['paths']
        if mode not in paths:
            raise ValueError(f"Mode '{mode}' không có trong config.")
        self.data_path = paths[mode]
        # Thay thế {paths.base} nếu cần
        base = paths.get("base", "")
        if "{paths.base}" in self.data_path:
            self.data_path = self.data_path.replace("{paths.base}", base)
        # Các tham số tiền xử lý
        bilstm_conf = self.config['preprocessing']['bilstm']
        self.max_num_words = bilstm_conf.get('max_features', 2500)
        self.max_seq_length = bilstm_conf.get('maxlen', 500)
        self.clean_text = bilstm_conf.get('clean_text', True)
        self.tokenizer = Tokenizer(num_words=self.max_num_words, oov_token="<OOV>")
        self.texts, self.labels = self.load_data()
        self.prepare_data()
        # Xử lý embedding ngoài nếu được yêu cầu
        self.embedding_matrix = None
        if embedding_type is not None:
            self.embedding_matrix = self.get_embedding_matrix(embedding_type)
```
==**Mục tiêu chính**==
Lớp `DataLoader` này là một “module trung gian” giúp bạn đọc config, tải và xử lý dữ liệu văn bản (text) theo các tham số thiết lập sẵn – thường dùng cho các dự án học sâu với mô hình như BiLSTM, CNN, RNN, v.v.
Nó tự động chuẩn hóa, token hóa, chuyển đổi văn bản thành chuỗi số, và có thể tích hợp embedding ngoài như word2vec hoặc fastText nếu cần.
==**Các thư viện được import**==
- `yaml`: Đọc file cấu hình YAML (config các tham số, đường dẫn dữ liệu)
- `pandas`: Xử lý dữ liệu dạng bảng (DataFrame)
- `numpy`: Tính toán số học, ma trận
- `re`: Xử lý, làm sạch văn bản bằng biểu thức chính quy
- `tensorflow.keras.preprocessing.text.Tokenizer`: Chuyển đổi văn bản thành chuỗi số (tokenization)
- `tensorflow.keras.preprocessing.sequence.pad_sequences`: Đảm bảo mọi chuỗi đều cùng chiều dài (padding)
==**Phần khởi tạo (**====`**__init__**`====**)**==
- **Nhận đường dẫn file config và mode (train/test/val/warmup)**
- Đọc file cấu hình bằng `yaml.safe_load`, lưu thành biến `self.config`
- Xác định đường dẫn file dữ liệu (data_path) dựa trên `mode`
    
    Ví dụ: Nếu mode='train', nó sẽ tìm trong config đoạn:
    
    ```YAML
    paths:
      train: "{paths.base}/train.csv"
      base: "/content/data"
    ```
    
    => Kết quả, `self.data_path` sẽ thành `/content/data/train.csv`
    
- Đọc các tham số tiền xử lý:
    - Số lượng từ tối đa: `max_features`
    - Độ dài chuỗi tối đa: `maxlen`
    - Có làm sạch văn bản không: `clean_text`
- Khởi tạo tokenizer của Keras để chuyển đổi văn bản thành số, hỗ trợ cả từ ngoài từ điển (OOV: out-of-vocabulary)
- Gọi `self.load_data()` để đọc dữ liệu, gán vào `self.texts` và `self.labels`
- Gọi `self.prepare_data()` để thực hiện tokenization + padding chuỗi về cùng chiều dài
- Nếu bạn truyền thêm tham số `embedding_type`, lớp sẽ tự động load embedding matrix bên ngoài (vd. từ word2vec, fastText, glove...) thông qua hàm `get_embedding_matrix()`
---
### `load_data()`
```Python
    def load_data(self):
        df = pd.read_csv(self.data_path)
        texts = df['post_message'].astype(str).tolist()
        if self.clean_text:
            texts = [vi_preprocess(t) for t in texts]
        if 'label' in df.columns:
            labels = df['label'].astype(int).values
        else:
            labels = None
        return texts, labels
```
  
- Đọc dữ liệu:
    - Sử dụng `pd.read_csv(self.data_path)` để đọc file dữ liệu (thường là .csv) thành DataFrame `df`.
- Lấy văn bản:
    - Trích cột `'post_message'` từ DataFrame, ép kiểu sang `str` để đảm bảo là chuỗi ký tự.
    - Chuyển thành list Python bằng `.tolist()`, lưu vào biến `texts`.
- Tiền xử lý văn bản (nếu có):
    - Nếu `self.clean_text` là `True`, áp dụng hàm `vi_preprocess` cho từng phần tử trong `texts`.
    - Hàm `vi_preprocess` thường dùng để chuẩn hóa tiếng Việt (bỏ dấu, chuyển chữ thường, xóa ký tự đặc biệt, v.v).
- Xử lý nhãn:
    - Nếu cột `'label'` tồn tại trong DataFrame:
        - Lấy cột `'label'`, ép kiểu sang số nguyên (`int`), chuyển thành mảng numpy `labels`.
    - Nếu không có cột `'label'`:
        - Gán `labels = None` (dùng cho dữ liệu test chưa gán nhãn).
- Trả về:
    - Trả về tuple `(texts, labels)`, phục vụ cho bước tiếp theo trong pipeline.
  
---
# III. BiLSTM Model
Đây là class `BiLSTMModel` được xây dựng bằng Keras, dùng để tạo mô hình **phân loại văn bản nhị phân** sử dụng mạng **Bi-directional LSTM**.
- **Dễ cấu hình:** Có thể chọn số chiều embedding, số units của LSTM, chiều dài chuỗi, loại loss, optimizer...
- **Linh hoạt loss:** Hỗ trợ `binary_crossentropy` hoặc `focal loss` tùy chọn.
- **Dễ dùng:** Có sẵn các hàm `summary()`, `save()`, `load()` như một wrapper gọn cho Keras.
### **init**()
```Python
from keras.models import Sequential, load_model
from keras.layers import Embedding, Bidirectional, LSTM, Dense, Dropout
class BiLSTMModel:
    def __init__(
        self,
        vocab_size,
        embedding_dim=128,
        max_seq_length=100,
        lstm_units=64,
        dropout_rate=0.2,
        output_dim=1,
        output_activation="sigmoid",
        loss="binary_crossentropy",
        optimizer="adam",
        metrics=["accuracy"]
    ):
        self.vocab_size = vocab_size
        self.embedding_dim = embedding_dim
        self.max_seq_length = max_seq_length
        self.lstm_units = lstm_units
        self.dropout_rate = dropout_rate
        self.output_dim = output_dim
        self.output_activation = output_activation
                # --- CHỌN LOSS THEO THAM SỐ ---
        if str(loss).lower() == "focal":
            self.loss = binary_focal_loss()
            print("=> Sử dụng Focal Loss")
        else:
            self.loss = loss  # "binary_crossentropy" hoặc tên loss khác
            print("=> Sử dụng loss:", loss)
        self.optimizer = optimizer
        self.metrics = metrics
        self.model = self.build_model()
        self.model.compile(
            loss=self.loss,
            optimizer=self.optimizer,
            metrics=self.metrics
        )
```
|Tham số|Mô tả|
|---|---|
|`vocab_size`|Kích thước từ vựng (số lượng token)|
|`embedding_dim`|Số chiều vector từ mỗi token|
|`max_seq_length`|Độ dài tối đa mỗi câu|
|`lstm_units`|Số units trong lớp LSTM|
|`dropout_rate`|Tỉ lệ dropout|
|`output_dim`|Số node đầu ra (1 với nhị phân)|
|`output_activation`|Hàm kích hoạt đầu ra, thường là `"sigmoid"`|
|`loss`|Tên hàm loss hoặc `"focal"` nếu có định nghĩa riêng|
|`optimizer`|Tối ưu hoá, ví dụ `"adam"`|
|`metrics`|Các metric để theo dõi, mặc định là `"accuracy"`|
  
- Cho phép chọn dùng `**focal loss**` nếu được chỉ định. Cần có hàm `binary_focal_loss()` định nghĩa trước đó trong code.
- Nếu không, dùng loss mặc định `"binary_crossentropy"`.
  
  
  
### `build_model()`
```Python
def build_model(self):
    model = Sequential()
    model.add(Embedding(self.vocab_size, self.embedding_dim, input_length=self.max_seq_length))
    model.add(Bidirectional(LSTM(self.lstm_units, return_sequences=False)))
    model.add(Dropout(self.dropout_rate))
    model.add(Dense(self.output_dim, activation=self.output_activation))
    return model
```
- `Embedding`: ánh xạ token ID → vector.
- `Bidirectional(LSTM)`: học thông tin từ cả trước và sau (giúp hiểu ngữ cảnh tốt hơn).
- `return_sequences=False`: chỉ lấy hidden state cuối cùng.
- `Dropout`: giảm overfitting.
- `Dense`: đầu ra sigmoid → phân loại nhị phân.
  
### summary()
```Python
def summary(self):
    return self.model.summary()
```
In kiến trúc mô hình như Keras thường dùng.
  
### Hàm `save()` và `load()`
```Python
def save(self, filepath):
    self.model.save(filepath)
def load(self, filepath):
    self.model = load_model(filepath)
```
Lưu và tải lại mô hình từ file `.h5` hoặc `.keras`.
  
  
---
  
# IV. Trainer
```Python
from keras.models import Sequential, load_model
from keras.layers import Embedding, Bidirectional, LSTM, Dense, Dropout
class BiLSTMModel:
    def __init__(
        self,
        vocab_size,
        embedding_dim=128,
        max_seq_length=100,
        lstm_units=64,
        dropout_rate=0.2,
        output_dim=1,
        output_activation="sigmoid",
        loss="binary_crossentropy",
        optimizer="adam",
        metrics=["accuracy"]
    ):
        self.vocab_size = vocab_size
        self.embedding_dim = embedding_dim
        self.max_seq_length = max_seq_length
        self.lstm_units = lstm_units
        self.dropout_rate = dropout_rate
        self.output_dim = output_dim
        self.output_activation = output_activation
                # --- CHỌN LOSS THEO THAM SỐ ---
        if str(loss).lower() == "focal":
            self.loss = binary_focal_loss()
            print("=> Sử dụng Focal Loss")
        else:
            self.loss = loss  # "binary_crossentropy" hoặc tên loss khác
            print("=> Sử dụng loss:", loss)
        self.optimizer = optimizer
        self.metrics = [
            "accuracy",
            tf.keras.metrics.Precision(name='precision'),
            tf.keras.metrics.Recall(name='recall'),
            tf.keras.metrics.AUC(name='auc')
        ]
        self.model = self.build_model()
        self.model.compile(
            loss=self.loss,
            optimizer=self.optimizer,
            metrics=self.metrics
        )
```
Các tham số quan trọng:
- `model`: mô hình Keras đã được khởi tạo (chưa cần `.fit()`).
- `X_train`, `y_train`: dữ liệu train.
- `X_val`, `y_val`: dữ liệu validation, nếu có.
- `batch_size`, `epochs`: cấu hình huấn luyện.
- `callbacks`: danh sách các callback như EarlyStopping, ModelCheckpoint,...
- `checkpoint_path`: nếu có, tự động lưu model tốt nhất.
- `log_path`: nếu có, lưu lịch sử loss/accuracy ra file CSV.
- `class_weights`: nếu bài toán mất cân bằng, bạn có thể gán trọng số cho từng class.
- Nếu `callbacks` không được truyền, nó tự gán mặc định là `EarlyStopping(patience=3)`.
  
  

> [!important] **Callback** là các **hàm gọi lại tự động được kích hoạt tại các thời điểm cụ thể** trong quá trình huấn luyện (ví dụ: sau mỗi epoch, sau mỗi batch, khi đạt điều kiện nào đó…).
> 
> **Một số callback phổ biến:**
> 
> |Callback|Mục đích chính|
> |---|---|
> |`EarlyStopping`|Dừng huấn luyện nếu loss/accuracy không cải thiện sau vài epoch|
> |`ModelCheckpoint`|Tự động lưu mô hình tốt nhất trong quá trình train|
> |`ReduceLROnPlateau`|Giảm learning rate nếu metric không cải thiện|
> |`TensorBoard`|Ghi log để visualize với TensorBoard|
> |`CSVLogger`|Ghi lại lịch sử huấn luyện vào file `.csv`|
  
Cách chạy
```Python
self.model.compile(
    loss=self.loss,
    optimizer=self.optimizer,
    metrics=self.metrics
)
```
### `_auto_build()`
```Python
def _auto_build(self):
    # Nếu model chưa được build (layers param=0), hãy build trước với shape phù hợp
    # Cách này hoạt động nếu bạn biết shape input (vd: max_seq_length)
    if hasattr(self.model, 'built') and not self.model.built:
        # Lấy shape từ layer đầu tiên nếu có, hoặc đoán theo chuẩn NLP
        for layer in self.model.layers:
            if hasattr(layer, 'input_length'):
                seq_len = getattr(layer, 'input_length', 100)
                self.model.build(input_shape=(None, seq_len))
                break
        # Hoặc dùng cứng nếu bạn biết chắc chiều dài:
        # self.model.build(input_shape=(None, 500))
    # In summary để kiểm tra, có thể comment nếu không cần
    self.model.summary()
```
- Nhiều khi bạn tạo mô hình nhưng **chưa gọi** `**.build()**` **hoặc** `**.fit()**`, thì `model.summary()` sẽ hiển thị `param=0`.
- Phương thức này cố gắng:
    
    - Tìm shape input (`input_length`) trong các layer đầu tiên.
    - Gọi `.build(input_shape=(None, seq_len))` nếu chưa build.
    - Sau đó in `model.summary()` (giúp debug nhanh).
    
      
    
### `default_callbacks()`
```Python
def default_callbacks(self):
    callbacks = [
    EarlyStopping(monitor='val_loss', patience=3, restore_best_weights=True)
]
    if self.log_path:
        log_dir = os.path.dirname(self.log_path)
        if log_dir and not os.path.exists(log_dir):
            os.makedirs(log_dir)
        # CSVLogger: ghi log training vào file CSV
        callbacks.append(CSVLogger(self.log_path, append=False))
        # TensorBoard: log cho TensorBoard
        tensorboard_dir = os.path.join(log_dir, "tensorboard")
        callbacks.append(TensorBoard(log_dir=tensorboard_dir, histogram_freq=1))
    return callbacks
```
`EarlyStopping`:
- Theo dõi `val_loss`.
- Nếu `val_loss` không giảm trong **3 epoch liên tiếp**, quá trình huấn luyện sẽ **dừng sớm**.
- `restore_best_weights=True`: sau khi dừng, mô hình sẽ khôi phục lại **trọng số tốt nhất** (val_loss thấp nhất).
  
Nếu bạn truyền `log_path="logs/training_log.csv"`, phần callback `CSVLogger` và `TensorBoard` sẽ được thêm vào.
  
- Tự động ghi log vào file `.csv` theo từng epoch: `epoch, loss, accuracy, val_loss,...`
- `append=False`: ghi đè (không ghi tiếp vào file cũ).
  
- Ghi log TensorBoard vào thư mục `logs/tensorboard/`.
- `histogram_freq=1`: lưu thống kê histogram của layer mỗi epoch (giúp xem sự phân bố của weight/bias nếu cần).
### `train()`
```Python
def train(self):
        if self.use_validation_data and self.X_val is not None and self.y_val is not None:
            history = self.model.fit(
                self.X_train, self.y_train,
                validation_data=(self.X_val, self.y_val),
                batch_size=self.batch_size,
                epochs=self.epochs,
                callbacks=self.callbacks,
                verbose=self.verbose,
                class_weight=self.class_weights
            )
        else:
            history = self.model.fit(
                self.X_train, self.y_train,
                validation_split=self.validation_split,
                batch_size=self.batch_size,
                epochs=self.epochs,
                callbacks=self.callbacks,
                verbose=self.verbose,
                class_weight=self.class_weights
            )
```
Gồm 2 nhánh:
1. **Dùng validation data riêng**: `validation_data=(X_val, y_val)`
2. **Không có validation set** → dùng `validation_split` từ `X_train`
Kết quả:
- Trả về `history` để vẽ biểu đồ loss/acc.
- Nếu có `log_path`, sẽ gọi `save_history_to_csv()` để lưu log ra file CSV.
  
## 2. Class BinaryMetricsCallback
```Python
from sklearn.metrics import f1_score, recall_score, precision_score, roc_auc_score
from tensorflow.keras.callbacks import Callback
import numpy as np
class BinaryMetricsCallback(Callback):
    def __init__(self, X_val, y_val, threshold=0.5):
        super().__init__()
        self.X_val = X_val
        self.y_val = y_val
        self.threshold = threshold
    def on_epoch_end(self, epoch, logs=None):
        # Dự đoán xác suất
        y_pred_prob = self.model.predict(self.X_val)
        
        # Nhãn dự đoán 0/1 dựa theo ngưỡng threshold
        y_pred = (y_pred_prob > self.threshold).astype(int).reshape(-1)
        
        # Nếu y_val là one-hot vector thì chuyển thành nhãn
        y_true = self.y_val if len(self.y_val.shape) == 1 else np.argmax(self.y_val, axis=1)
        # Tính các chỉ số
        f1 = f1_score(y_true, y_pred, average='binary')
        recall = recall_score(y_true, y_pred, average='binary')
        precision = precision_score(y_true, y_pred, average='binary')
        try:
            auc = roc_auc_score(y_true, y_pred_prob)
        except ValueError:
            auc = float('nan')  # xảy ra khi chỉ có 1 lớp trong batch val
        # In kết quả
        print(f" — val_precision: {precision:.4f} — val_recall: {recall:.4f} — val_f1: {f1:.4f} — val_auc: {auc:.4f}")
        # Ghi vào logs để có thể lưu hoặc dùng với TensorBoard/CSVLogger
        if logs is not None:
            logs['val_precision'] = precision
            logs['val_recall'] = recall
            logs['val_f1'] = f1
            logs['val_auc'] = auc
```
**Callback các chỉ số:**
- **Precision**
- **Recall**
- **F1-score**
- **AUC (Area Under ROC Curve)**
  
`__init__`  
- Hàm khởi tạo lưu lại tập validation (`X_val`, `y_val`) để dùng mỗi khi kết thúc một epoch.
  
`on_epoch_end()`
- Dự đoán kết quả trên tập `X_val` bằng mô hình (`self.model.predict()`).
  
|Chỉ số|Ý nghĩa|
|---|---|
|**Precision**|Tỷ lệ dự đoán dương chính xác: TP / (TP + FP)|
|**Recall**|Tỷ lệ phát hiện đúng dương tính: TP / (TP + FN)|
|**F1-score**|Trung bình hài hòa của Precision và Recall|
|**AUC**|Diện tích dưới đường ROC – độ phân biệt giữa class 0 và 1|
  
  
---
# V. Evaluation
### `__init__`
```Python
import matplotlib.pyplot as plt
from sklearn.metrics import classification_report, confusion_matrix, accuracy_score
class Evaluator:
	def __init__(self, model):
	    self.model = model
```
Gán mô hình Keras đã được huấn luyện (`model`) cho thuộc tính `self.model`.
### `evaluate()`
```Python
def evaluate(self, X_test, y_test, batch_size=32):
    loss, acc = self.model.evaluate(X_test, y_test, batch_size=batch_size, verbose=0)
    print(f"Test Loss: {loss:.4f} | Test Accuracy: {acc:.4f}")
    return loss, acc
```
- Gọi `.evaluate()` của Keras để tính `loss` và `accuracy` trên tập test.
- In ra kết quả loss và accuracy trên tập test.
  
### `**classification_report()**`
```Python
def classification_report(self, X_test, y_test, threshold=0.5):
    y_pred_prob = self.model.predict(X_test)
    if y_pred_prob.shape[-1] == 1:
        y_pred = (y_pred_prob > threshold).astype(int)
    else:
        y_pred = y_pred_prob.argmax(axis=-1)
    print(classification_report(y_test, y_pred))
```
- Dự đoán nhãn trên `X_test`.
- mô hình nhị phân (đầu ra 1 node sigmoid), so sánh xác suất với `threshold` để lấy label (`0` hoặc `1`).
- In ra báo cáo gồm Precision, Recall, F1-score cho từng class.
  
### `plot_history()`
```Python
def plot_history(self, history):
    # loss
    plt.plot(history.history['loss'], label='Train Loss')
    if 'val_loss' in history.history:
        plt.plot(history.history['val_loss'], label='Val Loss')
    # accuracy
    if 'accuracy' in history.history:
        plt.plot(history.history['accuracy'], label='Train Acc')
        if 'val_accuracy' in history.history:
            plt.plot(history.history['val_accuracy'], label='Val Acc')
```
- Nhận `history` trả về từ `.fit()` trong Keras.
- Vẽ biểu đồ loss và accuracy theo từng epoch cho cả train/val (nếu có).
  
### `confusion_matrix()`
```Python
def confusion_matrix(self, X_test, y_test, threshold=0.5):
    y_pred_prob = self.model.predict(X_test)
    if y_pred_prob.shape[-1] == 1:
        y_pred = (y_pred_prob > threshold).astype(int)
    else:
        y_pred = y_pred_prob.argmax(axis=-1)
    cm = confusion_matrix(y_test, y_pred)
    print("Confusion Matrix:\n", cm)
```
Tương tự như `classification_report()`, nhưng in ra ma trận nhầm lẫn (confusion matrix).
### `predict_single()`
```Python
def predict_single(self, input_sequence):
    y_prob = self.model.predict(input_sequence)
    if y_prob.shape[-1] == 1:
        return int(y_prob[0][0] > 0.5)
    else:
        return int(y_prob.argmax(axis=-1)[0])
```
- Dự đoán một mẫu đơn (đã padding sẵn).
- Trả về label `0` hoặc `1` (hoặc index class cao nhất nếu softmax).
  
### `plot_roc_auc`
```Python
    def plot_roc_auc(self, X_test, y_test):
        y_score = self.model.predict(X_test)
        # Binary classification
        fpr, tpr, _ = roc_curve(y_test, y_score)
        roc_auc = auc(fpr, tpr)
        plt.figure()
        plt.plot(fpr, tpr, color='darkorange', lw=2, label=f'ROC curve (AUC = {roc_auc:.2f})')
        plt.plot([0, 1], [0, 1], color='navy', lw=2, linestyle='--')
        plt.xlabel('False Positive Rate')
        plt.ylabel('True Positive Rate')
        plt.title('ROC Curve')
        plt.legend(loc="lower right")
        plt.show()
```
Hàm vẽ ROC và tính AUC
  
---
# Chạy chương trình
### Config