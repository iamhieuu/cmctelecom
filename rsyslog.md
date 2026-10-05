# Hướng dẫn cài đặt Rsyslog

KIẾN TRÚC:
Server:  Nhận logs từ nhiều clients
Client: Gửi /var/log/auth.log tới Server
Port: TCP/514

---

PHẦN 1: CÀI ĐẶT SERVER

Bước 1: Cài Rsyslog

```
sudo apt update
sudo apt install -y rsyslog
sudo systemctl start rsyslog
sudo systemctl enable rsyslog
```

Bước 2: Bật TCP Listener

```
sudo nano /etc/rsyslog.conf
```

Tìm và uncomment (xóa # ở đầu):

```
module(load="imtcp")
input(type="imtcp" port="514")
```

Lưu: Ctrl+X → Y → Enter

Bước 3: Tạo Config File Remote

```
sudo nano /etc/rsyslog.d/50-remote.conf
```

Copy & paste nội dung:

```
$template RemoteAuth,"/var/log/remote/%HOSTNAME%/auth.log"
$template RemoteAllClients,"/var/log/remote/all-clients.log"

if ($inputname == "imtcp") and \
   ($hostname != "localhost" and $hostname != "127.0.0.1") then {
    
    if ($programname startswith "client" and $programname endswith "-auth") then {
        action(type="omfile" dynafile="RemoteAuth"
               dirCreateMode="0755" fileCreateMode="0644"
               dirOwner="syslog" dirGroup="adm"
               fileOwner="syslog" fileGroup="adm" asyncWriting="on")
    }
    
    action(type="omfile" file="/var/log/remote/all-clients.log"
           fileCreateMode="0644" asyncWriting="on" flushInterval="5")
}
```

Lưu: Ctrl+X → Y → Enter

Bước 4: Tạo Thư Mục và Restart

```
sudo mkdir -p /var/log/remote
sudo chown -R syslog:adm /var/log/remote
sudo chmod -R 755 /var/log/remote
sudo touch /var/log/remote/all-clients.log
sudo chown syslog:adm /var/log/remote/all-clients.log
sudo rsyslogd -N1
sudo systemctl restart rsyslog
sudo ss -lntp | grep 514
```

Kết quả phải thấy: LISTEN ... rsyslogd
✓ SERVER READY!

---

PHẦN 2: CÀI ĐẶT CLIENT

Bước 1: Cài Rsyslog

```
sudo apt update
sudo apt install -y rsyslog
sudo systemctl start rsyslog
sudo systemctl enable rsyslog
```

Bước 2: Add syslog to adm Group

```
sudo usermod -a -G adm syslog
```

Bước 3: Tạo Config Forward

```
sudo nano /etc/rsyslog.d/50-forward-auth.conf
```

THAY 192.168.x.10 bằng IP của SERVER!

Copy & paste nội dung:

```
module(load="imfile")

$template ForwardTemplate,"<%PRI%>%TIMESTAMP% %HOSTNAME% [%PROGRAMNAME%]: %MSG%"

input(type="imfile"
      File="/var/log/auth.log"
      Tag="client-auth"
      StateFile="/var/log/imfile.auth.state"
      Facility="auth"
      Severity="notice")

action(type="omfwd"
       Target="192.168.x.10"
       Port="514"
       Protocol="tcp"
       Template="ForwardTemplate"
       queue.Type="LinkedList"
       queue.FileName="fwd-server"
       queue.MaxDiskSpace="1g"
       queue.SaveOnShutdown="on")
```

Lưu: Ctrl+X → Y → Enter

Bước 4: Restart Rsyslog

```
sudo rsyslogd -N1
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
nc -vz 192.168.x.10 514
```

Kết quả phải thấy: active (running) và succeeded!
✓ CLIENT READY!

---

PHẦN 3: KIỂM TRA (TEST)

Test 1: Connection (Từ CLIENT)

```
nc -vz 192.168.x.10 514
```

Kết quả: succeeded!

Test 2: Tạo Test Log (Từ CLIENT)

```
sudo logger -t "client-auth" "TEST LOG FROM CLIENT"
```

Test 3: Check Server (Từ SERVER)

```
sudo tail -f /var/log/remote/all-clients.log
```

Kết quả phải thấy:

```
Nov 17 14:35:45 client-hostname client-auth: TEST LOG FROM CLIENT
```

✓ LOGS TẬP TRUNG ĐƯỢC!

---

PHẦN 4: THEO DÕI LOGS

Cách 1: Simple Tail (Dễ Nhất)

```
sudo tail -f /var/log/remote/all-clients.log
```

Cách 2: Dashboard (Tốt Nhất)

```
sudo monitor-clients-dashboard.sh
```

Cách 3: Search

```
grep "pattern" /var/log/remote/all-clients.log
grep "client-01" /var/log/remote/all-clients.log
wc -l /var/log/remote/all-clients.log
```

---

LỆNH NHANH CHỌN

```
sudo bash install-rsyslog-fresh.sh server
→ Setup server (automated)

sudo bash install-rsyslog-fresh.sh client 192.168.x.10
→ Setup client (automated)

sudo ss -lntp | grep 514
→ Check server listening

nc -vz 192.168.x.10 514
→ Test client connection

sudo logger -t "client-auth" "TEST"
→ Create test log

sudo tail -f /var/log/remote/all-clients.log
→ Monitor all clients

sudo monitor-clients-dashboard.sh
→ Live dashboard

grep "pattern" /var/log/remote/all-clients.log
→ Search logs

du -sh /var/log/remote/
→ Check disk usage
```

---

TROUBLESHOOTING

Server not listening
Kiểm tra TCP uncommented trong /etc/rsyslog.conf
Sau đó:

```
sudo systemctl restart rsyslog
```

Client cannot connect

```
sudo ufw allow 514/tcp
```

Server not receiving logs

```
sudo rsyslogd -N1
sudo systemctl status rsyslog
sudo logger -t "client-auth" "TEST"
```

No auth.log on client

```
ssh localhost
```

Hoặc dùng logger:

```
sudo logger -t "test" "test log"
```

---

BẢNG TÓM HỢP

Step | Server | Client
-----|--------|--------
1    | apt install rsyslog | apt install rsyslog
2    | uncomment TCP | add syslog to adm
3    | create 50-remote.conf | create 50-forward-auth.conf
4    | mkdir /var/log/remote | (config done)
5    | restart rsyslog | restart rsyslog
6    | verify ss -lntp | verify nc connection
7    | tail -f all-clients.log | logger test
8    | see logs | check on server

---

XONG! Centralized logging ready.
Server theo dõi tất cả clients trong 1 file.
Không cần mở nhiều files.
Dễ monitor!
