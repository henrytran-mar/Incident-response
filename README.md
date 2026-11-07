# Đồ án Incident Response — Phân tích ARP Spoofing & trích xuất credential cleartext

> **Nhóm:** PWSUCO_N03 (3 người) · **Trưởng nhóm:** Minh
> **Deadline:** 26/11/2026
> **Khung báo cáo:** NIST SP 800-61 · **Lab nền tảng:** SEED Labs — ARP Cache Poisoning Attack Lab

Đồ án thực hiện lại một cuộc tấn công **ARP cache poisoning → MITM** trong môi trường lab đã công bố (SEED Labs, Docker), bắt **PCAP** chứng minh credential bị lộ dạng **cleartext** qua Telnet/HTTP, rồi viết báo cáo **dưới góc nhìn chuyên gia Incident Response**: phát hiện, phân tích, phòng chống 2 tầng và bài học sau sự cố.

> ⚠️ **Phạm vi:** Toàn bộ thao tác chỉ chạy trong lab Docker cô lập phục vụ mục đích học tập. Không áp dụng lên bất kỳ hệ thống/mạng nào ngoài lab.

---

## 1. Cấu trúc kho

```
PWSUCO_N03/
├── 1. Báo cáo/          # File .docx báo cáo chính (6 chương)
├── 2. Evidence/
│   ├── buoi1/           # Ảnh chụp: ARP cache, Wireshark, cấu hình Packet Tracer...
│   └── buoi2/
├── 3. Source/
│   ├── scripts/         # Script Scapy (tấn công + detector)
│   ├── pcap/            # File .pcap bắt được (bằng chứng cốt lõi)
│   └── video/           # (không dùng)
├── KE_HOACH_TONG.md             # Kế hoạch tổng, phân việc, lộ trình
├── GUIDE_CAI_SEED.md            # Hướng dẫn cài & chạy SEED ARP Lab (Docker)
├── NGUOI1_HUONGDAN_TANCONG.md   # Hướng dẫn chi tiết — Người 1 (tấn công)
├── NGUOI2_HUONGDAN_PHONGCHONG.md# Hướng dẫn chi tiết — Người 2 (phòng chống)
├── NGUOI3_HUONGDAN_PHANTICH.md  # Hướng dẫn chi tiết — Người 3 (phân tích)
└── CHEATSHEET_NGUOI{1,2,3}_GORUOI.md  # Cheatsheet gỡ rối nhanh từng người
```

**Bắt đầu từ đâu:** đọc `KE_HOACH_TONG.md` trước để nắm toàn cảnh và phần việc của mình, rồi mở file `NGUOIx_HUONGDAN_*.md` tương ứng.

---

## 2. Phân việc

| Người | Vai trò | Pha NIST | Sản phẩm chính |
|---|---|---|---|
| **Người 1** | Tấn công & trích bằng chứng | Detection (nguồn dữ liệu) | PCAP telnet + HTTP, ảnh ARP cache bị đầu độc, script Scapy |
| **Người 2** | Phòng chống & khôi phục | Containment / Eradication / Recovery | Tầng 1 (SEED): static ARP + detector; Tầng 2 (Packet Tracer): DAI + DHCP Snooping |
| **Người 3** | Phân tích & tổng hợp | Analysis + Post-Incident | Phân tích Wireshark, timeline, trích credential, ghép báo cáo 6 chương |

---

## 3. Lấy code về (clone)

```bash
git clone https://github.com/henrytran-mar/Incident-response.git
cd Incident-response
```

---

## 4. Quy trình làm việc — commit lên nhánh, mở PR, tự merge

> **Nhánh `main` đã được khoá:** không push thẳng vào `main`. Mọi thay đổi phải đi qua một **nhánh riêng** rồi **Pull Request (PR)**. Không bắt buộc người khác duyệt — bạn **tự merge PR của mình** được.

### Bước 1 — Cập nhật `main` mới nhất trước khi làm
```bash
git checkout main
git pull
```

### Bước 2 — Tạo nhánh riêng cho phần việc của mình
Đặt tên nhánh dễ hiểu, ví dụ:
```bash
git checkout -b nguoi1/pcap-telnet        # Người 1
git checkout -b nguoi2/phongchong-dai     # Người 2
git checkout -b nguoi3/phantich-timeline  # Người 3
```

### Bước 3 — Làm việc, commit bình thường
```bash
git add -A
git commit -m "Mô tả rõ thay đổi"
```

### Bước 4 — Đẩy nhánh lên GitHub
```bash
git push -u origin <tên-nhánh>
```

### Bước 5 — Mở Pull Request rồi tự merge
- Vào https://github.com/henrytran-mar/Incident-response → GitHub sẽ gợi ý **"Compare & pull request"** → bấm **Create pull request**.
- Khi không có xung đột, bấm **Merge pull request** → **Confirm merge** (tự merge, không cần chờ ai duyệt).
- Hoặc dùng dòng lệnh (cần cài `gh`):
  ```bash
  gh pr create --fill
  gh pr merge --merge --delete-branch
  ```

### Bước 6 — Dọn nhánh & đồng bộ lại
```bash
git checkout main
git pull
git branch -d <tên-nhánh>
```

---

## 5. Lưu ý

- **File lớn / nhị phân** (PCAP, ảnh, .docx, .pkt): commit được bình thường. Nếu PCAP quá lớn (hàng trăm MB), báo trưởng nhóm để cân nhắc nén hoặc chia nhỏ.
- **Mỗi người làm trên nhánh riêng** để tránh đụng độ; chỉ gộp vào `main` qua PR.
- **Trước khi mở PR**, chạy `git pull origin main` và rebase/merge nếu `main` đã đổi, để PR merge sạch.

---

## 6. Tài liệu tham khảo

- SEED Labs — *ARP Cache Poisoning Attack Lab* (Wenliang Du, Syracuse University)
- NIST SP 800-61 Rev.2 — *Computer Security Incident Handling Guide*
- RFC 826 — *An Ethernet Address Resolution Protocol*
- Cisco — *Dynamic ARP Inspection & DHCP Snooping configuration guide*
