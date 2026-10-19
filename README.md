# Bài tập DevOps - Nginx, Security Headers và Docker

Bài nộp **Session 12 DevOps** gồm 4 bài tập vận dụng. Mỗi bài có file cấu hình độc lập; phần giải thích, lệnh kiểm tra và kết quả thử nghiệm được ghi trong tài liệu này.

## Danh sách bài tập

| Bài | Chủ đề | File cấu hình |
|---|---|---|
| 1 | Sửa cú pháp Nginx | [bai-1/app.conf](bai-1/app.conf) |
| 2 | Xử lý lỗi 504 Gateway Timeout | [bai-2/app.conf](bai-2/app.conf) |
| 3 | HTTP Security Headers | [bai-3/secure.conf](bai-3/secure.conf) |
| 4 | UFW và Docker port binding | [bai-4/docker-compose.yml](bai-4/docker-compose.yml) |

> Các domain `example.com`, `api.example.com`, `secure.example.com`, image `my-spring-boot-app:latest` và đường dẫn chứng chỉ TLS là giá trị minh họa từ đề bài. Để triển khai thực tế cần thay bằng domain, image, certificate và backend phù hợp.

## Bài 1. Xử lý sự cố sập Nginx

**Nguyên nhân:** Dòng `proxy_pass http://localhost:8080` bị thiếu dấu chấm phẩy `;` cuối chỉ thị. Vì thế Nginx không thể phân tích chính xác chỉ thị tiếp theo và báo lỗi `unexpected "proxy_set_header"`.

**Cách sửa:** Bổ sung dấu `;`:

```nginx
proxy_pass http://localhost:8080;
```

Cấu hình đầy đủ: [bai-1/app.conf](bai-1/app.conf).

**Kiểm tra và nạp lại cấu hình:**

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx
```

Nếu Nginx đang dừng, sau khi test thành công sử dụng `sudo systemctl start nginx`.

## Bài 2. Tối ưu timeout cho proxy

**Nguyên nhân:** Spring Boot cần khoảng 90 giây để xuất báo cáo. Mặc định, `proxy_read_timeout` của Nginx là 60 giây; nếu không đọc được dữ liệu phản hồi từ upstream trong khoảng này, Nginx có thể trả về `504 Gateway Time-out`.

**Cách sửa:** Trong `location /api/export/` của [bai-2/app.conf](bai-2/app.conf) thêm:

```nginx
proxy_connect_timeout 10s;
proxy_send_timeout 120s;
proxy_read_timeout 120s;
```

| Chỉ thị | Giải thích |
|---|---|
| `proxy_connect_timeout 10s` | Tối đa 10 giây để thiết lập kết nối tới Backend. |
| `proxy_send_timeout 120s` | Tối đa 120 giây **giữa hai lần ghi** dữ liệu yêu cầu tới Backend. |
| `proxy_read_timeout 120s` | Tối đa 120 giây **giữa hai lần đọc** dữ liệu phản hồi từ Backend. |

**Lưu ý:** `proxy_read_timeout` không giới hạn tổng thời lượng xử lý request. Việc tăng timeout phía Nginx cũng không thay thế cấu hình timeout ở Backend hoặc các proxy khác.

**Kiểm tra:**

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -v http://api.example.com/api/export/
```

Ví dụ `curl` trên giả định API hỗ trợ GET, cần điều chỉnh nếu API thực tế yêu cầu phương thức khác.

## Bài 3. Cấu hình Header Security

Trong [bai-3/secure.conf](bai-3/secure.conf) thêm ba chỉ thị:

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

- **X-Frame-Options: SAMEORIGIN:** Chỉ cho phép nhúng trang trong frame cùng origin, giúp giảm nguy cơ Clickjacking.
- **X-Content-Type-Options: nosniff:** Giúp ngăn một số hành vi suy đoán MIME type của trình duyệt.
- **Strict-Transport-Security:** Yêu cầu trình duyệt dùng HTTPS trong 31.536.000 giây (khoảng 1 năm), gồm cả subdomain.
- **always:** Thêm header kể cả với một số phản hồi lỗi.

**Kiểm chứng:**

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -I https://secure.example.com/
```

Kết quả mong đợi trong response:

```http
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**Lưu ý bảo mật:** Chỉ dùng `includeSubDomains` nếu tất cả subdomain hỗ trợ HTTPS. Các header yêu cầu trong bài không loại bỏ hoàn toàn nguy cơ XSS; ở ứng dụng thực tế còn cần xử lý đầu vào/đầu ra và cân nhắc CSP. Chứng chỉ TLS cần tồn tại và hợp lệ khi triển khai.

## Bài 4. Tích hợp UFW và Docker

**Vấn đề:** Với mapping `"8080:8080"`, Docker thường publish cổng trên mọi interface của host. Docker sử dụng các quy tắc firewall/NAT riêng nên việc chỉ chặn bằng UFW có thể không bảo vệ cổng đã publish như mong đợi.

**Cách sửa:** [bai-4/docker-compose.yml](bai-4/docker-compose.yml) dùng mapping:

```yaml
services:
  backend:
    image: my-spring-boot-app:latest
    ports:
      - "127.0.0.1:8080:8080"
```

Giải thích: `127.0.0.1` là loopback của **host**; `8080` bên trái là host port và `8080` bên phải là container port. Nginx chạy **trên chính host** vẫn có thể kết nối `http://127.0.0.1:8080`, nhưng Docker không publish cổng đó trên public IP trong cấu hình bridge/NAT mặc định.

**Kiểm chứng:**

```bash
docker compose config
docker compose up -d
docker compose ps
docker compose port backend 8080
curl -v http://127.0.0.1:8080/
```

Dự kiến `docker compose port backend 8080` hiển thị `127.0.0.1:8080`. Cần có Docker và image Spring Boot thật để chạy thử. Các Docker Engine cũ hơn 28.0.0 có ngoại lệ liên quan đến truy cập từ thiết bị cùng mạng L2 tới cổng bind loopback; nên sử dụng phiên bản Docker được cập nhật và kiểm tra firewall/mạng thực tế.

## Kết quả kiểm tra cú pháp

| Bài | Phương pháp | Kết quả |
|---|---|---|
| 1 | `nginx -t` với cấu hình Nginx tạm | PASS |
| 2 | `nginx -t` với cấu hình Nginx tạm | PASS |
| 3 | `nginx -t` với chứng chỉ tự ký tạm dùng để test cú pháp | PASS |
| 4 | Phân tích YAML và xác nhận mapping `127.0.0.1:8080:8080` | PASS |

**Phạm vi:** Đây là kết quả kiểm tra tĩnh, **chưa triển khai Nginx, Docker Compose và Spring Boot trên server thực tế**. Chứng chỉ tự ký dùng để test không được đưa vào repository.

## Link nộp trên Portal

- Toàn bộ bài: https://github.com/lexuanbao258-collab/ss12-devops
- Bài 1: https://github.com/lexuanbao258-collab/ss12-devops/tree/main/bai-1
- Bài 2: https://github.com/lexuanbao258-collab/ss12-devops/tree/main/bai-2
- Bài 3: https://github.com/lexuanbao258-collab/ss12-devops/tree/main/bai-3
- Bài 4: https://github.com/lexuanbao258-collab/ss12-devops/tree/main/bai-4

## Tài liệu chính thức

- [Nginx command-line switches](https://nginx.org/en/docs/switches.html)
- [Nginx proxy module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Nginx headers module](https://nginx.org/en/docs/http/ngx_http_headers_module.html)
- [Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)
- [Docker and UFW/iptables](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
