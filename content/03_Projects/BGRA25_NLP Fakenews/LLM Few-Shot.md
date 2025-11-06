---
Author: HoàiAn Thái
Date: Invalid date
Status: Not started
tags:
  - Research
  - NLP
---
> [!important] **Table of Content**
> 
> ---
> 
> - [[#Tham khảo]]
>     - [[#Các thư viện cần thiết]]
> 
>   

> [!important]
> 
> # Tham khảo
  
  
  
### **Các thư viện cần thiết**
  
  
  
```Plain
from transformers import AutoModelForCausalLM, AutoTokenizer
model_id = "ngoan/Llama-2-7b-vietnamese-20k" 
tokenizer = AutoTokenizer.from_pretrained(model_id, use_fast=True)
tokenizer.pad_token = tokenizer.eos_token
```
Import 2 class từ thư viện **transformers**:
- `AutoModelForCausalLM`: Tải mô hình language model dạng “causal” (phù hợp cho sinh văn bản, như GPT, Llama).
- `AutoTokenizer`: Tải tokenizer (bộ chuyển đổi text → token id, ngược lại).
- Tải **tokenizer** đi kèm mô hình từ HuggingFace Hub, dùng bản “fast” (cài sẵn với Rust, nhanh hơn bản cũ).
**Thiết lập token PAD bằng token kết thúc câu (EOS)**.
- Llama (và hầu hết mô hình causal) không có pad token mặc định.
- Dùng `pad_token = eos_token` là cách phổ biến, để khi cần padding (ví dụ batch infer, fine-tune) sẽ không gây lỗi.
  
---
```Python
import torch
import gc
import pandas as pd
from tqdm import tqdm
from datetime import datetime
from transformers import AutoTokenizer, AutoModelForCausalLM

def predict_labels_on_dataset(
    texts: list,
    model,
    tokenizer,
    fewshot_examples=None,
    max_new_tokens=20,
    batch_size=4,
    device="cuda" if torch.cuda.is_available() else "cpu",
    save_csv=True,
    csv_path=None
):
    model.eval()
    prompts = []
    results = []
    clean_generated = []
    for idx, text in enumerate(tqdm(texts, desc="Predicting")):
        prompt = build_prompt(text, fewshot_examples=fewshot_examples)
        prompts.append(prompt)
        inputs = None
        output = None
        try:
            inputs = tokenizer(prompt, return_tensors="pt", truncation=True, max_length=1024).to(device)
            with torch.no_grad():
                output = model.generate(
                    **inputs,
                    max_new_tokens=max_new_tokens,
                    do_sample=False,
                    temperature=0.0,
                    pad_token_id=tokenizer.eos_token_id
                )
            # Toàn bộ output
            decoded = tokenizer.decode(output[0], skip_special_tokens=True)
            results.append(decoded)
            # Chỉ phần sinh thêm (loại bỏ prompt)
            input_ids = inputs["input_ids"][0]
            output_ids = output[0]
            generated_ids = output_ids[len(input_ids):]
            generated_text = tokenizer.decode(generated_ids, skip_special_tokens=True)
            clean_generated.append(generated_text.strip())
            print(f"[{idx}] → {generated_text.strip()}")
        except torch.cuda.OutOfMemoryError:
            print(f"⚠️ OOM on sample {idx}: {text[:30]}... Skipping.")
            torch.cuda.empty_cache()
            results.append("")
            clean_generated.append("")
        finally:
            if inputs is not None:
                del inputs
            if output is not None:
                del output
            torch.cuda.empty_cache()
            gc.collect()
    # Lưu CSV nếu yêu cầu
    if save_csv:
        df = pd.DataFrame({
            "text": texts,
            "prompt": prompts,
            "llm_output": results,
            "llm_generated_only": clean_generated
        })
        if csv_path is None:
            now = datetime.now().strftime("%Y%m%d_%H%M%S")
            csv_path = f"llm_predictions_{now}.csv"
        df.to_csv(csv_path, index=False)
        print(f"✅ Đã lưu kết quả vào: {csv_path}")
    return results
```
- `torch`, `gc`: xử lý tính toán và dọn dẹp bộ nhớ.
- `pandas`: tạo và lưu kết quả dưới dạng bảng CSV.
- `tqdm`: tạo thanh tiến trình hiển thị tiến độ xử lý.
- `datetime`: tạo timestamp cho tên file.
- `transformers`: dùng để load tokenizer và model LLM.
  
**Các tham số:**
- `texts`: danh sách văn bản đầu vào.
- `model`, `tokenizer`: LLM đã được load từ HuggingFace.
- `fewshot_examples`: nếu có, dùng thêm ví dụ mẫu để gợi ý mô hình.
- `max_new_tokens`: số token tối đa mô hình sinh ra.
- `device`: dùng GPU nếu có, nếu không thì CPU.
- `save_csv`: có lưu kết quả ra CSV không.
- `csv_path`: đường dẫn lưu kết quả (mặc định là theo thời gian).
  
**Các biến**
```Python
    model.eval()
    prompts = []
    results = []
    clean_generated = []
```
- Đưa model vào chế độ suy luận (`eval()`).
- Tạo list để lưu prompt đã dùng, kết quả toàn bộ, và chỉ phần model sinh thêm.
**Vòng lặp dự đoán**
```Python
    for idx, text in enumerate(tqdm(texts, desc="Predicting")):
        prompt = build_prompt(text, fewshot_examples=fewshot_examples)
```
- Gọi `build_prompt()` để tạo prompt từ text (có thể thêm ví dụ nếu few-shot).
- Sau đó:
    - `tokenizer(prompt, return_tensors="pt", truncation=True, max_length=1024).to(device)`: chuyển prompt thành tensor token (trên GPU/CPU).
    - `model.generate(...)`: sinh văn bản theo prompt.
  
**Xử lý kết quả:**
```Python
decoded = tokenizer.decode(output[0], skip_special_tokens=True)
results.append(decoded)
# Lấy phần sinh thêm
input_ids = inputs["input_ids"][0]
output_ids = output[0]
generated_ids = output_ids[len(input_ids):]
generated_text = tokenizer.decode(generated_ids, skip_special_tokens=True)
clean_generated.append(generated_text.strip())
```