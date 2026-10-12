# Dựng Hệ Thống Giám Sát Log Tập Trung: HAProxy SSL + Filebeat ➔ Kafka (KRaft) ➔ ELK Stack

---

## 1. Mô hình và thông tin chuẩn bị

| Vai trò | Hostname | IP thực tế | Dịch vụ cài đặt | Ghi chú |
|---|---|---|---|---|
| **VM01 (Proxy & Shipper)** | `edge-02` | `192.168.136.146` | HAProxy, Rsyslog, Filebeat | Nhận traffic HTTPS cổng 443, gom log đẩy đi |
| **VM02 (Data All-in-One)** | `cmc` | `192.168.136.131` | Kafka, Elasticsearch, Logstash, Kibana | Xử lý, lưu trữ và hiển thị dữ liệu |



**Yêu cầu hệ thống:**
- **VM01:** Tối thiểu 2GB RAM, 1 vCPU.
- **VM02:** Tối thiểu 6GB - 8GB RAM.
- Máy Client đã trỏ file `hosts` để phân giải domain:
  ```text
  192.168.136.146    hieucute.id.vn
  ```

### Bảng cổng dịch vụ và đường dẫn quan trọng

| Thành phần | Máy | Cổng mạng | File cấu hình / Thư mục log |
|---|---|---|---|
| HAProxy | VM01 | `80`, `443` | `/etc/haproxy/haproxy.cfg`, `/var/log/haproxy.log` |
| Filebeat | VM01 | - | `/etc/filebeat/filebeat.yml` |
| Kafka  | VM02 | `9092`, `9093` | `/usr/local/kafka/config/kraft/server.properties` |
| Elasticsearch | VM02 | `9200` | `/etc/elasticsearch/elasticsearch.yml` |
| Logstash | VM02 | - | `/etc/logstash/conf.d/haproxy.conf` |
| Kibana | VM02 | `5601` | `/etc/kibana/kibana.yml` |

---

## 2. Hiểu nhanh cơ chế hoạt động của luồng log

1. **Khách truy cập:** Trình duyệt gọi vào `https://hieucute.id.vn` ➔ HAProxy giải mã SSL và chuyển tiếp request tới backend Kibana.
2. **Ghi log:** HAProxy đẩy log truy cập qua socket UDP cục bộ vào Rsyslog ➔ Rsyslog ghi nối tiếp vào `/var/log/haproxy.log`.
3. **Thu thập log:** Filebeat bám theo file log, đóng gói message rồi đẩy qua giao thức Kafka Producer vào topic `haproxy-logs`.
4. **Xử lý log:** Logstash đóng vai trò Kafka Consumer, kéo message về, dùng Grok filter bóc tách IP, status code, URL, rồi nạp vào Elasticsearch.
5. **Hiển thị:** Quản trị viên mở `https://hieucute.id.vn` để xem biểu đồ và log chi tiết trực tiếp trên Kibana.

<img width="9084" height="432" alt="image" src="https://github.com/user-attachments/assets/17e477b1-52ee-4a4d-aac4-2b60d6e4b30b" />

---

## 3. Cài đặt trên VM02

### Bước 1. Cài đặt Java 17 và gói bổ trợ
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk wget curl gnupg2 net-tools
java -version
```
<img width="504" height="65" alt="image" src="https://github.com/user-attachments/assets/0ef60044-a49f-4a80-b96a-33d351a29566" />

### Bước 2. Cài đặt Kafka chế độ KRaft 
#### Vai trò: Tiếp nhận và lưu đệm log trước khi chuyển cho Logstash xử lý.  
- Topic: Kênh chứa log, ví dụ haproxy-logs.  
- Partition: Chia Topic thành các phần để hỗ trợ xử lý song song.  
- Offset: Vị trí của bản ghi trong một Partition, giúp Consumer theo dõi tiến độ đọc.  
- KRaft: Cơ chế Kafka tự quản lý metadata và đồng thuận, không cần ZooKeeper.  
Áp dụng: Khi Logstash xử lý chậm, Kafka có thể giữ log để Logstash đọc dần, giúp giảm áp lực lên các thành phần phía sau.

1. Tải và giải nén Kafka:
```bash
cd ~
wget https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz
tar -xzf kafka_2.13-3.7.0.tgz
sudo mv kafka_2.13-3.7.0 /usr/local/kafka
```
<img width="357" height="86" alt="image" src="https://github.com/user-attachments/assets/87b907d9-c790-44f5-a858-d0e81bffdb8c" />

2. Tạo file cấu hình KRaft tại `/usr/local/kafka/config/kraft/server.properties`:
```bash
sudo mkdir -p /usr/local/kafka/config/kraft
sudo tee /usr/local/kafka/config/kraft/server.properties > /dev/null << 'EOF'
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@127.0.0.1:9093

# Lắng nghe các kết nối
listeners=PLAINTEXT://0.0.0.0:9092,CONTROLLER://127.0.0.1:9093
inter.broker.listener.name=PLAINTEXT

advertised.listeners=PLAINTEXT://192.168.136.131:9092
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

log.dirs=/tmp/kraft-combined-logs
num.partitions=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
EOF
```
<img width="545" height="207" alt="image" src="https://github.com/user-attachments/assets/cf4ec89a-17be-4bd7-addb-47af00a78c1b" />

3. Định dạng ổ đĩa lưu trữ cho KRaft:
```bash
sudo rm -rf /tmp/kraft-combined-logs
# Lấy KAFKA_CLUSTER_ID
KAFKA_CLUSTER_ID="$(/usr/local/kafka/bin/kafka-storage.sh random-uuid)"
/usr/local/kafka/bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c /usr/local/kafka/config/kraft/server.properties --standalone
```
<img width="736" height="146" alt="image" src="https://github.com/user-attachments/assets/1c85afb1-8b85-4665-baf8-ffa74b65d2a1" />

4. Tạo service Systemd:
```bash
sudo tee /etc/systemd/system/kafka.service > /dev/null << 'EOF'
[Unit]
Description=Apache Kafka (KRaft Mode)
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/kraft/server.properties
ExecStop=/usr/local/kafka/bin/kafka-server-stop.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now kafka
```
<img width="730" height="173" alt="image" src="https://github.com/user-attachments/assets/aee7e07e-a0c7-4110-9532-2a655b7f166c" />

5. Kiểm tra Kafka đã mở cổng 9092:
```bash
ss -tulpn | grep 9092
```
<img width="478" height="44" alt="image" src="https://github.com/user-attachments/assets/8f91ba87-92ca-4eb7-ba98-23b9558e3c89" />

---

### Bước 3. Cài đặt Elasticsearch 8.x  
#### Elasticsearch là gì?  
- Là một Database NoSQL phân tán và công cụ tìm kiếm toàn văn, xây dựng trên thư viện Apache Lucene.  
#### Các khái niệm cốt lõi:  
- Document: Một bản ghi log đơn lẻ dưới dạng JSON.  
- Index: Tập hợp các Document có cấu trúc tương đương nhau. Ở bài này, mỗi ngày tạo 1 index: haproxy-logs-YYYY.MM.dd.  
Áp dụng: Tìm IP truy cập, mã HTTP 404, 500 hoặc thống kê số lượng request lỗi.  
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update && sudo apt install -y elasticsearch
```

Sửa `/etc/elasticsearch/elasticsearch.yml`, tắt xác thực tạm thời để lab:
```yaml
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
xpack.security.enabled: false
```

Khởi động Elasticsearch:
```bash
sudo systemctl enable --now elasticsearch
curl -s http://localhost:9200
```
<img width="724" height="355" alt="image" src="https://github.com/user-attachments/assets/6c66f27a-d46f-41b5-96bc-a836abf88e8e" />

<img width="693" height="233" alt="image" src="https://github.com/user-attachments/assets/5b0ed847-f690-4a8a-ad6e-d283c6d334c0" />

---

### Bước 4. Cài đặt Kibana 8.x

#### Kibana là gì?
- Là giao diện người dùng Web dành riêng cho Elasticsearch. Bản thân Kibana không lưu trữ dữ liệu log, mọi biểu đồ và truy vấn tìm kiếm Kibana đều gửi request REST API sang Elasticsearch để lấy dữ liệu về vẽ.
#### Thuộc tính server.publicBaseUrl:
- Khi Kibana nằm sau một Reverse Proxy chạy HTTPS, Kibana cần biết tên miền công khai chính thức của nó để tạo chính xác các đường dẫn redirect và cookie phiên đăng nhập.
- Data View: Khai báo mẫu Index để Kibana biết cần truy vấn dữ liệu nào, ví dụ haproxy-logs-*.  
- Discover: Xem và lọc log theo thời gian, IP, mã HTTP.  
- Dashboard: Hiển thị biểu đồ và số liệu giám sát.  
- Alerting: Phát hiện điều kiện bất thường và thực hiện hành động cảnh báo.  
```bash
sudo apt install -y kibana
```

Cấu hình `/etc/kibana/kibana.yml`:
```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
server.publicBaseUrl: "https://hieucute.id.vn"
```

Khởi động Kibana:
```bash
sudo systemctl enable --now kibana
```
<img width="740" height="291" alt="image" src="https://github.com/user-attachments/assets/b1e69ee4-aec2-4120-b0a5-802a0d6532dc" />
Trang đăng nhập
<img width="954" height="434" alt="image" src="https://github.com/user-attachments/assets/31afc307-b3a0-4304-8b90-4dad459a3063" />

---

### Bước 5. Cài đặt Logstash 8.x
Vai trò: Nhận log từ Kafka, phân tích dữ liệu và gửi sang Elasticsearch.  
Pipeline gồm 3 phần:  
- Input: Nhận dữ liệu từ Kafka Topic haproxy-logs.  
- Filter: Dùng Grok hoặc các bộ lọc khác để tách log thành các trường như IP, đường dẫn, mã HTTP.  
- Output: Gửi dữ liệu đã xử lý vào Elasticsearch.  
Áp dụng: Chuyển log thô thành dữ liệu có cấu trúc để dễ tìm kiếm và thống kê.

```bash
sudo apt install -y logstash
```

Tạo cấu hình pipeline tại `/etc/logstash/conf.d/haproxy.conf`:
```ruby
input {
  kafka {
    bootstrap_servers => "127.0.0.1:9092"
    topics => ["haproxy-logs"]
    group_id => "logstash-haproxy-group"
    codec => "json"
    auto_offset_reset => "latest"
  }
}

filter {
  if [message] {
    grok {
      match => { "message" => "%{HAPROXYHTTP}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://127.0.0.1:9200"]
    index => "haproxy-logs-%{+YYYY.MM.dd}"
  }
}
```

Khởi động Logstash:
```bash
sudo systemctl enable --now logstash
```

---

## 4. Cấu hình trên VM01 (HAProxy + SSL + Filebeat)

Chạy trên máy **VM01 (`192.168.136.146`)**.

### Bước 1. Cài đặt 
```bash
sudo apt update
sudo apt install -y haproxy rsyslog openssl filebeat
```

### Bước 2. Tạo chứng chỉ SSL cho domain `hieucute.id.vn`
```
Sử dụng file a Mạnh
# Gộp key và crt thành 1 file pem 
sudo cat /etc/haproxy/certs/hieucute.id.vn.crt /etc/haproxy/certs/hieucute.id.vn.key | sudo tee /etc/haproxy/certs/haproxy.pem

sudo chmod 600 /etc/haproxy/certs/haproxy.pem

```
<img width="262" height="373" alt="image" src="https://github.com/user-attachments/assets/a27638be-84c8-4182-9de7-abb8518ed722" />

---

### Bước 3. Cấu hình Rsyslog nhận log từ HAProxy
1. Mở `/etc/rsyslog.conf`, bỏ dấu comment `#` ở 2 dòng sau:
```text
module(load="imudp")
input(type="imudp" port="514")
```

2. Tạo file cấu hình log riêng `/etc/rsyslog.d/49-haproxy.conf`:
```text
local0.*    /var/log/haproxy.log
& stop
```

3. Khởi động lại Rsyslog:
```bash
sudo touch /var/log/haproxy.log
sudo chown syslog:adm /var/log/haproxy.log
sudo systemctl restart rsyslog
```

---

### Bước 4. Cấu hình HAProxy
Sửa file `/etc/haproxy/haproxy.cfg`:

```haproxy
global
    log 127.0.0.1 local0
    log 127.0.0.1 local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 4096
    tune.ssl.default-dh-param 2048

defaults
    log     global
    mode    http
    option  dontlognull
    log-format "%ci:%cp [%tr] %ft %b/%s %Tw/%Tc/%Tr/%Tt %ST %B \"%r\" Host:\"%[capture.req.hdr(0)]\" XFF:\"%[capture.req.>
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend http_in
    bind *:80
    mode http
    redirect scheme https code 301 if !{ ssl_fc }

frontend https_in
    bind *:443 ssl crt /etc/haproxy/certs/haproxy.pem
    mode http
    option httplog

    capture request header Host len 64
    capture request header X-Forwarded-For len 64

    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-Port 443

    acl is_kibana hdr(host) -i hieucute.id.vn
    use_backend kibana_backend if is_kibana
backend kibana_backend
    mode http
    balance roundrobin
    server kibana1 192.168.136.131:5601 check

listen stats
    bind *:8080
    mode http
    stats enable
    stats uri /stats
    stats auth admin:Admin@123
    stats refresh 5s
```

Kiểm tra cú pháp và kích hoạt service:
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```
<img width="710" height="296" alt="image" src="https://github.com/user-attachments/assets/5faaeee5-6a64-44dd-a7a8-c53321e26f19" />

<img width="949" height="368" alt="image" src="https://github.com/user-attachments/assets/0b9038df-db1f-4387-8344-1b625aead0cb" />

---

### Bước 5. Cấu hình Filebeat đẩy log sang Kafka
Vai trò: Theo dõi file log và gửi dữ liệu sang Kafka.  
- Đọc file /var/log/haproxy.log.  
- Lưu trạng thái đọc để tiếp tục xử lý khi dịch vụ khởi động lại.  
- Gửi log đến Kafka qua TCP cổng 9092.  
file `/etc/filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/haproxy.log
  fields:
    service: haproxy
  fields_under_root: true

output.kafka:
  # Trỏ tới IP máy VM02 cổng 9092
  hosts: ["192.168.136.131:9092"]
  topic: 'haproxy-logs'
  partition.round_robin:
    reachable_only: false
  required_acks: 1
  compression: gzip
```
<img width="539" height="250" alt="image" src="https://github.com/user-attachments/assets/6864c08c-19a0-4835-8d73-f06092fd623e" />

Kiểm tra kết nối và khởi động Filebeat:
```bash
sudo filebeat test output
sudo systemctl enable --now filebeat
```
<img width="320" height="88" alt="image" src="https://github.com/user-attachments/assets/1042372a-4b52-4d19-b857-8189d59b6c9b" />

---

## 5. Kiểm tra toàn bộ luồng hoạt động

### 5.1. Kiểm tra tiến trình trên các server

**VM01**:
```bash
systemctl is-active haproxy filebeat rsyslog
```
<img width="407" height="49" alt="image" src="https://github.com/user-attachments/assets/e8588828-856d-4600-b1d6-9a1e42a14ab6" />

**VM02**:
```bash
systemctl is-active kafka elasticsearch logstash kibana
```
<img width="439" height="62" alt="image" src="https://github.com/user-attachments/assets/172bf631-069f-4d5d-acc1-e4d1dd32fa2b" />

### 5.2. Đọc thử dữ liệu trong Kafka topic
Chạy trên **VM02** để nghe ngóng xem Filebeat có bắn message vào không:
```bash
/usr/local/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic haproxy-logs \
  --from-beginning
```
<img width="727" height="295" alt="image" src="https://github.com/user-attachments/assets/1ef1d653-85e1-4ebf-aecf-3e00366daf88" /> 

Thông suốt hoàn toàn từ Filebeat đẩy thành công vào Kafka.
### 5.3. Sinh log truy cập thực tế
Từ máy cá nhân hoặc trên terminal VM01, gọi liên tục vài request:
```bash
curl -k https://hieucute.id.vn
curl -k https://hieucute.id.vn/login
curl -k https://hieucute.id.vn/test-404
```

### 5.4. Kiểm tra chỉ mục (Index) trên Elasticsearch
Truy cập giao diện Kibana: `https://hieucute.id.vn` ➔ Vào **Dev Tools** (Console), gõ:
```http
GET _cat/indices/haproxy-logs*?v
```
<img width="959" height="219" alt="image" src="https://github.com/user-attachments/assets/15ca4172-4504-40ff-bf8f-c5a9911f647c" />


---

## 6. Xem log và tạo Dashboard trên Kibana

1. **Tạo Data View:**
   * Mở menu **☰** ➔ **Stack Management** ➔ **Data Views**.
   * Nhấn **Create data view**:
     * **Name:** `HAProxy Logs`
     * **Index pattern:** `haproxy-logs-*`
     * **Timestamp field:** `@timestamp`
   * Nhấn **Save data view to Kibana**.

<img width="941" height="428" alt="image" src="https://github.com/user-attachments/assets/b9dd001a-26c4-48dd-9634-553caac1f1bc" />

2. **Xem chi tiết log trên Discover:**
   * Menu **☰** ➔ **Analytics** ➔ **Discover**.
   * Bấm thêm các cột `message`, `log.file.path` để theo dõi request thời gian thực.

<img width="959" height="440" alt="image" src="https://github.com/user-attachments/assets/88263168-bbba-44af-b07d-4ae03af38a66" />

3. **Tạo Dashboard trực quan:**
   * Menu **☰** ➔ **Analytics** ➔ **Dashboard** ➔ **Create visualization**.
   * Biểu đồ 1: Kéo field `Records` để tạo ô Metric đếm tổng lượng request.
   * Biểu đồ 2: Kéo `@timestamp` trục ngang, `Records` trục dọc để xem lưu lượng theo thời gian.

<img width="957" height="391" alt="image" src="https://github.com/user-attachments/assets/2e256ca7-e25d-4096-b940-75b7aad3872b" />

---
## 8. Tóm tắt các bước triển khai (Checklist)

**Trên VM02 (Data Node - `192.168.136.131`):**
1. Cài Java 17, tải Kafka 3.7/4.x.
2. Cấu hình `server.properties` chuẩn KRaft (chú ý `advertised.listeners` là `192.168.136.131`).
3. Chạy `kafka-storage.sh format` với cờ `--standalone` và bật `kafka.service`.
4. Cài và bật Elasticsearch (tắt `xpack.security`).
5. Cài và bật Kibana (cấu hình `server.publicBaseUrl`).
6. Cài Logstash và tạo pipeline lấy từ topic `haproxy-logs` ném vào Elasticsearch.

**Trên VM01 (Proxy Node - `192.168.136.146`):**
7. Tạo chứng chỉ SSL gộp file `.pem` cho `hieucute.id.vn`.
8. Bật UDP 514 cho Rsyslog để ghi log HAProxy vào `/var/log/haproxy.log`.
9. Cấu hình HAProxy: Port 80 redirect sang 443, SSL cert, reverse proxy trỏ về `192.168.136.131:5601`.
10. Cấu hình Filebeat đọc `/var/log/haproxy.log` và xuất ra `192.168.136.131:9092`.

**Kiểm tra và hoàn tất:**
11. Dùng `curl` gọi vài request vào `https://hieucute.id.vn`.
12. Vào Kibana Dev Tools kiểm tra index bằng `GET _cat/indices/haproxy-logs*?v`.
13. Tạo Data View và xem kết quả trên Discover.
