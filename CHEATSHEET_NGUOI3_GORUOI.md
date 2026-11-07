# CHEATSHEET GỠ RỐI — NGƯỜI 3 (PHÂN TÍCH PCAP & TỔNG HỢP BÁO CÁO)

> Dành cho người **lần đầu** dùng Wireshark/tshark và lần đầu ráp báo cáo IR.
> Dạng: Triệu chứng → Chẩn đoán → Sửa.

---

## 🔑 BỘ LỌC WIRESHARK VÀNG (gõ vào ô Filter)

```
arp                              # xem toàn bộ ARP
arp.opcode == 2                  # chỉ ARP reply (dấu hiệu spoofing)
telnet                           # lưu lượng telnet (có credential)
http.request.method == "POST"    # form login HTTP (có password)
ip.addr == 10.9.0.5              # lọc theo 1 máy
eth.addr == <MAC_của_M>          # mọi gói dính tới kẻ tấn công
```
Thao tác vàng: chuột phải 1 gói → **Follow → TCP Stream** để đọc nguyên luồng.

---

## A. MỞ & ĐỌC FILE PCAP

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Không mở được `.pcap` | File hỏng/truncated khi copy | Nhờ Người 1 copy lại; kiểm tra dung lượng > vài KB |
| Mở ra **trống rỗng** | File bắt lỗi từ đầu (sai iface) | Nhờ Người 1 bắt lại đúng interface (xem cheatsheet Người 1 mục H) |
| Quá nhiều gói, rối | Chưa lọc | Dùng bộ lọc vàng ở trên; hoặc **Statistics → Conversations** |
| Không có Wireshark | Chưa cài | Cài Wireshark; hoặc dùng `tshark` dòng lệnh |

---

## B. TÌM DẤU HIỆU ARP SPOOFING

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Không thấy ARP reply bất thường | Lọc sai / PCAP không chứa giai đoạn đầu độc | Lọc `arp.opcode==2`; xác nhận PCAP bắt đúng lúc Người 1 chạy poison |
| Không rõ MAC nào là của kẻ tấn công | Chưa đối chiếu | **Statistics → Endpoints → Ethernet**: MAC nào gắn **nhiều IP** = kẻ tấn công |
| Thiếu cảnh báo "duplicate address" | Expert Info tắt | **Analyze → Expert Information**; hoặc tự chỉ ra 1 IP có 2 MAC khác nhau |

---

## C. TRÍCH CREDENTIAL TELNET

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Lọc `telnet` không ra gì | PCAP không có phiên telnet | Kiểm tra với Người 1 đã telnet lúc đang bắt chưa; thử lọc `tcp.port==23` |
| Follow TCP Stream thấy ký tự lạ/rời rạc | Telnet gửi từng ký tự + echo | Bình thường — đọc kỹ sẽ thấy `seed`/`dees` ghép lại; chọn hiển thị dạng ASCII |
| Không biết đâu là password | Lẫn trong lệnh | Password đi ngay sau prompt `Password:`; đối chiếu ghi chú mốc thời gian của Người 1 |

---

## D. TRÍCH CREDENTIAL HTTP

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Lọc POST không ra gì | Người 1 chưa gửi POST / sai cổng | Thử `tcp.port==8080`; kiểm tra Người 1 đã chạy `curl` login chưa |
| Thấy POST nhưng không thấy password | Chưa mở đúng nhánh | Mở gói → **HTML Form URL Encoded** → thấy `username=`, `password=` |
| Dữ liệu bị mã hóa | Nhầm sang HTTPS | Phải là HTTP (cổng 8080 cleartext); nếu HTTPS thì không đọc được — xác nhận lại kịch bản |

---

## E. TSHARK (dòng lệnh) KHÔNG CHẠY

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| `tshark: command not found` | Chưa cài | Cài Wireshark (kèm tshark); Linux: `apt install -y tshark` |
| `Permission denied` đọc file | Quyền file | `chmod +r file.pcap`; hoặc chạy nơi có quyền |
| Lọc `-Y` báo lỗi cú pháp | Thiếu nháy | Bọc biểu thức trong nháy đơn: `-Y 'http.request.method=="POST"'` |
| Ra rỗng dù Wireshark thấy | Sai tên trường `-e` | Kiểm tra tên field trong Wireshark (chuột phải field → Apply as Column) |

---

## F. DỰNG TIMELINE

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| Thời gian khó đọc (giây tương đối) | Định dạng mặc định | **View → Time Display Format → UTC Date and Time** (hoặc Time of Day) |
| Không khớp log của Người 2 | Lệch múi giờ/đồng hồ | Ghi rõ dùng giờ tương đối từ T0; đối chiếu theo **thứ tự sự kiện**, không cần khớp tuyệt đối |
| Không biết frame nào quan trọng | Thiếu chú thích | Dùng ghi chú frame# của Người 1; hoặc đánh dấu gói (Ctrl+M) khi tìm thấy |

---

## G. RÁP BÁO CÁO (.docx)

| Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|
| 3 người viết lệch giọng/thuật ngữ | Chưa thống nhất quy ước | Áp bảng thuật ngữ: M=kẻ tấn công, A/B=nạn nhân; cùng bộ IP; bạn biên tập lại cho đồng giọng |
| Hình không có số/chú thích | Chèn vội | Mỗi hình: "Hình X.Y — mô tả (nguồn: Người ?)"; tham chiếu trong văn |
| Khẳng định không có bằng chứng | Viết chay | Mỗi câu kết luận phải trỏ "(xem Hình ...)"; thiếu thì xin bổ sung |
| Phát hiện **thiếu bằng chứng** sát ngày nộp | Bàn giao muộn | Soát sớm theo checklist; thiếu gì báo ngay Người 1/2 chạy lại — **đừng để sát 26/11** |
| Chương không khớp pha NIST | Viết lệch khung | Đối chiếu bảng C1 trong guide: Ch3=tấn công, Ch4=phân tích, Ch5=phòng chống, Ch6=bài học |
| File .docx nặng/ảnh vỡ | Ảnh quá lớn | Nén ảnh vừa phải; giữ bản gốc ảnh trong `2. Evidence/` |

---

## 📌 3 LỖI "KINH ĐIỂN" CỦA NGƯỜI MỚI

1. **Đọc PCAP không lọc** → chìm trong hàng nghìn gói. Dùng bộ lọc vàng ngay. (Mục A)
2. **Viết kết luận không kèm bằng chứng** → báo cáo mất tính thuyết phục IR. Mọi khẳng định trỏ tới hình. (Mục G)
3. **Phát hiện thiếu bằng chứng quá trễ** → không kịp chạy lại lab. Soát checklist sớm, báo ngay. (Mục G)

---

## ✅ THỨ TỰ LÀM VIỆC ĐỀ XUẤT

1. Nhận PCAP + ghi chú từ Người 1 → trích đủ **3 bằng chứng** (ARP giả, telnet, HTTP POST) → chụp ảnh.
2. Dựng **timeline T0–T5**.
3. Đối chiếu log detector của Người 2.
4. Viết Chương 4 (phân tích) trước — vì đó là phần của bạn và là lõi báo cáo.
5. Thu Chương 3 (Người 1), Chương 5 (Người 2); viết Chương 1, 2, 6.
6. Biên tập đồng giọng → chèn hình → soát checklist → gửi Minh review trước 25/11.
