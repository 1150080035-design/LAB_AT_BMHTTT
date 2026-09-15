**Đoàn Trung Tấn**

**1150080035**

**Lab 1.2 - Phân tích tính bảo mật giữa Telnet và SSH bằng kỹ thuật Sniffing (Wireshark)**

### 1.Nội dung đã thực hiện

#### Bước 1:Cấu hình địa chỉ IP và môi trường mạng trên 3 máy ảo Kali Linux

* Thiết lập card mạng cả 3 máy ảo trên VMware Workstation về cùng mạng nội bộ **VMnet1 (Custom/Host-only)** để đảm bảo thông suốt tầng liên kết dữ liệu.
* **Máy Kali\_Server (10.0.0.1):**

  * Gán địa chỉ IP tĩnh: `sudo ip addr add 10.0.0.1/24 dev eth0`
  * Bật giao diện mạng: `sudo ip link set eth0 up`
* **Máy Kali\_Client (10.0.0.2):**

  * Gán địa chỉ IP tĩnh: `sudo ip addr add 10.0.0.2/24 dev eth0`
  * Bật giao diện mạng: `sudo ip link set eth0 up`
* **Máy Kali\_Attacker (10.0.0.3):**

  * Kiểm tra giao diện mạng `eth0` đảm bảo cùng thuộc dải mạng `10.0.0.0/24`.
* **Kiểm tra kết nối:**

  * Chạy lệnh `ping -c 2 10.0.0.1` từ máy Client để xác nhận mạng đã thông suốt (báo `64 bytes from 10.0.0.1`).

#### Bước 2:Khởi chạy các dịch vụ trên máy Server (10.0.0.1)

* **Khởi chạy Telnet Server giả lập (Port 23):**

  * Do máy không có Internet để cài `telnetd`, sử dụng Python script để lắng nghe cổng 23 và tạo phản hồi đăng nhập:

&#x20;   ```bash
    sudo python3 -c "import socket; s=socket.socket(); s.setsockopt(socket.SOL\_SOCKET, socket.SO\_REUSEADDR, 1); s.bind(('0.0.0.0', 23)); s.listen(1); print('=== TELNET SERVER READY ==='); conn, addr = s.accept(); conn.send(b'Login: '); user = conn.recv(1024); conn.send(b'Password: '); pwd = conn.recv(1024); conn.send(b'Success\\n'); conn.close()"

* Màn hình hiển thị `=== TELNET SERVER READY ===` sẵn sàng tiếp nhận kết nối.
* **Khởi chạy SSH Server (Port 22):**

  * Mở tab Terminal thứ hai trên Server và khởi động lại dịch vụ OpenSSH: `sudo systemctl restart ssh`

#### Bước 3:Khởi động công cụ bắt gói tin trên máy Attacker (10.0.0.3)

* Mở công cụ bắt gói tin Wireshark với quyền root: `sudo wireshark`
* Lựa chọn card mạng **`eth0`** và bắt đầu quá trình ghi lưu lượng mạng (Packet Capture).

#### Bước 4:Thực hiện kết nối dịch vụ từ máy Client (10.0.0.2)

* **Kết nối thử nghiệm Telnet (Văn bản rõ):**

  * Chạy lệnh: `nc 10.0.0.1 23`
  * Nhập tài khoản **Login**: `trungtan`
  * Nhập mật khẩu **Password**: `123`
  * Nhận được phản hồi `Success` từ Server.
* **Kết nối thử nghiệm SSH (Mã hóa):**

  * Chạy lệnh: `ssh kali@10.0.0.1`
  * Nhập `yes` để xác nhận thêm fingerprint của Server vào file `known\_hosts`.
  * Tiến hành nhập mật khẩu tài khoản hệ thống `kali` để thực hiện quá trình bắt tay và xác thực SSH.

### 2.Kết quả thực hiện

* **Đối với dịch vụ Telnet (Port 23):**

  * Lọc gói tin trên Wireshark bằng cú pháp: `tcp.port == 23`
  * Nhấp chuột phải vào gói tin bất kỳ $\\rightarrow$ Chọn **Follow** $\\rightarrow$ **TCP Stream**.
  * **Kết quả:** Đoạn thoại hiển thị rõ ràng tài khoản `trungtan` và mật khẩu `123` dạng văn bản rõ (**Clear-text**).
  * **Đánh giá:** Telnet không bảo mật, dễ dàng bị thu thập tài khoản/mật khẩu bằng kỹ thuật Sniffing đơn giản.
* **Đối với dịch vụ SSH (Port 22):**

  * Lọc gói tin trên Wireshark bằng cú pháp: `ssh`
  * Nhấp chuột phải vào gói tin bất kỳ $\\rightarrow$ Chọn **Follow** $\\rightarrow$ **TCP Stream**.
  * **Kết quả:** Toàn bộ dữ liệu hiển thị là các chuỗi ký tự mã hóa vô nghĩa (**Encrypted Payload**). Hoàn toàn không tìm thấy chuỗi mật khẩu hay thông tin đã nhập.
  * **Đánh giá:** SSH mã hóa toàn bộ dữ liệu end-to-end, vô hiệu hóa khả năng đọc lén gói tin của kẻ tấn công trong cùng mạng nội bộ.

### 3\.Lưu ý cần thiết

* **Lưu ý về Card mạng:**

  * Cần đảm bảo cả 3 máy ảo (`Server`, `Client`, `Attacker`) chọn đúng chung 1 card mạng **VMnet1** trong mục **Virtual Machine Settings** của VMware.
* **Lưu ý về địa chỉ IP:**

  * Nếu lệnh `sudo ip addr add` báo lỗi `Address already assigned`, chạy lệnh `sudo ip link set eth0 up` để bật giao diện mạng vì IP đã được gán sẵn.
  * Nên sử dụng Subnet Mask `/24` (ví dụ `10.0.0.1/24`) thay vì `/8` để tránh xung đột bảng định tuyến.
* **Lưu ý về Port 23 (Telnet):**

  * Nếu chạy lại lệnh Python mà gặp lỗi `Address already assigned (Errno 98)`, cần chạy lệnh `sudo fuser -k 23/tcp` trên máy Server để tắt tiến trình cũ đang chiếm giữ cổng 23.
* **Lưu ý khi kết nối SSH:**

  * Khi xuất hiện thông báo `Are you sure you want to continue connecting (yes/no/\[fingerprint])?`, bắt buộc phải gõ **`yes`** và nhấn Enter.
  * Dù đăng nhập thành công hay báo `Permission denied` (do gõ sai pass), gói tin trao đổi khóa SSH vẫn được tạo đầy đủ trên Wireshark để kiểm tra tính chất mã hóa.
* **Cách thao tác nhanh trên Wireshark tại máy Attacker:**

  * Bắt gói Telnet: Nhập Filter `tcp.port == 23` $\\rightarrow$ Chuột phải gói tin $\\rightarrow$ **Follow** $\\rightarrow$ **TCP Stream**.
  * Bắt gói SSH: Nhập Filter `ssh` $\\rightarrow$ Chuột phải gói tin $\\rightarrow$ **Follow** $\\rightarrow$ **TCP Stream**.

