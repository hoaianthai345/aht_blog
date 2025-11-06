### Bộ cấu hình nên chạy (ưu tiên theo thứ tự)

- Ước lượng VRAM: PhoBERT 256/384/512 tương ứng batch ~32/16/8 (12–24 GB VRAM). BiLSTM nhẹ hơn, batch có thể 64–128.

1) BiLSTM + FastText (baseline)
- Model. Type: bilstm
- Model. Embedding_type: fasttext
- Data. Max_seq_length: 512
- Model. Lstm_units: 128
- Model. Dropout: 0.3
- Training: { batch_size: 64, lr: 1 e-4, optimizer: adam }
- Imbalance. Strategy: baseline
- Output. Run_name: exp_bilstm_ft_512_baseline

2) BiLSTM + FastText + class_weight
- Như (1) nhưng imbalance. Strategy: class_weight
- Run_name: exp_bilstm_ft_512_classweight

3) BiLSTM + FastText + focal_loss
- Như (1) nhưng imbalance. Strategy: focal_loss
- Run_name: exp_bilstm_ft_512_focal

4) BiLSTM + FastText + oversample
- Như (1) nhưng imbalance. Strategy: oversample
- Run_name: exp_bilstm_ft_512_ros

5) BiLSTM mạnh hơn (ẩn vị lớn, dropout cao)
- Model. Lstm_units: 256
- Model. Dropout: 0.5
- Các mục còn lại như (1)
- Run_name: exp_bilstm_ft_512_lstm 256_do 50

6) PhoBERT (baseline, 256)
- Model. Type: phobert
- Data. Max_seq_length: 256
- Model. Phobert_name: vinai/phobert-base
- Training: { batch_size: 32, lr: 2 e-5, optimizer: adamw }
- Imbalance. Strategy: baseline
- Run_name: exp_phobert_256_baseline

7) PhoBERT (384)
- Như (6), data. Max_seq_length: 384, batch_size: 16
- Run_name: exp_phobert_384_baseline

8) PhoBERT (512)
- Như (6), data. Max_seq_length: 512, batch_size: 8
- Run_name: exp_phobert_512_baseline

9) PhoBERT + class_weight (256)
- Như (6) nhưng imbalance. Strategy: class_weight
- Run_name: exp_phobert_256_classweight

10) PhoBERT + focal_loss (256)
- Như (6) nhưng imbalance. Strategy: focal_loss
- Run_name: exp_phobert_256_focal

### Lưu ý quan trọng trước khi chạy
- Device: đặt training. Device = "cuda".
- Đường dẫn embedding:
  - Fasttext: data/Embeder/cc. Vi. 300. Vec (đúng với repo).
  - word 2 vec: nếu dùng file `.txt` hiện có, đổi config `model.embedding_path.word2vec` sang `.txt` (không phải `.bin`).
- Batch size/VRAM: nếu OOM, giảm batch_size hoặc max_seq_length (đặc biệt PhoBERT).
- Run_name: đặt khác nhau để tách log/ckpt.

### Cách chạy
- Mỗi lần sửa `config.yaml` theo cấu hình trên rồi chạy:
- `python main.py --config config.yaml`