# CHEATSHEET GỠ RỐI — NGƯỜI 1 (TẤN CÔNG & XUẤT PCAP)

> Dành cho người **lần đầu** dùng Docker/SEED/Scapy. Mỗi máy mỗi khác → đây là bộ "sửa nhanh" theo triệu chứng.
> Cách dùng: thấy lỗi ở đâu thì nhảy tới mục đó. Luôn làm theo thứ tự **Chẩn đoán → Sửa**.

---

## 🔑 LỆNH VÀNG — chạy khi "không biết lỗi ở đâu"

```bash
dockps                      # 3 container còn sống không?
docker ps -a                # container nào chết/exit? (xem cột STATUS)
ip addr                     # (trong container) tên interface + IP đúng chưa?
arp -n                      # bảng ARP hiện tại
sudo service docker status  # (hoặc) systemctl status docker — Docker còn chạy?
```

**Quy tắc reset khi bế tắc:** `dcdown` → `docker network prune -f` → `dcup`. 80% sự cố "lạ" biến mất sau bước này.

---

## A. DOCKER KHÔNG CHẠY / KHÔNG BUILD ĐƯỢC

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `Cannot connect to the Docker daemon` | Daemon chưa chạy | `sudo systemctl start docker` — WSL cũ: `sudo service docker start` |
| `permission denied ... docker.sock` | User chưa thuộc group `docker` | `sudo usermod -aG docker $USER` rồi **đăng xuất/đăng nhập lại**, hoặc `newgrp docker` |
| `dcbuild` lỗi tải gói / timeout | Mạng chập chờn lúc build | Chạy lại `dcbuild`; đổi mạng; hoặc `docker-compose build --no-cache` |
| `no space left on device` | Hết dung lượng Docker | `docker system prune -a` (xóa image/cache cũ) rồi build lại |
| (WSL) gõ `docker` không phản hồi | WSL không có systemd | `sudo service docker start`; kiểm tra `docker info` |

---

## B. CONTAINER KHÔNG LÊN ĐỦ 3 / TỰ TẮT

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `dockps` chỉ thấy 1–2 container | Một container exit ngay | `docker ps -a` xem STATUS → `docker logs <id>` đọc lỗi |
| `dcup` báo **port is already allocated** | Lần chạy trước chưa tắt sạch | `dcdown` → `docker network prune -f` → `dcup` |
| `network ... already exists` | Mạng Docker cũ còn kẹt | `docker network prune -f` rồi `dcup` |
| Container "Exited (0)" ngay sau khi lên | Bình thường nếu entrypoint xong — nhưng SEED cần container sống | Chạy `dcup` (không `-d`) ở 1 terminal và **giữ nguyên**, mở terminal khác để thao tác |
| Lỡ đóng terminal chạy `dcup` | Lab bị tắt | Mở lại terminal, `cd ~/seed-arp/Labsetup`, `dcup` |

---

## C. VÀO CONTAINER / MẠNG GIỮA CÁC MÁY

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `docksh <id>` báo không tìm thấy | Sai ID hoặc dùng tên thay ID | Lấy ID từ `dockps`; hoặc `docker exec -it <ID> /bin/bash` |
| Trong container không có `eth0` | Interface tên khác (vd `eth0@if..`) | `ip addr` lấy đúng tên → thay vào **mọi** lệnh có `-i eth0` và biến `IFACE` trong script |
| Attacker **ping không tới** A/B | Container chưa cùng mạng / chưa lên đủ | `dockps` đủ 3 chưa; `dcdown`+`dcup`; kiểm tra IP bằng `ip addr` |
| IP không phải 10.9.0.x | docker-compose bản khác dải IP | Mở `docker-compose.yml` xem dải mạng thật → cập nhật IP_A/IP_B trong script cho khớp |

---

## D. SCAPY / SCRIPT ĐẦU ĐỘC ARP

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `ModuleNotFoundError: scapy` | Container này chưa có Scapy | **Chạy script trong container ATTACKER** (nơi có sẵn); nếu thiếu: `pip3 install scapy` |
| `getmacbyip()` trả về `None` | M chưa biết MAC của A/B | Trong attacker: `ping -c2 10.9.0.5` và `ping -c2 10.9.0.6` trước, rồi chạy lại script |
| `Operation not permitted` khi gửi gói | Thiếu quyền raw socket | Chạy bằng `sudo python3 arp_poison.py` (trong container attacker thường đã là root) |
| Script chạy nhưng **ARP nạn nhân không đổi** | Sai interface, hoặc gói không tới | Kiểm tra `IFACE` đúng tên (`ip addr`); giảm `time.sleep(2)` xuống `1`; xác nhận `op=2` |
| ARP đổi được **1 chiều** | Chỉ 1 trong 2 gói tới đích | Kiểm tra cả `pkt_to_A` và `pkt_to_B`; MAC_A/MAC_B không `None` |
| Đổi xong vài giây lại về cũ | Nạn nhân tự làm mới ARP | Giữ script **chạy liên tục** (vòng while) — đừng tắt giữa chừng |
| `No such device` khi `sendp` | Tên iface sai | `ip addr` → sửa `IFACE` |

---

## E. MITM "ĐỨT MẠNG" (A và B mất kết nối sau khi đầu độc)

> Đây là lỗi **hay gặp nhất** và khiến tấn công "lộ".

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Đầu độc xong, A telnet/curl tới B bị treo/timeout | M chưa chuyển tiếp gói | Bật forwarding: `sysctl -w net.ipv4.ip_forward=1` → kiểm tra `cat /proc/sys/net/ipv4/ip_forward` = `1` |
| Bật forwarding rồi vẫn đứt | Script đầu độc đã dừng | Mở lại, giữ `arp_poison.py` **chạy suốt** trong lúc sinh traffic |
| Kết nối chập chờn | Tần suất đầu độc thấp | Giảm `sleep` còn 1s để cache nạn nhân luôn bị ghi đè |

---

## F. TELNET (Bước 5A)

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `telnet: command not found` trên Host A | Thiếu client | `apt update && apt install -y telnet` trong container A |
| `Connection refused` tới 10.9.0.6 | B chưa có telnet server | Thường SEED có sẵn; nếu không: dùng kịch bản HTTP thay thế (Bước 5B vẫn đủ bằng chứng cleartext) |
| Login không vào | Sai tài khoản | Thử `seed`/`dees`; nếu khác, xem `docker-compose.yml` hoặc README lab |
| Telnet vào được nhưng **PCAP không thấy** | tcpdump chạy sai interface / chưa chạy lúc login | Đảm bảo tcpdump đang chạy **trước** khi telnet; đúng `-i eth0` |

---

## G. HTTP / login_server.py (Bước 5B)

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `Address already in use` khi chạy server | Cổng 8080 đang bận | Đổi cổng trong script (vd 8000) và trong lệnh `curl`, hoặc tắt tiến trình cũ |
| `curl: Connection refused` | Server chưa chạy trên B / sai IP | Chạy `login_server.py` trên **Host B** trước; curl đúng `http://10.9.0.6:8080/login` |
| `curl: command not found` | Thiếu curl trên A | `apt install -y curl`; hoặc dùng `wget --post-data=...` |
| POST gửi được nhưng PCAP không thấy password | Traffic không qua M | Xác nhận đầu độc còn chạy + `ip_forward=1`; bắt đúng interface |

---

## H. BẮT & KIỂM TRA PCAP

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| File `.pcap` trống (0 byte / vài gói) | tcpdump sai interface hoặc bắt quá ngắn | Dùng đúng `-i <iface>`; để tcpdump chạy xuyên suốt bước sinh traffic |
| PCAP quá lớn, khó mở | Bắt cả nhiễu | Lọc khi bắt: `tcpdump -i eth0 'host 10.9.0.5 or host 10.9.0.6' -w file.pcap` |
| Không thấy `arp_mitm.pcap` trên host | Chưa ghi vào `/volumes` | tcpdump phải ghi vào `-w /volumes/arp_mitm.pcap` (thư mục mount chung) |
| Mở Wireshark không thấy telnet/password | Lọc sai | Lọc `telnet` → chuột phải → **Follow → TCP Stream** |
| Không thấy POST | Lọc sai | `http.request.method == "POST"` → mở gói → **HTML Form URL Encoded** |
| Không có Wireshark trên máy | — | Dùng `tshark -r file.pcap -Y 'telnet'` ; hoặc copy PCAP sang máy có Wireshark |

---

## I. COPY FILE RA THƯ MỤC ĐỒ ÁN

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `cp` báo `No such file or directory` (đích) | Sai đường dẫn Windows/WSL | Đường dẫn có dấu cách → **bọc trong nháy kép** cả path; kiểm tra `ls "/mnt/c/Users/..."` |
| File copy ra nhưng Windows không thấy | Chưa đúng ổ `/mnt/c` | Trong WSL ổ C là `/mnt/c/...`; mở lại Explorer để refresh |
| Permission denied khi copy | Quyền thư mục | Thêm `sudo` cho lệnh `cp`, hoặc copy về `~` rồi kéo qua |

---

## ⏱️ QUY TRÌNH SỐNG SÓT (khi hoảng, làm đúng thứ tự này)

1. 3 shell attacker: **(1)** tcpdump chạy, **(2)** arp_poison chạy, **(3)** rảnh để gõ lệnh.
2. `ip_forward=1` đã bật.
3. Sang A: `arp -n` thấy MAC của M → OK. Rồi telnet + curl.
4. Quay lại attacker: Ctrl+C tcpdump → có file.
5. Mở PCAP, lọc `telnet` và `http.request.method=="POST"` → thấy mật khẩu → **xong**.
6. Nếu 1 bước fail → tra đúng mục A–I ở trên, **không đập cả lab làm lại**.

---

## 📌 3 LỖI "KINH ĐIỂN" CỦA NGƯỜI MỚI (nhớ kỹ)

1. **Quên bật `ip_forward=1`** → A/B đứt mạng, tấn công lộ. (Mục E)
2. **Tắt script đầu độc quá sớm** → ARP cache nạn nhân về cũ, không bắt được gì. Giữ nó chạy suốt. (Mục D)
3. **tcpdump sai interface hoặc chạy sau khi đã login** → PCAP rỗng. Bắt trước, sinh traffic sau. (Mục H)
