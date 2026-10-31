# Thay đổi

## 0.4.0-beta — 2026-10-31

Bản phát hành công khai đầu tiên (chỉ bản dựng sẵn).

- **Thư viện ROM chỉ-đọc** (`library:read`): Pocket 1 duyệt, tải (tải tiếp được khi mất mạng, kiểm SHA-256) và cài game từ thư
  mục ROM trên PC/Mac/Pi qua LoongMate → Thư viện riêng.
- Agent cho Windows (x64), macOS (Apple Silicon), Linux (x64, ARM64); bộ cài kiểm checksum, tạo dịch vụ chạy nền, ghép đôi bằng
  mã dùng một lần + ghim chứng chỉ TLS.
- App PocketBridge cho Pocket 1 (bảng trạng thái, tác vụ cho phép trước).
- Đã kiểm: test tự động trên Linux; **chưa** cài thử trên mọi hệ điều hành — báo lỗi qua Issues.
