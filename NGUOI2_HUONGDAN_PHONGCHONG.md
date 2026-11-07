# NGƯỜI 2 — HƯỚNG DẪN CHI TIẾT: PHÒNG CHỐNG & KHÔI PHỤC

> **Nhiệm vụ của bạn:** sau khi Người 1 chứng minh tấn công thành công, bạn xây dựng lớp **phát hiện — ngăn chặn — khôi phục** theo mô hình **defense-in-depth 2 tầng**, và chứng minh nó thật sự chặn được tấn công.
>
> **Pha NIST phụ trách:** Containment → Eradication → Recovery (một phần Detection).
>
> **Phụ thuộc:** cần Người 1 bàn giao MAC thật của A/B và PCAP tấn công để làm mốc đối chiếu "trước/sau khi phòng chống".

---

## Tổng quan 2 tầng

| Tầng | Môi trường | Chặn được tấn công thật? | Bằng chứng |
|---|---|---|---|
| **Tầng 1 — Host-level** | Trong SEED Docker | ✅ Có (test trực tiếp) | PCAP + ảnh "tấn công thất bại sau khi áp dụng" |
| **Tầng 2 — Switch-level** | Packet Tracer | Mô phỏng (SEED không có switch) | Ảnh cấu hình + log switch drop gói ARP giả |

> **Thông điệp cho báo cáo:** Tầng 1 là biện pháp *host* (khả dụng ngay cả khi không kiểm soát hạ tầng mạng). Tầng 2 là biện pháp *hạ tầng doanh nghiệp* (DAI + DHCP Snooping) — mạnh nhất nhưng cần switch quản lý. Kết hợp = phòng thủ nhiều lớp.

---

# PHẦN A — TẦNG 1: PHÒNG CHỐNG HOST-LEVEL (trong SEED)

## A1. PHÁT HIỆN — Detector ARP spoofing bằng Scapy

Tạo file trên host `~/seed-arp/Labsetup/volumes/arp_detector.py`:

```python
#!/usr/bin/env python3
# arp_detector.py — phát hiện ARP spoofing: cảnh báo khi một IP đổi MAC bất thường
from scapy.all import sniff, ARP
import time

ip_mac = {}     # bảng ánh xạ IP -> MAC đã học

def handle(pkt):
    if pkt.haslayer(ARP) and pkt[ARP].op == 2:      # chỉ xét ARP reply
        ip  = pkt[ARP].psrc
        mac = pkt[ARP].hwsrc
        if ip in ip_mac and ip_mac[ip] != mac:
            print(f"[!] {time.strftime('%H:%M:%S')} CẢNH BÁO ARP SPOOFING: "
                  f"IP {ip} đổi MAC {ip_mac[ip]} -> {mac}")
        else:
            ip_mac[ip] = mac
            print(f"[+] Học: {ip} = {mac}")

print("[*] Đang theo dõi ARP... (Ctrl+C để dừng)")
sniff(filter="arp", prn=handle, store=0)
```

**Cách demo (bằng chứng Tầng 1 — phát hiện):**
1. Chạy detector trên **Host B** (hoặc host A): `python3 /volumes/arp_detector.py`
2. Nhờ Người 1 (hoặc tự) chạy lại `arp_poison.py` từ attacker.
3. Detector in cảnh báo `CẢNH BÁO ARP SPOOFING: IP 10.9.0.5 đổi MAC ...` → **CHỤP ẢNH**.

> Nguyên lý: khi bị đầu độc, một IP (của A) đột nhiên gắn với MAC khác (của M) → sự thay đổi ánh xạ chính là dấu hiệu.

## A2. (Bổ sung) Công cụ có sẵn — arpwatch / arp-scan

```bash
apt update && apt install -y arpwatch arp-scan
arp-scan --interface=eth0 --localnet      # liệt kê IP-MAC thật trong mạng (chụp làm "baseline")
arpwatch -i eth0                          # ghi log khi phát hiện MAC đổi (xem trong syslog)
```
→ Dùng để đối chiếu: so sánh MAC "baseline" với MAC lúc bị tấn công.

## A3. NGĂN CHẶN — Static ARP (gắn cứng ánh xạ IP↔MAC)

Lấy MAC thật của A và B (do Người 1 cung cấp, hoặc `arp-scan`), rồi gắn cứng:

```bash
# Trên Host A — gắn cứng MAC thật của B:
arp -s 10.9.0.6 <MAC_THẬT_CỦA_B>
# Trên Host B — gắn cứng MAC thật của A:
arp -s 10.9.0.5 <MAC_THẬT_CỦA_A>

arp -n        # xác nhận cột Flags hiện 'PERM' (permanent) -> CHỤP ẢNH
```

**Demo chứng minh ngăn chặn thành công (bằng chứng "đinh" của Tầng 1):**
1. Sau khi gắn static ARP trên A và B.
2. Người 1 chạy lại `arp_poison.py`.
3. Kiểm tra `arp -n` trên A/B → MAC **KHÔNG đổi** (vẫn là MAC thật, cờ PERM) → **CHỤP ẢNH**.
4. A telnet/curl tới B → traffic đi thẳng, **không qua M** → bắt PCAP mới thấy M không còn nhận được gì → tấn công **thất bại**.

## A4. KHÔI PHỤC — Recovery

```bash
# Xóa cache bị đầu độc, nạp lại ánh xạ đúng:
ip neigh flush all              # xóa toàn bộ ARP cache
arp -s 10.9.0.6 <MAC_THẬT_B>    # nạp lại entry tĩnh (trên A)
# Trên attacker: tắt forwarding để dừng MITM còn sót:
sysctl -w net.ipv4.ip_forward=0
```

---

# PHẦN B — TẦNG 2: DAI + DHCP SNOOPING (Packet Tracer)

> Vì SEED chạy trên Docker bridge không có switch quản lý, ta mô phỏng giải pháp hạ tầng doanh nghiệp trên Packet Tracer.

## B1. Dựng topology trong Packet Tracer

Kéo thả:
- **1 Switch** hỗ trợ DAI: chọn **3560** (Multilayer Switch) — hỗ trợ DHCP Snooping + Dynamic ARP Inspection.
- **1 Server** làm **DHCP Server**.
- **3 PC**: `PC-A` (nạn nhân), `PC-B` (nạn nhân/server), `PC-Attacker` (kẻ tấn công).
- Nối tất cả vào switch bằng cáp thẳng (Copper Straight-Through).

Ghi nhớ cổng: ví dụ Server→`Gig0/1`, PC-A→`Fa0/1`, PC-B→`Fa0/2`, Attacker→`Fa0/3`.

## B2. Cấu hình DHCP Server

Trên Server (tab Services → DHCP): bật DHCP, pool `192.168.1.0/24`, default gateway `192.168.1.1`, start `192.168.1.10`. Đặt mỗi PC về chế độ **DHCP** (tab Desktop → IP Configuration → DHCP) để nhận IP tự động.

> Quan trọng: DAI dựa vào **bảng binding** mà DHCP Snooping học được. Dùng DHCP để bảng này tự điền (IP↔MAC↔port hợp lệ).

## B3. Cấu hình DHCP Snooping

Vào CLI của Switch:

```
Switch> enable
Switch# configure terminal
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 1
! Cổng nối tới DHCP Server là cổng TIN CẬY:
Switch(config)# interface gigabitEthernet 0/1
Switch(config-if)# ip dhcp snooping trust
Switch(config-if)# exit
! (các cổng PC để mặc định = untrusted)
```

## B4. Cấu hình Dynamic ARP Inspection (DAI)

```
Switch(config)# ip arp inspection vlan 1
! Cổng uplink/tin cậy bỏ qua kiểm tra:
Switch(config)# interface gigabitEthernet 0/1
Switch(config-if)# ip arp inspection trust
Switch(config-if)# exit
! (cổng PC = untrusted -> mọi gói ARP bị kiểm tra đối chiếu bảng binding)
```

## B5. Kiểm chứng & bằng chứng Tầng 2

1. Để các PC nhận IP qua DHCP → trên switch xem bảng binding:
   ```
   Switch# show ip dhcp snooping binding
   ```
   → thấy IP↔MAC↔port hợp lệ. **CHỤP ẢNH**.

2. Trên **PC-Attacker**, mô phỏng gửi ARP giả (tab Desktop → có thể dùng tính năng gửi ARP, hoặc mô tả hành vi). Switch sẽ **drop** gói ARP không khớp binding trên cổng untrusted.

3. Xem thống kê DAI:
   ```
   Switch# show ip arp inspection
   Switch# show ip arp inspection statistics
   ```
   → cột **Dropped** tăng lên = gói ARP giả bị chặn. **CHỤP ẢNH**.

> ⚠️ **Nói thật trong báo cáo:** đây là demo mô phỏng trên Packet Tracer, không phải chặn trực tiếp cuộc tấn công SEED. Nêu rõ: "Trong môi trường có switch quản lý thật, DAI+DHCP Snooping sẽ loại bỏ gói ARP giả ngay tại tầng 2 trước khi tới nạn nhân." → trung thực về giới hạn môi trường.

---

## Checklist bàn giao (Người 2 → Người 3 để đưa vào Chương 5)

- [ ] `arp_detector.py` trong `3. Source/scripts/` + ảnh detector báo cảnh báo.
- [ ] Ảnh `arp -n` có cờ **PERM** sau khi gắn static ARP.
- [ ] Ảnh/PCAP chứng minh tấn công **thất bại** sau static ARP (M không nhận được traffic).
- [ ] File `.pkt` (Packet Tracer) + ảnh: topology, `show ip dhcp snooping binding`, `show ip arp inspection statistics` (cột Dropped).
- [ ] Ghi chú ngắn: mỗi biện pháp thuộc pha NIST nào (Detection / Containment / Eradication / Recovery).

---

## Bảng ánh xạ biện pháp → pha NIST (dùng viết Chương 5)

| Biện pháp | Pha NIST | Tầng |
|---|---|---|
| Detector Scapy / arpwatch | Detection | 1 |
| Static ARP | Containment + Eradication | 1 |
| Flush cache + nạp lại entry đúng | Recovery | 1 |
| DHCP Snooping + DAI | Containment (hạ tầng) | 2 |
| Khuyến nghị dùng HTTPS/SSH thay HTTP/Telnet | Eradication (gốc rễ: bỏ cleartext) | — |
