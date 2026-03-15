# 9router Docker Deployment (OpenClaw-ready)

Chạy **9router** bằng Docker Compose, tối ưu để giao tiếp mượt với **OpenClaw** trong cùng môi trường Docker.

## 1) Quick start

```bash
git clone https://github.com/nguyenduysphk3407051993/9router-docker-deployment.git
cd 9router-docker-deployment
cp .env.example .env
# sửa các biến secret/password trong .env

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

- `JWT_SECRET`, `INITIAL_PASSWORD`: bắt buộc đổi.
- `REQUIRE_API_KEY=true`: nên bật nếu có truy cập từ môi trường ngoài.
- `BASE_URL=http://9router:20128`: chuẩn cho internal jobs của 9router.
- `NEXT_PUBLIC_BASE_URL=http://127.0.0.1:20128`: để truy cập từ host/browser local.
- `DOCKER_SHARED_NETWORK=ai_stack`: network dùng chung với OpenClaw.

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
