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
> - [[#Tham khảo]]
> - [[#1. Setting]]
>     - [[#%%writefile ".../config_phobert.yaml"]]
>     - [[#mode: train]]
>     - [[#seed: 42]]
>     - [[#paths:]]
>     - [[#preprocessing.phobert.max_sequence_length: 256]]
>     - [[#models.phobert]]
>     - [[#training]]
>     - [[#cv – Cross Validation]]
> - [[#2. Data]]
> - [[#3. PhoBERT]]
> - [[#4. Trainer]]
>     - [[#5. Vòng lặp theo batch]]
>     - [[#6. Đánh giá train & val]]
>     - [[#7. Lưu model tốt nhất]]
>     - [[#8. Early stopping]]
> 
>   

> [!important]
> 
> # Tham khảo
# 1. Setting
```Python
%%writefile "/content/drive/MyDrive/1.PROJECTS/[TeamPe][BIT][BGRA2025]/NLP/Code/config_phobert.yaml"
mode: train
seed: 42
paths:
  train: "/content/drive/MyDrive/1.PROJECTS/[TeamPe][BIT][BGRA2025]/NLP/Code/reintel_dataset/train.csv"
  test: "/content/drive/MyDrive/1.PROJECTS/[TeamPe][BIT][BGRA2025]/NLP/Code/reintel_dataset/test.csv"
  val: "/content/drive/MyDrive/1.PROJECTS/[TeamPe][BIT][BGRA2025]/NLP/Code/reintel_dataset/val.csv"
preprocessing:
  phobert:
    max_sequence_length: 256
models:
  phobert:
    model_name: "vinai/phobert-base"
    hidden_size: 768
    dropout_rate: 0.3
    num_labels: 2
training:
  batch_size: 32
  epochs: 5
  checkpoint_path: "/content/drive/MyDrive/PROJECT/NLP/checkpoints/phobert_best.pt"
  log_path: "/content/drive/MyDrive/PROJECT/NLP/logs/phobert_training.csv"
  early_stopping:
    patience: 2
  device: auto
cv:
  n_splits: 5
  shuffle: true
  random_state: 42
```
  
### `%%writefile ".../config_phobert.yaml"`
- **Ý nghĩa:** Ghi toàn bộ nội dung phía dưới vào một file tại đường dẫn chỉ định.
- **Tác dụng:** Lưu file config YAML để sử dụng trong script huấn luyện hoặc dự đoán.
---
### `mode: train`
- **Giá trị mặc định khi chạy:** `"train"` (có thể là `"test"` hoặc `"val"` tùy context)
- **Tác dụng:** Giúp script biết đang ở chế độ nào để load file dữ liệu phù hợp.
---
### `seed: 42`
- **Ý nghĩa:** Đặt seed để tái lập kết quả (reproducibility)
- **Tác dụng:** Đảm bảo kết quả giống nhau nếu chạy lại code.
---
### `paths:`
- **Đường dẫn tới các file dữ liệu** (đã được chuẩn hóa theo Drive)
```YAML
yaml
CopyEdit
  train: "/.../train.csv"
  test: "/.../test.csv"
  val: "/.../val.csv"
```
- **Mục đích:** Load dữ liệu tương ứng cho từng phase huấn luyện/đánh giá.
---
### `preprocessing.phobert.max_sequence_length: 256`
- **Ý nghĩa:** Độ dài tối đa mỗi câu khi tokenized đưa vào model
- **Nếu quá dài sẽ bị cắt, quá ngắn sẽ được padding**
- Với PhoBERT base, bạn nên giữ ở mức 256 (không nên quá 512)
---
### `models.phobert`
```YAML
yaml
CopyEdit
model_name: "vinai/phobert-base"
hidden_size: 768
dropout_rate: 0.3
num_labels: 2
```
|Trường|Giải thích|
|---|---|
|`model_name`|Tên model Hugging Face để tải PhoBERT (`vinai/phobert-base`)|
|`hidden_size`|Kích thước đầu ra từ PhoBERT (PhoBERT base = 768)|
|`dropout_rate`|Mức dropout trước khi đưa vào classifier layer|
|`num_labels`|Số nhãn phân loại (2 → nhị phân)|
---
### `training`
```YAML
yaml
CopyEdit
batch_size: 32
epochs: 5
checkpoint_path: ".../phobert_best.pt"
log_path: ".../phobert_training.csv"
early_stopping:
  patience: 2
device: auto
```
|Trường|Giải thích|
|---|---|
|`batch_size`|Số sample mỗi batch|
|`epochs`|Số lần lặp qua toàn bộ tập train|
|`checkpoint_path`|Đường dẫn lưu mô hình tốt nhất (.pt - PyTorch)|
|`log_path`|File CSV lưu log kết quả huấn luyện từng epoch|
|`early_stopping`|Dừng sớm nếu không cải thiện sau `patience` epochs|
|`device`|`"auto"` sẽ chọn `"cuda"` nếu có GPU, `"cpu"` nếu không|
---
### `cv` – Cross Validation
```YAML
yaml
CopyEdit
n_splits: 5
shuffle: true
random_state: 42
```
- Dùng để thiết lập **k-fold cross-validation**
- `n_splits`: số fold (mặc định là 5)
- `shuffle`: xáo trộn dữ liệu trước khi chia fold
- `random_state`: để kết quả chia fold có thể lặp lại
```Python
def load_config(config_path: str) -> Namespace:
    with open(config_path, 'r') as f:
        cfg_dict = yaml.safe_load(f)
    
    def dict_to_namespace(d):
        if isinstance(d, dict):
            return Namespace(**{k: dict_to_namespace(v) for k, v in d.items()})
        elif isinstance(d, list):
            return [dict_to_namespace(x) for x in d]
        else:
            return d
    return dict_to_namespace(cfg_dict)
# Load ví dụ
config = load_config("/content/drive/MyDrive/1.PROJECTS/[TeamPe][BIT][BGRA2025]/NLP/Code/config_phobert.yaml")
```
  
---
# 2. Data
```Python
class PhoBERTDataset(Dataset):
    def __init__(self, texts, labels, tokenizer, max_len):
        self.texts = texts
        self.labels = labels
        self.tokenizer = tokenizer
        self.max_len = max_len
    def __len__(self):
        return len(self.texts)
    def __getitem__(self, idx):
        encoding = self.tokenizer(
            self.texts[idx],
            padding='max_length',
            truncation=True,
            max_length=self.max_len,
            return_tensors='pt'
        )
        return {
            'input_ids': encoding['input_ids'].squeeze(),
            'attention_mask': encoding['attention_mask'].squeeze(),
            'label': torch.tensor(self.labels[idx], dtype=torch.long)
        }
```
Biến danh sách văn bản và nhãn (text, label) thành format chuẩn mà mô hình PhoBERT (hoặc bất kỳ transformer nào) có thể sử dụng trong `DataLoader`.
**init:**
- **texts**: danh sách văn bản gốc (list[str])
- **labels**: danh sách nhãn tương ứng (list[int])
- **tokenizer**: `AutoTokenizer` của Hugging Face (đã load PhoBERT)
- **max_len**: độ dài tối đa cho mỗi input sequence
  
**getitem:**
- Hàm này sẽ được gọi mỗi khi `DataLoader` cần lấy 1 item.
- `idx`: chỉ số dòng cần truy cập.
  
  
# 3. **PhoBERT**
```Python
class PhoBERTClassifier(nn.Module):
    def __init__(self, model_name="vinai/phobert-base", num_labels=2):
        super().__init__()
        self.phobert = AutoModel.from_pretrained(model_name)
        self.dropout = nn.Dropout(0.3)
        self.classifier = nn.Linear(self.phobert.config.hidden_size, num_labels)
    def forward(self, input_ids, attention_mask=None):
        outputs = self.phobert(input_ids=input_ids, attention_mask=attention_mask)
        pooled = outputs.last_hidden_state[:, 0]  # CLS token
        return self.classifier(self.dropout(pooled))
```
Dùng PhoBERT để trích xuất đặc trưng từ văn bản, sau đó phân loại bằng một lớp `Linear`.
  
  
- **Dropout** giúp giảm overfitting
- Xác suất dropout: 30%
  
- `input_ids`: tensor `[batch_size, seq_len]`, đầu vào token đã mã hóa
- `attention_mask`: mặt nạ cho model biết đâu là token thật/pad
  
- Qua `Dropout` → `Linear` → **trả ra logits**
- Bạn sẽ dùng `CrossEntropyLoss()` cho nhiều lớp hoặc `BCEWithLogitsLoss()` nếu `num_labels = 1`
---
# 4. Trainer
```Python
def train_model(model, train_loader, val_loader, config):
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    model.to(device)
    optimizer = torch.optim.AdamW(model.parameters(), lr=2e-5)
    loss_fn = nn.CrossEntropyLoss()
    best_val_acc = 0
    patience_counter = 0
    logs = []
    for epoch in range(config.training.epochs):
        model.train()
        total_loss = 0
        all_preds, all_labels = [], []
        for batch in tqdm(train_loader, desc=f"[Epoch {epoch+1}] Training"):
            input_ids = batch['input_ids'].to(device)
            attention_mask = batch['attention_mask'].to(device)
            labels = batch['label'].to(device)
            outputs = model(input_ids, attention_mask)
            loss = loss_fn(outputs, labels)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            total_loss += loss.item()
            preds = torch.argmax(outputs, dim=1)
            all_preds.extend(preds.cpu().numpy())
            all_labels.extend(labels.cpu().numpy())
        train_acc = accuracy_score(all_labels, all_preds)
        # Validation
        val_acc = evaluate_model(model, val_loader)
        logs.append((epoch+1, train_acc, val_acc))
        print(f"[Epoch {epoch+1}] Train Acc: {train_acc:.4f} - Val Acc: {val_acc:.4f}")
        if val_acc > best_val_acc:
            best_val_acc = val_acc
            patience_counter = 0
            torch.save(model.state_dict(), config.training.checkpoint_path)
        else:
            patience_counter += 1
            if patience_counter >= config.training.early_stopping.patience:
                print("=> Early stopping.")
                break
```
1. **Khởi tạo**
- Chọn GPU nếu có, không thì dùng CPU.
- Chuyển mô hình lên thiết bị.
---
2. **Tối ưu hóa và loss**
- `AdamW`: tối ưu hóa thường dùng cho Transformers.
- `CrossEntropyLoss`: phù hợp với `num_labels >= 2`.
---
3. **Biến theo dõi**
```Python
best_val_acc = 0
patience_counter = 0
logs = []
```
- Theo dõi model tốt nhất (highest `val_acc`)
- Đếm số lần không cải thiện để dùng `early_stopping`
---
4. **Vòng lặp Epoch**
```Python
for epoch in range(config.training.epochs):
    model.train()
```
- Bắt đầu training cho mỗi epoch
- Kích hoạt chế độ training (tính gradient)
---
### 5. **Vòng lặp theo batch**
- Lấy từng batch từ DataLoader
- Đưa dữ liệu lên thiết bị
- Forward → tính loss
```Python
optimizer.zero_grad()
loss.backward()
optimizer.step()
```
- Backpropagation → cập nhật trọng số
```Python
preds = torch.argmax(outputs, dim=1)
```
- Lấy nhãn dự đoán từ `logits` → `preds`
---
### 6. **Đánh giá train & val**
```Python
train_acc = accuracy_score(all_labels, all_preds)
val_acc = evaluate_model(model, val_loader)
```
- Tính accuracy trên tập huấn luyện và validation
---
### 7. **Lưu model tốt nhất**
```Python
python
CopyEdit
if val_acc > best_val_acc:
    best_val_acc = val_acc
    patience_counter = 0
    torch.save(model.state_dict(), config.training.checkpoint_path)
```
- Nếu kết quả validation tốt hơn trước đó → lưu mô hình tốt nhất
---
### 8. **Early stopping**
```Python
python
CopyEdit
else:
    patience_counter += 1
    if patience_counter >= config.training.early_stopping.patience:
        print("=> Early stopping.")
        break
```
- Nếu kết quả không cải thiện liên tục `patience` lần → dừng sớm