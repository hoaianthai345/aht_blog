---
Author: HoàiAn Thái
Status: Not started
Tag:
  - Self-Study
---

Để **set up môi trường phát triển Python trên macOS** (đặc biệt cho dự án dùng HuggingFace + LangChain + xử lý tài liệu), cần làm theo quy trình chuẩn, tránh lỗi dependency hoặc conflict với hệ thống.
**Bước 1: Cài đặt Homebrew & Python**
```Shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install python
```
Kiểm tra:
```Shell
python3 --version
pip3 --version
```

> ⚠️ Dùng python3 và pip3, không override python mặc định của macOS.
---
**Bước 2: Tạo virtual environment**
```Shell
python3 -m venv .venv
source .venv/bin/activate
```
Kiểm tra:
```Shell
which python
# → nên trả về .../project_folder/.venv/bin/python
```
**Bước 3: Cài đặt dependencies**
```Shell
pip install --upgrade pip
pip install -r requirements.txt
```
**Bước 5: Xử lý lỗi macOS nếu có**
Một số lỗi thường gặp:
🔻 `bitsandbytes` không hỗ trợ macOS
- Thư viện này yêu cầu GPU NVIDIA CUDA, thường không dùng được trên Mac M1/M2/M3 → **skip hoặc thay thế bằng mô hình nhẹ hơn**.
- Nếu bạn không cần 4-bit quantization, **có thể bỏ dòng này ra khỏi** `**requirements.txt**`.
🔻 Lỗi khi cài `transformers` hoặc `torch`
- Thêm:
```Shell
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
```

> Vì bản mặc định đôi khi đòi CUDA mà macOS không có.
**Bước 6: Test đơn giản**
```Python
from transformers import pipeline
qa = pipeline("question-answering")
print(qa({
    "question": "What is LangChain?",
    "context": "LangChain is a framework to build LLM-powered applications."
}))
```