# Hướng dẫn cài đặt VMware ESXi 7.0 trên máy chủ vật lý bằng USB Boot

---

## 1. Yêu cầu chuẩn bị

| Thành phần | Yêu cầu |
|---|---|
| Phần cứng server | CPU 64-bit hỗ trợ ảo hóa (Intel VT-x/AMD-V), tối thiểu 4GB RAM (khuyến nghị 8GB+), CPU/mainboard nằm trong [VMware HCL](https://www.vmware.com/resources/compatibility/search.php) |
| Ổ USB | Dung lượng ≥ 8GB, dùng để tạo boot installer |
| File ISO | `VMware-VMvisor-Installer-7.0*.iso` (tải từ trang chủ VMware hoặc kho ISO) |
| Công cụ tạo USB boot | Rufus (Windows) hoặc `dd`/Etcher (Linux/macOS) |
| Ổ đĩa cài đặt | Ổ cứng/SSD trống trên server (tối thiểu 8GB, khuyến nghị ≥ 32GB để có chỗ cho log, scratch partition) |

---

## 2. Tạo USB Boot cài đặt ESXi

### Cách 1: Dùng Rufus (Windows)
1. Cắm USB vào máy tính, mở **Rufus**.
2. **Device:** chọn đúng USB của bạn.
3. **Boot selection:** chọn file ISO ESXi 7.0 vừa tải.
4. **Partition scheme:** MBR (cho BIOS/Legacy) hoặc GPT (cho UEFI) — chọn theo chế độ boot của server.
5. Nhấn **START**, xác nhận ghi đè dữ liệu USB (dữ liệu cũ trên USB sẽ bị xóa).
6. Chờ Rufus hoàn tất ghi ISO ra USB.

### Cách 2: Dùng lệnh `dd` (Linux/macOS)
```bash
# Xác định đúng thiết bị USB, ví dụ /dev/sdX (THAY ĐÚNG THIẾT BỊ, tránh ghi nhầm ổ đĩa)
sudo dd if=VMware-VMvisor-Installer-7.0.iso of=/dev/sdX bs=4M status=progress conv=fsync
```
> ⚠️ Kiểm tra kỹ tên thiết bị (`lsblk` hoặc `diskutil list`) trước khi chạy `dd`, ghi sai ổ có thể mất dữ liệu.

---

## 3. Cấu hình BIOS/UEFI trên server

1. Khởi động server, vào BIOS/UEFI Setup (thường phím **Del**, **F2**, **F10**, **F12** tùy hãng).
2. Bật **Virtualization Technology** (Intel VT-x / AMD-V).
3. Đặt **Boot mode** phù hợp (UEFI khuyến nghị cho ESXi 7).
4. Đặt **Boot Order** ưu tiên USB lên đầu, hoặc dùng phím tắt Boot Menu (thường **F11**/**F12**) để chọn boot từ USB.
5. Cắm USB installer vào cổng USB server, khởi động lại.

---

## 4. Tiến hành cài đặt ESXi 7

### Bước 1 – Boot vào USB
Server nhận diện USB và hiển thị màn hình **Loading ESXi installer**.

### Bước 2 – Nhấn Enter để tiến hành cài đặt
Ở màn hình Welcome, nhấn **Enter** để tiếp tục.

### Bước 3 – Nhấn F11 để chấp nhận điều khoản User
Đồng ý điều khoản EULA để tiếp tục.

### Bước 4 – Chọn ổ đĩa cài đặt
Trình cài đặt quét và liệt kê các ổ đĩa có trên server. Chọn ổ đĩa muốn cài ESXi 7 lên → **Enter**.
> ⚠️ Ổ đĩa được chọn sẽ bị format hoàn toàn, kiểm tra kỹ tránh chọn nhầm ổ chứa dữ liệu quan trọng.

### Bước 5 – Chọn bàn phím & nhập mật khẩu root
- Chọn layout bàn phím phù hợp.
- Nhập và xác nhận mật khẩu root (tối thiểu 7 ký tự, kết hợp chữ hoa/thường/số/ký tự đặc biệt).

### Bước 6 – Xác nhận cài đặt, nhấn F11 để bắt đầu
Xem lại thông tin cài đặt, nhấn **F11** để bắt đầu quá trình ghi dữ liệu lên ổ đĩa.

### Bước 7 – Chờ quá trình cài đặt
Chương trình copy các thành phần hệ thống lên ổ đĩa, mất vài phút.

### Bước 8 – Reboot
Khi cài đặt hoàn tất, **rút USB installer ra khỏi server**, nhấn **Enter** để khởi động lại.

---

## 5. Cấu hình mạng quản trị (Management Network)

### Bước 9-10 – Truy cập màn hình cấu hình (DCUI)
Tại màn hình console vàng-đen (Direct Console User Interface), nhấn **F2**, nhập mật khẩu root.

### Bước 11 – Configure Management Network
Chọn **Configure Management Network** → **Enter** để vào cấu hình mạng.

Trong menu này, bạn có thể chọn:
- **Network Adapters:** chọn card mạng vật lý dùng cho quản trị (nếu server có nhiều NIC)
- **VLAN (optional):** thiết lập VLAN ID nếu hạ tầng dùng VLAN

### Bước 12 – IPv4 Configuration
Chọn **IPv4 Configuration** → **Enter**.

### Bước 13 – Thiết lập địa chỉ IP tĩnh
Chọn **Set static IPv4 address and network configuration**, nhập:
- IP Address (địa chỉ IP quản trị của ESXi host)
- Subnet Mask
- Default Gateway

### Bước 14 – Cấu hình DNS
Chọn **DNS Configuration**, nhập:
- Primary/Secondary DNS Server
- Hostname (FQDN nếu có)

### Bước 15 – Lưu và Reboot
Nhấn **Esc** để thoát menu, sau đó nhấn **Y** để xác nhận áp dụng thay đổi — server sẽ restart lại dịch vụ mạng.

---

## 6. Truy cập ESXi Host Client và gán License

### Bước 16 – Truy cập web console
Từ máy tính khác cùng mạng, mở trình duyệt, truy cập:
```
https://<IP-ESXi-vua-cau-hinh>/
```
Đăng nhập bằng user `root` và mật khẩu đã thiết lập.

### Bước 17 – Assign license
1. Vào **Manage → Licensing → Assign license**.
2. Nhập license key hợp lệ của bạn → **Check license** → **Assign**.

> ⚠️ **Về bản quyền:** Mặc định sau khi cài, ESXi chạy chế độ **Evaluation** đầy đủ tính năng trong 60 ngày. Sau đó cần gán license hợp lệ để tiếp tục sử dụng: dùng bản **vSphere Hypervisor (Free license)** đăng ký miễn phí tại trang chủ VMware (giới hạn tính năng so với bản trả phí), hoặc mua license thương mại (Essentials, Standard, Enterprise Plus...) tùy nhu cầu sử dụng thực tế. Tránh dùng license key chia sẻ/crack không rõ nguồn gốc vì có thể vi phạm bản quyền và tiềm ẩn rủi ro bảo mật.

---

## 7. Checklist sau khi cài đặt

- [ ] Kiểm tra ping tới IP quản trị ESXi từ máy khác trong mạng
- [ ] Truy cập được Host Client qua HTTPS
- [ ] Đã đổi/lưu trữ an toàn mật khẩu root
- [ ] Đã gán license (evaluation/free/paid) phù hợp
- [ ] Đã cấu hình NTP time sync (Manage → System → Time & Date)
- [ ] Đã kiểm tra và cấu hình thêm datastore nếu có nhiều ổ đĩa

---

## Tài liệu tham khảo
- [congdonglinux.com – Hướng dẫn Cài Đặt Esxi 7.0](https://congdonglinux.com/huong-dan-cai-dat-esxi-7-0-full-license/)
- [Tổng hợp link download VMware ESXi ISO](https://congdonglinux.com/tong-hop-link-download-esxi-iso/)
- [VMware Compatibility Guide (HCL)](https://www.vmware.com/resources/compatibility/search.php)
