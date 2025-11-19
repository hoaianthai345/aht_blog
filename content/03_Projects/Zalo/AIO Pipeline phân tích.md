
# LoRA fine-tuning
Dùng **một Bash script hoàn chỉnh để fine-tune mô hình đa phương thức InternVL (Vintern-1 B-v 3.5)**.  Dựa theo repo: [internvl2_1b_qwen2_0_5b_dynamic_res_2nd_finetune_lora.sh](internvl_chat/shell/internvl2.0/2nd_finetune/internvl2_1b_qwen2_0_5b_dynamic_res_2nd_finetune_lora.sh)
***
1. Dòng mở đầu

```bash
set -x
```

* Bật chế độ **debug shell**.
* Khi chạy, mọi lệnh sẽ được in ra màn hình **trước khi thực thi** (giúp bạn biết chính xác script đang chạy gì).
* * *

2. Thiết lập biến huấn luyện

```bash
GPUS=${GPUS:-1}
BATCH_SIZE=${BATCH_SIZE:-1}
PER_DEVICE_BATCH_SIZE=${PER_DEVICE_BATCH_SIZE:-1}
GRADIENT_ACC=$((BATCH_SIZE / PER_DEVICE_BATCH_SIZE / GPUS))
```

* `GPUS=${GPUS:-1}`: nếu chưa định nghĩa `GPUS`, gán mặc định là **1 GPU**.
* `BATCH_SIZE=${BATCH_SIZE:-1}`: batch tổng (toàn hệ thống).
    
* `PER_DEVICE_BATCH_SIZE=${PER_DEVICE_BATCH_SIZE:-1}`: batch trên mỗi GPU.
    
* `GRADIENT_ACC=$((...))`: tính **gradient accumulation steps** theo công thức:
    

$$\text{GRADIENT\_ACC} = \frac{\text{BATCH\_SIZE}}{\text{PER\_DEVICE\_BATCH\_SIZE} \times \text{GPUS}}$$

→ giúp mô phỏng batch lớn hơn dù GPU nhỏ (tích lũy nhiều bước mới cập nhật).

* * *

3. Cấu hình môi trường

```bash
export PYTHONPATH="${PYTHONPATH}:$(pwd)"
export MASTER_PORT=34229
export TF_CPP_MIN_LOG_LEVEL=3
export LAUNCHER=pytorch
```

* `PYTHONPATH`: thêm thư mục hiện tại (`pwd`) vào Python path, để Python import được module nội bộ (vd: `internvl/...`).
    
* `MASTER_PORT=34229`: cổng kết nối cho quá trình distributed (torchrun).
    
* `TF_CPP_MIN_LOG_LEVEL=3`: tắt log cảnh báo của TensorFlow (nếu có import).
    
* `LAUNCHER=pytorch`: đặt tên launcher để dễ kiểm tra khi log.
    

* * *

 4. Đặt thư mục output

```bash
OUTPUT_DIR='/mnt/model-weights-vintern-1b-v3-5-zaic-track2'
```

Nơi lưu checkpoint và log huấn luyện.

* * *

 5. Tạo thư mục nếu chưa có

```bash
if [ ! -d "$OUTPUT_DIR" ]; then
  mkdir -p "$OUTPUT_DIR"
fi
```

Kiểm tra nếu chưa tồn tại → tạo thư mục rỗng.

* * *

6. Ghi chú cấu hình

```bash
# number of gpus: 1
# batch size per gpu: 4
# gradient accumulation steps: 2
# Total batch size: 16
# Epoch: 1
```

Comment minh họa (không thực thi), để người đọc biết thông số training dự kiến.

* * *

 7. Lệnh huấn luyện chính

```bash
Torchrun \
  --nnodes=1 \
  --node_rank=0 \
  --master_addr=127.0.0.1 \
  --nproc_per_node=${GPUS} \
  --master_port=${MASTER_PORT} \
  Internvl/train/internvl_chat_finetune. Py \
```

Chạy distributed training bằng **PyTorch launcher (`torchrun`)**.

| Option | Ý nghĩa |
| --- | --- |
| `--nnodes=1` | Chỉ chạy trên 1 máy (node). |
| `--node_rank=0` | Node hiện tại là node đầu tiên. |
| `--master_addr=127.0.0.1` | Địa chỉ master (localhost). |
| `--nproc_per_node=${GPUS}` | Số process spawn trên mỗi node (tức số GPU). |
| `--master_port=${MASTER_PORT}` | Cổng đồng bộ tiến trình. |
| `internvl/train/internvl_chat_finetune. Py` | Script Python chính thực hiện fine-tuning. |

* * *

 8. Các tham số truyền vào mô hình

Phần này là **toàn bộ hyperparameter và cấu hình train**.

 a. Cấu hình model & dataset

```bash
--model_name_or_path "/mnt/src/Vintern/pretrained/Vintern-1 B-v 3_5" \
--conv_style "Hermes-2" \
--output_dir ${OUTPUT_DIR} \
--meta_path "/mnt/zaic-data-merged-train/custom_ormatted_zalo_merged_dataset. Json" \
--overwrite_output_dir True \
```

* `model_name_or_path`: đường dẫn đến pretrained model.
    
* `conv_style`: định dạng conversation (tương tác kiểu “Hermes-2”).
    
* `output_dir`: nơi lưu model sau khi train.
    
* `meta_path`: file JSON chứa dữ liệu train (ảnh, câu hỏi, câu trả lời).
    
* `overwrite_output_dir True`: cho phép ghi đè thư mục nếu đã tồn tại.
    

* * *

 b. Cấu hình xử lý ảnh

```bash
--force_image_size 448 \
--max_dynamic_patch 6 \
--down_sample_ratio 0.5 \
--drop_path_rate 0.0 \
```

* Resize ảnh về 448×448.
    
* `max_dynamic_patch 6`: cho phép chia ảnh thành tối đa 6 patch.
    
* `down_sample_ratio 0.5`: giảm độ phân giải khi cần.
    
* `drop_path_rate`: tỷ lệ dropout đường dẫn (0 → không dùng).
    

* * *

 c. Đóng băng (freeze) mô hình

```bash
--freeze_llm True \
--freeze_mlp True \
--freeze_backbone True \
--use_llm_lora 16 \
--vision_select_layer -1 \
```

* **Freeze** toàn bộ phần LLM, MLP, backbone → chỉ train các adapter LoRA.
    
* `use_llm_lora 16`: LoRA rank = 16 (số chiều low-rank).
    
* `vision_select_layer -1`: dùng layer cuối của encoder hình ảnh.
    

* * *

 d. Cấu hình huấn luyện

```bash
--dataloader_num_workers 4 \
--bf 16 True \
--num_train_epochs 5 \
--per_device_train_batch_size ${PER_DEVICE_BATCH_SIZE} \
--gradient_accumulation_steps ${GRADIENT_ACC} \
```

* 4 worker cho dataloader.
    
* `bf 16 True`: dùng BFloat 16 (nếu GPU hỗ trợ).
    
* Train 5 epoch.
    
* Batch mỗi GPU = giá trị biến `PER_DEVICE_BATCH_SIZE`.
    
* Số bước tích lũy gradient = `GRADIENT_ACC`.
    

* * *

 e. Chiến lược save và eval

```bash
--evaluation_strategy "no" \
--save_strategy "epoch" \
--save_total_limit 5 \
```

* Không chạy evaluation.
    
* Mỗi epoch lưu 1 checkpoint.
    
* Giữ tối đa 5 checkpoint (xóa cái cũ nhất).
    

* * *

 f. Hyperparameter optimizer

```bash
--learning_rate 4 e-5 \
--weight_decay 0.01 \
--warmup_ratio 0.03 \
--lr_scheduler_type "cosine" \
```

* LR = 4 e-5
    
* Decay weight 0.01
    
* Warmup 3% số step đầu.
    
* Dùng scheduler dạng cosine decay.
    

* * *

 g. Logging và giới hạn

```bash
--logging_steps 10 \
--max_seq_length 700 \
--do_train True \
--grad_checkpoint True \
--group_by_length True \
--dynamic_image_size True \
--use_thumbnail True \
--ps_version 'v 2' \
--report_to "tensorboard" \
```

* Log mỗi 10 step.
    
* Giới hạn chuỗi tối đa 700 token.
    
* Bật train.
    
* Dùng **gradient checkpointing** để giảm VRAM.
    
* `group_by_length`: gom các sample có độ dài gần nhau để tối ưu batching.
    
* `dynamic_image_size`: resize ảnh động theo batch.
    
* `use_thumbnail`: dùng thumbnail để load nhanh hơn.
    
* `ps_version 'v 2'`: version của pipeline sampler.
    
* `report_to "tensorboard"`: gửi log sang TensorBoard.
    

* * *

 9. Ghi log ra file và hiển thị cùng lúc

```bash
2>&1 | tee -a "${OUTPUT_DIR}/training_log. Txt"
```

* `2>&1`: gộp stderr và stdout → không mất thông báo lỗi.
    
* `tee -a`: vừa **in ra màn hình**, vừa **ghi thêm vào file log** `training_log. Txt`.
    
* Giúp theo dõi quá trình huấn luyện và lưu lại lịch sử.