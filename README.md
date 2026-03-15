# 9router Docker Deployment (OpenClaw-ready)

Chạy **9router** bằng Docker Compose, tối ưu để giao tiếp mượt với **OpenClaw** trong cùng môi trường Docker.

## 1) Quick start

```bash
git clone https://github.com/nguyenduysphk3407051993/9router-docker-deployment.git
cd 9router-docker-deployment
cp .env.example .env
# tạo secret/password theo mục 3 bên dưới, rồi cập nhật vào file .env

docker compose up -d --build
```

Kiểm tra:

```bash
docker compose ps
curl http://127.0.0.1:20128/v1/models
```

Dashboard: `http://127.0.0.1:20128/dashboard`
API: `http://127.0.0.1:20128/v1`

---

## 2) Kết nối OpenClaw <-> 9router trong Docker network chung

Compose này tạo network dùng chung tên mặc định: `ai_stack`.
Service `9router` có hostname nội bộ: `9router`.

### Trường hợp OpenClaw chạy bằng Docker Compose khác

Đảm bảo OpenClaw container join cùng network:

```bash
docker network connect ai_stack <openclaw_container_name>
```

Sau đó cấu hình OpenClaw dùng endpoint nội bộ:

```bash
OPENAI_BASE_URL=http://9router:20128/v1
OPENAI_API_KEY=<api-key-tao-trong-dashboard-9router>
```

> Dùng `9router` (hostname nội bộ) giúp ổn định hơn `localhost` trong cross-container traffic.

---

## 3) Cấu hình quan trọng trong `.env`

- `JWT_SECRET`, `INITIAL_PASSWORD`, `API_KEY_SECRET`, `MACHINE_ID_SALT`: **bắt buộc đổi** trước khi chạy thật.
- `REQUIRE_API_KEY=true`: nên bật nếu có truy cập từ môi trường ngoài.
- `BASE_URL=http://9router:20128`: chuẩn cho internal jobs của 9router.
- `NEXT_PUBLIC_BASE_URL=http://127.0.0.1:20128`: để truy cập từ host/browser local.
- `DOCKER_SHARED_NETWORK=ai_stack`: network dùng chung với OpenClaw.

### 3.1) Ý nghĩa 4 biến secret/password và vì sao phải đổi

- `JWT_SECRET`  
  Dùng để ký/xác thực token đăng nhập (JWT).  
  Nếu để mặc định hoặc quá yếu: kẻ tấn công có thể giả mạo token, chiếm phiên đăng nhập.

- `API_KEY_SECRET`  
  Dùng để sinh/kiểm tra API key nội bộ.  
  Nếu để mặc định: API key dễ bị đoán hoặc bị giả mạo, dẫn đến truy cập trái phép API.

- `MACHINE_ID_SALT`  
  Salt để băm/định danh machine id theo cách khó đoán hơn.  
  Nếu để mặc định: tăng rủi ro bị suy luận định danh và các dữ liệu liên quan đến nhận diện máy.

- `INITIAL_PASSWORD`  
  Mật khẩu admin ban đầu để đăng nhập dashboard lần đầu.  
  Nếu để mặc định: bị quét/bruteforce rất nhanh, dễ mất quyền quản trị hệ thống.

> Gợi ý an toàn: mỗi môi trường (dev/staging/prod) dùng secret khác nhau, không tái sử dụng.

### 3.2) Cách tạo chuỗi ngẫu nhiên (copy-paste nhanh)

Bạn có thể dùng **một trong các cách** dưới đây.

#### Cách A - OpenSSL (nhanh, phổ biến)

```bash
# 64 ký tự hex (~32 bytes)
openssl rand -hex 32

# chuỗi base64 dài, phù hợp secret
openssl rand -base64 48
```

#### Cách B - Python (không phụ thuộc openssl)

```bash
# token URL-safe, dễ dùng cho .env
python3 -c "import secrets; print(secrets.token_urlsafe(48))"

# hoặc tạo mật khẩu mạnh 32 bytes dạng hex
python3 -c "import secrets; print(secrets.token_hex(32))"
```

#### Cách C - uuidgen (có sẵn trên nhiều máy)

```bash
# tạo 1 UUID
uuidgen

# tạo chuỗi dài hơn bằng cách ghép 4 UUID
printf "%s%s%s%s\n" "$(uuidgen)" "$(uuidgen)" "$(uuidgen)" "$(uuidgen)" | tr -d '-'
```

### 3.3) Ví dụ tạo sẵn cho từng biến

```bash
# JWT_SECRET (khuyến nghị dài)
JWT_SECRET=$(openssl rand -base64 64)

# API_KEY_SECRET
API_KEY_SECRET=$(python3 -c "import secrets; print(secrets.token_urlsafe(48))")

# MACHINE_ID_SALT
MACHINE_ID_SALT=$(openssl rand -hex 32)

# INITIAL_PASSWORD (mật khẩu mạnh, có thể copy vào password manager)
INITIAL_PASSWORD=$(python3 -c "import secrets,string; a=string.ascii_letters+string.digits+'!@#$%^&*()-_=+'; print(''.join(secrets.choice(a) for _ in range(24)))")
```

In thử để kiểm tra:

```bash
echo "$JWT_SECRET"
echo "$API_KEY_SECRET"
echo "$MACHINE_ID_SALT"
echo "$INITIAL_PASSWORD"
```

### 3.4) Cập nhật nhanh vào file `.env`

Sau khi đã `cp .env.example .env`, chạy:

```bash
sed -i "s|^JWT_SECRET=.*|JWT_SECRET=$JWT_SECRET|" .env
sed -i "s|^API_KEY_SECRET=.*|API_KEY_SECRET=$API_KEY_SECRET|" .env
sed -i "s|^MACHINE_ID_SALT=.*|MACHINE_ID_SALT=$MACHINE_ID_SALT|" .env
sed -i "s|^INITIAL_PASSWORD=.*|INITIAL_PASSWORD=$INITIAL_PASSWORD|" .env
```

Kiểm tra lại:

```bash
grep -E '^(JWT_SECRET|API_KEY_SECRET|MACHINE_ID_SALT|INITIAL_PASSWORD)=' .env
```

> Lưu ý: không commit `.env` thật lên GitHub.

---

## 4) Quản trị nhanh

```bash
# logs
docker compose logs -f 9router

# restart
docker compose restart 9router

# stop
docker compose down

# stop + remove volume data (CẨN THẬN: mất dữ liệu)
docker compose down -v
```

---

## 5) Lưu ý triển khai

- Compose đang build trực tiếp từ repo gốc `decolua/9router` branch `main`.
- Dữ liệu persist tại volume `9router_data` (map vào `/app/data`).
- Healthcheck gọi `GET /v1/models` để xác nhận API sống.
