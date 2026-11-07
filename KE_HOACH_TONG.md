# KẾ HOẠCH TỔNG — ĐỒ ÁN INCIDENT RESPONSE
## Phân tích lưu lượng mạng (PCAP) phát hiện ARP Spoofing và trích xuất credential cleartext trong mạng LAN

> **Nhóm:** PWSUCO_N03 (3 người) · **Trưởng nhóm:** Minh
> **Deadline nộp:** 26/11/2026
> **Cập nhật:** 07/11/2026 — hướng SEED Labs

---

## 1. Tóm tắt & định hướng

Đồ án **KHÔNG tự dựng homelab**. Thay vào đó dùng một lab đã công bố làm nền tảng chính, thực hiện lại, rồi viết báo cáo **dưới góc nhìn chuyên gia Incident Response (IR)**.

- **Lab nền tảng:** *SEED Labs — ARP Cache Poisoning Attack Lab* (ĐH Syracuse, GS. Wenliang Du). Môi trường Docker 3 container, tấn công bằng Scapy. Nguồn chính thống, có giáo trình kèm → độ tin cậy học thuật cao, rủi ro kỹ thuật thấp.
- **Kịch bản:** Kẻ tấn công đầu độc ARP cache → chèn mình vào giữa (MITM) → nạn nhân đăng nhập **Telnet** và **HTTP** → bắt được **PCAP** chứng minh username/password lộ dạng **cleartext** trong LAN.
- **Khung báo cáo:** vòng đời ứng phó sự cố **NIST SP 800-61** (xem mục 4).

> ⚠️ **Lưu ý trung thực:** SEED ARP Lab bản gốc là lab **tấn công**. Nhóm phải **tự bổ sung** 2 phần để thành một bài IR hoàn chỉnh: (a) lớp **phát hiện & phòng chống**, (b) lớp **phân tích & tổng hợp báo cáo IR**. Đây chính là phần tạo ra giá trị và điểm số của đồ án, không chỉ "chạy lại lab".

---

## 2. Kiến trúc kỹ thuật (đã chốt)

| Thành phần | Công cụ / môi trường | Loại bằng chứng |
|---|---|---|
| Tấn công ARP poisoning + MITM | SEED ARP Lab (Docker 3 container, Scapy) | **Thật** |
| Service bị tấn công | **Telnet** (SEED có sẵn) + **HTTP** (nhóm tự thêm form login) | **Thật** |
| Bằng chứng lộ credential | **PCAP** bắt bằng tcpdump/Wireshark | **Thật** |
| Phòng chống **Tầng 1 — host-level** | static ARP, `arpwatch`/`arp-scan`, detector Scapy tự viết (chạy trong SEED) | **Thật, test được** |
| Phòng chống **Tầng 2 — switch-level** | **Dynamic ARP Inspection (DAI) + DHCP Snooping** trên **Cisco Packet Tracer** | Mô phỏng (ảnh cấu hình + log chặn) |

**Vì sao 2 tầng:** SEED chạy trên Docker bridge (Linux), **không có switch quản lý** nên không thể demo DAI trực tiếp trong SEED. Do đó phòng chống chia 2 tầng theo mô hình **defense-in-depth**: Tầng 1 chặn được ngay trong SEED (bằng chứng thật), Tầng 2 mô phỏng trên Packet Tracer để chứng minh giải pháp doanh nghiệp.

> **Đã cân nhắc GNS3** (IOS thật, bằng chứng mạnh hơn) nhưng loại vì chi phí ẩn lớn: image IOSvL2 bản quyền/vùng xám, ngốn RAM, learning curve cao, dễ sa lầy sát deadline. → **Dùng Packet Tracer trước** cho chắc; nâng cấp GNS3 nếu còn dư thời gian.

**Môi trường vận hành:** mỗi thành viên dùng **1 máy riêng** (cài Docker + SEED để tự chạy lại được).

---

## 3. Mạng & địa chỉ trong SEED ARP Lab

Mạng Docker `10.9.0.0/24`:

| Vai trò trong kịch bản | Container SEED | IP |
|---|---|---|
| Nạn nhân A (client đăng nhập) | Host A | `10.9.0.5` |
| Nạn nhân B / Server (telnet + HTTP) | Host B | `10.9.0.6` |
| **Kẻ tấn công (MITM)** | Host M (attacker) | `10.9.0.105` |

Mục tiêu tấn công: đầu độc ARP cache của A và B để cả hai tin rằng MAC của đối phương là MAC của M → mọi lưu lượng A↔B đi qua M.

---

## 4. Khung báo cáo theo vòng đời NIST SP 800-61

| Pha NIST | Nội dung trong báo cáo | Ai phụ trách |
|---|---|---|
| **1. Preparation** | Bối cảnh, mô hình mạng, công cụ, giả định, phạm vi lab | Cả nhóm (Minh tổng hợp) |
| **2. Detection & Analysis** | Dấu hiệu tấn công, phân tích PCAP/log, dựng timeline sự cố, trích credential cleartext | Người 1 (dữ liệu) + Người 3 (phân tích) |
| **3. Containment, Eradication & Recovery** | Phòng chống 2 tầng: cô lập, chặn ARP giả, khôi phục | Người 2 |
| **4. Post-Incident Activity** | Bài học, khuyến nghị dài hạn, phòng ngừa tái diễn | Người 3 + cả nhóm |

---

## 5. Phân việc (map theo vòng đời IR)

| Người | Vai trò | Pha NIST | Sản phẩm chính |
|---|---|---|---|
| **Người 1** | Tấn công & trích bằng chứng | Detection (nguồn dữ liệu) | PCAP telnet + HTTP, ảnh ARP cache bị đầu độc, script Scapy, log tấn công |
| **Người 2** | Phòng chống & khôi phục | Containment / Eradication / Recovery | Tầng 1 (SEED): static ARP + detector; Tầng 2 (PT): DAI + DHCP Snooping; ảnh + log chặn |
| **Người 3** | Phân tích Log/PCAP & tổng hợp | Analysis + Post-Incident | Phân tích Wireshark, timeline, trích credential, ghép báo cáo 6 chương |

**⚠️ Rủi ro điều phối cần quản lý:**
1. **Người 1 là nút thắt phụ thuộc** — Người 2 và Người 3 đều cần PCAP/bằng chứng của Người 1 mới làm được. → Người 1 phải chạy SEED và xuất PCAP **sớm nhất**.
2. **Người 3 dễ quá tải** (vừa phân tích vừa ráp báo cáo). → Người 1 bàn giao PCAP **kèm chú thích sẵn** (mốc thời gian, packet quan trọng) để giảm tải cho Người 3.

---

## 6. Cấu trúc thư mục đồ án

```
PWSUCO_N03/
├── 1. Báo cáo/          # File .docx báo cáo chính (6 chương)
├── 2. Evidence/
│   ├── buoi1/           # Ảnh chụp: ARP cache, Wireshark, cấu hình PT...
│   └── buoi2/
├── 3. Source/
│   ├── scripts/         # Script Scapy (tấn công + detector)
│   ├── pcap/            # File .pcap bắt được (bằng chứng cốt lõi)
│   └── video/           # (không dùng — nhóm chọn chỉ ảnh + PCAP)
├── KE_HOACH_TONG.md     # File này
├── GUIDE_CAI_SEED.md    # Hướng dẫn cài & chạy SEED ARP Lab
└── NGUOI1_HUONGDAN_TANCONG.md  # Hướng dẫn chi tiết cho Người 1
```

---

## 7. Lộ trình & mốc thời gian (gợi ý, deadline 26/11)

| Tuần | Việc | Ghi chú |
|---|---|---|
| **07–11/11** | Cài SEED, Người 1 chạy tấn công → xuất PCAP telnet + HTTP | **Ưu tiên cao nhất** — gỡ nút thắt |
| **12–16/11** | Người 2 làm phòng chống Tầng 1 + Tầng 2 (PT); Người 3 bắt đầu phân tích PCAP | Chạy song song sau khi có PCAP |
| **17–21/11** | Viết nội dung 6 chương; ghép bằng chứng vào báo cáo | Người 3 chủ trì ráp |
| **22–25/11** | Rà soát, thống nhất, kiểm tra tính nhất quán, chốt | Minh review tổng |
| **26/11** | **Nộp** | |

---

## 8. Tài liệu tham khảo nền tảng

- SEED Labs — ARP Cache Poisoning Attack Lab (Wenliang Du, Syracuse University).
- NIST SP 800-61 Rev.2 — *Computer Security Incident Handling Guide*.
- RFC 826 — *An Ethernet Address Resolution Protocol* (cơ chế ARP).
- Cisco — *Dynamic ARP Inspection & DHCP Snooping* configuration guide.
