
**Sinh viên:** Tạ Phạm Đình Hòa  
**Domain:** https://odoo.taphamdinhhoa.io.vn

---

## Bài toán

Xây dựng hệ thống quản lý thư viện sách cá nhân sử dụng Odoo làm giao diện quản lý và PostgreSQL làm cơ sở dữ liệu. Hệ thống cho phép quản lý danh sách sách, theo dõi việc mượn/trả sách và xem báo cáo tồn kho — tất cả truy cập được từ internet qua domain riêng.

**Yêu cầu cụ thể:**
- Quản lý danh sách sách (tên, tác giả)
- Quản lý mượn/trả sách
- Xem báo cáo sách theo thể loại

---

## Kiến trúc hệ thống
Internet
│
▼
Cloudflare Tunnel (odoo.taphamdinhhoa.io.vn)
│
▼
Nginx :80  (reverse proxy)
│
▼
Odoo :8069 (giao diện quản lý)
│
▼
PostgreSQL :5432 (cơ sở dữ liệu)
Toàn bộ chạy trên Ubuntu Server 24.04 + Docker, tái sử dụng hạ tầng từ bài 1.

---

## Cấu hình Docker Compose

Thêm service Odoo và PostgreSQL vào `docker-compose.yml`, tái sử dụng Nginx và Cloudflare Tunnel từ bài 1:

```yaml
services:
  mypostgres:
    image: postgres:15
    container_name: mypostgres
    restart: always
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo123
    volumes:
      - ./postgres_data:/var/lib/postgresql/data

  myodoo:
    image: odoo:17
    container_name: myodoo
    restart: always
    depends_on:
      - mypostgres
    ports:
      - "8069:8069"
    volumes:
      - ./odoo_data:/var/lib/odoo
      - ./odoo_config:/etc/odoo
    command: -- --db_host=mypostgres --db_port=5432 --db_user=odoo --db_password=odoo123

  mycloudflared:
    image: cloudflare/cloudflared:latest
    container_name: mycloudflared
    restart: always
    command: tunnel --no-autoupdate run --token <TOKEN>
```

Chạy `docker compose up -d` để khởi động. Tất cả 5 container đang chạy ổn định:

<!-- Chèn ảnh: docker ps --format "table..." hiện 5 container Running -->

---

## Cấu hình Cloudflare Tunnel

Tái sử dụng tunnel `myapp` từ bài 1, thêm subdomain mới `odoo.taphamdinhhoa.io.vn` trỏ vào Nginx:

**Thêm DNS record trên Cloudflare:**
- Type: CNAME
- Name: odoo
- Target: `<tunnel-id>.cfargotunnel.com`

**Cấu hình ingress rule qua API:**

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/cfd_tunnel/<TUNNEL_ID>/configurations" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"config":{"ingress":[
    {"hostname":"odoo.taphamdinhhoa.io.vn","service":"http://mynginx:80"},
    {"hostname":"web.taphamdinhhoa.io.vn","service":"http://mynginx:80"},
    {"service":"http_status:404"}
  ]}}'
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/25406952-a321-42dc-a08b-b0cb5ba0926d" />


---

## Cấu hình Nginx Reverse Proxy

Thêm server block cho Odoo vào file `nginx.conf`:

```nginx
upstream odoo {
  server myodoo:8069;
}

server {
  listen 80;
  server_name odoo.taphamdinhhoa.io.vn;

  location / {
    proxy_pass http://odoo;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  location /websocket {
    proxy_pass http://odoo;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

Reload Nginx: `docker exec mynginx nginx -s reload`
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/ce3d7156-4104-49d9-81f3-51b84f68f414" />


---

## Cài đặt và Cấu hình Odoo

Truy cập `http://192.168.1.x:8069` để vào trang tạo database lần đầu.

Điền thông tin:
- Database Name: `odoo`
- Email: `admin@admin.com`
- Password: `admin123`
- Language: Vietnamese

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/bf18cddf-9e66-40de-aacd-56ee3b333ebc" />


Sau khi tạo database xong, vào **Apps** → cài module **Inventory** để quản lý kho sách:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/d4830824-b45e-4bf0-a197-79375059d980" />


---

## Quản lý Danh Sách Sách

Vào **Inventory → Products → Products → New** để thêm sách mới.

Tạo 3 sách với thông tin:

| Mã sách | Tên sách | Số lượng |
|---------|----------|----------|
| BOOK001 | Tạ Phạm Đình Hòa | 10 cuốn |
| BOOK002 | Cơ Sở Dữ Liệu | 20 cuốn |
| BOOK003 | Mạng Máy Tính | 30 cuốn |

Sau khi tạo sản phẩm, bấm **Update Quantity** để nhập số lượng tồn kho ban đầu:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/e55a3de2-1f10-40bf-a993-4d8d813b9f37" />


<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/e66954a5-2038-4370-b250-0313f64d01c6" />

---

## Quản lý Mượn Sách

Vào **Operations → Transfers → Deliveries → New** để tạo phiếu mượn sách.

Điền:
- **Delivery Address:** Tên người mượn
- **Product:** Chọn sách cần mượn
- **Demand:** Số lượng mượn

Bấm **Validate** để xác nhận phiếu mượn. Hệ thống tự động trừ số lượng tồn kho:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/73b1bdfe-f4a6-441f-83ae-f5ee0cc81cbd" />


Tạo 3 phiếu mượn cho 3 người mượn khác nhau:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/f55befb9-4a15-4be5-92ff-56c3661b4c29" />


---

## Quản lý Trả Sách

Mở phiếu mượn đã Done → bấm **Return** → popup Reverse Transfer hiện ra → bấm **Return** để xác nhận.

Hệ thống tự tạo phiếu nhập kho `WH/IN/...` và số lượng được cộng lại:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/cde9ef37-4ec1-4dc3-9f77-4d4ae833d6e3" />


---

## Báo cáo Tồn Kho

Vào **Reporting → Inventory** để xem số lượng sách hiện tại:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/b175f16c-a112-4068-af41-4ce03153791a" />


Vào **Reporting → Move Analysis** → chuyển sang dạng biểu đồ để xem phân tích giao dịch:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/260b1a1e-3a52-4170-b158-be224b673235" />


---

## Kết quả

Website truy cập thành công qua domain `https://odoo.taphamdinhhoa.io.vn` với HTTPS tự động từ Cloudflare:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/0720323f-9638-4751-a06b-ed4d6dad239e" />

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/fe228b1e-7cf4-43cd-8663-6a9e7c00ed72" />


| Thành phần | Trạng thái |
|-----------|-----------|
| PostgreSQL | Running |
| Odoo 17 | https://odoo.taphamdinhhoa.io.vn |
| Nginx | Reverse proxy hoạt động |
| Cloudflare Tunnel | HEALTHY |

---

## Lỗi gặp phải

**Nginx trỏ sai — hiện trang bài 1 thay vì Odoo**

Nguyên nhân: File `nginx.conf` chính (mount từ bài 1) override hết các file trong `conf.d/`, nên server block mới thêm vào `conf.d/odoo.conf` không được đọc.

Fix: Thêm trực tiếp server block Odoo vào file `/home/dinhhoa/myapp/nginx/nginx.conf` rồi reload Nginx.

---

## Kết Luận

Bài tập đã triển khai thành công hệ thống quản lý thư viện sách sử dụng Odoo và PostgreSQL trên Ubuntu Server với Docker. Hệ thống hoạt động ổn định, truy cập được từ internet qua domain `https://odoo.taphamdinhhoa.io.vn` với HTTPS tự động từ Cloudflare.

**Những gì đã đạt được:**
- Tái sử dụng hạ tầng từ bài 1 (Ubuntu Server, Docker, Nginx, Cloudflare Tunnel) một cách hiệu quả
- Triển khai Odoo 17 với PostgreSQL 15 bằng Docker Compose
- Cấu hình Nginx làm reverse proxy đứng giữa Cloudflare Tunnel và Odoo
- Quản lý danh sách sách, phiếu mượn/trả và báo cáo tồn kho đầy đủ

**Bài học kinh nghiệm:**
- Cloudflare Tunnel token mode không hỗ trợ ingress rule trực tiếp, phải dùng API để cấu hình
- Nginx có thể mount nhiều file config nhưng file `nginx.conf` chính sẽ override các file trong `conf.d/`
- Docker network giữa các container độc lập cần được kết nối thủ công bằng `docker network connect`
