---
Author: HoàiAn Thái
Date: Invalid date
Status: In progress
Tag:
  - RSNA24
  - Research
---
# **Giới thiệu**
  
**Table of Content**
- [[#Giới thiệu]]
- [[#Import & configure]]
- [[#0. Initial stage (Create meta file)]]
    - [[#Create meta file using dicom's meta data]]
    - [[#Thực hiện]]
- [[#I. First stage: Depth inference]]
    - [[#1. Class Dataset:]]
        - [[#Khởi tạo]]
        - [[#For_scs]]
        - [[#For_nfn]]
        - [[#Các hàm khác]]
    - [[#2. Models: 3D ConvNeXt]]
        - [[#ConvNextStem]]
        - [[#LayerScaler]]
        - [[#BottleNeckBlock]]
        - [[#ConvNexStage]]
        - [[#ConvNextEncoder]]
        - [[#ClassificationHead]]
    - [[#3. Lightning module]]
    - [[#5. Inference]]
    - [[#6. Create label coordinate & align]]
- [[#II. Second Stage (xy inference)]]
    - [[#1. Coordinate prediction dataset]]
        - [[#Khởi tạo]]
        - [[#Func for_scs]]
    - [[#2. Coordinate prediction models]]
        - [[#3. Coordinate detection lightning module]]
        - [[#4. Coordinate inference]]
- [[#III. Third Stage (calc. location of axial t2)]]
    - [[#1. Calculate axial slice]]
        - [[#2. Subarticular stenosis coordinate prediction dataset]]
---
# Import & configure
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
import gc
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
import torch.nn as nn
import pytorch_lightning as pl
from pytorch_lightning.callbacks import ModelCheckpoint, EarlyStopping, TQDMProgressBar
import torchvision.transforms as T
import albumentations as A
import pandas.api.types
import sklearn.metrics
import timm
import scipy
import albumentations as A
from torchvision.transforms import v2
from torchvision import models
from tqdm.auto import tqdm
from joblib import Parallel, delayed
from torch.utils.data import default_collate
import pydicom as dcm
import transformers
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
debug: False
train_bs: 4
valid_bs: 4
test_bs: 8
workers: 1
progress_bar_refresh_rate: 1
pseudo_train: 0
save_topk: 1
fold: 5
task:
    kind: 'detect'
    \#kind: 'classify'
    \#kind: 'depth'
    
    condition: 'nfn'
    \#condition: 'scs'
    \#condition: 'scs'
    \#condition: 'all'
    
    \#direction: 'sagt2'
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
model:
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
# use train data when debug mode on
if config['debug']: 
    IMAGE_PATH = '/kaggle/input/rsna-2024-lumbar-spine-degenerative-classification/train_images/'
    series = pd.read_csv('/kaggle/input/rsna-2024-lumbar-spine-degenerative-classification/train_series_descriptions.csv')
else: 
    IMAGE_PATH = '/kaggle/input/rsna-2024-lumbar-spine-degenerative-classification/test_images/'
    series = pd.read_csv('/kaggle/input/rsna-2024-lumbar-spine-degenerative-classification/test_series_descriptions.csv')
```
```Python
series.head()
```
||study_id|series_id|series_description|
|---|---|---|---|
|0|44036939|2828203845|Sagittal T1|
|1|44036939|3481971518|Axial T2|
|2|44036939|3844393089|Sagittal T2/STIR|
---
# 0. Initial stage (Create meta file)
## Create meta file using dicom's meta data
```Python
def create_dcm_df(study_id, series_id, series_desc): 
    try: 
        path_list = glob(IMAGE_PATH + f'{study_id}/{series_id}/*.dcm')
        in_list = sorted([int(s.split('/')[-1].split('.')[0]) for s in path_list])
        dcm_list = []
        for i in in_list: 
            dcm_list.append(dcm.dcmread(IMAGE_PATH + f'{study_id}/{series_id}/{i}.dcm'))
        \#dcm_list = [dcm.dcmread(IMAGE_PATH + f'{study_id}/{series_id}/{i}.dcm') for i in in_list]
        ipp = np.asarray([d.ImagePositionPatient for d in dcm_list]).astype('float')
        iop = [d.ImageOrientationPatient for d in dcm_list]
        iop = [[float(d[0]), float(d[1]), float(d[2]), 
                float(d[3]), float(d[4]), float(d[5])] for d in iop]
        ipp_x = ipp[:, 0]
        ipp_y = ipp[:, 1]
        ipp_z = ipp[:, 2]
        shape = np.array([d.pixel_array.shape for d in dcm_list])
        sbs = np.asarray([d.SpacingBetweenSlices for d in dcm_list]).astype('float')
        ps = np.asarray([d.PixelSpacing for d in dcm_list]).astype('float')
        ps_x = ps[:, 0]
        ps_y = ps[:, 1]
        meta_dict = {'instance_number': in_list, 'ipp_x': ipp_x, 'ipp_y': ipp_y, 'ipp_z': ipp_z, 'sbs': sbs, 'ps_x': ps_x, 'ps_y': ps_y}
        meta_df = pd.DataFrame(meta_dict)
        meta_df['series_id'] = series_id
        meta_df['study_id'] = study_id
        meta_df['series_description'] = series_desc
        meta_df['height'] = shape[:, 0]
        meta_df['width'] = shape[:, 1]
        meta_df['iop'] = pd.Series(iop)
        del dcm_list, ipp, iop, sbs, ps
        gc.collect()
        return meta_df[['study_id', 'series_id', 'series_description', 'instance_number', 'height', 'width', 'ipp_x', 'ipp_y', 'ipp_z', 'iop', 'sbs', 'ps_x', 'ps_y']]
    except: 
        print(study_id, series_id, series_desc)
        return None
```
![[image 15 6.png|image 15 6.png]]
  
![[image 17 6.png|image 17 6.png]]
  
![[image 16 6.png|image 16 6.png]]
  
**Đọc thêm:**
**[[Tổng quan về giải phẩu học và phân tích hình ảnh RSNA24]]**
Geometry in Medical Imaging :[https://discovery.ucl.ac.uk/id/eprint/10146893/1/geometry_medim.pdf](https://discovery.ucl.ac.uk/id/eprint/10146893/1/geometry_medim.pdf)
  
Hàm `create_dcm_df(...)` dùng để **đọc thông tin metadata từ một chuỗi ảnh DICOM (.dcm)** trong một series MRI/CT và **tạo thành DataFrame có cấu trúc** chứa các đặc trưng không gian của từng ảnh.
Từ `study_id`, `series_id` và `series_desc` → tạo ra một `pandas.DataFrame` chứa thông tin như:
- Kích thước ảnh
- Vị trí trong không gian (`ImagePositionPatient`)
- Hướng trục (`ImageOrientationPatient`)
- Độ phân giải (`PixelSpacing`)
- Khoảng cách giữa các lát cắt (`SpacingBetweenSlices`)
  
**1. Lấy danh sách file DICOM**
- `glob(...)`: tìm tất cả file `.dcm` trong thư mục tương ứng với `study_id/series_id`
- `in_list`: lấy số thứ tự từ tên file `.dcm` (ví dụ `23.dcm` → `23`) và sắp xếp tăng dần theo `instance_number.`
  
**2. Đọc tất cả file DICOM theo thứ tự**
- Tạo list `dcm_list` chứa các object DICOM đã đọc.
  
**3. Trích xuất metadata từ các DICOM**
`ipp` - `ImagePositionPatient`: là một **chuỗi 3 giá trị (x, y, z)** trong DICOM tag, thể hiện:

> Vị trí tọa độ (mm) của góc trên trái của ảnh trong không gian bệnh nhân (Patient Coordinate System).
- Output `ipp`: có shape `[N, 3]` – N là số lát (slices), mỗi dòng là tọa độ của lát đó.
`iop = [...]` – Image Orientation Patient: là một vector chỉ hướng 6 chiều `[r1, r2, r3, c1, c2, c3]`:
- `r1, r2, r3`: vector đơn vị theo **hướng hàng (row direction)**
- `c1, c2, c3`: vector đơn vị theo **hướng cột (column direction)**
- Mỗi lát ảnh MRI là một mặt phẳng 2D nằm đâu đó trong không gian 3D cơ thể bệnh nhân. `IOP` nói cho bạn biết **mặt phẳng đó đang hướng về đâu**.
    
    Ví dụ: Ảnh chụp **Axial** (trục ngang – từ đầu đến chân)
    
    `axial_iop = [1, 0, 0, 0, 1, 0] # Axial plane`
    
    `sagittal_iop = [0, 1, 0, 0, 0, -1] # Sagittal plane`
    
    `coronal_iop = [1, 0, 0, 0, 0, -1] # Coronal plane`
    
`shape = np.array([...])` – kích thước ảnh
- `d.pixel_array.shape` → trả về `(height, width)` của từng ảnh.
- Output: mảng `[N, 2]`, dùng để xác nhận consistency hoặc xây ma trận 3D.
`sbs = np.asarray([...])` – Spacing Between Slices
- `SpacingBetweenSlices`: khoảng cách (mm) giữa **tâm của 2 lát cắt liên tiếp**.
- Dùng để dựng đúng thể tích 3D, nhất là khi lát cắt không đều.
- ⚠️ Có thể khác với `SliceThickness`, vì `SliceThickness` là độ dày ảnh, còn `SpacingBetweenSlices` là **khoảng cách giữa các ảnh**.
`ps = np.asarray([...])` – Pixel Spacing
- `PixelSpacing`: là `[dy, dx]` – khoảng cách thực tế giữa 2 pixel liên tiếp theo chiều dọc và ngang, đơn vị mm.
- Output: mảng `[N, 2]` → dùng để tính kích thước vật lý ảnh (khác với số pixel).
  
**4. Tách từng thành phần thành mảng riêng → Dùng để truy cập trực tiếp tọa độ không gian.**
ipp_x = ipp[:, 0]  
ipp_y = ipp[:, 1]  
ipp_z = ipp[:, 2]  
ps_x = ps[:, 0]  
ps_y = ps[:, 1]
  
**5. Tạo DataFrame từ metadata**
- Gộp lại thành DataFrame để dễ xử lý bằng `pandas`.
- `del dcm_list, ipp, iop, sbs, ps gc.collect()`
    - Xóa biến lớn khỏi RAM (đặc biệt quan trọng khi xử lý hàng nghìn DICOM).
- `return meta_df[['study_id', 'series_id', ..., 'ps_y']]`
  
**6.**`**except**` **– khi lỗi xảy ra:**
Bắt lỗi trong trường hợp file DICOM lỗi hoặc thiếu tag → tránh crash toàn bộ pipeline.
## Thực hiện
```Python
%%time
if config['debug']: 
    \#meta_df = pd.read_parquet('/kaggle/input/rsna-newmeta/meta.parquet')
    meta_df_list = []
    meta_df_list = Parallel(n_jobs=-1)([delayed(create_dcm_df)(row.study_id, row.series_id, row.series_description) for _, row in series.iterrows()])
    \#for _, row in tqdm(test_series.iterrows(), total=len(test_series)): 
    #    meta_df_list.append(create_dcm_df(row.study_id, row.series_id, row.series_description))
    meta_df = pd.concat(meta_df_list)
    del meta_df_list
    gc.collect()
    meta_df.to_parquet('meta.parquet')
else: 
    # spend about 20 min for train data
    meta_df_list = []
    meta_df_list = Parallel(n_jobs=-1)([delayed(create_dcm_df)(row.study_id, row.series_id, row.series_description) for _, row in series.iterrows()])
    \#for _, row in tqdm(test_series.iterrows(), total=len(test_series)): 
    #    meta_df_list.append(create_dcm_df(row.study_id, row.series_id, row.series_description))
    meta_df = pd.concat(meta_df_list)
    del meta_df_list
    gc.collect()
    meta_df.to_parquet('meta.parquet')
```
CPU times: user 290 ms, sys: 99.4 ms, total: 390 ms  
Wall time: 3.09 s
  
`%%time`: Jupyter magic để đo thời gian thực thi toàn bộ cell.
Khi `debug = True`, chạy trên **một phần dữ liệu nhỏ hơn. Nếu dùng tập train sẽ mất tầm 20’.**
- `Parallel(n_jobs=-1)`: chạy song song tất cả CPU core hiện có.
- `delayed(...)`: wrapper để chạy hàm `create_dcm_df(...)`
- `series.iterrows()`: duyệt từng dòng, mỗi dòng là một series MRI/CT.
Output: `meta_df_list` là list các DataFrame tương ứng với từng series.
- `meta_df = pd.concat(meta_df_list)`
    
    Ghép toàn bộ metadata từ nhiều series thành một `DataFrame` lớn.
    
- `del meta_df_list`, `gc.collect()`: Dọn dẹp bộ nhớ để giải phóng RAM gg colab
- `meta_df.to_parquet('meta.parquet')`: Lưu dưới định dạng `.parquet` – nhẹ hơn `.csv`, tối ưu cho xử lý bảng lớn, và hỗ trợ tốt trong Spark hoặc Pandas.
  
**Nói thêm về** `**Parallel**`
```Python
from joblib import Parallel, delayed
results = Parallel(n_jobs=4)(delayed(func)(arg1, arg2) for ... in iterable)
```
→ **Chạy lặp (loop) nhanh hơn** bằng cách **tự động phân chia công việc cho nhiều CPU core**.
- `n_jobs= 4`: sử dụng 4 core CPU (hoặc `-1` để dùng tất cả)
- `delayed(...)`: gói hàm lại thành “tác vụ có thể song song”.
- `(...) for ...`:Tạo list các “tác vụ” cần xử lý
---
# I. First stage: Depth inference
## 1. Class Dataset:
![[image 56.png|image 56.png]]
### Khởi tạo
```Python
class DepthDetectDataset(Dataset):
    def __init__(self, meta, condition, usage='sub'):
        if condition == 'scs': 
            meta = meta.loc[meta.series_description=='Sagittal T2/STIR']
        else: 
            meta = meta.loc[meta.series_description=='Sagittal T1']
        self.id = list(meta.study_id.unique())
        if 3637444890 in self.id: 
            self.id.remove(3637444890)
        self.meta = meta
        self.condition = condition
        self.usage = usage
        
        self.resize = v2.Resize((384, 384))
		def __getitem__(self, index):
        study_id = self.id[index]
        \#print(study_id)
        \#try:
        if self.condition == 'scs':
            volume = self.for_scs(study_id)
        elif self.condition == 'nfn':
            try: 
                volume = self.for_nfn(study_id)
            except: 
                print(study_id)
        return volume, torch.tensor([study_id])
    def __len__(self):
        return len(self.id)
```
  
  
Lớp `DepthDetectDataset`, kế thừa từ `torch.utils.data.Dataset`, dùng để tạo **dataset 3D volume từ ảnh DICOM**
**Mục tiêu:** Từ `meta` chứa thông tin DICOM, chọn ra đúng `series` theo `condition`, rồi load ảnh thành `volume 3D` cho từng `study_id`.
**__init__**
**Tham số:**
- `meta`: DataFrame chứa thông tin metadata các file DICOM (được tạo từ create_dcm_df)
- `condition`: Loại bài toán (scs, nfn, v.v.)
- `usage`: Có thể là `sub`, `train`, `val`, v.v. – dùng để phân tách dữ liệu
**Bộ lọc theo hướng chụp (**`**series_description**`**):**
- Nếu task là **scs** (Spinal Canal Stenosis) → chọn ảnh `Sagittal T2/STIR`
- Ngược lại → chọn ảnh `Sagittal T1` (thường dùng cho NFN - normal finding)
**Lưu danh sách study:**
- `self.id`: danh sách các **study_id duy nhất** (mỗi patient scan).
- Loại bỏ `study_id = 3637444890` vì nó có thể bị lỗi, corrupt hoặc không đầy đủ lát ảnh.
**Khởi tạo thuộc tính:** Lưu lại metadata, loại điều kiện và hàm resize dùng cho ảnh 2D.
- self.meta = meta
- self.condition = condition
- self.usage = usage
- self.resize = v2.Resize((384, 384))
  
**__getitem__(self, index)**
- `study_id = self.id[index]`: Lấy `study_id` tại vị trí `index`.
**Load volume tương ứng theo task:**
- Nếu là `scs` → gọi hàm `for_scs(study_id)` để load volume MRI T2
- Nếu là `nfn` → gọi hàm `for_nfn(study_id)` để load volume MRI T1
**Trả về kết quả**
- `volume`: tensor dạng `[C, D, H, W]` hoặc `[1, D, 384, 384]` – toàn bộ ảnh 3D của bệnh nhân.
- `study_id`: để trace lại ảnh nào thuộc bệnh nhân nào (tránh label nhầm).
  
**__len__(self)**
- Trả về **tổng số volume MRI/CT mà dataset này quản lý**.
### For_scs
```Python
		def for_scs(self, study_id):
        depth = 32
        meta = self.meta.loc[(self.meta.study_id==study_id) & (self.meta.series_description=='Sagittal T2/STIR')]
        meta = meta.sort_values('ipp_x', ascending=True).reset_index(drop=True)
        img = [self.load_dicom(IMAGE_PATH + f'{row.study_id}/{row.series_id}/{row.instance_number}.dcm') for _, row in meta.iterrows()]
        volume = self.normalize(torch.cat([self.resize(torch.tensor(i.astype(np.float32))[None, ...]).to(torch.float32) for i in img]).contiguous())
        if volume.shape[0] < depth:
            volume = torch.cat([volume, torch.zeros(depth-volume.shape[0], volume.shape[1], volume.shape[2])])
        elif volume.shape[0] > depth:
            volume = torch.nn.functional.interpolate(volume[None, None, ...], (depth, volume.shape[1], volume.shape[2])).squeeze()
        return volume.to(torch.float32)
```
Hàm dùng cho task SCS
Trả về 1 **tensor ảnh 3D (volume)** có shape cố định `(depth=32, H=384, W=384)` để đưa vào mô hình học sâu.
- `depth = 32`: Thiết lập số lát (slices) của ảnh.
- `meta`: Lấy series ảnh theo `study_id` và là ảnh Sagittal T2/STIR.
- `meta.sort:` Sắp xếp lát ảnh theo trục không gian (tăng dần ipp_x) → Giúp đảm bảo thứ tự lát ảnh đúng (quan trọng trong volume 3D).
- `img:` Gọi `self.load_dicom(...)` để đọc từng ảnh `.dcm` → kết quả: list các mảng ảnh `numpy`.
- Chuyển ảnh thành tensor 3D stack: `volume`
    - Chuyển `numpy → tensor`
    - Thêm chiều channel (1) → `[1, H, W]`
    - Resize về `[1, 384, 384]`
    - Sau đó `cat` lại theo chiều depth → ra tensor shape: `[D, 384, 384]`
    - Áp dụng normalization
- Điều chỉnh độ sâu (chuẩn hóa số lát ảnh):
    - Nếu ảnh có **ít lát hơn 32**, thì **padding thêm các lát đen (0)**.
    - Nếu ảnh có **nhiều lát hơn 32**, thì **interpolate giảm số lát xuống 32** (dùng nội suy tuyến tính 3D).
  
### For_nfn
```Python
		def for_nfn(self, study_id):
        depth = 32
        meta = self.meta.loc[(self.meta.study_id==study_id) & (self.meta.series_description=='Sagittal T1')]
        meta = meta.sort_values('ipp_x', ascending=True).reset_index(drop=True)
        img = [self.load_dicom(IMAGE_PATH + f'{row.study_id}/{row.series_id}/{row.instance_number}.dcm') for _, row in meta.iterrows()]
        volume = self.normalize(torch.cat([self.resize(torch.tensor(i.astype(np.float32))[None, ...]).to(torch.float32) for i in img]).contiguous())
        if volume.shape[0] < depth:
            volume = torch.cat([volume, torch.zeros(depth-volume.shape[0], volume.shape[1], volume.shape[2])])
        elif volume.shape[0] > depth:
            volume = torch.nn.functional.interpolate(volume[None, None, ...], (depth, volume.shape[1], volume.shape[2])).squeeze()
        return volume.to(torch.float32)
```
Hàm `for_nfn(self, study_id)` gần như giống hệt `for_scs(...)`, nhưng được dùng cho **ảnh MRI Sagittal T1** — phục vụ cho các mẫu thuộc loại **NFN (Normal Finding Negative)**
  
### Các hàm khác
```Python
		def normalize(self, x):
        upper = torch.quantile(x, torch.tensor([0.99]))
        lower = torch.quantile(x, torch.tensor([0.01]))
        x = torch.clip(x, lower, upper)
        x = x - torch.min(x)
        x = x / (torch.max(x)+1e-6)
        return x
    def load_dicom(self, path):
        dicom = dcm.read_file(path)
        data = dicom.pixel_array
        return data
```
==**normalize(self, x)**==
Chuẩn hóa ảnh `x` về khoảng `[0, 1]`, tăng độ tương phản, loại bỏ outlier.
- `upper/lower = torch.quantile(x, 0.99)` : Giới hạn trên/dưới để cắt 1% pixel sáng/tối bất thường.
- `torch.clip(x, lower, upper)`: Cắt ngưỡng đã set
- Chuẩn hóa về [0, 1] :
    
    x = x - torch.min(x)  
    x = x / (torch.max(x) + 1e-6)
    
**load_dicom(self, path)**
- `dcm.read_file(path)` (thường là `pydicom.dcmread(path)`): đọc file `.dcm` và trả về một object DICOM.
- `dicom.pixel_array`: trích xuất **ảnh 2D gốc** (numpy array) từ DICOM. Output → Ảnh 2D numpy (kiểu `uint16` hoặc `int16`), chưa chuẩn hóa.
---
## 2. Models: 3D ConvNeXt
### ConvNextStem
```Python
from torchvision.ops import StochasticDepth
from typing import List, Dict
from torch import Tensor
class ConvNextStem(nn.Sequential):
    def __init__(self, in_features: int, out_features: int):
        super().__init__(
            nn.Conv3d(in_features, out_features, kernel_size=(1, 2, 2), stride=(1, 2, 2)),
            nn.GroupNorm(num_groups=1, num_channels=out_features)
        )
```
  
**Khối khởi đầu (stem block)**
Mục tiêu lớp ConvNextStem: Áp dụng một lớp convolution 3D (trên ảnh 3D volume) để giảm chiều không gian, sau đó chuẩn hóa đặc trưng đầu ra.
  
Kế thừa từ `nn.Sequential` → tức là khối này là **chuỗi layer** được thực hiện nối tiếp nhau.
- Dễ dùng: có thể truyền tensor vào class như một layer bình thường: `x = ConvNextStem(...)(x)`
**__init__**(self, in_features: int, out_features: int):
- `in_features`: số kênh đầu vào (ví dụ: `1` nếu là ảnh xám, `3` nếu là RGB).
- `out_features`: số kênh đầu ra sau `Conv3d` (tăng độ trừu tượng).
**Các layer bên trong:**
- nn.Conv3d(...)
    
    - Là convolution 3D: hoạt động trên tensor dạng `[B, C, D, H, W]`
    - `kernel_size=(1,2,2)` → chỉ **giảm kích thước không gian** (H × W), **giữ nguyên chiều depth (D)**.
    - `stride=(1,2,2)` → giảm 2 lần chiều cao và rộng (downsample), giống như "stem block" trong ConvNeXt hoặc ViT.
    - **Output shape:** input `[B, C=1, D=32, H=384, W=384]` → output `[B, out_features, D=32, H=192, W=192]`
    
    ![[image 1 22.png|image 1 22.png]]
    
- nn.GroupNorm(num_groups=1, num_channels=out_features)
    
    - GroupNorm giúp **chuẩn hóa đầu ra** từ `Conv3d`.
    - `num_groups=1` tương đương với **LayerNorm theo kênh**, tức chuẩn hóa trên toàn bộ `C×H×W` với từng batch/sample.
    - Ổn định hơn BatchNorm trong huấn luyện 3D và khi batch size nhỏ.
    
    ![[image 2 21.png|image 2 21.png]]
    
  
### LayerScaler
```Python
class LayerScaler(nn.Module):
    def __init__(self, init_value: float, dimensions: int):
        super().__init__()
        self.gamma = nn.Parameter(init_value * torch.ones((dimensions)),
                                    requires_grad=True)
		def forward(self, x):
        return self.gamma[None,...,None,None] * x
```
**Đây là lớp chuẩn hóa hiện đại** được dùng trong các kiến trúc như **ConvNeXt**, **Swin Transformer**, v.v. — nhằm **khống chế độ lớn của đầu ra mỗi layer**, giúp mô hình học ổn định hơn.
Mục tiêu:  
Nhân đầu ra của layer với một vector hệ số $γ$ (trainable), gọi là LayerScale, để điều chỉnh độ lớn của feature trước khi cộng skip connection hoặc truyền tiếp.
**Tham số**
- Kế thừa từ `nn.Module`
- `init_value`: giá trị khởi tạo cho scale (ví dụ `1e-6` hoặc `1.0`)
- `dimensions`: số kênh đầu ra (`C`) của tensor cần nhân
**Khởi tạo tham số** `**gamma**`**:**
- `self.gamma`: vector $γ ∈ ℝ^C$ – trainable
- `gamma` sẽ **được học trong quá trình training**.
- Ban đầu, gamma = `[init_value, ..., init_value]` (dài `dimensions` phần tử)
**Hàm** `**forward**`**:**
Giả sử:
- `x` có shape `[B, C, H, W]` hoặc `[B, C, D, H, W]`
- `self.gamma` shape: `[C]`
→ `self.gamma[None, ..., None, None]` sẽ reshape thành `[1, C, 1, 1]` (2D) hoặc `[1, C, 1, 1, 1]` (3D)
→ broadcasting cho phép nhân từng kênh **riêng biệt**
✅ Tức là: mỗi channel trong `x` được scale bởi một hệ số khác nhau (trainable).
### BottleNeckBlock
```Python
class BottleNeckBlock(nn.Module):
    def __init__(
        self,
        in_features: int,
        out_features: int,
        expansion: int = 4,
        drop_p: float = .0,
        layer_scaler_init_value: float = 1e-6,
    ):
        super().__init__()
        expanded_features = out_features * expansion
        self.block = nn.Sequential(
            # narrow -> wide (with depth-wise and bigger kernel)
            nn.Conv3d(
                in_features, in_features, kernel_size=(2, 7, 7), padding='same', bias=False, groups=in_features
            ),
            # GroupNorm with num_groups=1 is the same as LayerNorm but works for 2D data
            nn.GroupNorm(num_groups=in_features, num_channels=in_features),
						# wide -> wide
            nn.Conv3d(in_features, expanded_features, kernel_size=1),
            nn.GELU(),
            # wide -> narrow
            nn.Conv3d(expanded_features, out_features, kernel_size=1),
        )
        \#self.layer_scaler = LayerScaler(layer_scaler_init_value, out_features)
        \#self.drop_path = StochasticDepth(drop_p, mode="batch")

    def forward(self, x: Tensor) -> Tensor:
        res = x
        x = self.block(x)
        \#x = self.layer_scaler(x)
        \#x = self.drop_path(x)
        x += res
        return x
```
**Mục tiêu:**  
Tạo một khối residual block có khả năng học hiệu quả với ảnh 3D, giúp mô hình sâu hơn nhưng vẫn ổn định khi huấn luyện.
**Kiến trúc:**
```Python
Input ──╮
        │
        ├─ Conv3d (DW, kernel 2x7x7, same padding)
        ├─ GroupNorm
        ├─ Conv3d (1x1, expand)
        ├─ GELU
        ├─ Conv3d (1x1, reduce)
        │
        └─ (optional LayerScaler → DropPath)
        │
    Add Residual
        ↓
     Output
```
==**__init__**==
`in_features` Số kênh đầu vào
`out_features` Số kênh đầu ra (cũng là số kênh residual)
`expansion=4` Hệ số mở rộng số kênh ở giữa
`drop_p=0.0` Xác suất DropPath (chưa dùng)
`layer_scaler_init_value=1e-6` Giá trị khởi tạo của LayerScaler (chưa bật)
**Tính số kênh mở rộng:**
`expanded_features = out_features * expansion`
**Khối** `**self.block**`
Depthwise Conv3d:
- `groups=in_features` ⇒ **depthwise convolution**
- Kernel `(2, 7, 7)` → nhìn rộng theo chiều không gian nhưng ngắn theo chiều sâu
- `padding='same'` → giữ nguyên kích thước
GroupNorm:
- Chuẩn hóa theo từng channel riêng biệt (giống `LayerNorm` cho 3D)
Conv3d 1×1 (expand)
GELU activation
Conv3d 1×1 (reduce)
**Các bước hiện đang comment (chưa dùng):**
- **Ổn định đầu ra** (LayerScaler)
- **Giảm overfitting** (StochasticDepth)
  
==**forward(...)**==
- `res`: lưu lại input để dùng cho **skip connection**
- `x += res`: cộng đầu vào và đầu ra (residual learning)
### ConvNexStage
```Python
class ConvNexStage(nn.Sequential):
    def __init__(
        self, in_features: int, out_features: int, depth: int, **kwargs
    ):
        super().__init__(
            # add the downsampler
            nn.Sequential(
                nn.GroupNorm(num_groups=in_features, num_channels=in_features),
                nn.Conv3d(in_features, out_features, kernel_size=(2, 2, 2), stride=(2, 2, 2))
            ),
            *[
                BottleNeckBlock(out_features, out_features, **kwargs)
                for _ in range(depth)
            ],
        )
```
Đây là một module trong kiến trúc ConvNeXt 3D, gồm:
- **1 layer downsampling (reduce spatial/temporal resolution, increase channels)**
- **N nhiều block BottleNeck (học đặc trưng sâu hơn ở resolution mới)**
**Kế thừa** `**nn.Sequential**`
- Toàn bộ khối là một chuỗi module, truyền input qua lần lượt Downsample → Block1 → Block2 → ... → BlockN.
**Downsampling**
- **GroupNorm**: Chuẩn hóa, ổn định learning (đặc biệt khi batch size nhỏ).
- **Conv3d**: Giảm mỗi chiều của tensor (D, H, W) xuống một nửa, đồng thời tăng số channel lên `out_features`.
- Output: `[B, out_features, D/2, H/2, W/2]`.
**Chuỗi BottleNeckBlock**
- Tạo `depth` block nối tiếp nhau, mỗi block đầu vào và ra cùng số channel (`out_features`).
- Có thể truyền các tham số như `expansion`, `drop_p`... vào block.
### ConvNextEncoder
```Python
class ConvNextEncoder(nn.Module):
    def __init__(
        self,
        in_channels: int,
        stem_features: int,
        depths: List[int],
        widths: List[int],
        drop_p: float = .0,
    ):
        super().__init__()
        self.stem = ConvNextStem(in_channels, stem_features)
        in_out_widths = list(zip(widths, widths[1:]))
        # create drop paths probabilities (one for each stage)
        drop_probs = [x.item() for x in torch.linspace(0, drop_p, sum(depths))]
				self.stages = nn.ModuleList(
            [
                ConvNexStage(stem_features, widths[0], depths[0], drop_p=drop_probs[0]),
                *[
                    ConvNexStage(in_features, out_features, depth, drop_p=drop_p)
                    for (in_features, out_features), depth, drop_p in zip(
                        in_out_widths, depths[1:], drop_probs[1:]
                    )
                ],
            ]
        )

    def forward(self, x):
        x = self.stem(x)
        for stage in self.stages:
            x = stage(x)
        return x
```
Tổ chức một **chuỗi khối xử lý đặc trưng** gồm:
- 1 **stem** đầu vào
- N **stage** (mỗi stage giảm độ phân giải, tăng số kênh, gồm nhiều bottleneck block)
**Stem**
- Stem = 1 block Conv3d + chuẩn hóa → tăng kênh lên `stem_features`, giảm resolution ban đầu.
**Chuẩn bị cho các stage**
- `in_out_widths`: tạo các cặp `(in_channels, out_channels)` cho từng stage.
- `drop_probs`: nếu muốn drop path stochastic, tạo ra mảng xác suất drop đều từ 0 đến `drop_p` trên toàn bộ số block (dùng cho regularization kiểu ConvNeXt/Swin).
**Xây dựng các stage**
- Stage đầu tiên: `stem_features → widths[0]`, `depths[0]` block, drop path theo `drop_probs[0]`.
- Các stage tiếp theo: lần lượt `(in_features, out_features)`, mỗi stage `depth` block.
- Tất cả các stage được xâu chuỗi bằng `ModuleList` (cho phép forward tuần tự).
**Hàm** `**forward**`
- Dữ liệu đi qua stem, rồi lần lượt qua từng stage.
  
### `ClassificationHead`
```Python
class ClassificationHead(nn.Sequential):
    def __init__(self):
        super().__init__(
            nn.AdaptiveAvgPool3d((1, 1, 1)),
            nn.Flatten(1),
            nn.LayerNorm(512),
            nn.Linear(512, 3)
        )
```
Đóng vai trò **đầu ra (output head) cho bài toán phân loại** khi sử dụng backbone ConvNeXt3D hoặc tương tự.
Biến feature map 3D (sau backbone) thành **vector phân loại** (logit cho từng class).
`**nn.AdaptiveAvgPool3d((1, 1, 1))**`
- Lấy **trung bình toàn bộ không gian 3D** (D, H, W) của feature map.
- Output shape: `[B, C, 1, 1, 1]` (C thường là 512).
`**nn.Flatten(1)**`
- Bỏ chiều không gian còn lại, chỉ giữ lại vector channel:
- Output shape: `[B, C]`, tức `[B, 512]`.
`**nn.LayerNorm(512)**`
- Chuẩn hóa từng sample, giúp mạng ổn định hơn, đặc biệt sau pooling.
`**nn.Linear(512, 3)**`
- Fully connected layer để map vector 512 chiều thành vector logits 3 chiều, mỗi chiều là 1 class.
- Dùng cho bài toán phân loại 3 lớp.
  
```Python
class Flatten(nn.Sequential):
    def __init__(self):
        super().__init__(
            nn.AdaptiveAvgPool3d((1, 1, 1)),
            nn.Flatten(1),
            nn.LayerNorm(512)
        )
```
  
```Python
class ConvNextSSDepthDetect(nn.Module):
    def __init__(self):
        super().__init__()
        self.encoder = ConvNextEncoder(in_channels=1, stem_features=32, depths=[3,3,9,3], widths=[64, 128, 256, 512])
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool3d((1,1,1)),
                                    nn.Flatten(1),
                                    nn.LayerNorm(512))
        self.ll1 = nn.Linear(512, 96)
        self.ll2 = nn.Linear(512, 96)
        self.ll3 = nn.Linear(512, 96)
        self.ll4 = nn.Linear(512, 96)
        self.ll5 = nn.Linear(512, 96)
        self.rl1 = nn.Linear(512, 96)
        self.rl2 = nn.Linear(512, 96)
        self.rl3 = nn.Linear(512, 96)
        self.rl4 = nn.Linear(512, 96)
        self.rl5 = nn.Linear(512, 96)
	  def forward(self, x, label=None):
        x = x.unsqueeze(1)
        x = self.encoder(x)
        x = self.flatten(x)
        ll1 = self.ll1(x)
        ll2 = self.ll2(x)
        ll3 = self.ll3(x)
        ll4 = self.ll4(x)
        ll5 = self.ll5(x)
        rl1 = self.rl1(x)
        rl2 = self.rl2(x)
        rl3 = self.rl3(x)
        rl4 = self.rl4(x)
        rl5 = self.rl5(x)
        return {'left_L1/L2': ll1,'left_L2/L3': ll2,'left_L3/L4': ll3, 'left_L4/L5': ll4, 'left_L5/S1': ll5,
                'right_L1/L2': rl1, 'right_L2/L3': rl2, 'right_L3/L4': rl3, 'right_L4/L5': rl4, 'right_L5/S1': rl5}
```
  
```Python
class PositionalEncoding(nn.Module):
    def __init__(self, d_model: int, dropout: float = 0.1, max_len: int = 5000):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)
        position = torch.arange(max_len).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, d_model, 2) * (-math.log(10000.0) / d_model))
        pe = torch.zeros(max_len, 1, d_model)
        pe[:, 0, 0::2] = torch.sin(position * div_term)
        pe[:, 0, 1::2] = torch.cos(position * div_term)
        self.register_buffer('pe', pe)
		def forward(self, x: Tensor) -> Tensor:
        """
        Args:
            x: Tensor, shape [batch_size, seq_len, embedding_dim]
        """
        x = x.permute(1, 0, 2)
        x = x + self.pe[:x.size(0)]
        return self.dropout(x.permute(1, 0, 2))
```
  
```Python
class AttentionSSDepthDetect(nn.Module):
    def __init__(self):
        super().__init__()
        self.encoder = timm.create_model('convnext_base.fb_in22k_ft_in1k_384', in_chans=3, pretrained=True, num_classes=0)
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool2d((1,1)),
                                    nn.Flatten(1),
                                    nn.LayerNorm(512))
        self.in_features = self.encoder.num_features
        self.rpe = PositionalEncoding(self.in_features, dropout=0., max_len=64)
        self.transformer0 = nn.TransformerEncoderLayer(d_model=self.in_features, nhead=8, activation='gelu',dropout=0.1, batch_first=True)
        self.transformer1 = nn.TransformerEncoderLayer(d_model=self.in_features, nhead=8, activation='gelu',dropout=0.1, batch_first=True)
        
        self.ll1 = nn.Linear(512, 64)
        self.ll2 = nn.Linear(512, 64)
        self.ll3 = nn.Linear(512, 64)
        self.ll4 = nn.Linear(512, 64)
        self.ll5 = nn.Linear(512, 64)
        self.rl1 = nn.Linear(512, 64)
        self.rl2 = nn.Linear(512, 64)
        self.rl3 = nn.Linear(512, 64)
        self.rl4 = nn.Linear(512, 64)
        self.rl5 = nn.Linear(512, 64)
    def forward(self, x, label=None):
        x = x.unsqueeze(1)
        x = self.encoder(x)
        x = self.flatten(x)
        ll1 = self.ll1(x)
        ll2 = self.ll2(x)
        ll3 = self.ll3(x)
        ll4 = self.ll4(x)
        ll5 = self.ll5(x)
        rl1 = self.rl1(x)
        rl2 = self.rl2(x)
        rl3 = self.rl3(x)
        rl4 = self.rl4(x)
        rl5 = self.rl5(x)
        return {'left_L1/L2': ll1,'left_L2/L3': ll2,'left_L3/L4': ll3, 'left_L4/L5': ll4, 'left_L5/S1': ll5,
                'right_L1/L2': rl1, 'right_L2/L3': rl2, 'right_L3/L4': rl3, 'right_L4/L5': rl4, 'right_L5/S1': rl5}
```
  
```Python
class ConvNextSCSDepthDetect(nn.Module):
    def __init__(self):
        super().__init__()
        self.encoder = ConvNextEncoder(in_channels=1, stem_features=32, depths=[3,3,9,3], widths=[64, 128, 256, 512])
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool3d((1,1,1)),
                                    nn.Flatten(1),
                                    nn.LayerNorm(512))
        self.l1 = nn.Linear(512, 32)
        self.l2 = nn.Linear(512, 32)
        self.l3 = nn.Linear(512, 32)
        self.l4 = nn.Linear(512, 32)
        self.l5 = nn.Linear(512, 32)
    def forward(self, x, label=None):
        x = x.unsqueeze(1)
        x = self.encoder(x)
        x = self.flatten(x)
        l1 = self.l1(x)
        l2 = self.l2(x)
        l3 = self.l3(x)
        l4 = self.l4(x)
        l5 = self.l5(x)
        return {'L1/L2': l1, 'L2/L3': l2, 'L3/L4': l3, 'L4/L5': l4, 'L5/S1': l5}
```
  
```Python
class ConvNextSCSDepthDetect(nn.Module):
    def __init__(self, widths):
        super().__init__()
        self.encoder = ConvNextEncoder(in_channels=1, stem_features=widths[0]//2, depths=[3,3,9,3], widths=widths)
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool3d((1,1,1)),
                                    nn.Flatten(1),
                                    nn.LayerNorm(widths[-1]),
                                     )
        self.l1 = nn.Linear(widths[-1], 32)
        self.l2 = nn.Linear(widths[-1], 32)
        self.l3 = nn.Linear(widths[-1], 32)
        self.l4 = nn.Linear(widths[-1], 32)
        self.l5 = nn.Linear(widths[-1], 32)
    def forward(self, x, label=None):
        x = x.unsqueeze(1)
        x = self.encoder(x)
        x = self.flatten(x)
        l1 = self.l1(x)
        l2 = self.l2(x)
        l3 = self.l3(x)
        l4 = self.l4(x)
        l5 = self.l5(x)
        return {'L1/L2': l1, 'L2/L3': l2, 'L3/L4': l3, 'L4/L5': l4, 'L5/S1': l5}
```
  
  
```Python
class ConvNextNFNDepthDetect(nn.Module):
    def __init__(self, widths):
        super().__init__()
        self.encoder = ConvNextEncoder(in_channels=1, stem_features=widths[0]//2, depths=[3,3,9,3], widths=widths)
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool3d((1,1,1)),
                                    nn.Flatten(1),
                                    nn.LayerNorm(widths[-1]))
        self.ll1 = nn.Linear(widths[-1], 32)
        self.ll2 = nn.Linear(widths[-1], 32)
        self.ll3 = nn.Linear(widths[-1], 32)
        self.ll4 = nn.Linear(widths[-1], 32)
        self.ll5 = nn.Linear(widths[-1], 32)
        self.rl1 = nn.Linear(widths[-1], 32)
        self.rl2 = nn.Linear(widths[-1], 32)
        self.rl3 = nn.Linear(widths[-1], 32)
        self.rl4 = nn.Linear(widths[-1], 32)
        self.rl5 = nn.Linear(widths[-1], 32)
		def forward(self, x, label=None):
        x = x.unsqueeze(1)
        x = self.encoder(x)
        x = self.flatten(x)
        ll1 = self.ll1(x)
        ll2 = self.ll2(x)
        ll3 = self.ll3(x)
        ll4 = self.ll4(x)
        ll5 = self.ll5(x)
        rl1 = self.rl1(x)
        rl2 = self.rl2(x)
        rl3 = self.rl3(x)
        rl4 = self.rl4(x)
        rl5 = self.rl5(x)
        return {'left_L1/L2': ll1,'left_L2/L3': ll2,'left_L3/L4': ll3, 'left_L4/L5': ll4, 'left_L5/S1': ll5,
                'right_L1/L2': rl1, 'right_L2/L3': rl2, 'right_L3/L4': rl3, 'right_L4/L5': rl4, 'right_L5/S1': rl5}
```
  
```Python
class RegConvNextNFNDepthDetect(nn.Module):
    def __init__(self, widths):
        super().__init__()
        self.encoder = ConvNextEncoder(in_channels=1, stem_features=widths[0]//2, depths=[3,3,9,3], widths=widths)
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool3d((1,1,1)),
                                    nn.Flatten(1),
                                    nn.LayerNorm(1024))
        self.ll1 = nn.Linear(1024, 3)
        self.ll2 = nn.Linear(1024, 3)
        self.ll3 = nn.Linear(1024, 3)
        self.ll4 = nn.Linear(1024, 3)
        self.ll5 = nn.Linear(1024, 3)
        self.rl1 = nn.Linear(1024, 3)
        self.rl2 = nn.Linear(1024, 3)
        self.rl3 = nn.Linear(1024, 3)
        self.rl4 = nn.Linear(1024, 3)
        self.rl5 = nn.Linear(1024, 3)
        
		def forward(self, x, label=None):
        x = x.unsqueeze(1)
        x = self.encoder(x)
        x = self.flatten(x)
        ll1 = self.ll1(x).sigmoid()
        ll2 = self.ll2(x).sigmoid()
        ll3 = self.ll3(x).sigmoid()
        ll4 = self.ll4(x).sigmoid()
        ll5 = self.ll5(x).sigmoid()
        rl1 = self.rl1(x).sigmoid()
        rl2 = self.rl2(x).sigmoid()
        rl3 = self.rl3(x).sigmoid()
        rl4 = self.rl4(x).sigmoid()
        rl5 = self.rl5(x).sigmoid()
        return {'left_L1/L2': ll1,'left_L2/L3': ll2,'left_L3/L4': ll3, 'left_L4/L5': ll4, 'left_L5/S1': ll5,
                'right_L1/L2': rl1, 'right_L2/L3': rl2, 'right_L3/L4': rl3, 'right_L4/L5': rl4, 'right_L5/S1': rl5}
```
```Python
class RegConvNextSCSDepthDetect(nn.Module):
    def __init__(self, widths):
        super().__init__()
        self.encoder = ConvNextEncoder(in_channels=1, stem_features=widths[0]//2, depths=[3,3,9,3], widths=widths)
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool3d((1,1,1)),
                                    nn.Flatten(1),
                                    nn.LayerNorm(1024),
                                     )
        self.l1 = nn.Linear(1024, 3)
        self.l2 = nn.Linear(1024, 3)
        self.l3 = nn.Linear(1024, 3)
        self.l4 = nn.Linear(1024, 3)
        self.l5 = nn.Linear(1024, 3)
    def forward(self, x, label=None):
        x = x.unsqueeze(1)
        x = self.encoder(x)
        x = self.flatten(x)
        l1 = self.l1(x).sigmoid()
        l2 = self.l2(x).sigmoid()
        l3 = self.l3(x).sigmoid()
        l4 = self.l4(x).sigmoid()
        l5 = self.l5(x).sigmoid()
        return {'L1/L2': l1, 'L2/L3': l2, 'L3/L4': l3, 'L4/L5': l4, 'L5/S1': l5}
```
  
  
---
## 3. L**ightning module**
```Python
class DepthDetectModule(pl.LightningModule):
    def __init__(self, condition, widths=None, model_type='regression'):
        super().__init__()
        self.config = config
        if condition == 'scs':
            if model_type == 'regression': 
                self.model = RegConvNextSCSDepthDetect(widths)
            else: 
                self.model = ConvNextSCSDepthDetect(widths)
        elif condition == 'nfn':
            if model_type == 'regression': 
                self.model = RegConvNextNFNDepthDetect(widths)
            else: 
                self.model = ConvNextNFNDepthDetect(widths)
    def forward(self, batch):
        preds = self.model(batch)
        return preds
```
Class `**DepthDetectModule**`, kế thừa từ `pl.LightningModule` (PyTorch Lightning), dùng để đóng gói toàn bộ pipeline mô hình học sâu.
**Ý nghĩa chính:**
- **Tự động chọn backbone và loại bài toán (regression hay classification)** dựa trên biến `condition` (bệnh học) và `model_type`.
- **Tiện lợi cho quản lý, huấn luyện, thử nghiệm nhiều mô hình khác nhau** trong cùng một pipeline.
**Constructor** `**__init__**`
Nhận tham số:
- `condition`: xác định loại bệnh (ví dụ `'scs'` – Spinal Canal Stenosis, `'nfn'` – Normal Finding Negative,...)
- `widths`: cấu hình số lượng channel từng stage của backbone.
- `model_type`: `'regression'` hoặc `'classification'` (chọn loại bài toán).
**Chọn mô hình phù hợp tự động**
- Nếu bệnh là `'scs'`:
    - Nếu là bài toán hồi quy: dùng `RegConvNextSCSDepthDetect`
    - Nếu là phân loại: dùng `ConvNextSCSDepthDetect`
- Nếu bệnh là `'nfn'`:
    - Tương tự với hai mô hình cho task `'nfn'`
**Hàm forward**
- Forward chỉ việc gọi forward của backbone phù hợp, trả về dự đoán (`preds`).
- Hỗ trợ cả inference lẫn training.
  
---
## 5. I**nference**
```Python
%%time
prefix = ''
import warnings
warnings.filterwarnings("ignore")
depth_predict = {'scs': {
                     'L1/L2':[], 
                     'L2/L3': [], 
                     'L3/L4': [], 
                     'L4/L5': [], 
                     'L5/S1': []
                     }, 
                 'nfn': {
                     'left_L1/L2': [], 
                     'left_L2/L3': [], 
                     'left_L3/L4': [], 
                     'left_L4/L5': [], 
                     'left_L5/S1': [], 
                     'right_L1/L2': [], 
                     'right_L2/L3': [], 
                     'right_L3/L4': [], 
                     'right_L4/L5': [], 
                     'right_L5/S1': [], 
                     }
                    }
model_path_dict = {
    'scs': [
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_4.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_4.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_l1_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_l1_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_l1_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_l1_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_depth_1024_ssr_l1_4.ckpt', 
    ], 
		'nfn': [
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_4.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_4.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_l1_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_l1_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_l1_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_l1_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_depth_1024_ssr_l1_4.ckpt', 
    ]
}
#############\#DEPTH DETECT#########################
for condition in ['nfn', 'scs']:
    print(condition)
    model_path_list = model_path_dict[condition]
    for model_path in model_path_list:
        _meta_df = meta_df.copy()
        \#_series = series.copy()
        dataset_test = DepthDetectDataset(_meta_df, condition, 'sub')
        data_loader_test = DataLoader(
            dataset_test,
            batch_size=config["test_bs"], 
            shuffle=False,
            num_workers=4,
            pin_memory=False
        )
        model_name = model_path.split('/')[-1]
        if '1024' in model_name: 
            widths = [128, 256, 512, 1024]
        else: 
            widths = [64, 128, 256, 512]
        if 'l1' in model_name: 
            model_type = 'regression'
        else: 
            model_type = 'classification'
        model = DepthDetectModule.load_from_checkpoint(model_path, condition=condition, widths=widths, model_type=model_type)
        model.eval()
        model.zero_grad()
        model.to(device)
        pred_temp = {}
        for k in depth_predict[condition].keys(): 
            pred_temp[k] = []
        study_id_list = []
        with torch.no_grad():
            for data in tqdm(data_loader_test, total=len(data_loader_test)):
                images, study_id = data
                images = images.to(device)
                preds = model.forward(images)
                \#print(preds)
                if model_type == 'regression': 
                    for k, v in preds.items(): 
                        pred_temp[k].append((v[:, -1]*32).to('cpu').detach().numpy())
                else: 
                    for k, v in preds.items(): 
                        pred_temp[k].append(torch.argmax(v, dim=1).to('cpu').detach().numpy())
                study_id_list.append(study_id.to('cpu').reshape(-1).detach().numpy())
                del images, study_id, preds
                gc.collect()
				for k, v in pred_temp.items(): 
            depth_predict[condition][k].append(np.concatenate(v))
        study_id = np.concatenate(study_id_list)
        del pred_temp, study_id_list
        gc.collect()
        
    for k, v in depth_predict[condition].items(): 
        depth_predict[condition][k] = np.median(np.array(depth_predict[condition][k]), axis=0)
    depth_predict[condition]['study_id'] = study_id
    del study_id
    gc.collect()
```
**Pipeline inference và ensemble** cho mô hình **Depth Detection**, với các mô hình và checkpiont khác nhau trên cả hai condition: `'scs'` và `'nfn'`.
Ensemble là **kỹ thuật kết hợp nhiều mô hình** lại với nhau để tạo ra **kết quả cuối cùng tốt hơn, ổn định hơn** so với chỉ dùng một mô hình duy nhất.
Tổng thể:
- **Duyệt qua từng loại mô hình (**`**scs**`**,** `**nfn**`**),** load từng checkpoint mô hình, chạy dự đoán trên toàn bộ test set.
- **Gộp (ensemble)** kết quả dự đoán từ nhiều mô hình khác nhau (nhiều checkpoint, nhiều fold) → lấy giá trị trung vị (median) của từng đốt sống cho mỗi study (patient).
- **Tối ưu cho đánh giá chính xác nhất trên leaderboard hoặc trong thực tế.**
  
**1. Khởi tạo dictionary lưu kết quả dự đoán**
```Python
depth_predict = {
		'scs': {'L1/L2':[], ...},
		'nfn': {'left_L1/L2':[], ..., 'right_L5/S1':[]}
}
```
- Mỗi key sẽ lưu lại tất cả dự đoán từng mô hình, từng batch, từng study.
- Giúp gộp lại cho bước ensemble.
**2.** `**model_path_dict**`
- Dict chứa danh sách các checkpoint mô hình đã train cho từng condition (`scs`, `nfn`).
- Các checkpoint này có thể là nhiều fold, nhiều config, nhiều loại (classification/regression).
**3. Vòng lặp chính qua 2 condition:**
`for condition in ['nfn', 'scs']: ….`
→ Chạy lần lượt cho từng loại bệnh hoặc label khác nhau.
**4. Lặp qua từng checkpoint mô hình**
Với mỗi mô hình đã train (checkpoint khác nhau), sẽ chạy dự đoán full test set.
**5. Chuẩn bị dữ liệu test, xác định backbone và loại task:**
```Python
dataset_test = DepthDetectDataset(_meta_df, condition, 'sub')
data_loader_test = DataLoader(dataset_test, batch_size=config["test_bs"], ...)
if '1024' in model_name: 
    widths = [128, 256, 512, 1024]
else: 
    widths = [64, 128, 256, 512]
if 'l1' in model_name: 
    model_type = 'regression'
else: 
    model_type = 'classification'
```
- Chọn cấu trúc backbone phù hợp với mô hình (dựa vào tên file checkpoint).
- Dùng đúng loại regression/classification tương ứng.
**6. Load model từ checkpoint**
```Python
model = DepthDetectModule.load_from_checkpoint(
    model_path, condition=condition, widths=widths, model_type=model_type
)
model.eval()
model.zero_grad()
model.to(device)
```
- Load mô hình đã huấn luyện.
- Đưa sang chế độ eval, chuyển lên GPU/CPU tùy cấu hình.
**7. Vòng lặp inference từng batch**
```Python
with torch.no_grad():
    for data in tqdm(data_loader_test, total=len(data_loader_test)):
        images, study_id = data
        images = images.to(device)
        preds = model.forward(images)
        if model_type == 'regression': 
            for k, v in preds.items(): 
                pred_temp[k].append((v[:, -1]*32).to('cpu').detach().numpy())
        else: 
            for k, v in preds.items(): 
                pred_temp[k].append(torch.argmax(v, dim=1).to('cpu').detach().numpy())
        study_id_list.append(study_id.to('cpu').reshape(-1).detach().numpy())
```
- Nếu là regression, lấy giá trị cuối cùng trên trục class, nhân lại với depth chuẩn (`32`), convert về numpy.
- Nếu classification, lấy index lớn nhất (class dự đoán).
- Lưu lại dự đoán và study_id từng batch.
**8. Gộp kết quả từng model**
```Python
for k, v in pred_temp.items(): 
    depth_predict[condition][k].append(np.concatenate(v))
study_id = np.concatenate(study_id_list)
```
- Sau mỗi mô hình, gộp kết quả các batch thành một mảng lớn cho mỗi đốt sống.
**9. Ensemble kết quả nhiều model/checkpoint**
```Python
for k, v in depth_predict[condition].items(): 
    depth_predict[condition][k] = np.median(np.array(depth_predict[condition][k]), axis=0)
depth_predict[condition]['study_id'] = study_id
```
- Sau khi chạy hết tất cả checkpoint, tính **giá trị trung vị (median) trên trục model** cho mỗi đốt sống, mỗi study.
- Kết quả là dự đoán đã được ensemble tối ưu.
**10. Dọn bộ nhớ liên tục với** `**gc.collect()**`
  
---
## **6. Create label coordinate & align**
```Python
def create_label_ins(study_id, depth, level, condition, desc): 
    coor_dict = {'study_id': [], 'series_id': [], 'instance_number': []}
    _meta = meta_df.loc[meta_df.series_description==desc]
    for s, d in zip(study_id, depth): 
        sub_meta = _meta.loc[_meta.study_id==s]
        sub_meta = sub_meta.sort_values('ipp_x', ascending=True).reset_index(drop=True)
        if len(sub_meta) > 32: 
            d = (d/32)*len(sub_meta)
        try: 
            row = sub_meta.iloc[round(d)]
        except: 
            if condition == 'Spinal Canal Stenosis': 
                row = sub_meta.iloc[int(len(sub_meta)//2)]
            elif condition == 'Left Neural Foraminal Narrowing': 
                row = sub_meta.iloc[int(2*(len(sub_meta)//3))]
            elif condition == 'Right Neural Foraminal Narrowing': 
                row = sub_meta.iloc[int(len(sub_meta)//3)]
            print(s)
        coor_dict['study_id'].append(s)
        coor_dict['series_id'].append(row.series_id)
        coor_dict['instance_number'].append(row.instance_number)
    coor_dict['condition'] = condition
    coor_dict['level'] = level.split('_')[-1]
    return pd.DataFrame(coor_dict)
```
Từ một list `study_id` và dự đoán `depth` (tọa độ slice/lát), **tìm lại thông tin DICOM slice tương ứng** (series_id, instance_number, ...) cho từng case.
Dùng để:
- join với label thật trong evaluate
- submit lên hệ thống chấm điểm tự động
- hiển thị hình ảnh lát cắt đúng trên giao diện
  
  
```Python
scs_study_id = depth_predict['scs']['study_id']
scs_coor_list = []
for k, v in depth_predict['scs'].items(): 
    if k != 'study_id': 
        scs_coor_list.append(create_label_ins(scs_study_id, v, k, 'Spinal Canal Stenosis', 'Sagittal T2/STIR'))
nfn_study_id = depth_predict['nfn']['study_id']
nfn_coor_list = []
for k, v in depth_predict['nfn'].items(): 
    if k != 'study_id': 
        if k.split('_')[0] == 'left': 
            condition = 'Left Neural Foraminal Narrowing'
        else: 
            condition = 'Right Neural Foraminal Narrowing'
        nfn_coor_list.append(create_label_ins(nfn_study_id, v, k, condition, 'Sagittal T1'))
scs_coor = pd.concat(scs_coor_list)
nfn_coor = pd.concat(nfn_coor_list)
pred_coor = pd.concat([scs_coor, nfn_coor]).sort_values(['study_id', 'series_id', 'level'])
```
- Lấy kết quả dự đoán từ hai nhóm bệnh: **SCS (Spinal Canal Stenosis)** và **NFN (Neural Foraminal Narrowing)**.
- Duyệt qua từng vị trí (level) của từng nhóm, sử dụng hàm `create_label_ins` để **tìm lại thông tin lát ảnh thật** trong DICOM ứng với dự đoán.
- Gộp toàn bộ kết quả lại thành một bảng duy nhất, sắp xếp chuẩn bị cho các bước xử lý tiếp theo.
  
```Python
del scs_coor, nfn_coor, scs_coor_list, nfn_coor_list, depth_predict
gc.collect()
pred_coor.head()
pred_coor.to_csv('stage1_coor.csv', index=False)
```
- Giải phóng RAM, tránh chiếm dụng không cần thiết (rất quan trọng khi làm việc với dữ liệu y tế lớn).
- `gc.collect()`: ép Python thực hiện “garbage collection” ngay.
- Hiển thị 5 dòng đầu của DataFrame `pred_coor` để kiểm tra lại dữ liệu trước khi lưu (optional, hay dùng trong notebook).
- Ghi toàn bộ DataFrame `pred_coor` ra file CSV (không kèm chỉ số dòng)
---
# II. Second Stage (xy inference)
**Infer xy-coordinate of locations of sagittal t1 & t2**
- **Mục tiêu:** Với mỗi lát ảnh sagittal (T1 hoặc T2), mô hình dự đoán **tọa độ (x, y)** của vị trí cần xác định (ví dụ: vị trí tổn thương, vị trí giữa hai đốt sống,...).
**Ensemble or align (rule base)**
- **Ensemble:** Kết hợp nhiều mô hình hoặc nhiều phép thử lại để lấy kết quả ổn định hơn (trung bình, trung vị, voting...).
- **Align (rule base):** Có thể dùng các quy tắc (rule-based) để điều chỉnh lại vị trí, ví dụ:
    - Nếu T1 và T2 khác biệt nhiều về tọa độ, chọn điểm gần trung tâm nhất.
    - Nếu kết quả nằm ngoài vùng hợp lệ của ảnh, ép vào vùng hợp lệ.
    - Nếu giữa hai lát (slice) dự đoán gần nhau, có thể lấy trung bình, hoặc chọn điểm đã có trong ground-truth.
## 1. C**oordinate prediction dataset**
### Khởi tạo
```Python
class CoorDetectDataset(Dataset):
    def __init__(self, coor, meta, condition, usage='train'):
        if condition == 'scs':
            coor = coor.loc[coor.condition=='Spinal Canal Stenosis']
        elif condition == 'ss':
            coor = coor.loc[(coor.condition=='Left Subarticular Stenosis') | (coor.condition=='Right Subarticular Stenosis')]
        elif condition == 'nfn':
            coor = coor.loc[(coor.condition=='Right Neural Foraminal Narrowing') | (coor.condition=='Left Neural Foraminal Narrowing')]
        \#g_coor = coor.groupby('study_id').count()
        \#if condition == 'scs':
        #    self.id = g_coor.loc[g_coor.series_id==5].reset_index().study_id.unique()
        \#else:
        #    self.id = g_coor.loc[g_coor.series_id==10].reset_index().study_id.unique()
        self.id = coor.study_id.unique()
        self.coor = coor
        self.meta = meta
        self.condition = condition
        self.usage = usage
        if 3637444890 in self.id: 
            self.id.remove(3637444890)
        \#self.id = [2773343225]
        \#self.id = [1782095928]
        self.resize = v2.Resize((384, 384))
        
		def __getitem__(self, index):
        study_id = self.id[index]
        \#print(study_id)
        \#try:
        if self.condition == 'scs':
            volume = self.for_scs(study_id)
        elif self.condition == 'nfn':
            volume = self.for_nfn(study_id)
        if self.condition == 'ss':
            volume = self.for_ss(study_id)
        return volume, torch.tensor(study_id)
```
  
**Mục tiêu chính:**
- **Chọn ra các study phù hợp theo từng loại bệnh học (**`**condition**`**)**.
- **Chuẩn bị sample để huấn luyện hoặc inference:** với mỗi `study_id`, trả về một volume ảnh đã tiền xử lý sẵn, cùng label hoặc thông tin kèm theo.
- Dễ dàng mở rộng/biến đổi theo từng loại task (`scs`, `ss`, `nfn`).
  
**1. Lọc dữ liệu theo condition**
- Lấy đúng các dòng label trong DataFrame `coor` thuộc nhóm bệnh cần train/infer.
**2. Lấy danh sách study hợp lệ**
`self.id = coor.study_id.unique()`
- Tạo mảng gồm tất cả study_id **không trùng lặp** cho bài toán.
**3. Loại bỏ study lỗi hoặc không hợp lệ**
- Tránh lỗi khi train/infer (study này thiếu hoặc hỏng file).
**4. Lưu các thuộc tính cần thiết**
- `self.coor`: label filtered
- `self.meta`: metadata của DICOM (để truy xuất file, series, vị trí slice)
- `self.condition` : condition
- `self.usage` : usage
- `self.resize`: transform resize ảnh về kích thước chuẩn
  
**__getitem__**
- Dựa vào `condition`, gọi đúng hàm load volume (ảnh 3D) đã tiền xử lý:
    - `for_scs(study_id)` cho Spinal Canal Stenosis
    - `for_nfn(study_id)` cho Neural Foraminal Narrowing
    - `for_ss(study_id)` cho Subarticular Stenosis
- Trả về:
    - `volume`: tensor ảnh 3D, đã chuẩn hóa kích thước (thường là `[D, H, W]`)
    - `study_id`: tensor chứa id bệnh nhân, dùng để join kết quả hoặc debug.
### Func for_scs
```Python
    def for_scs(self, study_id):
        meta = self.meta.loc[(self.meta.study_id==study_id) & (self.meta.series_description=='Sagittal T2/STIR')]
        meta = meta.sort_values('ipp_x', ascending=True).reset_index(drop=True)
        \#img = [self.normalize(self.load_dicom(f'/content/train_images/{row.study_id}/{row.series_id}/{row.instance_number}.dcm')) for _, row in meta.iterrows()]
        coor = self.coor.loc[(self.coor.study_id==study_id) & (self.coor.condition=='Spinal Canal Stenosis')]
        meta_list = []
        for _, row in coor.iterrows():
            series_id, instance_number = row.series_id, row.instance_number
            meta_list.append(meta.loc[(meta.series_id==series_id) & (meta.instance_number==instance_number)])
        sub_meta = pd.concat(meta_list)
        idx = meta.loc[meta.ipp_x == sub_meta.ipp_x.median()].index[0]
        \#print(old_idx)
        img_row = meta.iloc[idx]
        before_img_row = meta.iloc[idx-1]
        after_img_row = meta.iloc[idx+1]
        img = self.normalize(self.load_dicom(IMAGE_PATH + f'{img_row.study_id}/{img_row.series_id}/{img_row.instance_number}.dcm'))
        bimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{before_img_row.study_id}/{before_img_row.series_id}/{before_img_row.instance_number}.dcm'))
        aimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{after_img_row.study_id}/{after_img_row.series_id}/{after_img_row.instance_number}.dcm'))
        img = self.resize(torch.tensor(img[None, ...]))
        bimg = self.resize(torch.tensor(bimg[None, ...]))
        aimg = self.resize(torch.tensor(aimg[None, ...]))
        img = torch.cat([bimg, img, aimg]).to(torch.float32)
        return img
```
  
**1. Lọc metadata của study SCS với đúng series**
```Python
meta = self.meta.loc[
    (self.meta.study_id==study_id) &
    (self.meta.series_description=='Sagittal T2/STIR')
]
meta = meta.sort_values('ipp_x', ascending=True).reset_index(drop=True)
```
- Lấy tất cả lát ảnh thuộc đúng study và đúng loại MRI (`Sagittal T2/STIR`).
- Sắp xếp theo trục không gian (ipp_x) để đảm bảo đúng thứ tự lát.
2. **Tìm vị trí các lát liên quan đến tổn thương**
```Python
coor = self.coor.loc[
    (self.coor.study_id==study_id) &
    (self.coor.condition=='Spinal Canal Stenosis')
]
meta_list = []
for _, row in coor.iterrows():
    series_id, instance_number = row.series_id, row.instance_number
    meta_list.append(meta.loc[
        (meta.series_id==series_id) & (meta.instance_number==instance_number)
    ])
sub_meta = pd.concat(meta_list)
```
- Tìm trong bảng label (self.coor) tất cả các instance của study này với condition SCS.
- Với từng label, tìm lại metadata của lát ảnh đó.
- Ghép lại thành bảng con `sub_meta` – tập hợp các lát ảnh liên quan.
3. **Tìm slice trung tâm nhất trong vùng tổn thương**
```Python
idx = meta.loc[meta.ipp_x == sub_meta.ipp_x.median()].index[0]
```
- Chọn tọa độ ipp_x trung vị trong tập các lát liên quan (giữa vùng tổn thương) → `idx` là chỉ số lát trung tâm.
4. **Lấy 3 lát ảnh: trung tâm, trước và sau**
```Python
img_row = meta.iloc[idx]
before_img_row = meta.iloc[idx-1]
after_img_row = meta.iloc[idx+1]
```
- `img_row`: lát trung tâm
- `before_img_row`: lát trước (idx-1)
- `after_img_row`: lát sau (idx+1)
5. **Đọc và chuẩn hóa 3 lát ảnh**
```Python
img = self.normalize(self.load_dicom(IMAGE_PATH + f'{img_row.study_id}/{img_row.series_id}/{img_row.instance_number}.dcm'))
bimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{before_img_row.study_id}/{before_img_row.series_id}/{before_img_row.instance_number}.dcm'))
aimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{after_img_row.study_id}/{after_img_row.series_id}/{after_img_row.instance_number}.dcm'))
```
- Đọc từng file DICOM, chuẩn hóa giá trị pixel (theo hàm bạn định nghĩa).
6. **Resize các lát về kích thước chuẩn**
```Python
img = self.resize(torch.tensor(img[None, ...]))
bimg = self.resize(torch.tensor(bimg[None, ...]))
aimg = self.resize(torch.tensor(aimg[None, ...]))
```
- Đưa ảnh từ numpy → tensor, thêm chiều channel `[1, H, W]`, resize về `[1, 384, 384]`
7. **Ghép 3 lát thành một tensor đầu vào**
```Python
img = torch.cat([bimg, img, aimg]).to(torch.float32)
```
- Output cuối: tensor shape `[3, 384, 384]` (3 channel: lát trước, trung tâm, sau).
- Dùng làm input cho mô hình detect xy (mô hình sẽ nhìn cận cảnh hơn, tăng độ chính xác).
8. **Trả về ảnh** `return img`
```Python
		def for_ss(self, study_id):
        meta = self.meta.loc[(self.meta.study_id==study_id) & (self.meta.series_description=='Axial T2')]
        meta = meta.sort_values('ipp_z', ascending=False).reset_index(drop=True)
        img = [self.normalize(self.load_dicom(f'/content/train_images/{row.study_id}/{row.series_id}/{row.instance_number}.dcm')) for _, row in meta.iterrows()]
        coor = self.coor.loc[(self.coor.study_id==study_id)]
        coor_dict = {}
        for _, row in coor.iterrows():
            series_id, instance_number = row.series_id, row.instance_number
            target_row = meta.loc[(meta.series_id==series_id) & (meta.instance_number==instance_number)]
            idx = target_row.index[0]
            \#print(row.level, idx, idx/len(img))
            \#plt.title(row.level)
            \#plt.imshow(img[idx])
            \#mask = torch.zeros(img[idx].shape)
            \#mask[int(row.y)-10:int((row.y))+10, int(row.x)-10:int((row.x))+10] = 1
            \#plt.imshow(mask, alpha=0.5)
            \#plt.show()
            height, width = img[idx].shape
            z = idx/depth if len(img) < depth else idx/len(img)
            x = row.x/width
            y = row.y/height
            if row.condition == 'Right Subarticular Stenosis':
                coor_dict['right_' + row.level] = torch.tensor([x, y, z]).to(torch.float32)
						else:
                coor_dict['left_' + row.level] = torch.tensor([x, y, z]).to(torch.float32)
        volume = torch.cat([self.resize(torch.tensor(i)[None, ...]).to(torch.float32) for i in img]).contiguous()
        if volume.shape[0] < depth:
            volume = torch.cat([volume, torch.zeros(depth-volume.shape[0], volume.shape[1], volume.shape[2])])
        elif volume.shape[0] > depth:
            volume = torch.nn.functional.interpolate(volume[None, None, ...], (depth, volume.shape[1], volume.shape[2])).squeeze()
        return volume, coor_dict
```
  
  
  
```Python
def for_nfn(self, study_id):
        meta = self.meta.loc[(self.meta.study_id==study_id) & (self.meta.series_description=='Sagittal T1')]
        meta = meta.sort_values('ipp_x', ascending=True).reset_index(drop=True)
        \#img = [self.normalize(self.load_dicom(f'/content/train_images/{row.study_id}/{row.series_id}/{row.instance_number}.dcm')) for _, row in meta.iterrows()]
        coor = self.coor.loc[(self.coor.study_id==study_id)]
        right_meta_list = []
        left_meta_list = []
        for _, row in coor.iterrows():
            series_id, instance_number = row.series_id, row.instance_number
            if row.condition == 'Right Neural Foraminal Narrowing':
                right_meta_list.append(meta.loc[(meta.series_id==series_id) & (meta.instance_number==instance_number)])
            else: 
                left_meta_list.append(meta.loc[(meta.series_id==series_id) & (meta.instance_number==instance_number)])
        right_sub_meta = pd.concat(right_meta_list)
        left_sub_meta = pd.concat(left_meta_list)
        ridx = meta.loc[meta.ipp_x == right_sub_meta.ipp_x.median()].index[0]
        lidx = meta.loc[meta.ipp_x == left_sub_meta.ipp_x.median()].index[0]
        right_img_row = meta.iloc[min(max(ridx, 0), len(meta)-1)]
        \#display(right_img_row)
        right_before_img_row = meta.iloc[min(max(ridx-1, 0), len(meta)-1)]
        rightafter_img_row = meta.iloc[min(max(ridx+1, 0), len(meta)-1)]
        left_img_row = meta.iloc[min(max(lidx, 0), len(meta)-1)]
        left_before_img_row = meta.iloc[min(max(lidx-1, 0), len(meta)-1)]
        leftafter_img_row = meta.iloc[min(max(lidx+1, 0), len(meta)-1)]
        rimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{right_img_row.study_id}/{right_img_row.series_id}/{right_img_row.instance_number}.dcm'))
        rbimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{right_before_img_row.study_id}/{right_before_img_row.series_id}/{right_before_img_row.instance_number}.dcm'))
        raimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{rightafter_img_row.study_id}/{rightafter_img_row.series_id}/{rightafter_img_row.instance_number}.dcm'))
        limg = self.normalize(self.load_dicom(IMAGE_PATH + f'{left_img_row.study_id}/{left_img_row.series_id}/{left_img_row.instance_number}.dcm'))
        lbimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{left_before_img_row.study_id}/{left_before_img_row.series_id}/{left_before_img_row.instance_number}.dcm'))
        laimg = self.normalize(self.load_dicom(IMAGE_PATH + f'{leftafter_img_row.study_id}/{leftafter_img_row.series_id}/{leftafter_img_row.instance_number}.dcm'))
        rimg = torch.cat([self.resize(torch.tensor(i)[None, ...]).to(torch.float32) for i in [rbimg, rimg, raimg]])
        limg = torch.cat([self.resize(torch.tensor(i)[None, ...]).to(torch.float32) for i in [lbimg, limg, laimg]])
        img = torch.stack([limg, rimg]).to(torch.float32).contiguous()
        return img
```
  
  
  
  
```Python
    def normalize(self, x):
        lower, upper = np.percentile(x, (1, 99))
        x = np.clip(x, lower, upper)
        x = x - np.min(x)
        x = x / np.max(x)
        return x
    def __len__(self):
        return len(self.id)
    def load_dicom(self, path):
        dicom = dcm.read_file(path)
        data = dicom.pixel_array
        return data
```
  
  
  
  
---
## 2. C**oordinate prediction models**
```Python
class ConvNextSCSDetect(nn.Module):
    def __init__(self, encoder):
        super().__init__()
        \#self.size = 384
        if encoder == 'convnext': 
            self.encoder = timm.create_model('convnext_base.fb_in22k_ft_in1k_384', in_chans=3, pretrained=False, num_classes=0)
        elif encoder == 'efficientnetv2-l': 
            self.encoder = timm.create_model('tf_efficientnetv2_l.in21k_ft_in1k', in_chans=3, pretrained=False, num_classes=0, drop_rate=0.)
        self.in_features = self.encoder.num_features
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool2d((1,1)),
                                    nn.Flatten(1),
                                    \#nn.LayerNorm(self.in_features)
                                    )
        self.l1 = nn.Linear(self.in_features, 2)
        self.l2 = nn.Linear(self.in_features, 2)
        self.l3 = nn.Linear(self.in_features, 2)
        self.l4 = nn.Linear(self.in_features, 2)
        self.l5 = nn.Linear(self.in_features, 2)
		def forward(self, x, label=None):
        \#for loc, img in x.items():
            \#print(img.shape)
        #    img = self.encoder.forward_features(img)
        #    img = self.flatten(img)
        #    x[loc] = img
        x = self.encoder.forward_features(x)
        x = self.flatten(x)
        l1 = self.l1(x)
        l2 = self.l2(x)
        l3 = self.l3(x)
        l4 = self.l4(x)
        l5 = self.l5(x)
        return {'L1/L2': l1.sigmoid(), 'L2/L3': l2.sigmoid(), 'L3/L4': l3.sigmoid(), 'L4/L5': l4.sigmoid(), 'L5/S1': l5.sigmoid()}
```
  
  
```Python
class ConvNextNFNDetect(nn.Module):
    def __init__(self, encoder):
        super().__init__()
        if encoder == 'convnext': 
            self.encoder = timm.create_model('convnext_base.fb_in22k_ft_in1k_384', in_chans=3, pretrained=False, num_classes=0)
        elif encoder == 'efficientnetv2-l': 
            self.encoder = timm.create_model('tf_efficientnetv2_l.in21k_ft_in1k', in_chans=3, pretrained=False, num_classes=0, drop_rate=0.)
        self.in_features = self.encoder.num_features
        self.flatten = nn.Sequential(nn.AdaptiveAvgPool2d((1,1)),
                                    nn.Flatten(1),
                                    \#nn.LayerNorm(self.in_features)
                                    )
        self.ll1 = nn.Linear(self.in_features, 2)
        self.ll2 = nn.Linear(self.in_features, 2)
        self.ll3 = nn.Linear(self.in_features, 2)
        self.ll4 = nn.Linear(self.in_features, 2)
        self.ll5 = nn.Linear(self.in_features, 2)
        self.rl1 = nn.Linear(self.in_features, 2)
        self.rl2 = nn.Linear(self.in_features, 2)
        self.rl3 = nn.Linear(self.in_features, 2)
        self.rl4 = nn.Linear(self.in_features, 2)
        self.rl5 = nn.Linear(self.in_features, 2)
    
		def forward(self, x, label=None):
        shape = x.shape
        x = x.reshape(shape[0]*shape[1], 3, shape[-2], shape[-1])
        x = self.encoder.forward_features(x)
        x = self.flatten(x)
        x = x.reshape(shape[0], shape[1], -1)
        x_left = x[:, 0, :]
        x_right = x[:, 1, :]
        ll1 = self.ll1(x_left)
        ll2 = self.ll2(x_left)
        ll3 = self.ll3(x_left)
        ll4 = self.ll4(x_left)
        ll5 = self.ll5(x_left)
        rl1 = self.rl1(x_right)
        rl2 = self.rl2(x_right)
        rl3 = self.rl3(x_right)
        rl4 = self.rl4(x_right)
        rl5 = self.rl5(x_right)
        return {'left_L1/L2': ll1.sigmoid(),'left_L2/L3': ll2.sigmoid(),'left_L3/L4': ll3.sigmoid(), 'left_L4/L5': ll4.sigmoid(), 'left_L5/S1': ll5.sigmoid(),
                'right_L1/L2': rl1.sigmoid(), 'right_L2/L3': rl2.sigmoid(), 'right_L3/L4': rl3.sigmoid(), 'right_L4/L5': rl4.sigmoid(), 'right_L5/S1': rl5.sigmoid()}
```
  
---
### 3. C**oordinate detection lightning module**
```Python
class DetectModule(pl.LightningModule):
    def __init__(self, condition, encoder):
        super().__init__()
        self.config = condition
        if condition == 'scs':
            self.model = ConvNextSCSDetect(encoder)
        elif condition == 'nfn':
            self.model = ConvNextNFNDetect(encoder)
        elif  condition == 'ss': 
            pass
        \#self.ema = ExponentialMovingAverage(self.model.parameters(), decay=0.995)
        \#self.ema.to(device)
        \#self.model = torch.optim.swa_utils.AveragedModel(self.model,
        #                                                 multi_avg_fn=torch.optim.swa_utils.get_ema_multi_avg_fn(0.999))
    def forward(self, batch):
        preds = self.model(batch)
        return preds
```
  
---
### **4. Coordinate inference**
```Python
%%time
prefix = ''
import warnings
warnings.filterwarnings("ignore")
coor_predict = {'scs': {
                     'L1/L2':[], 
                     'L2/L3': [], 
                     'L3/L4': [], 
                     'L4/L5': [], 
                     'L5/S1': []
                     }, 
                 'nfn': {
                     'left_L1/L2': [], 
                     'left_L2/L3': [], 
                     'left_L3/L4': [], 
                     'left_L4/L5': [], 
                     'left_L5/S1': [], 
                     'right_L1/L2': [], 
                     'right_L2/L3': [], 
                     'right_L3/L4': [], 
                     'right_L4/L5': [], 
                     'right_L5/S1': [], 
                     }
                    }
model_path_dict = {
  'scs': [
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_4.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_effv2l_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_effv2l_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_effv2l_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_effv2l_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/scs_detect_pre_effv2l_4.ckpt', 
    ], 
    'nfn': [
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_4.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_effv2l_0.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_effv2l_1.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_effv2l_2.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_effv2l_3.ckpt', 
        '/kaggle/input/rsna-spine-final-models/nfn_detect_pre_effv2l_4.ckpt', 
    ]
#############\#COOR DETECT#########################
for condition in ['nfn', 'scs']:
    print(condition)
    model_path_list = model_path_dict[condition]
    for path in model_path_list:
        if 'effv2l' in path.split('/')[-1]: 
            encoder = 'efficientnetv2-l'
        else: 
            encoder = 'convnext'
        _meta_df = meta_df.copy()
        _coor = pred_coor.copy()
        dataset_test = CoorDetectDataset(_coor, _meta_df, condition, 'sub')
        data_loader_test = DataLoader(
            dataset_test,
            batch_size=config["test_bs"],
            shuffle=False,
            num_workers=4,
            pin_memory=False
        )
        print(path, encoder)
        model = DetectModule.load_from_checkpoint(path, condition=condition, encoder=encoder)
        model.eval()
        model.zero_grad()
        model.to(device)
        pred_temp = {}
        for k in coor_predict[condition].keys(): 
            pred_temp[k] = []
        study_id_list = []
        with torch.no_grad():
            for data in tqdm(data_loader_test, total=len(data_loader_test)):
                images, study_id = data
                images = images.to(device)
                preds = model.forward(images)
                \#print(preds)
                for k, v in preds.items(): 
                    pred_temp[k].append(v.to('cpu').detach().numpy())
                study_id_list.append(study_id.to('cpu').reshape(-1).detach().numpy())
        for k, v in pred_temp.items(): 
            coor_predict[condition][k].append(np.concatenate(v))
        del pred_temp
        gc.collect()
        study_id = np.concatenate(study_id_list)
    for k, v in coor_predict[condition].items(): 
        coor_predict[condition][k] = np.mean(np.array(coor_predict[condition][k]), axis=0)
    coor_predict[condition]['study_id'] = study_id
    del study_id
    gc.collect()
```
  
```Python
pred_coor.head(1)
```
||study_id|series_id|instance_number|condition|level|
|---|---|---|---|---|---|
|0|44036939|2828203845|9|Left Neural Foraminal Narrowing|L1/L2|
```Python
%%time
def create_label_coor(study_id, coor_df, coor, level, condition, desc): 
    _meta = meta_df.loc[meta_df.series_description==desc].copy()
    _coor = coor_df.loc[coor_df.condition == condition]
    _coor_df = {'study_id': [], 'series_id': [], 'x': [], 'y': []}
    for s, c in zip(study_id, coor): 
        sub_meta = _meta.loc[_meta.study_id == s]
        sub_coor = _coor.loc[(_coor.study_id==s) & (_coor.level==level.split('_')[-1])].squeeze(axis=0)
        \#display(sub_coor)
        meta_row = sub_meta.loc[(sub_meta.instance_number==sub_coor.instance_number) & (sub_meta.series_id==sub_coor.series_id)].squeeze(axis=0)
        x = round(meta_row.width * c[0])
        y = round(meta_row.height * c[1])
        _coor_df['study_id'].append(s)
        _coor_df['series_id'].append(sub_coor.series_id)
        _coor_df['x'].append(x)
        _coor_df['y'].append(y)
    _coor_df['level'] = level.split('_')[-1]
    _coor_df['condition'] = condition
    del _meta, _coor, sub_meta, sub_coor, meta_row
    return pd.DataFrame(_coor_df)
```
```Python
scs_study_id = coor_predict['scs']['study_id']
scs_coor_list = []
for k, v in coor_predict['scs'].items(): 
    if k != 'study_id': 
        scs_coor_list.append(create_label_coor(scs_study_id, pred_coor, v, k, 'Spinal Canal Stenosis', 'Sagittal T2/STIR'))
nfn_study_id = coor_predict['nfn']['study_id']
nfn_coor_list = []
for k, v in coor_predict['nfn'].items(): 
    if k != 'study_id': 
        if k.split('_')[0] == 'left': 
            condition = 'Left Neural Foraminal Narrowing'
        else: 
            condition = 'Right Neural Foraminal Narrowing'
        nfn_coor_list.append(create_label_coor(nfn_study_id, pred_coor, v, k, condition, 'Sagittal T1'))
scs_coor = pd.concat(scs_coor_list)
nfn_coor = pd.concat(nfn_coor_list)
_pred_coor = pd.concat([scs_coor, nfn_coor]).sort_values(['study_id', 'series_id', 'level'])
```
```Python
pred_coor_stage2 = pd.merge(pred_coor, _pred_coor, on=['study_id', 'series_id', 'level', 'condition'], how='inner')
```
```Python
display(pred_coor_stage2.head())
pred_coor_stage2.to_csv('stage2_coor.csv', index=False)
```
||study_id|series_id|instance_number|condition|level|x|y|
|---|---|---|---|---|---|---|---|
|0|44036939|2828203845|9|Left Neural Foraminal Narrowing|L1/L2|387|201|
|1|44036939|2828203845|18|Right Neural Foraminal Narrowing|L1/L2|384|179|
|2|44036939|2828203845|9|Left Neural Foraminal Narrowing|L2/L3|348|267|
|3|44036939|2828203845|18|Right Neural Foraminal Narrowing|L2/L3|350|243|
|4|44036939|2828203845|9|Left Neural Foraminal Narrowing|L3/L4|311|326|
# **III. Third Stage (calc. location of axial t2)**
- calcurate depth of axial t2 for each location roughly, using xyz-coordinate (refered to @hengck's transformation from sagittal t2 to axial t2)
- roughly separate each locations
- infer instance number
- infer xy-coordinate
## **1. Calculate axial slice**
```Python
# project 2d to 3d
def project_to_3d(row):
    sx, sy, sz = row.ipp_x, row.ipp_y, row.ipp_z
    x, y = row.x, row.y
    o0, o1, o2, o3, o4, o5 = row.iop
    delx, dely = row.ps_x, row.ps_y
    xx = o0 * delx * x + o3 * dely * y + sx
    yy = o1 * delx * x + o4 * dely * y + sy
    zz = o2 * delx * x + o5 * dely * y + sz
    return xx,yy,zz
```
```Python
def sag_to_ax(sub_coor, sub_meta): 
    point = sub_coor[['ipp_x', 'ipp_y', 'ipp_z']].values \#2d
    level_list = sub_coor.level.tolist()    
    # here we project 2d to 3d
    center=[] 
    for _, row in sub_coor.iterrows():
        xx,yy,zz = project_to_3d(row)
        center.append([xx,yy,zz])
    center = np.array(center) \#3d
    # == 2. we get closest axial slices to the CSC points =================
    \#df = valid_data[0].axial_t2[0].df
    orientation = np.array(sub_meta.iop.values.tolist())
    position= np.array(sub_meta[['ipp_x', 'ipp_y', 'ipp_z']].values.tolist())
    ox = orientation[:, :3]
    oy = orientation[:, 3:]
    oz = np.cross(ox,oy)
    t = center.reshape(-1,1,3) - position.reshape(1,-1,3)
    dis = (oz.reshape(1,-1,3) * t).sum(-1)  # np.dot(point-s,oz)
    dis = np.fabs(dis)
    closest = dis.argmin(-1)
    closest_df = sub_meta.iloc[closest]
    closest_df['level'] = level_list#['L1/L2', 'L2/L3', 'L3/L4', 'L4/L5', 'L5/S1']
    closest_df['x'] = 0
    closest_df['y'] = 0
    \#closest_df = pd.concat([closest_df, closest_df])
    \#closest_df['condition'] = ['Left Subarticular Stenosis']*5 + ['Right Subarticular Stenosis']*5
    return closest_df[['study_id', 'series_id', 'instance_number', 'level']]
```
```Python
# sagittal t2 => axial t2
scs_coor = pred_coor_stage2.loc[pred_coor_stage2.condition=='Spinal Canal Stenosis'].copy()
scs_coor = scs_coor.merge(meta_df, on=['study_id', 'series_id', 'instance_number'], how='left')
study_id = scs_coor.study_id.unique()
ax_meta  = meta_df.loc[(meta_df.series_description=='Axial T2')]
closest_ax_list = []
for s in tqdm(study_id, total=len(study_id)): 
    sub_coor = scs_coor.loc[scs_coor.study_id==s]
    sub_meta = ax_meta.loc[ax_meta.study_id==s]
    closest_ax_list.append(sag_to_ax(sub_coor, sub_meta)) 
```
```Python
closest_ax = pd.concat(closest_ax_list)
closest_ax.head()
```
||study_id|series_id|instance_number|level|
|---|---|---|---|---|
|16|44036939|3481971518|17||
|22|44036939|3481971518|23||
|27|44036939|3481971518|28||
|33|44036939|3481971518|34||
|38|44036939|3481971518|39||
### **2. Subarticular stenosis coordinate prediction dataset**
```Python
class SSDetectDataset(Dataset):
    def __init__(self, ax, usage='train'):
        self.ax = ax
        self.id = ax.study_id.unique()
        self.usage = usage
        self.id = list(set(self.id) - set([3637444890]))
        \#self.id = [2773343225]
        \#self.id = [1782095928]
        self.resize = v2.Resize((384, 384))
        
    def __getitem__(self, index):
        study_id = self.id[index]
        volume = self.for_ss(study_id)
        return volume, torch.tensor(study_id)
```
```Python
   def for_ss(self, study_id):
        ax = self.ax.loc[self.ax.study_id==study_id]
        img_dict = {}
        for _, row in ax.iterrows():
            series_id, instance_number = row.series_id, row.instance_number
            img = self.load_dicom(IMAGE_PATH + f'{study_id}/{series_id}/{instance_number}.dcm').astype(np.float32)
            img = self.resize(torch.tensor(img)[None, ...])
            img = self.normalize(img)
            img_dict[row.level] = img
        img_list = []
        for k in ['L1/L2', 'L2/L3', 'L3/L4', 'L4/L5', 'L5/S1']: 
            img_list.append(img_dict[k])
        volume = torch.stack(img_list).contiguous()
        return volume
    def normalize(self, x):
        upper = torch.quantile(x, torch.tensor([0.99]))
        lower = torch.quantile(x, torch.tensor([0.01]))
        x = torch.clip(x, lower, upper)
        x = x - torch.min(x)
        x = x / (torch.max(x)+1e-6)
        return x
```