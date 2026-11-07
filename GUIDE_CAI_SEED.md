# HƯỚNG DẪN CÀI & CHẠY SEED ARP CACHE POISONING LAB

> Mục tiêu: dựng được môi trường SEED ARP Lab (3 container Docker) trên máy, sẵn sàng để Người 1 thực hiện tấn công và bắt PCAP.
> Áp dụng cho: **mỗi thành viên** (ai cũng nên chạy được để tự kiểm chứng).

---

## 0. Yêu cầu máy

- OS: **Ubuntu 20.04/22.04** (khuyến nghị — SEED chuẩn hóa trên Ubuntu) hoặc **Windows + WSL2 (Ubuntu)**, hoặc máy ảo Ubuntu trong VirtualBox/VMware.
- RAM: ≥ 4 GB (8 GB thoải mái).
- Dung lượng trống: ≥ 10 GB.
- Quyền `sudo`.

> 💡 Nếu dùng Windows: cài **WSL2** rồi làm theo hướng dẫn bên trong Ubuntu của WSL. Mọi lệnh dưới đây chạy trong terminal Ubuntu.

---

## 1. Cài Docker & Docker Compose

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
```

Cho user hiện tại dùng Docker không cần `sudo` mỗi lần:

```bash
sudo usermod -aG docker $USER
# ĐĂNG XUẤT/ĐĂNG NHẬP LẠI (hoặc chạy: newgrp docker) để có hiệu lực
```

Khởi động Docker và kiểm tra:

```bash
sudo systemctl enable --now docker   # trên WSL nếu không có systemd: sudo service docker start
docker --version
docker-compose --version
```

---

## 2. Tải Lab setup từ SEED

Trang lab: tìm *"SEED Labs ARP Cache Poisoning Attack Lab"* trên **seedsecuritylabs.org** → mục **Labsetup** → tải file `Labsetup.zip`.

```bash
mkdir -p ~/seed-arp && cd ~/seed-arp
# Giải nén file Labsetup.zip đã tải (giả sử để trong ~/Downloads)
unzip ~/Downloads/Labsetup.zip -d ~/seed-arp
cd ~/seed-arp/Labsetup
ls
```

Trong thư mục `Labsetup` sẽ có:
- `docker-compose.yml` — định nghĩa 3 container.
- thư mục `volumes/` — nơi chứa script Scapy (chia sẻ giữa host và container attacker).

---

## 3. Nạp các lệnh tắt của SEED (quan trọng)

SEED cung cấp bộ alias giúp thao tác Docker nhanh. Thêm vào `~/.bashrc`:

```bash
cat >> ~/.bashrc <<'EOF'
# SEED Labs aliases
alias dcbuild='docker-compose build'
alias dcup='docker-compose up'
alias dcdown='docker-compose down'
alias dockps='docker ps --format "{{.ID}}  {{.Names}}"'
dockps2() { docker ps --format "{{.ID}}  {{.Names}}"; }
docksh() { docker exec -it "$1" /bin/bash; }
EOF
source ~/.bashrc
```

> Giải thích nhanh: `dcbuild` build image, `dcup` khởi động lab, `dcdown` tắt, `dockps` liệt kê container đang chạy (ID + tên), `docksh <ID>` mở shell vào 1 container.

---

## 4. Build & khởi động lab

Trong thư mục `~/seed-arp/Labsetup`:

```bash
dcbuild        # build lần đầu (vài phút)
dcup           # khởi động — giữ terminal này chạy, KHÔNG đóng
```

Mở **terminal thứ 2**, kiểm tra 3 container đã chạy:

```bash
dockps
```

Kết quả kỳ vọng (ID sẽ khác trên máy bạn) — 3 container:

```
<id1>  seed-attacker        # Host M  — kẻ tấn công  — 10.9.0.105
<id2>  hostA-10.9.0.5        # Host A  — nạn nhân     — 10.9.0.5
<id3>  hostB-10.9.0.6        # Host B  — nạn nhân/svc  — 10.9.0.6
```

---

## 5. Vào từng container & kiểm tra kết nối

Mở shell vào attacker (Host M):

```bash
docksh <id của seed-attacker>
```

Trong container, kiểm tra IP và mạng:

```bash
ip addr          # xem IP (vd 10.9.0.105) và tên interface (thường eth0)
ping -c 2 10.9.0.5     # ping Host A
ping -c 2 10.9.0.6     # ping Host B
```

Xem bảng ARP hiện tại (để so sánh trước/sau khi tấn công):

```bash
arp -n
```

> 💡 **Mẹo chụp bằng chứng:** chụp `arp -n` trên Host A và Host B **TRƯỚC** khi tấn công (trạng thái sạch) để đối chiếu với **SAU** khi đầu độc. Đây là bằng chứng trực quan quan trọng cho báo cáo.

---

## 6. Nơi đặt script Scapy

Container attacker được **mount chung** thư mục `volumes/` với máy host. Nghĩa là:
- Bạn viết/sửa script `.py` ở `~/seed-arp/Labsetup/volumes/` trên máy host (dùng editor quen thuộc).
- Trong container attacker, file xuất hiện ở `/volumes/`.
- Chạy script **bên trong** container attacker (vì nó mới có địa chỉ 10.9.0.105 và quyền gửi gói raw).

```bash
# Trong container attacker:
cd /volumes
ls
python3 ten_script.py
```

---

## 7. Bắt PCAP (bằng chứng cốt lõi)

Có 2 cách, nên dùng **cách A** cho đơn giản:

**Cách A — tcpdump ngay trên container attacker (Host M):**
```bash
# Trong container attacker, interface thường là eth0:
tcpdump -i eth0 -w /volumes/capture.pcap
# Ctrl+C để dừng -> file nằm ở ~/seed-arp/Labsetup/volumes/capture.pcap trên host
```
Sau đó copy file ra thư mục đồ án `3. Source/pcap/`.

**Cách B — Wireshark trên máy host:** bắt trên interface bridge Docker (thường tên `br-xxxxxxxx`). Dùng khi muốn phân tích GUI ngay lúc bắt. Người 3 sẽ phân tích file này bằng Wireshark.

---

## 8. Tắt lab khi xong

```bash
# Quay lại terminal đang chạy dcup -> Ctrl+C, rồi:
dcdown
```

Dữ liệu trong `volumes/` (script, pcap) vẫn còn trên host sau khi `dcdown`.

---

## 9. Xử lý sự cố thường gặp

| Triệu chứng | Xử lý |
|---|---|
| `permission denied` khi chạy docker | Chưa add group `docker` hoặc chưa đăng nhập lại → `newgrp docker` |
| `dcup` báo cổng/bridge trùng | `dcdown` rồi `docker network prune`, chạy lại |
| Container không ping được nhau | Kiểm tra cả 3 container cùng chạy (`dockps`); build lại `dcbuild` |
| WSL: `docker` không chạy | `sudo service docker start` (WSL cũ không có systemd) |
| Không thấy interface `eth0` trong container | Chạy `ip addr` để lấy đúng tên interface rồi thay vào lệnh tcpdump |

---

## 10. Checklist trước khi chuyển sang bước tấn công

- [ ] 3 container chạy (`dockps` thấy đủ attacker + Host A + Host B).
- [ ] Attacker ping được 10.9.0.5 và 10.9.0.6.
- [ ] Đã chụp `arp -n` sạch của Host A và Host B (ảnh "trước").
- [ ] Viết/chạy được script `.py` trong `/volumes`.
- [ ] Bắt thử `tcpdump` ra file `.pcap` thành công.

→ Khi tick đủ, chuyển sang **NGUOI1_HUONGDAN_TANCONG.md**.
