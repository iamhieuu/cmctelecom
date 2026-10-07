

Postgresql streaming replication · MD
# Dựng PostgreSQL Streaming Replication: 1 Master + 1 Replica
 
> Tài liệu tham khảo: [Percona – How to Set Up PostgreSQL Streaming Replication](https://www.percona.com/blog/setting-up-streaming-replication-postgresql/)
>
> Bài Percona viết theo cú pháp cũ (PostgreSQL 9.x/10, dùng `recovery.conf`, `wal_keep_segments`). Tài liệu này giữ nguyên **luồng làm việc** của bài, nhưng đổi sang **cú pháp PostgreSQL 12 trở lên**. Các lệnh đã được tôi chạy thử thật trên **PostgreSQL 16** (Ubuntu 24.04): dựng master, replica, kiểm tra đồng bộ dữ liệu, bật chế độ synchronous và promote.
 
---
 
## 1. Mô hình và thông tin chuẩn bị
 
| Vai trò | Hostname | IP (ví dụ) | Ghi chú |
|---|---|---|---|
| Master (primary) | `pg-master` | `192.168.0.10` | Nhận đọc và ghi |
| Replica (standby) | `pg-replica` | `192.168.0.11` | Chỉ đọc, nhận WAL từ master |
 
Hãy thay IP, mật khẩu và phiên bản cho đúng với môi trường của bạn.
 
```
          WAL (streaming, cổng 5432)
 [Master] ---------------------------> [Replica]
 192.168.0.10                          192.168.0.11
 wal sender                            wal receiver + startup process
```
 
**Yêu cầu:**
- 2 server cùng phiên bản **major** của PostgreSQL (ví dụ cùng 16), cùng hệ điều hành/kiến trúc.
- Hai server ping và kết nối được nhau qua cổng `5432` (mở firewall).
- Quyền `root`/`sudo` để cài đặt, và dùng user `postgres` để chạy các lệnh PostgreSQL.
### Đường dẫn theo hệ điều hành
 
| Mục | Ubuntu / Debian | RHEL / Rocky / Alma (PGDG) |
|---|---|---|
| Thư mục dữ liệu (`PGDATA`) | `/var/lib/postgresql/16/main` | `/var/lib/pgsql/16/data` |
| File cấu hình | `/etc/postgresql/16/main/` | nằm trong `PGDATA` |
| Tên service | `postgresql@16-main` (hoặc `postgresql`) | `postgresql-16` |
 
Trong tài liệu dùng biến `PGDATA` để chỉ thư mục dữ liệu. Bạn gán đúng đường dẫn theo bảng trên.
 
---
 
## 2. Hiểu nhanh cơ chế (để biết mình đang làm gì)
 
Mọi thay đổi trong PostgreSQL đều được ghi vào **WAL (write-ahead log)** trước. Streaming replication là việc replica liên tục nhận WAL từ master rồi chạy lại (replay) các thay đổi đó.
 
Có 3 tiến trình tham gia:
 
| Tiến trình | Chạy ở đâu | Vai trò |
|---|---|---|
| `wal sender` | Master | Gửi WAL cho replica |
| `wal receiver` | Replica | Nhận WAL và ghi vào file WAL của replica |
| `startup process` | Replica | Replay WAL vào dữ liệu của replica |
 
**LSN** (Log Sequence Number) là con trỏ vị trí trong WAL. Replica báo cho master biết nó đã replay tới LSN nào, master gửi tiếp phần WAL từ vị trí đó.
 
---
 
## 3. Cài đặt PostgreSQL (cả 2 server)
 
Ubuntu / Debian:
```bash
sudo apt update
sudo apt install -y postgresql postgresql-client
```
 
RHEL / Rocky / Alma (dùng repo PGDG, ví dụ bản 16):
```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable --now postgresql-16
```
 
Mở firewall cổng 5432 
```bash
chỉ cho phép IP của replica vào master
sudo ufw allow from 192.168.0.11 to any port 5432 proto tcp
 
```
 <img width="511" height="66" alt="image" src="https://github.com/user-attachments/assets/1aafda1b-c956-4822-aa5e-a505b3b21e3b" />

---
 
## 4. Cấu hình trên MASTER
 
Tất cả lệnh SQL dưới đây chạy trên **master** (`sudo -u postgres psql`).
 
### Bước 1. Tạo user dùng riêng cho replication
 
```sql
CREATE ROLE replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD '123';
```
 
- User này chỉ cần quyền `REPLICATION` (không cần superuser).
- Nên dùng mật khẩu mạnh, không dùng mật khẩu đơn giản như trong bài mẫu.
### Bước 2. Đặt các tham số cho master
 
```sql
ALTER SYSTEM SET listen_addresses = '*';          -- hoặc IP cụ thể: '192.168.0.10'
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET wal_keep_size = '512MB';         -- thay cho wal_keep_segments của PG <= 12
ALTER SYSTEM SET hot_standby = 'on';
```
 
| Tham số | Ý nghĩa |
|---|---|
| `listen_addresses` | Cho phép PostgreSQL nhận kết nối từ IP khác. Mặc định chỉ `localhost` nên replica không kết nối được. |
| `wal_level` | Phải từ `replica` trở lên để có đủ thông tin cho replication (mặc định đã là `replica` từ PG10). |
| `max_wal_senders` | Số kết nối gửi WAL tối đa. Mỗi replica cần ít nhất 1 (lúc chạy `pg_basebackup` với `-Xs` cần thêm 1), nên để dư. |
| `max_replication_slots` | Số replication slot tối đa (xem mục 5). |
| `wal_keep_size` | Giữ lại tối thiểu bấy nhiêu WAL trên master để replica bị chậm vẫn bắt kịp. |
| `hot_standby` | Có hiệu lực trên replica, cho phép chạy `SELECT` trên replica. Đặt luôn trên master để sau này promote vẫn đúng. |
 
Khởi động lại để các tham số có hiệu lực (`listen_addresses`, `wal_level`, `max_wal_senders` bắt buộc restart):
```bash
# Ubuntu / Debian
sudo systemctl restart postgresql@16-main
# RHEL
sudo systemctl restart postgresql-16
```
 
> **Về archive WAL (không bắt buộc):** bài Percona khuyên bật `archive_mode` và `archive_command`. Streaming replication vẫn chạy được khi không có archive. Archive đáng bật trên production để replica bị tụt quá xa vẫn lấy lại được WAL (dùng `restore_command`) và để làm backup/PITR. Nếu dùng replication slot (mục 5) thì rủi ro replica mất WAL giảm đi nhiều.
 
### Bước 3. Cho phép replica kết nối vào master (`pg_hba.conf`)
 
Mở file `pg_hba.conf`:
- Ubuntu/Debian: `/etc/postgresql/16/main/pg_hba.conf`
- RHEL: `$PGDATA/pg_hba.conf`
Thêm 1 dòng (IP phải đúng IP của **replica**):
```
host    replication    replicator    192.168.0.11/32    scram-sha-256
```
 
Lưu ý: từ khóa database phải là `replication` (đây không phải tên database thật). Dùng `scram-sha-256` cho PostgreSQL 14 trở lên; bản cũ hơn có thể dùng `md5`.
 
Nạp lại cấu hình (không cần restart):
```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```
 
---
 
## 5. Cấu hình trên REPLICA
 
### Bước 4. Dừng PostgreSQL và làm sạch thư mục dữ liệu
 
> **Cảnh báo:** các lệnh dưới **xóa toàn bộ dữ liệu cũ** của PostgreSQL trên replica. Chỉ làm trên server replica mới.
 
```bash
# Ubuntu / Debian
sudo systemctl stop postgresql@16-main
sudo -u postgres bash -c 'rm -rf /var/lib/postgresql/16/main/*'
 
# RHEL
sudo systemctl stop postgresql-16
sudo -u postgres bash -c 'rm -rf /var/lib/pgsql/16/data/*'
```
 
Thư mục dữ liệu phải **rỗng** và thuộc quyền user `postgres` thì `pg_basebackup` mới ghi vào được.
 
### Bước 5. Sao chép dữ liệu từ master bằng `pg_basebackup`
 
Chạy trên **replica**, dưới user `postgres`:
 
```bash
sudo -u postgres pg_basebackup \
  -h 192.168.0.10 -p 5432 -U replicator \
  -D /var/lib/postgresql/16/main \
  -X stream -P -R \
  -C -S replica1_slot
```
 
(RHEL đổi `-D` thành `/var/lib/pgsql/16/data` và gọi `/usr/pgsql-16/bin/pg_basebackup`.)
 
Nhập mật khẩu của user `replicator` khi được hỏi. Ý nghĩa các tùy chọn:
 
| Tùy chọn | Ý nghĩa |
|---|---|
| `-h`, `-p`, `-U` | Địa chỉ, cổng và user của master |
| `-D` | Thư mục dữ liệu đích trên replica |
| `-X stream` (hay `-Xs`) | Vừa copy dữ liệu vừa stream WAL phát sinh trong lúc copy |
| `-P` | Hiện tiến độ |
| `-R` | Tự tạo file `standby.signal` và ghi `primary_conninfo` vào `postgresql.auto.conf` |
| `-C -S replica1_slot` | Tạo luôn một **replication slot** tên `replica1_slot` trên master và gắn replica vào slot đó |
 
**Khác biệt quan trọng so với bài Percona:** từ PostgreSQL 12, file `recovery.conf` đã bị bỏ. Thay vào đó:
- File rỗng `standby.signal` trong `PGDATA` cho PostgreSQL biết đây là standby.
- Thông tin kết nối master (`primary_conninfo`, `primary_slot_name`) nằm trong `postgresql.auto.conf` hoặc `postgresql.conf`.
`-R` đã tự làm cả hai việc. Kiểm tra:
```bash
sudo -u postgres ls /var/lib/postgresql/16/main | grep standby
sudo -u postgres cat /var/lib/postgresql/16/main/postgresql.auto.conf
```
Kết quả mong đợi: có file `standby.signal`, và có dòng `primary_conninfo = '...host=192.168.0.10...'` cùng `primary_slot_name = 'replica1_slot'`.
 
> **Replication slot là gì?** Slot báo cho master giữ lại WAL cho tới khi replica nhận xong, nên replica không bị mất WAL khi tạm ngắt kết nối. Đổi lại, nếu replica **tắt lâu**, WAL tích tụ trên master có thể làm đầy ổ đĩa. Cần giám sát (xem mục 8).
 
**Lưu ý cho Ubuntu/Debian:** file cấu hình nằm ở `/etc/postgresql/16/main/`, không nằm trong `PGDATA` nên `pg_basebackup` không copy sang. Replica dùng cấu hình `/etc` của chính nó. Hãy đảm bảo `max_connections` trên replica **lớn hơn hoặc bằng** master, và `hot_standby = on` (mặc định đã bật từ PG10).
 
### Bước 6. Khởi động replica
 
```bash
# Ubuntu / Debian
sudo systemctl start postgresql@16-main
# RHEL
sudo systemctl start postgresql-16
```
 
Xem log nếu cần:
```bash
sudo tail -n 30 /var/log/postgresql/postgresql-16-main.log      # Ubuntu / Debian
sudo tail -n 30 /var/lib/pgsql/16/data/log/*.log                # RHEL
```
Dòng log báo thành công thường có nội dung `started streaming WAL from primary`.
 
---
 
## 6. Kiểm tra replication đã chạy và dữ liệu đồng bộ
 
### 6.1. Kiểm tra các tiến trình
 
Trên **master**:
```bash
ps -eo args | grep "wal sender" | grep -v grep
```
Mong đợi thấy: `postgres: walsender replicator 192.168.0.11(...) streaming 0/...`
 
Trên **replica**:
```bash
ps -eo args | grep -E "walreceiver|wal receiver|startup" | grep -v grep
```
Mong đợi thấy `startup recovering ...` và `walreceiver streaming 0/...`.
 
### 6.2. Xem trạng thái trên master
 
```sql
SELECT pid, usename, application_name, client_addr, state,
       sent_lsn, replay_lsn, sync_state
FROM pg_stat_replication;
```
Cột `state` phải là `streaming`. Cột `sync_state` ban đầu là `async` (bất đồng bộ).
 
### 6.3. Xác nhận replica đang ở chế độ standby
 
Trên **replica**:
```sql
SELECT pg_is_in_recovery();   -- trả về t (true) nghĩa là đang là standby
```
 
### 6.4. Test dữ liệu thực tế
 
Trên **master**, tạo dữ liệu mẫu:
```sql
CREATE DATABASE demo;
\c demo
CREATE TABLE khach_hang (id serial PRIMARY KEY, ten text);
INSERT INTO khach_hang (ten) VALUES ('Nguyen Van A'), ('Tran Thi B');
```
 
Trên **replica**, đọc lại (chờ khoảng 1 giây):
```sql
\c demo
SELECT * FROM khach_hang;
```
Hai dòng vừa tạo ở master phải xuất hiện trên replica. Đây là bằng chứng dữ liệu đã được replicate.
 
Thử ghi trên replica để thấy replica chỉ đọc:
```sql
INSERT INTO khach_hang (ten) VALUES ('Test');
-- ERROR:  cannot execute INSERT in a read-only transaction
```
Lỗi này là bình thường và đúng thiết kế.
 
---
 
## 7. (Tùy chọn) Chuyển sang replication ĐỒNG BỘ (synchronous)
 
Mặc định replication là **bất đồng bộ (async)**: master commit xong là trả kết quả, không chờ replica. Nếu master hỏng đúng lúc, vài giao dịch cuối có thể chưa tới replica.
 
Với **synchronous**, master chỉ báo commit thành công **sau khi replica đã nhận** WAL, nên không mất dữ liệu đã commit, đổi lại chậm hơn một chút.
 
**Bước 1 – trên replica:** đặt `application_name` để master nhận diện replica.
```sql
ALTER SYSTEM SET primary_conninfo =
  'host=192.168.0.10 port=5432 user=replicator password=Doi_Mat_Khau_Manh_Cho_Replica application_name=replica1';
SELECT pg_reload_conf();
```
 
**Bước 2 – trên master:** chỉ định replica đồng bộ.
```sql
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (replica1)';
SELECT pg_reload_conf();
```
 
**Bước 3 – kiểm tra trên master:**
```sql
SELECT application_name, state, sync_state FROM pg_stat_replication;
-- sync_state = sync
```
 
> **Cảnh báo quan trọng:** khi bật synchronous với 1 replica, **nếu replica tắt hoặc mất mạng thì mọi lệnh commit trên master sẽ bị treo** (tôi đã thử nghiệm thấy đúng như vậy). Muốn khẩn cấp cho master chạy lại, tắt chế độ đồng bộ:
> ```sql
> ALTER SYSTEM SET synchronous_standby_names = '';
> SELECT pg_reload_conf();
> ```
> Production nên dùng từ 2 replica trở lên (ví dụ `ANY 1 (replica1, replica2)`) để không bị treo khi mất một con.
 
---
 
## 8. Giám sát replication
 
### Độ trễ (lag) trên master
```sql
SELECT application_name,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```
 
### Độ trễ trên replica
```sql
SELECT now() - pg_last_xact_replay_timestamp() AS replay_delay;
SELECT status, sender_host, slot_name FROM pg_stat_wal_receiver;
```
 
### Replication slot trên master (tránh đầy ổ đĩa)
```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_giu_lai
FROM pg_replication_slots;
```
Nếu một slot `active = f` lâu ngày mà `wal_giu_lai` tăng mãi, hãy khôi phục replica hoặc xóa slot không dùng:
```sql
SELECT pg_drop_replication_slot('replica1_slot');
```
 
---
 
## 9. Chuyển replica thành master (promote)
 
Khi master gặp sự cố và cần cho replica nhận vai trò chính (thao tác trên **replica**):
```sql
SELECT pg_promote();
```
hoặc:
```bash
sudo -u postgres pg_ctlcluster 16 main promote          # Ubuntu / Debian
sudo -u postgres /usr/pgsql-16/bin/pg_ctl -D /var/lib/pgsql/16/data promote   # RHEL
```
Sau đó `SELECT pg_is_in_recovery();` trả về `f`, replica đã ghi được (tôi đã chạy thử). Việc đưa master cũ trở lại làm replica cần dựng lại hoặc dùng `pg_rewind`, và việc tự động failover cần công cụ như Patroni, repmgr. Các phần này nằm ngoài phạm vi tài liệu.
 
---
 
## 10. Lỗi thường gặp
 
| Hiện tượng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| `pg_basebackup: could not connect to server` | Master chưa `listen_addresses = '*'`, firewall chặn 5432, hoặc chưa restart | Kiểm tra mục 4 bước 2, mở firewall, `telnet 192.168.0.10 5432` |
| `no pg_hba.conf entry for replication connection` | Thiếu hoặc sai dòng `replication` trong `pg_hba.conf` hoặc sai IP | Sửa IP/user trong dòng `host replication ...`, rồi `pg_reload_conf()` |
| `password authentication failed for user "replicator"` | Sai mật khẩu, hoặc kiểu mã hóa không khớp (md5 và scram) | Đặt lại mật khẩu, dùng đúng `scram-sha-256`/`md5` trong `pg_hba.conf` |
| `directory ... exists but is not empty` | `PGDATA` trên replica chưa được làm rỗng | Dừng service, xóa nội dung `PGDATA` (bước 4) |
| `database system identifier differs between the primary and standby` | Replica không được tạo từ `pg_basebackup` của chính master này (đã `initdb` riêng) | Làm lại từ bước 4: xóa `PGDATA` và chạy lại `pg_basebackup` |
| `requested WAL segment ... has already been removed` | Replica tụt quá xa, master đã xóa WAL | Dùng replication slot hoặc tăng `wal_keep_size`, hoặc dựng lại replica bằng `pg_basebackup` |
| Không thấy dòng nào trong `pg_stat_replication` | Replica chưa chạy, hoặc `standby.signal`/`primary_conninfo` chưa có | Xem log của cả hai bên, kiểm tra lại bước 5 và 6 |
| Master đầy ổ đĩa vì WAL | Slot không hoạt động giữ WAL | Mục 8, xử lý replica hoặc xóa slot |
 
---
 
## 11. Khác biệt so với bài Percona (đối chiếu nhanh)
 
| Nội dung | Bài Percona (cũ) | Tài liệu này (PG12+) |
|---|---|---|
| File cấu hình standby | `recovery.conf` (`standby_mode = 'on'`) | `standby.signal` + `primary_conninfo` trong `postgresql.auto.conf` |
| Giữ WAL | `wal_keep_segments` | `wal_keep_size` |
| `wal_level` | `hot_standby` | `replica` |
| Copy dữ liệu | `pg_basebackup ... -Xs -R` | Giữ nguyên, thêm `-C -S` để tạo slot |
| Archive WAL | Đặt `archive_mode`, `archive_command`, `restore_command` làm bước chuẩn | Không bắt buộc cho streaming; nên có trên production; slot giúp giảm rủi ro mất WAL |
| Trạng thái đồng bộ | Mặc định async | Thêm mục 7 để chuyển sang sync |
 
---
 
## 12. Tóm tắt các bước (checklist)
 
**Master**
1. `CREATE ROLE replicator WITH REPLICATION LOGIN ...`
2. Đặt `listen_addresses`, `wal_level`, `max_wal_senders`, `max_replication_slots`, `wal_keep_size`, rồi restart.
3. Thêm dòng `host replication replicator <IP replica>/32 scram-sha-256` vào `pg_hba.conf`, rồi reload.
**Replica**
4. Dừng service và làm rỗng `PGDATA`.
5. Chạy `pg_basebackup -h <IP master> -U replicator -D <PGDATA> -X stream -P -R -C -S replica1_slot`.
6. Khởi động service.
 
**Kiểm tra**
7. `pg_stat_replication` trên master có `state = streaming`.
8. `pg_is_in_recovery()` trên replica trả `t`.
9. Ghi dữ liệu ở master, đọc được ở replica.
 




