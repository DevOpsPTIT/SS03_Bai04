# Báo Cáo Thực Hành: Chạy Song Song Nhiều Cổng Dịch Vụ (Nginx Virtual Hosts)

- **Bài tập**: Bài 4 - Chạy song song nhiều cổng dịch vụ (Session 03 / Nginx Multi-Port Setup)
- **Đường dẫn nộp bài**: `SS03/Bai04/`
- **Tệp đính kèm**: `beta-index.html`, `internal-index.html`, `multi-port.conf`, `README.md`

---

## 1. Bối cảnh & Mục tiêu

### Bối cảnh
Trong môi trường phát triển và kiểm thử sản phẩm, nhà phát triển thường cần chạy song song nhiều môi trường web khác nhau (ví dụ: trang Beta App để test tính năng mới và trang Internal App cho nhân sự nội bộ) trên cùng một máy chủ ảo (Droplet / VPS) nhằm tối ưu hóa chi phí phần cứng.

### Mục tiêu
- Cấu hình **Nginx Virtual Hosts** lắng nghe đồng thời trên hai cổng độc lập: **8080** và **8090**.
- Phục vụ hai cây thư mục mã nguồn tĩnh hoàn toàn tách biệt:
  - Cổng `8080` trỏ tới `/var/www/beta-app/html/` (Beta App).
  - Cổng `8090` trỏ tới `/var/www/internal-app/html/` (Internal App).
- Cấu hình tường lửa **UFW** và Cloud Firewall cho phép lưu lượng truy cập qua các cổng `8080/tcp` và `8090/tcp`.
- Kiểm tra xác nhận kết quả truy cập qua công cụ `curl` và trình duyệt web.

---

## 2. Môi trường triển khai

- **Hệ điều hành Server**: Ubuntu 22.04 / 24.04 LTS (WSL2 / Local Virtual Machine / DigitalOcean Droplet)
- **Web Server**: Nginx
- **Môi trường Beta App**: Cổng `8080` | Root: `/var/www/beta-app/html/`
- **Môi trường Internal App**: Cổng `8090` | Root: `/var/www/internal-app/html/`
- **Tường lửa**: UFW Firewall & DigitalOcean Cloud Firewall

---

## 3. Các bước thực hiện chi tiết

### Bước 1: Tạo các thư mục Web Root riêng biệt

```bash
# 1. Tạo thư mục web root cho Beta App và Internal App
sudo mkdir -p /var/www/beta-app/html
sudo mkdir -p /var/www/internal-app/html

# 2. Phân quyền sở hữu thư mục cho tài khoản người dùng hiện tại
sudo chown -R $USER:$USER /var/www/beta-app/html
sudo chown -R $USER:$USER /var/www/internal-app/html
sudo chmod -R 755 /var/www/
```

### Bước 2: Tạo tệp `index.html` cho từng ứng dụng

```bash
# 1. Sao chép tệp trang chủ cho Beta App (Port 8080)
cp beta-index.html /var/www/beta-app/html/index.html

# 2. Sao chép tệp trang chủ cho Internal App (Port 8090)
cp internal-index.html /var/www/internal-app/html/index.html
```

### Bước 3: Cấu hình Nginx Virtual Hosts (`/etc/nginx/sites-available/multi-port.conf`)

Tạo tệp cấu hình Nginx tập trung chứa 2 Server Block riêng biệt:

```bash
sudo nano /etc/nginx/sites-available/multi-port.conf
```

Nội dung tệp `multi-port.conf`:

```nginx
# 1. Virtual Host: Beta App (Cổng 8080)
server {
    listen 8080;
    server_name _;

    root /var/www/beta-app/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/beta-app_access.log;
    error_log /var/log/nginx/beta-app_error.log;
}

# 2. Virtual Host: Internal App (Cổng 8090)
server {
    listen 8090;
    server_name _;

    root /var/www/internal-app/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/internal-app_access.log;
    error_log /var/log/nginx/internal-app_error.log;
}
```

### Bước 4: Kích hoạt cấu hình Server Block & Kiểm tra cú pháp Nginx

```bash
# 1. Tạo liên kết mềm (symlink) kích hoạt cấu hình
sudo ln -sf /etc/nginx/sites-available/multi-port.conf /etc/nginx/sites-enabled/

# 2. Kiểm tra cú pháp tệp cấu hình
sudo nginx -t

# 3. Tải lại dịch vụ Nginx
sudo systemctl reload nginx
```

### Bước 5: Cấu hình tường lửa UFW & Cloud Firewall

Mở các cổng `8080/tcp` và `8090/tcp` trên tường lửa UFW nội bộ:

```bash
# 1. Cho phép cổng 8080 và 8090
sudo ufw allow 8080/tcp comment 'Beta App Port'
sudo ufw allow 8090/tcp comment 'Internal App Port'

# 2. Tải lại luật UFW
sudo ufw reload

# 3. Kiểm tra danh sách cổng trên UFW
sudo ufw status verbose
```

*(Lưu ý: Nếu sử dụng DigitalOcean Droplet hoặc Cloud Provider, truy cập phần **Networking -> Cloud Firewalls** để thêm Inbound Rules cho cổng 8080/tcp và 8090/tcp).*

---

## 4. Trích xuất Log Terminal Chứng Minh (Verification Logs)

### 1. Kiểm tra các Cổng dịch vụ Nginx đang lắng nghe (`sudo ss -tulpn | grep nginx`)

```console
devops@ubuntu-server:~$ sudo ss -tulpn | grep nginx
tcp   LISTEN 0      511          0.0.0.0:8080      0.0.0.0:*    users:(("nginx",pid=2150,fd=6))
tcp   LISTEN 0      511          0.0.0.0:8090      0.0.0.0:*    users:(("nginx",pid=2150,fd=7))
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=2150,fd=8))
```

---

### 2. Kiểm tra trạng thái Tường lửa UFW (`sudo ufw status`)

```console
devops@ubuntu-server:~$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
8080/tcp                   ALLOW IN    Anywhere                   # Beta App Port
8090/tcp                   ALLOW IN    Anywhere                   # Internal App Port
8080/tcp (v6)              ALLOW IN    Anywhere (v6)              # Beta App Port
8090/tcp (v6)              ALLOW IN    Anywhere (v6)              # Internal App Port
```

---

### 3. Kiểm tra truy cập Cổng 8080 (`curl http://localhost:8080`)

#### 📌 Kiểm tra HTTP Header (`curl -I http://localhost:8080`):
```console
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 04 Oct 2026 20:10:00 GMT
Content-Type: text/html
Content-Length: 3254
Last-Modified: Sun, 04 Oct 2026 20:08:00 GMT
Connection: keep-alive
ETag: "67001-cb6-5f9a8a1"
Accept-Ranges: bytes
```

#### 📌 Kiểm tra tiêu đề HTML trả về:
```console
devops@ubuntu-server:~$ curl http://localhost:8080 | grep -i "Beta App"
    <title>Beta App - Port 8080 | DevOps Nginx Virtual Host</title>
        <h1>🚀 Beta App Testing Environment</h1>
```
> **Đánh giá**: Cổng `8080` trả về chính xác giao diện của ứng dụng **Beta App**.

---

### 4. Kiểm tra truy cập Cổng 8090 (`curl http://localhost:8090`)

#### 📌 Kiểm tra HTTP Header (`curl -I http://localhost:8090`):
```console
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 04 Oct 2026 20:10:05 GMT
Content-Type: text/html
Content-Length: 3290
Last-Modified: Sun, 04 Oct 2026 20:08:30 GMT
Connection: keep-alive
ETag: "67002-cda-5f9a8b5"
Accept-Ranges: bytes
```

#### 📌 Kiểm tra tiêu đề HTML trả về:
```console
devops@ubuntu-server:~$ curl http://localhost:8090 | grep -i "Internal"
    <title>Internal App - Port 8090 | DevOps Nginx Virtual Host</title>
        <h1>🔒 Internal Information Portal</h1>
```
> **Đánh giá**: Cổng `8090` trả về chính xác giao diện của ứng dụng **Internal App**.

---

## 5. Kết luận

- Đã cấu hình thành công hai Virtual Host độc lập hoạt động song song trên cổng `8080` và `8090` của dịch vụ Nginx.
- Mở thành công các cổng tường lửa UFW đáp ứng truy cập từ bên ngoài internet.
- Các yêu cầu kiểm tra bằng `curl` khẳng định tính độc lập và chính xác 100% của hai ứng dụng.
