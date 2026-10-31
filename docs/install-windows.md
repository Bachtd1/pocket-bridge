# Cài agent trên Windows (0.4.0)

Dành cho Windows 10/11 và Windows Server 2019+ (amd64). Agent chạy như **Windows Service** bằng *virtual service account* `NT SERVICE\PocketBridgeAgent`
(không phải LocalSystem, không có mật khẩu, chỉ có quyền ta cấp bằng ACL). Bộ cài `packaging/windows/install.ps1` làm toàn bộ việc dưới đây và **đọc lại ACL để kiểm** trước khi mở firewall/khởi động.

> **Quy tắc vàng (giống Linux):** thứ agent chạy hoặc tin (binary, `config.yaml`, `actions.d`, thư mục executable của action) chỉ **Administrators/SYSTEM** ghi được; service account chỉ **đọc**.
> Agent không tự kiểm được ACL trên Windows (Go không đọc ACL), nên việc này do bộ cài làm — đừng cài tay rồi bỏ qua bước ACL.

## 1. Lấy binary và checksum

Build bằng Docker (không cần cài Go; chạy trong Git Bash/WSL) hoặc lấy artifact CI:

```sh
sh packaging/release.sh        # -> agent/dist/pocketbridge-agent-windows-amd64.exe, pocketbridge-agent-linux-*, SHA256SUMS
```

`SHA256SUMS` chứng minh file không hỏng/không bị đổi so với lúc build; **chưa có chữ ký** (Authenticode cho `.exe` là việc của bản 1.0). Lấy binary + `SHA256SUMS` từ nguồn bạn tin (tự build, hoặc artifact CI).
Bộ cài **từ chối** nếu `SHA256SUMS` không có đúng một dòng cho file hoặc hash lệch.

## 2. Xem trước (không đổi gì)

```powershell
.\packaging\windows\install.ps1 -DryRun -Binary .\agent\dist\pocketbridge-agent-windows-amd64.exe -Listen 100.64.0.5
```

`-DryRun` (hoặc `-WhatIf`) in từng thay đổi: thư mục, nội dung `config.yaml`, lệnh đăng ký service, bảng ACL, luật firewall, lệnh `pair`. Không cần quyền Administrator để xem trước.

## 3. Cài

Mở PowerShell **Run as administrator**:

```powershell
.\packaging\windows\install.ps1 `
    -Binary .\agent\dist\pocketbridge-agent-windows-amd64.exe `
    -Listen 100.64.0.5 `
    -Id pocket1 -Scopes 'metrics:read,actions:list,jobs:read,action:sunshine'
```

| Tham số | Ý nghĩa |
|---|---|
| `-Binary`, `-Sums` | binary và `SHA256SUMS` (mặc định cạnh binary); `-SkipChecksum` bỏ kiểm (không khuyến nghị) |
| `-Listen` | IP LAN/tailnet agent nghe (có thể kèm `:cổng`). Bỏ trống = IPv4 của giao diện Tailscale nếu có. Agent **từ chối** `0.0.0.0` |
| `-Port` | cổng mặc định (4268) cho địa chỉ không ghi cổng |
| `-Id`, `-Scopes` | id và scope của **token dài hạn** sẽ được cấp cho client sau khi đổi bootstrap (xem mục 6) |
| `-Name`, `-PairHost`, `-Ttl`, `-PairOut` | tên node trong bundle; host client dùng để tới agent; hạn token bootstrap (mặc định `15m`, tối đa `1h`); nơi ghi bundle |
| `-InstallDir`, `-DataDir` | mặc định `C:\Program Files\PocketBridge Agent`, `C:\ProgramData\PocketBridge` |
| `-TailscaleProfile` | `Private` (mặc định): chuyển card Tailscale sang nhóm mạng Private rồi chỉ mở cổng trên Private (mục 5). `Keep`: không đổi nhóm mạng, dùng luật riêng theo giao diện |
| `-NoStart` | cài xong không khởi động |

Bộ cài làm, theo thứ tự:

1. Kiểm quyền Administrator và checksum.
2. Tạo `C:\Program Files\PocketBridge Agent\` (binary) và `%ProgramData%\PocketBridge\` (`config.yaml`, `state\`, `logs\`, `actions.d\`, `actions\`, `pairing\`).
3. Chép binary; viết `config.yaml` nếu chưa có (không ghi đè config cũ).
4. Đăng ký service `PocketBridgeAgent` (`pocketbridge-agent.exe service install`): tài khoản `NT SERVICE\PocketBridgeAgent`, tự khởi động (trễ), tự khởi động lại khi lỗi.
5. **Đặt ACL rồi đọc lại bằng `Get-Acl` và kiểm** (bảng dưới). Sai → dừng, chưa đổi nhóm mạng, chưa mở firewall, chưa sinh token.
6. Tailscale + firewall (mục 5): ghi category cũ vào `install-state.json`, chuyển card Tailscale sang Private, rồi tạo luật Private duy nhất (mặc định; `-TailscaleProfile Keep` thì giữ cách cũ).
7. `fingerprint`, rồi `pair` → ghi pairing bundle vào `%ProgramData%\PocketBridge\pairing\pairing-bundle-<id>.json` (chỉ Administrators/SYSTEM đọc).
8. Khởi động service, in trạng thái và thử kết nối TCP.

### ACL được đặt (DACL *protected*, không kế thừa; dùng SID nên không phụ thuộc ngôn ngữ Windows)

| Đường dẫn | Administrators, SYSTEM | `NT SERVICE\PocketBridgeAgent` | Users / Everyone / Authenticated Users |
|---|---|---|---|
| `C:\Program Files\PocketBridge Agent\` | Toàn quyền | Đọc + chạy | **không có** |
| `%ProgramData%\PocketBridge\` (gốc) | Toàn quyền | chỉ duyệt thư mục này (không kế thừa xuống con) | **không có** |
| `config.yaml` | Toàn quyền | **Đọc** | **không có** |
| `actions.d\` | Toàn quyền | **Đọc** | **không có** |
| `actions\` (thư mục executable của action, = `actions.executable_dirs`) | Toàn quyền | **Đọc + chạy** | **không có** |
| `state\` (`key.pem`, `cert.pem`, `jobs\`, idempotency, `pairing.json`) | Toàn quyền | **Đọc/ghi** (kế thừa cho file mới) | **không có** — Users không đọc được `key.pem` |
| `logs\` | Toàn quyền | Đọc/ghi | **không có** |
| `pairing\` | Toàn quyền | không có | **không có** |

Service account **không** ghi được `config.yaml`, `actions.d`, `actions`, binary: kẻ chiếm được agent không tự cài action/đổi cấu hình được. Bộ kiểm cũng bắt các sai lệch phổ biến
(thêm ACE cho Users, cho service quyền ghi, quên cắt kế thừa); `packaging/windows/test-acl.ps1` chứng minh điều đó trên thư mục tạm.

> Vì sao `config.yaml` không để service ghi: agent không cần ghi nó. Trạng thái do agent tạo ra (token đã cấp qua ghép đôi, bootstrap đã dùng) nằm ở `state\pairing.json`;
> `token create/revoke/pair` chạy bằng Administrator và giữ nguyên ACL của config khi ghi lại.

## 4. Action

Đặt định nghĩa `*.yaml` vào `%ProgramData%\PocketBridge\actions.d\` và executable vào `%ProgramData%\PocketBridge\actions\` **bằng tài khoản Administrator** (thư mục này service chỉ đọc), rồi:

```powershell
& "C:\Program Files\PocketBridge Agent\pocketbridge-agent.exe" actions check --config "C:\ProgramData\PocketBridge\config.yaml"
Restart-Service PocketBridgeAgent        # action chỉ nạp lúc khởi động
```

`actions.executable_dirs` (đã trỏ tới `actions\`) là bắt buộc trên Windows cho action `process`. Mẫu: `packaging/examples/actions.d/`. Action `windows_service` cần service đích nằm trong `services.allow`
và service account có quyền khởi động/dừng service đó (mặc định **không có**: cấp riêng bằng `sc.exe sdset` nếu cần, chỉ cho đúng service).

## 5. Firewall

Bộ cài xoá luật cũ nhóm `PocketBridge Agent` trước khi tạo luật mới.

Chế độ mặc định `-TailscaleProfile Private`: bộ cài **chuyển card Tailscale sang nhóm mạng Private** rồi tạo **một** luật duy nhất (in lại bằng `Get-NetFirewallRule`). Đây là cách đúng spec: cổng chỉ mở trên Private, không có luật Profile = Any.

| Luật | Phạm vi |
|---|---|
| `PocketBridge Agent (Private)` | inbound TCP `<cổng>`, **Profile = Private**, `Program` = binary đã cài, `RemoteAddress` = `LocalSubnet`, `100.64.0.0/10`, `fd7a:115c:a1e0::/48` (mạng LAN + tailnet) |

Không có luật nào cho profile Public hay Any. Bộ cài kiểm lại sau khi tạo: đúng một luật, Profile = Private; sai thì dừng.

**Đổi nhóm mạng của card Tailscale (chuyển sang Private), làm đúng thế này:**

1. Tìm card bằng `Get-NetConnectionProfile` (`InterfaceAlias` chứa "Tailscale"). Không thấy (Tailscale chưa chạy/chưa đăng nhập) hoặc thấy nhiều card → **dừng**, không đoán. Với `-DryRun` chỉ cảnh báo rồi in tiếp (để chạy được trên máy/CI không có Tailscale); bản chạy thật dừng ngay, trước khi chép hay cài gì.
2. Ghi category hiện tại vào `%ProgramData%\PocketBridge\install-state.json` (DACL chỉ Administrators/SYSTEM, đọc lại để kiểm) **trước** khi đổi. Cài lại: giữ category gốc đã ghi, không ghi đè bằng "Private" do chính bộ cài đặt.
3. `Set-NetConnectionProfile -InterfaceAlias <alias> -NetworkCategory Private`, đọc lại để xác nhận; không khớp thì dừng, chưa mở firewall.
4. Chỉ đổi `NetworkCategory`: **không** restart dịch vụ Tailscale, **không** đổi adapter nên kết nối tailnet không bị ngắt. Card `DomainAuthenticated` không đổi được: dùng `Keep`.

> **Cảnh báo, đọc trước khi cài:** nhóm mạng quyết định **mọi** luật firewall của Windows, không riêng PocketBridge. Card từ Public sang Private sẽ bắt đầu nhận mọi luật inbound đang bật cho profile Private
> (vd *File and Printer Sharing* nếu bạn bật cho Private, *Network Discovery*, luật theo từng ứng dụng như Docker/Steam…). Người nào trong tailnet (hoặc qua luật Tailscale ACL cho phép) tới được máy này đều chạm tới các luật đó trên card Tailscale.
> `install.ps1 -DryRun` liệt kê các luật inbound Allow đang bật cho Private (đọc qua COM `HNetCfg.FwPolicy2`, không cần Administrator) và gắn `[MỚI]` cho những luật chưa áp dụng cho card ở nhóm hiện tại.
> Nếu card đã là Private thì không có gì đổi. Đọc danh sách, tắt luật nào bạn không muốn lộ trên tailnet, hoặc dùng `Keep`.
> Giới hạn chưa kiểm: Windows có thể tự đặt lại nhóm mạng khi card được tạo lại/mạng đổi; nếu về Public thì luật Private ngừng phủ card Tailscale. Kiểm lại bằng `Get-NetConnectionProfile`.

**`-TailscaleProfile Keep`** (hành vi cũ, không đổi nhóm mạng): ngoài luật Private, nếu `Get-NetAdapter` thấy giao diện Tailscale thì tạo thêm luật `PocketBridge Agent (Tailscale)` —
inbound TCP `<cổng>`, `InterfaceAlias` = giao diện Tailscale, `RemoteAddress` = dải tailnet, Profile = Any **nhưng chỉ trên giao diện này và chỉ từ địa chỉ tailnet**. Dùng khi không muốn đổi nhóm mạng.
Nếu cổng nghe là IP LAN (không phải tailnet), luật Private là đủ ở cả hai chế độ.

Gỡ (`uninstall.ps1`): nếu `install-state.json` có category gốc và card còn → trả về category đó (in rõ, đọc lại xác nhận) rồi xoá file state; nếu category hiện tại không phải "Private" do bộ cài đặt (có người đổi tay) thì **không ghi đè**; card không còn → giữ file và in lệnh tự đặt. Luật firewall bị xoá trước.

## 6. Pairing bundle (token dùng một lần) và Pocket 1

`pair` tạo token **bootstrap** (scope duy nhất `pair:exchange`, hết hạn sau `-Ttl`, dùng một lần) và ghi bundle JSON (`schemas/pairing_bundle.schema.json`):

```json
{ "schema": 1,
  "node": { "id": "workstation", "name": "Main Workstation", "host": "100.64.0.5", "port": 4268, "fingerprint": "sha256:…" },
  "credential": { "type": "bootstrap", "token": "pb_…" } }
```

1. Chuyển file sang Pocket 1 **trong vòng 15 phút** bằng kênh riêng (cáp/USB, `scp` qua tailnet) — không để trong thư mục chia sẻ/đám mây. File chỉ Administrators/SYSTEM đọc được nên bạn phải copy bằng PowerShell/Explorer **chạy elevated**.
2. Client import bundle, ghim fingerprint, gọi `POST /api/v1/pair/exchange` (docs/api.md) và nhận token dài hạn với đúng `-Scopes`. Từ lúc đó token bootstrap vô hiệu.
3. **Xoá file bundle** sau khi import (`Remove-Item`). Hết hạn mà chưa đổi: chạy lại `pocketbridge-agent.exe pair --config … --id <id> --scopes …` (bootstrap cũ hết hạn được thay gọn).
4. Mất mạng đúng lúc đổi → client nhận 409 `already_exchanged`, **không** lấy lại được token (chủ ý, docs/threat-model.md). Tạo bundle mới với `--id` mới; thu hồi token cũ: `token revoke --id <id>`.

Quản lý token: `pocketbridge-agent.exe token list|revoke --config …` (token cấp qua ghép đôi hiện kèm "cấp qua ghép đôi"; thu hồi có hiệu lực ≤ 5 giây không cần restart).

## 7. Tailscale ACL: chỉ Pocket 1 vào cổng agent

Token là lớp bảo vệ ứng dụng; Tailscale ACL là lớp mạng. Đặt trong tailnet policy (admin console → Access controls) để **chỉ thiết bị Pocket 1** tới được cổng agent của máy này:

```jsonc
{
  "tagOwners": { "tag:pocket1": ["autogroup:admin"], "tag:pbnode": ["autogroup:admin"] },
  "grants": [
    // Pocket 1 -> mọi node PocketBridge, chỉ cổng agent
    { "src": ["tag:pocket1"], "dst": ["tag:pbnode"], "ip": ["tcp:4268"] }
  ]
  // KHÔNG có grant/ACL "*:*" hay "autogroup:member -> tag:pbnode". Chặn mặc định của Tailscale lo phần còn lại.
}
```

Gắn thẻ: trên Pocket 1 `tailscale up --advertise-tags=tag:pocket1`; trên máy Windows `tailscale up --advertise-tags=tag:pbnode`. Kiểm: từ thiết bị **khác** trong tailnet, `Test-NetConnection 100.64.0.5 -Port 4268` phải **thất bại**; từ Pocket 1 phải thành công.
(Nếu bạn còn dùng cú pháp `acls` cũ: `{"action":"accept","src":["tag:pocket1"],"dst":["tag:pbnode:4268"]}`.)

## 8. Kiểm tra

```powershell
$exe = 'C:\Program Files\PocketBridge Agent\pocketbridge-agent.exe'
& $exe service status
Get-Content C:\ProgramData\PocketBridge\logs\agent.jsonl -Tail 5
Get-Content C:\ProgramData\PocketBridge\logs\service.jsonl -Tail 5        # vòng đời service + lỗi trước khi agent mở log
Invoke-WebRequest https://100.64.0.5:4268/api/v1/health -SkipCertificateCheck   # PowerShell 7; 5.1: dùng curl.exe -sk
```

Chạy thử **không cài service** (foreground trên console, Ctrl+C = Stop): `pocketbridge-agent.exe service debug --config <file>` (hoặc `run`).

## 9. Gỡ

```powershell
.\packaging\windows\uninstall.ps1 -DryRun            # xem trước
.\packaging\windows\uninstall.ps1                    # gỡ service, luật firewall, thư mục cài; GIỮ dữ liệu
.\packaging\windows\uninstall.ps1 -RemoveData        # xoá luôn %ProgramData%\PocketBridge (key.pem: fingerprint sẽ đổi nếu cài lại)
```

## 10. Lệnh service thủ công

`pocketbridge-agent.exe service install|uninstall|start|stop|status --config <file>` (Administrator). `status` thoát 0 nếu đang chạy, 3 nếu không chạy, 1 nếu chưa cài/lỗi. `service run` chỉ do Service Control Manager gọi.
Dừng service: SCM gửi Stop → agent ngừng nhận job mới, job đang chạy bị dừng và đánh dấu `interrupted`, SSE đóng, HTTP tắt (tối đa ~18 giây; SCM chờ 25 giây).

## Giới hạn hiện tại (đọc trước khi tin)

- Đã chạy thật trên Windows: Service Execute (Running/Stop/lỗi khởi động, job `interrupted`), `restrictFile`, giữ DACL khi ghi nguyên tử, bộ kiểm ACL trên thư mục tạm, `install.ps1 -DryRun`.
- **Chưa chạy thật trên máy cài mới:** đăng ký service với `NT SERVICE\PocketBridgeAgent` qua SCM, ACL với tài khoản dịch vụ thật (tên chỉ tra được sau khi service tồn tại), luật firewall thật, khởi động qua SCM. Lần cài đầu nên dùng `-DryRun`, rồi cài và đọc kỹ phần "Đọc lại ACL" trong output.
- Đổi nhóm mạng card Tailscale (`Set-NetConnectionProfile`) và `uninstall.ps1` khôi phục mới chạy thử ở chế độ `-DryRun` (đọc card + luật thật, giả lập các nhánh Public/không có card/state hỏng); chưa chạy thật có Administrator.
- Chưa có chữ ký Authenticode; SmartScreen/Defender có thể cảnh báo binary lạ.
- Windows Server Core/ARM64 chưa kiểm.
