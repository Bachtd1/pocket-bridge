# PocketBridge

Cầu nối giữa **Miniloong Pocket 1** (Leaf) và máy của bạn (PC Windows, Mac, Linux, Raspberry Pi, NAS) qua LAN hoặc **Tailscale**:

- **Thư viện game riêng**: duyệt, tải và cài game từ thư mục ROM trên máy của chính bạn ngay trên Pocket 1 (qua
  [LoongMate](https://github.com/Bachtd1/Loongmate) → Thư viện riêng).
- **Bảng trạng thái** máy chủ và chạy các tác vụ đã cho phép trước (app PocketBridge trên Pocket 1).

PocketBridge là **phần mềm miễn phí (freeware)**: chỉ phát hành bản dựng sẵn, không phát hành mã nguồn.

## Tải về

Trang [Releases](../../releases):

| Gói | Dùng cho |
|---|---|
| `PocketBridge-Agent-Windows.zip` | PC Windows 10/11 (x64) |
| `PocketBridge-Agent-macOS.zip` | Mac Apple Silicon |
| `PocketBridge-Agent-Linux.zip` | PC Linux (x64), Raspberry Pi / máy ARM64 |
| `PocketBridge-Pocket1.zip` | app PocketBridge trên Pocket 1 (chép `THE-NHO` vào gốc thẻ Leaf) |
| `SHA256SUMS-agent.txt` | mã băm để kiểm file agent |

## Cách dùng nhanh (thư viện game riêng)

1. Giải nén gói Agent, cài bằng quyền quản trị (thay `<IP>` bằng IP Tailscale `100.x.y.z` hoặc IP LAN):
   - Windows (PowerShell Administrator): `.\install.ps1 -Listen <IP> -Id pocket1 -Scopes library:read`
   - Linux / Pi: `sudo sh linux/install.sh --listen <IP> --id pocket1 --scopes library:read`
   - macOS: `sudo sh macos/install.sh --listen <IP> --id pocket1 --scopes library:read`
2. Thêm thư mục ROM vào `config.yaml` (mục `library.roots`, xem `config.example.yaml`), khởi động lại agent.
   Cấu hình: Windows `C:\ProgramData\PocketBridge\config.yaml`, Linux `/etc/pocketbridge/config.yaml`,
   macOS `/Library/Application Support/PocketBridge/config.yaml`.
3. Chép file ghép đôi vào thẻ Leaf: `<thẻ>/PocketBridge-import/`.
4. Pocket 1: **LoongMate → Thư viện → Thư viện riêng → Ghép đôi**, rồi duyệt và cài game.

Hướng dẫn đầy đủ (đường dẫn, Tailscale, xử lý lỗi): [docs/POCKETBRIDGE.md của LoongMate](https://github.com/Bachtd1/Loongmate/blob/main/docs/POCKETBRIDGE.md)
và [docs/install-windows.md](docs/install-windows.md), [docs/install-linux.md](docs/install-linux.md), [docs/install-macos.md](docs/install-macos.md).

## An toàn và pháp lý

- Agent chỉ **cho đọc** đúng các thư mục bạn khai báo, chỉ nghe trên IP bạn chọn (không mở ra Internet), TLS ghim chứng chỉ +
  khoá truy cập riêng từng máy. Không tải gì từ Internet.
- PocketBridge **không cung cấp ROM**; chỉ dùng nội dung bạn có quyền sử dụng (bản dump của chính bạn, homebrew).
- Binary **chưa có chữ ký số** (SHA256SUMS chỉ chống tải lỗi) — chỉ tải từ trang Releases này.
- Phần mềm cung cấp **"nguyên trạng"**, không bảo hành. Điều khoản: [LICENSE](LICENSE). Giấy phép bên thứ ba: thư mục `GIAY-PHEP` trong mỗi gói.

Phát triển bởi **BACHTD**.
