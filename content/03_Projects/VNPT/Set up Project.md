# Database Docker

Lỗi tại:

```
cd infra
docker compose up -d        
```

1. Cảnh báo: `version` trong docker-compose. Yml đã lỗi thời
```
the attribute `version` is obsolete, it will be ignored
```
Từ Docker Compose v 2 trở lên, **không cần khai báo `version:`** trong file `docker-compose.yml`. Nếu còn dòng như:
```yaml
version: "3.8"
```
thì Docker sẽ bỏ qua và cảnh báo.  
**Không ảnh hưởng tới việc chạy**, nhưng nên xoá cho sạch và tránh hiểu lầm.

* * *
2. Lỗi chính: Không kết nối được với Docker daemon

```
unable to get image 'infra-postgres':
Cannot connect to the Docker daemon at unix:///Users/anhoaithai/.docker/run/docker.sock.
Is the docker daemon running?
```

Docker CLI không tìm thấy Docker daemon (engine).  
Điều này xảy ra khi:

1. **Docker Desktop chưa chạy**
    
2. Docker daemon đang tắt hoặc crash
    
3. Quyền truy cập file docker. Sock bị lỗi
    
4. Bạn đang chạy trong môi trường shell chưa được Docker Desktop export PATH/socket
    

**Về bản chất:**  
`docker compose` chỉ là client. Nó cần kết nối tới daemon qua socket:

```bash
/Users/anhoaithai/.docker/run/docker.sock
```

Nhưng socket này không tồn tại → daemon **chưa chạy**.

# Setup Virtual env

```
# Di chuyển vào thư mục backend
cd /Users/anhoaithai/Documents/AHT/1. PROJECTS/AIO25/3. Project/VNPT/vnpt_ai_hackathon_meetmate/backend

# Tạo virtual environment (nếu chưa có)
python -m venv venv

# Kích hoạt venv (macOS/Linux)
source venv/bin/activate

# Cài đặt dependencies
pip install -r ../requirements.txt

# Set PYTHONPATH
export PYTHONPATH="."

# Chạy backend
uvicorn app.main:app --reload --port 8000
```

Lỗi khi chạy 
```
uvicorn app.main:app --reload --port 8000
```

```
PydanticImportError: `BaseSettings` has been moved to the `pydantic-settings` package.
```

Nguyên nhân:
- Đang dùng Pydantic v2 (2.x)
- Trong Pydantic v2, BaseSettings đã được tách ra thành package riêng là pydantic-settings

- Code hiện tại đang import theo cách của Pydantic v1 (cũ)
Cách sửa đúng chuẩn cho Pydantic v2
Bước 1: Cài package mới
`pip install pydantic-settings`
Bước 2: Thay đổi import trong code

**Sai:**
`from pydantic import BaseSettings`

**Đúng (v2):**

`from pydantic_settings import BaseSettings`

## Pydantic
**Pydantic** là một thư viện Python dùng để:

- Định nghĩa **data models** bằng class Python
    
- **Validate dữ liệu** tự động
    
- **Parse dữ liệu** từ JSON, dict, request body
    
- Bảo đảm kiểu dữ liệu (**type safety**) thông qua Python type hints
    

Điểm mạnh nhất của Pydantic:  
**Dữ liệu được ép kiểu (type coercion) và validate cực mạnh nhưng vẫn rất nhanh.**

### 2. Pydantic hoạt động như thế nào?

Bạn định nghĩa một model:

`from pydantic import BaseModel  class User(BaseModel):     id: int     age: int     name: str`

Khi bạn truyền dữ liệu vào:

`u = User(id="123", age="20", name=123)`

Pydantic sẽ:

1. **Parse** chuỗi "123" thành số 123
    
2. **Parse** "20" thành số 20
    
3. **Parse** 123 thành chuỗi "123"
    

Kết quả:

`User(id=123, age=20, name='123')`

→ Đây là lý do Pydantic cực kỳ tiện cho **FastAPI**, nơi dữ liệu thô từ request thường sai kiểu.

---

### 3. Tại sao FastAPI chọn Pydantic?

Vì Pydantic hỗ trợ:

- Sinh **OpenAPI schema** tự động
    
- Validate dữ liệu request/response
    
- Parse JSON nhanh (sử dụng Rust hoặc C khi có thể)
    
- Tích hợp mạnh mẽ với Python typing
    

Ví dụ FastAPI:

`@app.post("/users") def create_user(user: User):     return user`

FastAPI tự validate request body theo model `User`.