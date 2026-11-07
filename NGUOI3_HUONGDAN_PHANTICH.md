# NGƯỜI 3 — HƯỚNG DẪN CHI TIẾT: PHÂN TÍCH PCAP/LOG & TỔNG HỢP BÁO CÁO IR

> **Nhiệm vụ của bạn:** biến bằng chứng thô (PCAP, log, ảnh) thành **phân tích của một chuyên gia Incident Response** — dựng timeline sự cố, chứng minh credential lộ cleartext, rút bài học — rồi **ráp toàn bộ thành báo cáo 6 chương** hoàn chỉnh.
>
> **Pha NIST phụ trách:** Detection & Analysis (phân tích) + Post-Incident Activity (bài học).
>
> ⚠️ **Bạn dễ quá tải nhất nhóm.** Yêu cầu Người 1 & Người 2 bàn giao bằng chứng **kèm chú thích** (mốc thời gian, số frame, biện pháp↔pha NIST) để bạn không phải tự dò từ đầu.

---

# PHẦN A — PHÂN TÍCH PCAP (Wireshark / tshark)

## A1. Chuẩn bị

- Cài **Wireshark** (Windows/Linux đều có bản GUI).
- Mở file `3. Source/pcap/arp_mitm.pcap` do Người 1 bàn giao.
- Mở sẵn ghi chú của Người 1 (mốc thời gian telnet login, POST HTTP, frame quan trọng).

## A2. Bằng chứng 1 — Phát hiện ARP spoofing trong PCAP

**Bộ lọc & thao tác:**
```
arp
```
- Tìm các **ARP reply (opcode 2)** từ MAC của attacker (M) khai rằng nó sở hữu **cả hai IP** (10.9.0.5 và 10.9.0.6) → dấu hiệu kinh điển.
- Wireshark thường tự gắn cảnh báo **"duplicate use of ... detected"** (Expert Info) — mở **Analyze → Expert Information** để chụp.
- Vào **Statistics → Endpoints → tab Ethernet**: thấy 1 MAC (của M) gắn với nhiều IP → bằng chứng trực quan.

→ **CHỤP:** danh sách ARP reply bất thường + Expert Info.

## A3. Bằng chứng 2 — Credential Telnet cleartext

```
telnet
```
- Chọn 1 gói telnet → chuột phải → **Follow → TCP Stream**.
- Trong luồng sẽ thấy username `seed` và password `dees` hiện nguyên văn (telnet gửi từng ký tự, đọc theo thứ tự).

→ **CHỤP:** cửa sổ Follow TCP Stream có username/password.

## A4. Bằng chứng 3 — Credential HTTP cleartext

```
http.request.method == "POST"
```
- Mở gói POST tới `10.9.0.6:8080/login`.
- Mở nhánh **HTML Form URL Encoded** → thấy `username=admin` và `password=SecretP@ss123`.

→ **CHỤP:** chi tiết form data.

## A5. Trích nhanh bằng dòng lệnh (tshark) — tùy chọn, tự động hóa

```bash
# Liệt kê gói ARP reply (dấu hiệu spoofing):
tshark -r arp_mitm.pcap -Y 'arp.opcode==2' -T fields -e eth.src -e arp.src.proto_ipv4

# Lấy nội dung POST HTTP (chứa credential):
tshark -r arp_mitm.pcap -Y 'http.request.method=="POST"' -T fields -e http.file_data

# Thống kê hội thoại:
tshark -r arp_mitm.pcap -q -z conv,tcp
```

## A6. Dựng TIMELINE sự cố (phần "chất IR" nhất)

Lập bảng thời gian từ cột Time của Wireshark (chuyển về giờ tuyệt đối: **View → Time Display Format → UTC Date and Time**):

| Thời điểm | Sự kiện | Bằng chứng (frame #) |
|---|---|---|
| T0 | Lưu lượng bình thường A↔B | frame ... |
| T1 | Bắt đầu ARP reply giả từ M | frame ... |
| T2 | ARP cache A/B bị ghi đè (MITM thành lập) | frame ... |
| T3 | A đăng nhập Telnet → lộ seed/dees | frame ... |
| T4 | A gửi POST HTTP → lộ password | frame ... |
| T5 | (nếu có) áp dụng static ARP → tấn công thất bại | frame ... |

→ Đây là **dòng thời gian sự cố** theo chuẩn điều tra IR, đưa thẳng vào Chương 4.

---

# PHẦN B — PHÂN TÍCH LOG (bổ trợ)

- Log detector của Người 2 (`arp_detector.py`): trích dòng `CẢNH BÁO ARP SPOOFING...` kèm timestamp → đối chiếu với timeline PCAP (khớp thời điểm T1/T2).
- Log arpwatch (nếu có): ghi nhận "changed ethernet address".
- Mục tiêu: chứng minh **cùng một sự kiện** nhìn thấy ở cả 2 nguồn (PCAP + log) → tăng độ tin cậy phân tích.

---

# PHẦN C — TỔNG HỢP BÁO CÁO 6 CHƯƠNG

## C1. Khung 6 chương (map NIST)

| Chương | Nội dung | Pha NIST | Nguồn dữ liệu |
|---|---|---|---|
| **1. Tổng quan** | Bối cảnh, mục tiêu, phạm vi, đối tượng, cấu trúc báo cáo | Preparation | Kế hoạch tổng |
| **2. Cơ sở lý thuyết** | ARP & RFC 826, ARP spoofing, MITM, giao thức cleartext (Telnet/HTTP), vòng đời NIST SP 800-61 | Preparation | Tài liệu |
| **3. Triển khai môi trường & kịch bản tấn công** | SEED Lab, topology, công cụ, các bước tấn công | — | Người 1 |
| **4. Phát hiện & phân tích sự cố** | Phân tích PCAP, timeline, trích credential, đối chiếu log | Detection & Analysis | Người 1 + bạn |
| **5. Phòng chống & khôi phục** | Defense-in-depth 2 tầng, demo chặn thành công | Containment/Eradication/Recovery | Người 2 |
| **6. Kết luận & bài học** | Đánh giá, khuyến nghị dài hạn, phòng ngừa tái diễn | Post-Incident | Cả nhóm |

## C2. Nguyên tắc nhất quán (tránh "3 người viết 3 kiểu")

- **Thuật ngữ thống nhất:** M = "kẻ tấn công", A/B = "nạn nhân"; dùng cùng một bộ IP xuyên suốt.
- **Đánh số hình nhất quán:** "Hình 4.1", "Hình 5.2"... mỗi hình có **chú thích** + **nguồn** (ai tạo).
- **Mọi khẳng định phải trỏ tới bằng chứng:** câu "password lộ cleartext" phải kèm "(xem Hình 4.3 — Follow TCP Stream)".
- **Giọng văn chuyên gia IR:** mô tả theo "sự cố — phát hiện — ứng phó", không viết như hướng dẫn thực hành.
- **Trung thực về giới hạn:** nêu rõ Tầng 2 (DAI) là mô phỏng Packet Tracer, không chặn trực tiếp SEED.

## C3. Quy trình ráp báo cáo

1. Tạo khung 6 chương trong file `.docx` ở `1. Báo cáo/` (đã có sẵn file tiêu đề).
2. Thu nội dung thô: Người 1 → Chương 3; Người 2 → Chương 5; bạn → Chương 4.
3. Bạn viết Chương 1, 2, 6 + biên tập lại Chương 3, 5 cho đồng giọng.
4. Chèn bằng chứng (ảnh + trích PCAP) vào đúng chỗ, đánh số, chú thích.
5. Rà bảng mục lục, danh mục hình, tài liệu tham khảo.
6. Gửi Minh review tổng trước 25/11.

## C4. Checklist hoàn thành báo cáo

- [ ] Đủ 6 chương, mỗi chương map đúng pha NIST.
- [ ] Timeline sự cố (bảng T0–T5) có trong Chương 4.
- [ ] 3 bằng chứng cleartext/spoofing (ARP reply giả, telnet seed/dees, POST password) đều có ảnh.
- [ ] Chương 5 có cả Tầng 1 (ảnh chặn thật) và Tầng 2 (ảnh PT + statistics Dropped).
- [ ] Mọi hình có số + chú thích + được tham chiếu trong văn.
- [ ] Danh mục tài liệu tham khảo (SEED, NIST SP 800-61, RFC 826, Cisco DAI).
- [ ] Đọc soát chính tả, thuật ngữ, IP nhất quán.

---

## Bàn giao ngược (nếu thiếu dữ liệu)

Nếu khi phân tích phát hiện thiếu bằng chứng (vd PCAP không có POST, thiếu ảnh static ARP), **báo ngay** Người 1/Người 2 chạy bổ sung — đừng để tới sát deadline. Bạn là người thấy lỗ hổng bằng chứng sớm nhất vì bạn ráp toàn bộ.
