# BÁO CÁO THỰC HÀNH – TRIỂN KHAI RSYSLOG TẬP TRUNG AUTH.LOG  
link truy cập: https://github.com/iamhieuu/cmctelecom/blob/main/rsyslog.md

> **Kiến trúc:**
> - **Server:** Nhận logs từ nhiều clients
> - **Client:** Gửi `/var/log/auth.log` tới Server
> - **Port:** TCP/514

---

## 1. Mục Tiêu Bài Lab

#### Về mặt kiến thức

- Nắm vững kiến trúc và cơ chế hoạt động của mô hình quản lý log tập trung Rsyslog Client/Server.
- Hiểu rõ cơ chế theo dõi, đọc dữ liệu realtime và đẩy log từ file tĩnh thông qua module `imfile`.

#### Về mặt triển khai & cấu hình

- **Phía Client:** Cấu hình đẩy log an toàn, đảm bảo tính toàn vẹn qua giao thức TCP (Port 514).
- **Phía Server:** Thiết lập lắng nghe, tiếp nhận log từ remote client và tự động bóc tách, băm luồng lưu trữ theo từng hostname.

---
Flow hoạt động từ client  
<img width="7448" height="2968" alt="image" src="https://github.com/user-attachments/assets/8b706012-5621-48db-aca9-800f3d4e5eb5" />

Mô hình lab và địa chỉ IP tailscale  

<img width="692" height="757" alt="image" src="https://github.com/user-attachments/assets/52357d94-e949-489a-a693-e6001bc0f5d6" />

Ping thông từ client đến server  

<img width="417" height="141" alt="image" src="https://github.com/user-attachments/assets/4696681b-3ff8-4a87-b62d-a6793c74911c" />

## PHẦN 1: CÀI ĐẶT SERVER

#### Bước 1: Cài Rsyslog

```
sudo apt update
sudo apt install -y rsyslog
sudo systemctl start rsyslog
sudo systemctl enable rsyslog
```

Check status rsyslog:

```
sudo systemctl status rsyslog
```

<img width="749" height="218" alt="image" src="https://github.com/user-attachments/assets/8505cf57-745e-4a75-88be-cb32f7b85a5d" />

#### Bước 2: Tạo Config File Remote

```
sudo nano /etc/rsyslog.d/50-remote.conf
```

```
module(load="imtcp")

template(name="RemoteAuthLog" type="string" string="/var/log/remote/%HOSTNAME%/auth.log")

ruleset(name="remote") {
    action(type="omfile" dynaFile="RemoteAuthLog")
}

input(type="imtcp" port="514" ruleset="remote")
```

<img width="608" height="185" alt="image" src="https://github.com/user-attachments/assets/ed78ed28-4349-4db8-898d-9b41d8abf93b" />

#### Bước 3: Tạo Thư Mục và Restart

```
sudo mkdir -p /var/log/remote
sudo chown -R syslog:adm /var/log/remote
sudo chmod -R 755 /var/log/remote
sudo rsyslogd -N1
sudo systemctl restart rsyslog
sudo ss -lntp | grep 514
```

<img width="743" height="152" alt="image" src="https://github.com/user-attachments/assets/742869cf-b22b-4f7d-ae4d-543009b2571c" />

---

## PHẦN 2: CÀI ĐẶT CLIENT

#### Bước 1: Cài Rsyslog

```
sudo apt update
sudo apt install -y rsyslog
sudo systemctl start rsyslog
sudo systemctl enable rsyslog
```

Check status rsyslog:

```
sudo systemctl status rsyslog
```

<img width="728" height="182" alt="image" src="https://github.com/user-attachments/assets/416359a1-49c8-4a4b-a5b0-10380ce4fbd8" />

#### Bước 2: Add syslog to adm Group

```
sudo usermod -a -G adm syslog
```

#### Bước 3: Tạo Config Forward

```
sudo nano /etc/rsyslog.d/50-forward-auth.conf
```

```
module(load="imfile" mode="polling" pollingInterval="2")

ruleset(name="fwd_auth") {
    action(type="omfwd" target="100.70.80.10"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.20"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.30"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.40"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.50"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.60"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.70"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.80"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.90"  port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
    action(type="omfwd" target="100.70.80.100" port="514" protocol="tcp" queue.type="LinkedList" action.resumeRetryCount="-1")
}

input(type="imfile"
      File="/var/log/auth.log"
      Tag="thanhhieudz"
      Severity="info"
      Facility="auth"
      ruleset="fwd_auth")
```

<img width="749" height="311" alt="image" src="https://github.com/user-attachments/assets/739f1052-5ec4-46ae-bc64-97e64b4f2091" />

#### Bước 4: Restart Rsyslog

```
sudo rsyslogd -N1
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

<img width="734" height="365" alt="image" src="https://github.com/user-attachments/assets/71a363f8-c08d-4784-aed3-f57789bea405" />

Check gói tin:

```
nc -vz 100.70.80.60 514
```

<img width="361" height="49" alt="image" src="https://github.com/user-attachments/assets/41a90640-ea2e-4804-829c-6a30ab23271e" />

---

## PHẦN 3: TEST

#### Test 1: Tạo test log check của chính bản thân

Đăng nhập sai:
Bên client :
```
sudo tail -f /var/log/auth.log
```
<img width="1509" height="626" alt="image" src="https://github.com/user-attachments/assets/bc50211c-dc3d-4366-8151-140c5b5983d2" />

Bên server xem log:  
```
sudo tail -f /var/log/remote/thanhhieu/auth.log
```
<img width="1507" height="549" alt="image" src="https://github.com/user-attachments/assets/e9ff6144-82e0-4479-b63f-cb883da9a720" />

#### Test 2: Tạo test log check xem của người khác

<img width="1520" height="558" alt="image" src="https://github.com/user-attachments/assets/fa5d354a-0efa-4709-8496-c89d35a27f3a" />

#### Sử dụng tcpdump để bắt gói tin mạng trên máy Client

```
sudo tcpdump -ni any port 514 -nn
```

<img width="1508" height="228" alt="image" src="https://github.com/user-attachments/assets/24aa931c-2dc7-4f54-abf2-e09a591f7911" />

---

## Câu Hỏi & Giải Đáp

#### 1. `@` và `@@` trong Rsyslog khác nhau như thế nào?

- `@IP:port` — Chuyển tiếp log qua giao thức **UDP**.
- `@@IP:port` — Chuyển tiếp log qua giao thức **TCP**.

#### 2. `imfile` dùng để làm gì?

`imfile` (**Input Module for Text Files**) là module cho phép Rsyslog đọc, theo dõi các file văn bản tĩnh (ví dụ: file log của Nginx, Apache, MySQL hoặc file log hệ thống) biến chúng thành các sự kiện Syslog chuẩn để xử lý hoặc chuyển tiếp qua mạng.

#### 3. Tại sao cần Template trên Rsyslog Server?

Template giúp định dạng nội dung log và tự động cấu trúc lại đường dẫn/tên file lưu trữ dựa trên các biến môi trường (như `%HOSTNAME%`, `%YEAR%`, `%syslogfacility-text%`). Nhờ có Template, Server có thể phân loại log của hàng trăm Client tự động mà không cần viết thủ công từng dòng cấu hình cho từng IP.

#### 4. Làm thế nào để phân biệt log của 10 Client?

Dựa vào trường thông tin **Hostname** (`%HOSTNAME%`) hoặc **IP nguồn** (`%FROMHOST-IP%`) được đóng gói sẵn trong Header của mỗi gói tin Syslog truyền đi qua mạng.

#### 5. Nếu Server mất kết nối 5 phút thì log có bị mất không?

**Không bị mất** nếu Client đã cấu hình chuyển tiếp bằng TCP kết hợp với Action Queue (`queue.type="LinkedList"` và `queue.filename="..."`). Trong khoảng thời gian 5 phút Server DOWN, log sẽ được tích lũy vào bộ nhớ đệm/ổ đĩa trên Client và tự động gửi bù lại khi kết nối Server khôi phục.

#### 6. TCP và UDP có ưu/nhược điểm gì khi truyền Syslog?

| Tiêu chí | UDP (`@`) | TCP (`@@`) |
|---|---|---|
| Bảo đảm dữ liệu | Không (Dễ mất gói khi nghẽn mạng) | Có (Truyền tin cậy, có ACK xác nhận) |
| Hỗ trợ Queue | Không hiệu quả | Tốt (Dễ phát hiện mất kết nối để Queue) |
| Tài nguyên | Nhẹ, nhanh, tốn ít CPU | Tốn tài nguyên thiết lập bắt tay 3 bước |

#### 7. Nếu rsyslogd -N1 báo lỗi thì xử lý thế nào?

1. Kiểm tra dòng bị báo lỗi hiển thị trên màn hình.
2. Kiểm tra các lỗi cú pháp thường gặp.
3. Sửa trực tiếp file cấu hình bị lỗi và chạy lại `rsyslogd -N1` cho tới khi kết quả trả về *syntax check OK*.

#### 8. Khi `tcpdump` thấy packet TCP/514 nhưng Server không tạo file log, có thể kiểm tra những thành phần nào?

- **Module nhận log:** Xem Server đã bật module `imtcp` và mở `input(type="imtcp" port="514")` chưa.
- **Tường lửa local:** Kiểm tra Firewall/UFW trên Server có đang chặn tiến trình `rsyslogd` ghi dữ liệu ra thư mục đĩa không.
- **Bộ lọc cấu hình:** Kiểm tra các câu lệnh `if...then` xem điều kiện lọc có bị lệch so với log gửi tới không.
- **Phân quyền thư mục:** Kiểm tra thư mục `/var/log/remote/` xem user `syslog` có đủ quyền ghi hay không.
