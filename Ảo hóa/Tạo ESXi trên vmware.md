# Hướng dẫn cài đặt VMware ESXi 7.0 trên VMware Workstation (Nested VM)

---

## 1. Yêu cầu chuẩn bị

| Thành phần | Yêu cầu tối thiểu |
|---|---|
| Phần mềm ảo hóa | VMware Workstation Pro 15.5+ / Player 15.5+ (bản mới hỗ trợ tốt nested ESXi) |
| CPU | Hỗ trợ ảo hóa phần cứng (Intel VT-x/EPT hoặc AMD-V), bật **Virtualization Engine** trong VM Settings |
| RAM host | Tối thiểu 16GB (cấp cho VM ESXi ít nhất 4GB, khuyến nghị 8GB+) |
| Ổ đĩa | Tối thiểu 40GB dung lượng trống cho ổ đĩa ảo |
| File ISO | Bộ cài `VMware-VMvisor-Installer-7.0*.iso` (tải từ trang chủ VMware hoặc kho lưu trữ ISO) |

 **Lưu ý:** Nested ESXi chỉ dùng để học/test. Không khuyến nghị chạy production trên nested VM vì hiệu năng và một số tính năng (SR-IOV, passthrough...) không hoạt động đầy đủ.

---

## 2. Tạo máy ảo mới trong VMware Workstation

1. Mở **VMware Workstation** → **File** → **New Virtual Machine**.
2. Chọn **Custom (advanced)** → **Next**.
3. Chọn **I will install the operating system later** → **Next**.
4. Ở mục Guest Operating System, chọn:
   - **Guest OS:** VMware ESXi
   - **Version:** VMware ESXi 7
5. Đặt tên VM, ví dụ `ESXi-7.0-Lab`, chọn nơi lưu trữ.
6. Cấu hình phần cứng:
   - **Processors:** tối thiểu 2 core, bật **Virtualize Intel VT-x/EPT or AMD-V/RVI**
   - **Memory:** tối thiểu 4096 MB (khuyến nghị 8192 MB trở lên)
   - **Network:** chọn **Bridged** (để ESXi lấy IP cùng dải mạng LAN, dễ truy cập từ trình duyệt)
   - **Disk:** tạo ổ đĩa mới, dung lượng tối thiểu 40GB, định dạng **SCSI**
7. Sau khi tạo xong, vào **Edit virtual machine settings** → **CD/DVD** → trỏ đến file ISO ESXi 7.0 vừa tải.
8. Bật máy ảo (**Power on this virtual machine**).

---

## 3. Tiến hành cài đặt ESXi 7

### Bước 1 – Boot vào trình cài đặt
Máy ảo sẽ boot từ ISO, hiển thị màn hình **Loading ESXi installer**.

### Bước 2 – Nhấn Enter để bắt đầu cài đặt
Ở màn hình Welcome, nhấn **Enter** để tiếp tục.

### Bước 3 – Chấp nhận điều khoản sử dụng
Nhấn **F11** (hoặc **F1** tùy phiên bản build) để đồng ý điều khoản EULA.

### Bước 4 – Chọn ổ đĩa cài đặt
Trình cài đặt sẽ quét và liệt kê ổ đĩa ảo đã tạo. Chọn ổ đĩa (thường chỉ có 1 ổ SCSI bạn vừa tạo) → **Enter**.

### Bước 5 – Chọn bàn phím & đặt mật khẩu root
- Chọn layout bàn phím (mặc định US Default).
- Nhập mật khẩu root, đủ mạnh (tối thiểu 7 ký tự, có chữ hoa/thường/số).

### Bước 6 – Xác nhận cài đặt
Nhấn **F11** để bắt đầu quá trình cài đặt lên ổ đĩa.

### Bước 7 – Chờ quá trình cài đặt hoàn tất
Quá trình copy và cài đặt hệ thống diễn ra trong vài phút.

### Bước 8 – Khởi động lại
Khi hoàn tất, gỡ ISO ra khỏi ổ CD ảo (**VM → Removable Devices → CD/DVD → Disconnect**) rồi nhấn **Enter** để reboot.

---

## 4. Cấu hình mạng cho ESXi

1. Sau khi khởi động lại, tại màn hình console (DCUI), nhấn **F2**, nhập mật khẩu root vừa tạo.
2. Chọn **Configure Management Network** → **Enter**.
3. Chọn **IPv4 Configuration** → **Enter**.
4. Chọn **Set static IPv4 address and network configuration**, nhập:
   - IP Address
   - Subnet Mask
   - Default Gateway
   (phù hợp với dải mạng Bridged trên máy host)
5. Quay lại menu, chọn **DNS Configuration**, nhập IP DNS Server phù hợp.
6. Nhấn **Esc**, sau đó nhấn **Y** để lưu và áp dụng thay đổi (server sẽ restart network).

---

## 5. Truy cập ESXi qua trình duyệt (Host Client)

1. Trên máy host (hoặc máy cùng mạng), mở trình duyệt, truy cập:
   ```
   https://<IP-ESXi-vua-cau-hinh>/
   ```
2. Đăng nhập với user `root` và mật khẩu đã đặt.
3. Giao diện **VMware Host Client** sẽ hiển thị, từ đây bạn có thể tạo máy ảo, quản lý datastore, network...

---

## 6. Gán License (Assign License)

1. Vào **Manage → Licensing**.
2. Chọn **Assign license**, nhập license key hợp lệ của bạn (mua từ VMware/đại lý, hoặc dùng license **Evaluation Mode** miễn phí 60 ngày mặc định khi chưa gán license).
3. Nhấn **Check license** → **Assign**.

>  **Về bản quyền:** ESXi khi cài mới mặc định chạy ở chế độ **Evaluation** (đầy đủ tính năng, giới hạn 60 ngày) — đủ dùng để học/test. Để dùng lâu dài, hãy dùng key **vSphere Hypervisor (Free)** đăng ký miễn phí trên trang VMware, hoặc mua license thương mại chính hãng. Tài liệu này không cung cấp và không khuyến khích sử dụng license key chia sẻ/crack trái phép.

---

## 7. Một số lưu ý khi chạy ESXi lồng (Nested)

- Bật **Virtualize Intel VT-x/EPT or AMD-V/RVI** trong `VM Settings → Processors` — nếu không bật, ESXi sẽ báo lỗi CPU không hỗ trợ ảo hóa khi tạo VM con bên trong.
- Nếu cần tạo thêm VM Windows/Linux bên trong ESXi lồng này, nên tăng RAM/CPU cấp cho VM ESXi tương ứng.
- Card mạng nên để **Bridged** để dễ truy cập Host Client từ máy thật; nếu dùng NAT cần thêm cấu hình port-forward.

---

## Tài liệu tham khảo
- [congdonglinux.com – Hướng dẫn Cài Đặt Esxi 7.0](https://congdonglinux.com/huong-dan-cai-dat-esxi-7-0-full-license/)
- [Tổng hợp link download VMware ESXi ISO](https://congdonglinux.com/tong-hop-link-download-esxi-iso/)
