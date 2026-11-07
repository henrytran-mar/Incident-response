# CHEATSHEET GỠ RỐI — NGƯỜI 2 (PHÒNG CHỐNG & KHÔI PHỤC)

> Dành cho người **lần đầu** dùng Scapy detector + static ARP (SEED) và DAI/DHCP Snooping (Packet Tracer).
> Dạng: Triệu chứng → Chẩn đoán → Sửa. Thấy lỗi ở đâu nhảy tới mục đó.

---

## 🔑 LỆNH VÀNG

```bash
# Trong SEED:
arp -n                  # xem cache + cờ PERM
ip neigh                # cách khác xem ARP (Linux mới)
ip addr                 # tên interface đúng chưa (eth0?)
# Trong Packet Tracer (CLI switch):
show ip dhcp snooping binding       # bảng binding có dữ liệu chưa?
show ip arp inspection              # DAI bật trên VLAN nào, cổng trust/untrust
show ip arp inspection statistics   # cột Dropped tăng = đang chặn
```

---

## A. TẦNG 1 — DETECTOR SCAPY (arp_detector.py)

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Chạy nhưng **không in gì** | Không bắt được ARP trên iface này | Kiểm tra `IFACE`/filter; chạy detector **trên máy nạn nhân** (A hoặc B), không phải attacker |
| `ModuleNotFoundError: scapy` | Thiếu Scapy | `pip3 install scapy` trong container đang chạy detector |
| `Operation not permitted` khi sniff | Thiếu quyền | Chạy `sudo python3 arp_detector.py` (trong container thường đã root) |
| Không thấy cảnh báo dù đang bị tấn công | Detector học MAC giả **trước** khi học MAC thật | Khởi động detector **trước**, để nó học baseline sạch, **rồi** mới cho Người 1 tấn công |
| Cảnh báo quá nhiều/nhiễu | Mạng có ARP hợp lệ thay đổi | Chỉ xét `op==2`; ghi log ra file để lọc; tập trung cặp IP 10.9.0.5/.6 |

---

## B. TẦNG 1 — STATIC ARP (ngăn chặn)

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `arp -s` báo lỗi / không có lệnh | Thiếu `net-tools` | `apt install -y net-tools`; hoặc dùng `ip neigh add 10.9.0.6 lladdr <MAC> dev eth0 nud permanent` |
| Gắn xong nhưng **không thấy cờ PERM** | Gắn sai cú pháp/iface | Kiểm tra lại: `arp -s <IP> <MAC>`; xác nhận bằng `arp -n` cột Flags = `PERM` |
| Dùng **sai MAC** (MAC của M chứ không phải MAC thật) | Lấy MAC lúc đã bị đầu độc | Lấy MAC **trước** khi tấn công, hoặc từ `arp-scan`, hoặc do Người 1 cung cấp MAC gốc |
| Static ARP rồi mà cache **vẫn đổi** | Entry không thực sự permanent | Xác nhận PERM; trên Linux mới ưu tiên `ip neigh ... nud permanent` |
| Sau static ARP, A vẫn không nối được B | Gắn nhầm IP↔MAC chéo | Trên A gắn MAC của **B**; trên B gắn MAC của **A** — đừng gắn ngược |
| Muốn chứng minh "tấn công thất bại" nhưng không rõ | Thiếu đối chứng | Bắt PCAP mới trên attacker M lúc có static ARP → M **không** còn nhận traffic A↔B = thành công |

---

## C. TẦNG 1 — KHÔI PHỤC

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `ip neigh flush all` xong vẫn thấy MAC giả | Script đầu độc còn chạy | Dừng `arp_poison.py` trên attacker trước, rồi flush |
| MITM vẫn còn dù đã flush | `ip_forward` còn bật trên M | Trên attacker: `sysctl -w net.ipv4.ip_forward=0` |

---

## D. PACKET TRACER — TOPOLOGY & DHCP

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Switch không có lệnh `ip dhcp snooping` | Model switch không hỗ trợ | Dùng **3560 (Multilayer Switch)**, không dùng switch 2950 đời cũ |
| PC không nhận IP DHCP | Server DHCP chưa bật/pool sai | Bật DHCP service trên Server; kiểm tra pool, gateway; PC để chế độ **DHCP** |
| Link giữa thiết bị màu đỏ (down) | Sai loại cáp | PC↔Switch & Server↔Switch dùng **Copper Straight-Through** |
| `show ip dhcp snooping binding` **trống** | PC chưa nhận IP qua DHCP (đang để Static) | Cho PC xin DHCP lại (Desktop → IP Config → Static→DHCP) để binding tự điền |

---

## E. PACKET TRACER — DHCP SNOOPING / DAI

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Sau khi bật snooping, PC **mất IP/không xin được** | Cổng Server để untrusted → DHCP offer bị chặn | Đặt cổng nối Server: `ip dhcp snooping trust` (và `ip arp inspection trust`) |
| Bật DAI xong **tất cả** ARP bị chặn kể cả hợp lệ | Binding table trống hoặc cổng hợp lệ chưa trust | Đảm bảo binding có dữ liệu (D ở trên); trust đúng cổng uplink |
| Gửi ARP giả nhưng cột **Dropped không tăng** | DAI chưa áp đúng VLAN / cổng attacker đang trust | `ip arp inspection vlan 1`; cổng attacker phải **untrusted** (mặc định) |
| Không biết DAI có hoạt động không | Thiếu lệnh kiểm tra | `show ip arp inspection statistics` xem Dropped; `show ip arp inspection interfaces` xem trust state |
| Lệnh gõ bị từ chối | Chưa vào enable/config mode | `enable` → `configure terminal` trước khi cấu hình |
| PT bản cũ không có DAI | Phiên bản Packet Tracer cũ | Dùng PT ≥ 7.x/9.x; nếu vẫn thiếu, mô tả cấu hình + nêu rõ hạn chế mô phỏng trong báo cáo |

---

## F. BẰNG CHỨNG & BÀN GIAO

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Ảnh chụp không rõ đang chặn gì | Thiếu lệnh show kèm theo | Luôn chụp **kèm** `show ip arp inspection statistics` (có số Dropped) |
| Người 3 không hiểu bằng chứng | Thiếu chú thích | Ghi rõ mỗi ảnh: biện pháp gì, thuộc pha NIST nào, chứng minh điều gì |
| File `.pkt` mở máy khác bị lỗi | Khác phiên bản PT | Ghi rõ phiên bản PT đã dùng; xuất thêm ảnh để không phụ thuộc file |

---

## 📌 3 LỖI "KINH ĐIỂN" CỦA NGƯỜI MỚI

1. **Quên trust cổng DHCP Server** → bật snooping là cả mạng mất DHCP. (Mục E)
2. **Lấy MAC để static ARP lúc đã bị đầu độc** → gắn nhầm MAC của kẻ tấn công. Lấy MAC sạch trước. (Mục B)
3. **Chạy detector sau khi tấn công đã bắt đầu** → nó học nhầm MAC giả làm baseline, không báo động. Chạy detector trước. (Mục A)
