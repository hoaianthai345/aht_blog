---
Author: HoàiAn Thái
Date: Invalid date
Status: Done
Tag:
  - RSNA24
  - Research
---
# **Giới thiệu**
  
**Table of Content**
- [[#Giới thiệu]]
- [[#Coordinate pretrained models]]
    - [[#1. Set up]]
    - [[#2. Class Dataset]]
    - [[#3. Class pre-train model ConvNextSCSDetect]]
    - [[#4. Loss functions]]
    - [[#5. Class module DetectModule]]
    - [[#6. Chạy model]]
    - [[#7. Validation]]
---
# C**oordinate pretrained models**
## 1. Set up
```Python
!pip install kaggle
from google.colab import files
uploaded = files.upload()
for fn in uploaded.keys():
  print('User uploaded file "{name}" with length {length} bytes'.format(
      name=fn, length=len(uploaded[fn])))
# Then move kaggle.json into the folder where the API expects to find it.
!mkdir -p ~/.kaggle/ && mv kaggle.json ~/.kaggle/ && chmod 600 ~/.kaggle/kaggle.json
from google.colab import drive
drive.mount('./gdrive')
```
  
```Python
!kaggle datasets download -d brendanartley/lumbar-coordinate-pretraining-dataset
!unzip -qq "/content/lumbar-coordinate-pretraining-dataset.zip"
```
  
```Python
!pip install -q pytorch-lightning & pip install -q -U albumentations & pip install -q iterative-stratification
!pip install -q timm & pip install -q einops & pip install -q pytorch-lightning wandb & pip install torch-ema
!git clone https://github.com/mlpc-ucsd/CoaT
!pip install -q pydicom
```
  
```Python
import wandb
from pytorch_lightning.loggers import WandbLogger
import os
import yaml
import sys
import cv2
import random
import numpy as np
import matplotlib.pyplot as plt
import pandas as pd
import torch
from glob import glob
from torch.utils.data import Dataset
from torch.utils.data import DataLoader
from torch.optim import AdamW, Adam
from torch.optim.lr_scheduler import CosineAnnealingLR, StepLR, ReduceLROnPlateau
import torch.nn as nn
import pytorch_lightning as pl
from pytorch_lightning.callbacks import ModelCheckpoint, EarlyStopping, TQDMProgressBar
import torchvision.transforms as T
import albumentations as A
import pandas.api.types
import sklearn.metrics
from sklearn.model_selection import train_test_split
from sklearn.model_selection import StratifiedKFold, KFold, GroupKFold, StratifiedGroupKFold
from iterstrat.ml_stratifiers import MultilabelStratifiedKFold
\#from torchvision.models.maxvit import MaxVit
from timm.models.maxxvit import MaxxVit
import timm
import scipy
import albumentations as A
from torchvision.transforms import v2
from torchvision import models
from einops import repeat
from einops.layers.torch import Rearrange
from tqdm.auto import tqdm
sys.path.append('/content/CoaT')
from CoaT.src.models.coat import coat_lite_medium
from joblib import Parallel, delayed
from torch.utils.data import default_collate
from torch_ema import ExponentialMovingAverage
import pydicom as dcm
import transformers
from PIL import Image, ImageFilter
```
  
```Python
SEED = 126 # friend's birthday
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
def seed_everything(seed):
    random.seed(seed)
    os.environ['PYTHONHASHSEED'] = str(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed(seed)
    torch.backends.cudnn.deterministic = True # Fix the network according to random seed
    print('Finish seeding with seed {}'.format(seed))
seed_everything(SEED)
print('Training on device {}'.format(device))
```
  
```Python
%%writefile config.yaml
data_path: "/content/"
output_dir: "/content/gdrive/MyDrive/RSNA_SPINE/models/"
seed: 1101
train_bs: 4
valid_bs: 4
workers: 1
progress_bar_refresh_rate: 1
pseudo_train: 0
save_topk: 1
fold: 5
task:
    \#kind: 'detect'
    kind: 'classify'
    \#kind: 'depth'
    \#condition: 'nfn'
    condition: 'scs'
    \#condition: 'ss'
    \#condition: 'all'
    \#direction: 'sagt1'
    direction: 'ax'
    \#direction: 'sagt1'
    position:
        - 'L1/L2'
        - 'L2/L3'
        - 'L3/L4'
        - 'L4/L5'
        - 'L5/S1'
in_chans: 3
image_size: 384
early_stop:
    monitor: "val_loss"
    mode: "min"
    patience: 999
    verbose: 1
trainer:
    max_epochs: 25
    min_epochs: 25
    enable_progress_bar: True
    precision: "16-mixed"
    devices: 1
model:
    name: "eff"
    loss_smooth: 0.0
    optimizer_params:
        lr: 0.0001
        \#lr: 0.001
        weight_decay: 0.0001
    scheduler:
        name: "CosineAnnealingLR"
        \#name: "ReduceLROnPlateau"
        \#name: "cosine_with_warmup"
        \#name: "ChrisLR"
        params:
            CosineAnnealingLR:
                T_max: 25
                eta_min: 5.0e-5
                last_epoch: -1
            ReduceLROnPlateau:
                mode: "min"
                factor: 0.5
                patience: 2
                min_lr: 0.00001
                verbose: True
            ChrisLR:
                freeze_iter: 2
            cosine_with_warmup:
                num_training_steps: 25
                num_warmup_steps: 2
                num_cycles: 0.5
                last_epoch: -1
```
**Đường dẫn và cấu hình cơ bản**
- `data_path`: thư mục chứa dữ liệu huấn luyện.
- `output_dir`: nơi lưu checkpoint model.
- `seed`: để tái lập (reproducibility), kiểm soát ngẫu nhiên.
  
**Huấn luyện**
- `train_bs`, `valid_bs`: batch size cho huấn luyện và validation.
- `workers`: số luồng xử lý data loader.
- `pseudo_train`: bật/tắt pseudo labeling (0 là không dùng).
- `save_topk`: số lượng mô hình tốt nhất được lưu.
- `fold`: chỉ số fold hiện tại trong k-fold cross-validation.
  
**Nhiệm vụ học:**
- `kind`: loại task, ví dụ:
    - `'classify'`: phân loại mức độ bệnh.
    - `'detect'`: phát hiện vị trí tổn thương.
    - `'depth'`: phân tích độ sâu tổn thương.
- `condition`: loại bệnh, ví dụ:
    - `'scs'`: _spinal canal stenosis_.
    - `'nfn'`: _neural foraminal narrowing_.
    - `'ss'`: _subarticular stenosis_.
- `direction`: hướng ảnh MRI (axial, sagittal,...).
- `position`: các vị trí đốt sống cần dự đoán.
  
**Xử lý ảnh**
- `in_chans`: số kênh ảnh đầu vào (3 kênh như RGB hoặc 3 slice liên tiếp).
- `image_size`: kích thước ảnh đầu vào sau resize.
  
**Chiến lược dừng sớm**
- Dừng sớm nếu `val_loss` không cải thiện sau số `patience` epoch.
- Với `patience = 999` thì tính ra là không dừng sớm gì cả (dùng khi muốn train đủ số epoch).
  
**Huấn luyện (Trainer)**
- `max_epochs`, `min_epochs`: số epoch huấn luyện.
- `precision`: "16-mixed" dùng FP16 mixed precision để tăng tốc và giảm bộ nhớ.
- `devices`: số GPU dùng.
  
**Mô hình và bộ tối ưu hóa**
- `name`: loại backbone, ví dụ `eff` là EfficientNet.
- `loss_smooth`: regularization cho label smoothing (0.0 là không dùng).
- `optimizer_params`: tham số cho optimizer.
  
**Lịch trình điều chỉnh learning rate**
- `CosineAnnealingLR`: lịch giảm lr hình sin.
- `ReduceLROnPlateau`: giảm lr khi `val_loss` không cải thiện.
- `cosine_with_warmup`: kết hợp khởi động chậm và cosine decay.
- `ChrisLR`, `freeze_iter`: dùng nếu muốn freeze các layer đầu tiên một số epoch.
  
  
  
```Python
with open("config.yaml", "r") as file_obj:
    config = yaml.safe_load(file_obj)
```
  
```Python
coor = pd.read_csv('/content/coords_pretrain.csv')
coor.head(3)
```
|**filename**|**source**|**x**|**y**|**level**|**relative_x**|**relative_y**|
|---|---|---|---|---|---|---|
|**0**|1_t2.jpg|spider|139|175|L5/S1|0.542969|
|**1**|1_t2.jpg|spider|133|157|L4/L5|0.519531|
|**2**|1_t2.jpg|spider|132|131|L3/L4|0.515625|
```Python
coor.groupby(['filename']).count()['x'].describe()
```
||x|
|---|---|
|**count**|1121|
|**mean**|5|
|**std**|0.0|
|**min**|5.0|
|**25%**|5.0|
|**50%**|5.0|
|**75%**|5.0|
|**max**|5.0|
**dtype:** float64
```Python
source='spider'
name='1_t2.jpg'
path = f'/content/data/processed_{source}_jpgs/{name}'
img = np.array(Image.open(path))
plt.imshow(img)
x = 0.515625
y = 0.511719
y = img.shape[0]*y
x = img.shape[1]*x
mask = np.zeros(img.shape)
mask[int(y-3):int(y+3), int(x-3):int(x+3)] = 1
plt.imshow(mask, alpha=0.3)
img.shape
```
(256, 256)
![[image 55.png|image 55.png]]
  
→ Tạo đường dẫn ảnh: `/content/data/processed_spider_jpgs/1_t2.jpg`
`img = np.array(Image.open(path)) plt.imshow(img)`
→ Đọc ảnh và chuyển thành mảng NumPy. Hiển thị ảnh bằng matplotlib.
→ `y = img.shape[0]*y`: Tọa độ `x, y` ban đầu nằm trong khoảng `[0,1]` (chuẩn hóa), được chuyển thành tọa độ pixel thực.
```Python
mask = np.zeros(img.shape)
mask[int(y-3):int(y+3), int(x-3):int(x+3)] = 1
plt.imshow(mask, alpha=0.3)
```
- Tạo mask (ma trận toàn 0 cùng shape với ảnh).
- Đánh dấu vùng nhỏ 6×6 pixel xung quanh điểm `(x, y)` bằng 1.
- Vẽ mask lên ảnh với `alpha=0.3` (trong suốt 30%).
    
    ![[image 1 21.png|image 1 21.png]]
    
---
## 2. Class Dataset
```Python
class CoorDetectDataset(Dataset):
    def __init__(self, coor):
        self.coor = coor
        self.path = []
        for _, row in self.coor.iterrows():
            self.path.append(f'/content/data/processed_{row.source}_jpgs/{row.filename}')
        self.resize = v2.Resize((384, 384))
    def __getitem__(self, index):
        path = self.path[index]
        img = np.array(Image.open(path))
        img = img.astype(np.float32)
        img = (img - img.min()) / (img.max() - img.min())
        img = self.resize(torch.from_numpy(img)[None, ...])
        img = img.repeat(3, 1, 1)
        filename = path.split('/')[-1]
        sub_coor = self.coor.loc[self.coor.filename==filename]
        coor_dict = {}
        for _, row in sub_coor.iterrows():
            coor_dict[row.level] = torch.tensor([row.relative_x, row.relative_y])
        return img, coor_dict, path
    def __len__(self):
        return len(self.path)
```
Sử dụng:
```Python
dataset = CoorDetectDataset(coor)
img, coor_dict, path = dataset[0]
# img.shape: torch.Size([3, 384, 384])
# coor_dict: {'L2/L3': tensor([0.51, 0.49]), 'L3/L4': tensor([0.50, 0.52]), ...}
# path: '/content/data/processed_spider_jpgs/1_t2.jpg'
```
  
**Mục tiêu:**
- Load từng ảnh MRI đã resize.
- Chuẩn hóa ảnh về dạng tensor `[3, 384, 384]`.
- Trả về:
    - tensor ảnh,
    - dictionary các tọa độ tương đối (`relative_x`, `relative_y`) theo từng mức đốt sống (level),
    - đường dẫn ảnh gốc.
**1.** `**__init__**` **– khởi tạo dataset**
- `coor`: là một `DataFrame` chứa các cột như:
    - `source`, `filename`, `level`, `relative_x`, `relative_y`.
- Tạo danh sách đường dẫn ảnh theo từng dòng.
- Thiết lập `resize` ảnh về kích thước chuẩn (384x384).
2. `__getitem__` – lấy một mẫu dữ liệu
- Đọc ảnh bằng `PIL`, chuyển thành NumPy array.
- Chuẩn hóa về [0, 1].
- Convert sang tensor (thêm chiều batch `1`) → rồi lặp lại 3 lần để thành ảnh 3 kênh giả lập.
- Tạo dict lưu tọa độ các đốt sống:
    - Tìm tất cả dòng trong `self.coor` ứng với `filename` hiện tại.
    - Với mỗi đốt sống (`level`), lưu lại cặp tọa độ tương đối `[x, y]` dưới dạng tensor.
- Trả về kết quả:
    - `img`: Tensor ảnh `[3, 384, 384]`.
    - `coor_dict`: `{level: tensor([x, y]), ...}`.
    - `path`: chuỗi đường dẫn ảnh, để debug hoặc visualize.
**3.** `**__len__**` **– số lượng mẫu**
---
## 3. Class pre-train model `ConvNextSCSDetect`
```Python
class ConvNextSCSDetect(nn.Module):
    def __init__(self):
        super().__init__()
        \#self.size = 384
        self.encoder = timm.create_model('tf_efficientnetv2_l.in21k_ft_in1k', in_chans=3, pretrained=True, num_classes=0, drop_rate=0.)
        self.in_features = self.encoder.num_features
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool2d((1,1)),
                                    nn.Flatten(1),
                                    )
        self.l1 = nn.Linear(self.in_features, 2)
        self.l2 = nn.Linear(self.in_features, 2)
        self.l3 = nn.Linear(self.in_features, 2)
        self.l4 = nn.Linear(self.in_features, 2)
        self.l5 = nn.Linear(self.in_features, 2)
        \#self.out = nn.Linear(self.in_features, 10)
    def forward(self, x, label=None):
        x = self.encoder.forward_features(x)
        x = self.flatten(x)
        l1 = self.l1(x)
        l2 = self.l2(x)
        l3 = self.l3(x)
        l4 = self.l4(x)
        l5 = self.l5(x)
        return {'L1/L2': l1.sigmoid(), 
				        'L2/L3': l2.sigmoid(), 
				        'L3/L4': l3.sigmoid(), 
				        'L4/L5': l4.sigmoid(), 
				        'L5/S1': l5.sigmoid()}
```
Output mẫu:
{  
'L1/L2': tensor([[0.51, 0.49]]),  
'L2/L3': tensor([[0.50, 0.52]]),  
...  
}
  
Dùng kiến trúc backbone `EfficientNetV2` từ `timm`, được thiết kế để **dự đoán tọa độ tương đối (x, y)** cho từng đốt sống ở các vị trí cụ thể trong cột sống.
  
**Mục tiêu của mô hình:**

> Dự đoán tọa độ tương đối (x, y) cho 5 đốt sống:
> 
> `'L1/L2'`, `'L2/L3'`, `'L3/L4'`, `'L4/L5'`, `'L5/S1'`
> 
> → mỗi nhãn đầu ra là **2 giá trị**: `[x, y]` ∈ [0, 1] (chuẩn hóa theo ảnh).
  
**1.** `**__init__**`**: Khởi tạo**
- `self.encoder`: Tải mô hình `EfficientNetV2-L` đã pretrain (dùng ImageNet21k) từ thư viện `timm`.
    - `in_chans=3`: input là ảnh RGB.
    - `pretrained=True`: Tải trọng số đã huấn luyện trước (pretrained weights), không huấn luyện từ đầu
    - `num_classes=0`: loại bỏ lớp phân loại cuối.
        - Mặc định `timm` sẽ tạo mô hình có lớp cuối `nn.Linear(..., num_classes)` nếu bạn chỉ định `num_classes>0`.
        - Nhưng ở đây bạn chỉ muốn **dùng phần encoder (trích đặc trưng)** → `num_classes=0` sẽ **bỏ luôn phần classifier cuối**. → Đầu ra của mô hình là feature map (thường có shape `[B, C, H', W']`).
    - `drop_rate=0`: Xác định **tỷ lệ dropout** trong các lớp linear cuối mô hình. Vì **không dùng head phân loại**, nên `drop_rate=0.` là an toàn (không ảnh hưởng).
- `self.in_features:` Output là tensor `feature map`.
- `self.flatten`
    - Lấy trung bình không gian (Global Average Pooling) → chuyển thành vector đặc trưng 1D.
- `self.l1 = nn.Linear(self.in_features, 2) …`
    - **Cú pháp nn.Linear:** nn.Linear(in_features, out_features, bias=True)
        - `in_features`: Số chiều đầu vào
        - `out_features`: Số chiều đầu ra
        - `bias`: thêm bias không, default: True
    - Nhận đầu vào là 1 vector đặc trưng có `self.in_features` chiều (ví dụ: 1280, 2048,...). → xuất ra tọa độ tương đối `[x, y]]`.
**2.** `**forward**`**: Dự đoán đầu ra**
- `x = self.encoder.forward_features(x) x = self.flatten(x)`
    - Trích xuất đặc trưng từ ảnh → vector kích thước `[batch_size, in_features]`.
- l1 = self.l1(x)  
    l2 = self.l2(x)  
    ...
    - Dự đoán tọa độ từng vị trí đốt sống. Mỗi output là `[x, y]`.
- Return: Dùng `sigmoid()` để đảm bảo giá trị trong khoảng `[0, 1]` → đúng với yêu cầu **tọa độ tương đối**.
  
**Giải thích thêm nn.Linear**
Gọi là **fully connected layer** (FC layer) vì: Mỗi **đầu ra** (output neuron) **liên kết với tất cả** các **đầu vào** (input neuron). Mỗi kết nối có **trọng số riêng biệt**.
Nếu bạn có:
- 3 đầu vào: $x_1, x_2, x_3$ (feature map)
- 2 đầu ra: $y_1, y_2$ (tọa độ)
Thì:
- $y_1 = w_{11}x_1 + w_{12}x_2 + w_{13}x_3 + b_1$
- $y_2 = w_{21}x_1 + w_{22}x_2 + w_{23}x_3 + b_2$
---
## 4. Loss functions
```Python
class SCSDetectLoss(nn.Module):
    def __init__(self):
        super(SCSDetectLoss, self).__init__()
    def forward(self, outputs, targets):
        loss = 0
        for level in ['L1/L2', 'L2/L3', 'L3/L4', 'L4/L5', 'L5/S1']:
            _loss = nn.functional.l1_loss(outputs[level], targets[level])
            loss += _loss
        return loss/5
```
```Python
class SCSDetectLoss(nn.Module):
    def __init__(self):
        super(SCSDetectLoss, self).__init__()
    def forward(self, outputs, targets):
        loss = 0
        for level in ['L1/L2', 'L2/L3', 'L3/L4', 'L4/L5', 'L5/S1']:
            _loss = nn.functional.l1_loss(outputs[level], targets[level])
            loss += _loss
        return loss/5
class SCSDetectRelativePositionLoss(nn.Module):
    def __init__(self):
        super(SCSDetectRelativePositionLoss, self).__init__()
    def forward(self, outputs, targets):
        loss = 0
        for level in ['L1/L2', 'L2/L3', 'L3/L4', 'L4/L5', 'L5/S1']:
            _loss = nn.functional.l1_loss(outputs[level], targets[level])
            loss += _loss
        loss = loss/5
        relative_positional_loss = 0
        for l in ['L1/L2', 'L2/L3', 'L3/L4', 'L4/L5', 'L5/S1']:
            relative_positional_pred_list = []
            relative_positional_tar_list = []
            for ll in ['L1/L2', 'L2/L3', 'L3/L4', 'L4/L5', 'L5/S1']:
                if l == ll:
                    continue
                pred1, pred2 = outputs[l], outputs[ll]
                tar1, tar2 = targets[l], targets[ll]
                dist_pred = torch.sqrt((pred1[:, 0]-pred2[:, 0])**2 + (pred1[:, 1]-pred2[:, 1])**2)
                dist_tar = torch.sqrt((tar1[:, 0]-tar2[:, 0])**2 + (tar1[:, 1]-tar2[:, 1])**2)
                relative_positional_pred_list.append(dist_pred)
                relative_positional_tar_list.append(dist_tar)
            relative_positional_pred = torch.cat(relative_positional_pred_list)
            relative_positional_tar = torch.cat(relative_positional_tar_list)
            relative_positional_loss += nn.functional.l1_loss(relative_positional_pred, relative_positional_tar)
        return loss + relative_positional_loss/5
```
`**SCSDetectLoss**`**:**
Tính trung bình **L1 loss** (còn gọi là **Mean Absolute Error**) giữa các tọa độ dự đoán và ground truth tại từng đốt sống.
1. `__init__`
    - Kế thừa `nn.Module`.
    - Không có tham số học, nên `__init__` để trống.
2. `def forward(self, outputs, targets)`
    - Các tham số:
        - `outputs`: dict chứa các output của mô hình, mỗi key là 1 đốt sống, giá trị là tensor `[B, 2]` chứa tọa độ dự đoán.
        - `targets`: dict chứa ground truth tương ứng, cũng là `[B, 2]`.
    - Duyệt qua từng **đốt sống** cần dự đoán.
        - Với mỗi level: Tính **L1 Loss** giữa dự đoán và ground truth tọa độ.
            
            $\text{MAE} = |x_{\text{pred}} - x_{\text{true}}| + |y_{\text{pred}} - y_{\text{true}}|$
            
        - Cộng dồn vào tổng loss.
    - `return loss / 5`: Trả về là loss trung bình trên 5 đốt sống
  
`**SCSDetectRelativePositionLoss**`**:**
Hàm loss kết hợp giữa: **Loss tuyệt đối** tại mỗi đốt sống và **Loss tương đối** về khoảng cách giữa các đốt sống
**1. Tính absolute loss (giống loss cơ bản)**
```Python
loss = 0
for level in ['L1/L2', 'L2/L3', 'L3/L4', 'L4/L5', 'L5/S1']:
    _loss = nn.functional.l1_loss(outputs[level], targets[level])
    loss += _loss
loss = loss/5
```
Phần này giống với ở trên
  
**2. Tính relative positional loss**
- Với mỗi cặp đốt sống khác nhau (`l ≠ ll`), tính khoảng cách Euclidean giữa tọa độ dự đoán và thật.
- So sánh khoảng cách này để xem mô hình có giữ được **khoảng cách tương đối đúng** giữa các đốt sống không.
**Ví dụ tính tay:**
```Python
# Dự đoán từ mô hình
outputs = {
    'L1/L2': torch.tensor([[0.50, 0.20]]),
    'L2/L3': torch.tensor([[0.52, 0.30]]),
    'L3/L4': torch.tensor([[0.53, 0.40]]),
    'L4/L5': torch.tensor([[0.55, 0.50]]),
    'L5/S1': torch.tensor([[0.58, 0.60]]),
}
# Ground truth
targets = {
    'L1/L2': torch.tensor([[0.50, 0.20]]),
    'L2/L3': torch.tensor([[0.51, 0.30]]),
    'L3/L4': torch.tensor([[0.52, 0.40]]),
    'L4/L5': torch.tensor([[0.54, 0.50]]),
    'L5/S1': torch.tensor([[0.57, 0.60]]),
}
```
Tính khoản cách Euclidean giữa các cặp điểm: $d = \sqrt{(x_1 - x_2)^2 + (y_1 - y_2)^2}$
|Cặp|`dist_pred`|`dist_tar`|
|---|---|---|
|L1–L2/L3|√((0.50−0.52)² + (0.20−0.30)²) ≈ 0.102|√((0.50−0.51)² + (0.20−0.30)²) ≈ 0.100|
|L1–L3/L4|√((0.50−0.53)² + (0.20−0.40)²) ≈ 0.224|√((0.50−0.52)² + (0.20−0.40)²) ≈ 0.200|
|L1–L4/L5|√((0.50−0.55)² + (0.20−0.50)²) ≈ 0.305|√((0.50−0.54)² + (0.20−0.50)²) ≈ 0.283|
|L1–L5/S1|√((0.50−0.58)² + (0.20−0.60)²) ≈ 0.412|√((0.50−0.57)² + (0.20−0.60)²) ≈ 0.389|
|L2–L3/L4|√((0.52−0.53)² + (0.30−0.40)²) ≈ 0.100|√((0.51−0.52)² + (0.30−0.40)²) ≈ 0.100|
|L2–L4/L5|√((0.52−0.55)² + (0.30−0.50)²) ≈ 0.224|√((0.51−0.54)² + (0.30−0.50)²) ≈ 0.212|
|L2–L5/S1|√((0.52−0.58)² + (0.30−0.60)²) ≈ 0.316|√((0.51−0.57)² + (0.30−0.60)²) ≈ 0.283|
|L3–L4/L5|√((0.53−0.55)² + (0.40−0.50)²) ≈ 0.224|√((0.52−0.54)² + (0.40−0.50)²) ≈ 0.224|
|L3–L5/S1|√((0.53−0.58)² + (0.40−0.60)²) ≈ 0.206|√((0.52−0.57)² + (0.40−0.60)²) ≈ 0.206|
|L4–L5/S1|√((0.55−0.58)² + (0.50−0.60)²) ≈ 0.111|√((0.54−0.57)² + (0.50−0.60)²) ≈ 0.100|
Sau đó lưu lại khoảng cách giữa cặp đốt sống đang xét cho cả `pred` và `target:`
relative_positional_pred_list.append(dist_pred)
relative_positional_tar_list.append(dist_tar)
Gộp tất cả khoảng cách lại thành vector kích thước `[B × num_pairs]`, ví dụ: `[1 × 10]`
relative_positional_pred = torch.cat(relative_positional_pred_list)
[0.102, 0.224, 0.305, 0.412, 0.100,
0.224, 0.316, 0.224, 0.206, 0.111]
relative_positional_tar = torch.cat(relative_positional_tar_list)
[0.100, 0.200, 0.283, 0.389, 0.100,
0.212, 0.283, 0.224, 0.206, 0.100]
Cuối cùng là so sánh sự khác biệt giữa các **khoảng cách hình học** của dự đoán và ground truth bằng L1 loss: `relative_positional_pred` - `relative_positional_tar`
diff = tensor( [0.002, 0.024, 0.022, 0.023, 0.000, 0.012, 0.033, 0.000, 0.000, 0.011] )
  
---
## 5. Class module `DetectModule`
```Python
class DetectModule(pl.LightningModule):
    def __init__(self, config):
        super().__init__()
        self.config = config
        self.model = ConvNextSCSDetect()
        \#self.model = SwinSCSDetect()
        \#self.loss_module = SCSDetectRelativePositionLoss()
        self.loss_module = SCSDetectLoss()
        self.val_step_outputs = []
        self.val_step_labels = []
        \#self.ema = ExponentialMovingAverage(self.model.parameters(), decay=0.995)
        \#self.ema.to(device)
        \#self.model = torch.optim.swa_utils.AveragedModel(self.model,
        #                                                 multi_avg_fn=torch.optim.swa_utils.get_ema_multi_avg_fn(0.999))
    def forward(self, batch):
        preds = self.model(batch)
        return preds
    def configure_optimizers(self):
        optimizer = AdamW(self.parameters(), **self.config['model']["optimizer_params"])
        if self.config['model']["scheduler"]["name"] == "CosineAnnealingLR":
            scheduler = CosineAnnealingLR(
                optimizer,
                **self.config['model']["scheduler"]["params"]["CosineAnnealingLR"],
            )
            lr_scheduler_dict = {"scheduler": scheduler, "interval": "step"}
            return {"optimizer": optimizer, "lr_scheduler": lr_scheduler_dict}
            
        elif self.config['model']["scheduler"]["name"] == "ReduceLROnPlateau":
            scheduler = ReduceLROnPlateau(
                optimizer,
                **self.config['model']["scheduler"]["params"]["ReduceLROnPlateau"],
            )
            lr_scheduler = {"scheduler": scheduler, "monitor": "val_loss"}
            return {"optimizer": optimizer, "lr_scheduler": lr_scheduler}
            
        elif self.config['model']['scheduler']['name'] == 'ChrisLR':
            scheduler = ChrisLR(
                optimizer,
                **self.config['model']["scheduler"]["params"]["ChrisLR"],
            )
            lr_scheduler = {"scheduler": scheduler, "monitor": "val_loss"}
            return {"optimizer": optimizer, "lr_scheduler": lr_scheduler}
            
        elif self.config['model']['scheduler']['name'] == 'cosine_with_warmup':
            print('cosine with warmup')
            print(self.config['model']['scheduler']['params']['cosine_with_warmup'])
            scheduler = transformers.get_cosine_schedule_with_warmup(
                optimizer,
                **self.config['model']['scheduler']['params']['cosine_with_warmup'],
            )
            lr_scheduler_dict = {"scheduler": scheduler, "interval": "step"}
            return {"optimizer": optimizer, "lr_scheduler": lr_scheduler_dict}
            
        else:
            return {"optimizer": optimizer}
    def training_step(self, batch, batch_idx):
        imgs, label, _ = batch
        preds = self.model(imgs)
        loss = self.loss_module(preds, label)
        self.log("train_loss", loss, on_step=True, on_epoch=True, prog_bar=True, batch_size=self.config['train_bs'])
        for param_group in self.trainer.optimizers[0].param_groups:
            lr = param_group["lr"]
        self.log("lr", lr, on_step=True, on_epoch=False, prog_bar=True)
        return loss
    def validation_step(self, batch, batch_idx):
        """Add TTA"""
        volume, label, _ = batch
        preds = self.model.forward(volume)
        loss = self.loss_module(preds, label)
        self.log("val_loss", loss, on_step=False, on_epoch=True, prog_bar=True)
        return loss
    def on_validation_epoch_end(self):
        if self.trainer.global_rank == 0:
            print(f"\nEpoch: {self.current_epoch}", flush=True)
        return 0
    def optimizer_step(self, *args, **kwargs):
        super().optimizer_step(*args, **kwargs)
        \#self.ema.update()
```
Kế thừa từ `pl.LightningModule` của PyTorch Lightning để huấn luyện mô hình phát hiện tọa độ đốt sống.
**Gói toàn bộ logic huấn luyện vào một mô-đun chuẩn hóa gồm:**
- Model: `ConvNextSCSDetect()`
- Loss: `SCSDetectLoss()` hoặc `SCSDetectRelativePositionLoss()`
- Optimizer & Scheduler: từ file `config.yaml`
- Huấn luyện (training) + Đánh giá (validation)
  
**1.** `**__init__(self, config)**`**:**
- `config`: cấu hình từ file YAML.
- `self.model`: mô hình backbone (dự đoán [x, y] cho từng đốt sống).
    - ConvNextSCSDetect()
    - SwinSCSDetect()
- `self.loss_module`: hàm mất mát,
    - SCSDetectLoss()
    - `SCSDetectRelativePositionLoss`
  
**2.** `**forward(self, batch)**`**:**
- Gọi model forward (dự đoán).
- Dùng trong cả training và inference.
  
3. `**configure_optimizers(self)**`**:**
Cấu hình `optimizer` + `lr_scheduler` theo config:
- Sử dụng `AdamW` optimizer với các tham số như `lr`, `weight_decay`.
- 4 loại scheduler hỗ trợ:
    - `CosineAnnealingLR`
    - `ReduceLROnPlateau`
    - `ChrisLR` (custom)
    - `cosine_with_warmup` (transformers)
- Trả về dictionary đúng chuẩn PyTorch Lightning.
  
4. `**training_step(self, batch, batch_idx)**`:
- Nhận ảnh và nhãn (label).
- Tính loss và log:
    - `train_loss`: theo từng step + epoch
    - `lr`: log learning rate hiện tại
  
**5.** `**validation_step(self, batch, batch_idx)**`
- Tính `val_loss` và log cho epoch.

> có thể thêm Test-Time Augmentation ở đây.
  
**6.** `**on_validation_epoch_end(self)**`
- In số epoch sau mỗi vòng `validation` (chỉ ở process chính khi dùng multi-GPU).
  
**7.** `**optimizer_step(...)**`
- Ghi đè bước cập nhật optimizer.
- Có hỗ trợ `EMA` (Exponential Moving Average), nhưng đang tắt.
---
## 6. Chạy model
```Python
train, valid = train_test_split(coor.filename.unique(), test_size=0.2, random_state=42)
train_coor = coor.loc[coor.filename.isin(train)]
valid_coor = coor.loc[coor.filename.isin(valid)]
```
```Python
train_coor.shape, valid_coor.shape
```
→ ((4480, 7), (1125, 7))
```Python
T_MAX = config["model"]["scheduler"]["params"]["CosineAnnealingLR"]["T_max"]
num_training_steps = config["model"]["scheduler"]["params"]["cosine_with_warmup"]["num_training_steps"]
num_warmup_steps = config["model"]["scheduler"]["params"]["cosine_with_warmup"]["num_warmup_steps"]
```
  
```Python
%%time
import warnings
warnings.filterwarnings("ignore")
#############\#COORDINATE DETECT#########################
print(T_MAX)
\#for i in [5]:
for i in [0]:
    dataset_train = CoorDetectDataset(train_coor)
    dataset_validation = CoorDetectDataset(valid_coor)
    data_loader_train = DataLoader(
        dataset_train,
        batch_size=4,
        shuffle=True,
        num_workers=8,
        pin_memory=True,
    )
    data_loader_validation = DataLoader(
        dataset_validation,
        batch_size=8,
        shuffle=False,
        num_workers=8,
        pin_memory=True
    )
    checkpoint_callback = ModelCheckpoint(
        save_weights_only=True,
        monitor="val_loss",
        dirpath="/content/gdrive/MyDrive/RSNA_SPINE/pretrained_models",
        mode='min',
        filename=f"scs_detect_pretrained_efficientnetv2l_1e-4",
        save_top_k=config["save_topk"],
        verbose=1,
    )
    progress_bar_callback = TQDMProgressBar(
        refresh_rate=config["progress_bar_refresh_rate"]
    )
    early_stop_callback = EarlyStopping(**config["early_stop"])
    _c, _k = config['task']['condition'], config['task']['kind']
    wandb_logger = WandbLogger(project=f'rsna_spine_coor_pretrained', # group runs in "MNIST" project
                            log_model=False) # log all new checkpoints during training
    trainer = pl.Trainer(
        logger=wandb_logger,
        callbacks=[checkpoint_callback, early_stop_callback, progress_bar_callback],
        **config["trainer"],
    )
    config["model"]["scheduler"]["params"]["CosineAnnealingLR"]["T_max"] = T_MAX*len(data_loader_train)/config["trainer"]["devices"]
    config["model"]["scheduler"]["params"]["cosine_with_warmup"]["num_training_steps"] = int(num_training_steps*len(data_loader_train)/config["trainer"]["devices"])
    config["model"]["scheduler"]["params"]["cosine_with_warmup"]["num_warmup_steps"] = int(num_warmup_steps*len(data_loader_train)/config["trainer"]["devices"])
    model = DetectModule(config=config)
    trainer.fit(model, data_loader_train, data_loader_validation)
    wandb.finish()
```
**Run history:**
epoch ▁▁▁▁▂▂▂▂▂▂▃▃▃▃▃▄▄▄▄▅▅▅▅▅▅▆▆▆▆▆▇▇▇▇▇▇▇███
lr ███████▇▇▇▇▇▆▆▆▆▅▅▅▅▄▄▄▄▃▃▃▃▂▂▂▂▂▁▁▁▁▁▁▁
train_loss_epoch █▄▃▃▃▂▂▂▂▂▂▂▁▁▁▁▁▁▁▁▁▁▁▁▁
train_loss_step█▆▅▃▃▃▂▅▃▂▃▂▃▃▂▂▂▂▂▂▂▂▂▂▂▂▁▂▁▂▁▁▁▁▁▂▁▁▁▁
trainer/global_step ▁▁▁▁▂▂▂▂▂▃▃▃▃▃▃▄▄▄▄▄▅▅▅▅▅▅▆▆▆▆▆▇▇▇▇▇▇███
val_loss █▅▆▄▃▃▂▂▂▂▂▁▂▁▂▁▁▁▁▁▁▁▁▁▁
  
**Run summary:**
epoch 24  
lr 5e-05  
train_loss_epoch 0.0015  
train_loss_step 0.00166  
trainer/global_step 27999  
val_loss 0.00516
`train, valid = train_test_split(coor.filename.unique(), test_size=0.2, random_state=42)`
- `coor`: là một `DataFrame` chứa thông tin label tọa độ cho các ảnh (có thể nhiều dòng ứng với nhiều đốt sống trong cùng 1 ảnh).
- `coor.filename.unique()`: lấy danh sách các tên file ảnh **không trùng lặp**.
- `train_test_split(...)`: chia danh sách ảnh thành 2 phần:
    - `train`: 80% file ảnh
    - `valid`: 20% file ảnh
- `random_state=42`: cố định seed để chia dữ liệu lặp lại được.
Sau đó lọc lại toàn bộ các dòng trong `coor` mà `filename` nằm trong danh sách `train`.
Tương tự, lọc lại các dòng có `filename` nằm trong danh sách `valid`.
  
`T_MAX` – cho `CosineAnnealingLR`
- `T_max`: Số bước (hoặc epochs) cần để giảm learning rate theo hình **cosine từ giá trị khởi đầu xuống** `**eta_min**`.
- Trong `CosineAnnealingLR`, công thức:
    
    $\text{lr}_t = \eta_{\text{min}} + \frac{1}{2}(\eta_{\text{max}} - \eta_{\text{min}})(1 + \cos(\pi \cdot t / T_{\text{max}}))$
    
- Thường đặt `T_max = num_epochs` để learning rate về gần 0 ở cuối training.
`num_training_steps` – cho `cosine_with_warmup`
- Tổng số bước huấn luyện (tức tổng batch update).
- Dùng cho scheduler của HuggingFace Transformers: `get_cosine_schedule_with_warmup`.
- Phần **sau warm-up**, learning rate sẽ giảm dần theo hình cosine.
`num_warmup_steps` – cho `cosine_with_warmup`
- Số bước đầu tiên (warm-up phase) learning rate sẽ **tăng tuyến tính** từ 0 → lr ban đầu.
- Sau đó mới bắt đầu giảm theo cosine.
  
**Giai đoạn huấn luyện:**
`%%time` & `warnings`
- Đo thời gian chạy toàn bộ cell (Jupyter).
- Bỏ qua cảnh báo (cho sạch log khi huấn luyện).
Vòng lặp `for i in [0]:`
- chỉ chạy 1 fold duy nhất (`i = 0`) để test, có thể thay bằng `for i in [5]`
Tạo dataset & DataLoader:
- Tạo 2 dataset từ dataframe chứa annotation tọa độ.
- Dataloader cho train và validation. `pin_memory=True` giúp tăng tốc nếu dùng GPU.
  
**Cài đặt callback:**
- **ModelCheckpoint:**
    - Lưu model tốt nhất (val_loss thấp nhất).
    - File lưu tại đường dẫn.
- Progress bar:
    - Hiển thị tiến trình với tốc độ cập nhật theo config.
- EarlyStopping:
    - Dừng sớm nếu `val_loss` không cải thiện sau `patience` epoch.
  
**Logging với Weights & Biases:**
- Dùng [Wandb](https://wandb.ai/) để log kết quả huấn luyện.
- `log_model=False`: không lưu model vào Wandb.
  
**Tạo Trainer**
- Trainer Lightning với logger + callbacks + config (`max_epochs`, `devices`, ...).
  
**Điều chỉnh lại scheduler theo batch**
- Vì `CosineAnnealingLR` và `cosine_with_warmup` cần biết **tổng số bước cập nhật**, chứ không chỉ epoch.
- `len(data_loader_train)` = số batch / epoch.
- Chia cho số GPU (`devices`) nếu dùng multi-GPU (Lightning tự xử lý).
  
**Huấn luyện mô hình**
- Tạo module `DetectModule` chứa mô hình + loss + optimizer.
- Huấn luyện với `.fit()`.
- Kết thúc Wandb run: `wandb.finish()`
---
## 7. Validation
```Python
dataset_validation = CoorDetectDataset(valid_coor)
data_loader = DataLoader(
        dataset_validation,
        batch_size=8,
        shuffle=True,
        num_workers=8,
        pin_memory=True,
    )
# prediction
model = DetectModule.load_from_checkpoint(checkpoint_path='/content/gdrive/MyDrive/RSNA_SPINE/pretrained_models/scs_detect_pretrained_convnext-base.ckpt', config=config)
model.eval()
model.zero_grad()
model.to(device)
coor_predict = {'L1/L2': [], 'L2/L3': [], 'L3/L4': [], 'L4/L5': [], 'L5/S1': []}
coor_label = {'L1/L2': [], 'L2/L3': [], 'L3/L4': [], 'L4/L5': [], 'L5/S1': []}
path_list = []
with torch.no_grad():
    for data in tqdm(data_loader, total=len(data_loader)):
        images, label, path = data
        path_list.append(path)
        images = images.to(device)
        preds = model.forward(images)
        \#print(preds)
        for k, v in preds.items():
            coor_predict[k].append(v.to('cpu').detach().numpy())
        for k, v in label.items():
            coor_label[k].append(v.detach().numpy())
```
**Post processing:**
```Python
for k, v in coor_predict.items():
    coor_predict[k] = np.concatenate(v)
for k, v in coor_label.items():
    coor_label[k] = np.concatenate(v)
path_list = np.concatenate(path_list)
```
```Python
coor_label['L5/S1'][0]
```
**Load mô hình đã huấn luyện** và thực hiện **dự đoán (inference)** trên tập `validation`, rồi **lưu lại kết quả tọa độ dự đoán và ground truth** cho từng đốt sống.
**1. Tạo dataset & dataloader cho validation**
- `valid_coor`: là DataFrame chứa ground truth của tập validation.
- `CoorDetectDataset`: trả về tuple `(image_tensor, coor_dict, path)` cho mỗi ảnh.
- `DataLoader`: load ảnh theo batch (ở đây batch_size = 8).
**2. Load mô hình đã huấn luyện từ checkpoint**
- `load_from_checkpoint(...)`: khôi phục lại mô hình đã được huấn luyện với tham số cấu hình.
- `eval()`: chuyển sang chế độ evaluation (vô hiệu hóa dropout, batchnorm...).
- `zero_grad()`: xóa gradient (cẩn thận, không cần thiết ở inference).
- `to(device)`: chuyển model sang `cuda` nếu đang dùng GPU.
3. Khởi tạo dictionary để lưu kết quả
- Mỗi đốt sống sẽ lưu lại danh sách các dự đoán (`predict`) và ground truth (`label`).
- `path_list`: lưu đường dẫn ảnh tương ứng để so sánh trực quan sau này.
4. Vòng lặp dự đoán
- `torch.no_grad()`: tắt gradient để tiết kiệm bộ nhớ.
- `tqdm`: hiển thị progress bar.
- `data` trả về: ảnh batch, label dạng dict, và list các đường dẫn ảnh.
5. Dự đoán & lưu kết quả
- Đưa ảnh vào GPU và truyền qua model để dự đoán.
- `preds` là dictionary `{level: tensor(batch_size, 2)}`.
6. Lưu dự đoán và ground truth vào dict
- Với mỗi `level` (đốt sống), tách giá trị tensor thành numpy array rồi append vào list.
- `.to('cpu')` để chuyển về CPU trước khi convert sang numpy.
- `detach()` ngắt khỏi computational graph.
→ array([0.47265625, 0.73046875], dtype=float32)