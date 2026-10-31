# Cài agent trên macOS (Apple Silicon, 0.4.0)

Agent chạy trên **macOS 14 (Sonoma) và 15 (Sequoia), Apple Silicon (arm64)** với cùng API v1 và cùng các chốt an toàn như Linux/Windows (`/info.os = "darwin"`). Đây là phần **ngoài spec gốc**
(spec chỉ có Linux/Windows), thêm theo yêu cầu riêng. Mac Intel và macOS ≤ 13 **không** được hỗ trợ/kiểm.

> **Trạng thái kiểm chứng (đọc trước):** khi viết tài liệu này chưa có Mac thật để cài. Đã kiểm: biên dịch/`go vet` cho `darwin/arm64`, toàn bộ logic parser/executor bằng test Go trên Linux (với lệnh giả),
> cú pháp và nhánh `--dry-run` của script bằng `dash`. **Chưa chạy trên Mac thật** — việc đó do job CI `test-macos` (runner Apple Silicon) làm: test Go native, `install.sh` thật (`dscl`, `launchctl bootstrap system`),
> so số đo với `sysctl`/`vm_stat`/`df`, rồi `uninstall.sh --purge`. Mục "Giới hạn và điều chưa kiểm" ở cuối liệt kê từng giả định.

## 1. Lấy binary và checksum

```sh
sh packaging/release.sh        # Docker; ra agent/dist/pocketbridge-agent-darwin-arm64 và SHA256SUMS (cùng với linux/windows)
```

Hoặc build ngay trên Mac có Go: `cd agent && CGO_ENABLED=0 go build -trimpath -buildvcs=false -ldflags "-s -w" -o dist/pocketbridge-agent-darwin-arm64 ./cmd/pocketbridge-agent`
rồi `cd dist && shasum -a 256 pocketbridge-agent-darwin-arm64 > SHA256SUMS`.

`install.sh` **bắt buộc** kiểm `SHA256SUMS` bằng `shasum -a 256 -c` (đúng một dòng cho binary, sai → không cài). Checksum chỉ chống hỏng/tải lỗi, **không** chứng minh nguồn gốc (xem mục 4).

## 2. Xem trước (không đổi gì)

```sh
sh packaging/macos/install.sh --dry-run --listen 100.64.0.5
```

## 3. Cài

```sh
sudo sh packaging/macos/install.sh --listen 100.64.0.5 --id pocket1 --scopes metrics:read,actions:list,jobs:read
# --listen: IP tailnet (hoặc LAN) của Mac; agent từ chối 0.0.0.0. Bỏ qua thì tự lấy `tailscale ip -4` nếu có Tailscale.
# --pair-out FILE ghi bundle ra file 0600 thay vì in màn hình;  --no-start cài nhưng chưa nạp vào launchd;  --skip-pair khi cài lại.
```

Script làm 8 bước: kiểm checksum → tạo user `_pocketbridge` → cài binary/thư mục → viết config → cài LaunchDaemon → chứng chỉ TLS + pairing bundle → in hướng dẫn firewall → `launchctl bootstrap system` và đợi `GET /health` = 200.
Cài lại an toàn (giữ config có sẵn; đang nạp thì `bootout` rồi `bootstrap`).

### Bố cục

| Thứ | Đường dẫn | Chủ / quyền |
|---|---|---|
| Binary | `/usr/local/pocketbridge/bin/pocketbridge-agent` | root:wheel 0755 |
| Executable cho action process | `/usr/local/pocketbridge/actions/` | root:wheel 0755 (agent không ghi được) |
| Config | `/Library/Application Support/PocketBridge/config.yaml` | root:_pocketbridge 0640 |
| Action | `/Library/Application Support/PocketBridge/actions.d/` | root:_pocketbridge 0750 |
| State (TLS key, jobs, `pairing.json`) | `/Library/Application Support/PocketBridge/state/` | _pocketbridge:_pocketbridge 0750 |
| Log (`agent.jsonl`, `audit.jsonl`) | `/Library/Logs/PocketBridge/` | _pocketbridge:_pocketbridge 0750 |
| LaunchDaemon | `/Library/LaunchDaemons/com.pocketbridge.agent.plist` | root:wheel 0644 |

**Vì sao `/Library/Application Support` mà không phải `/usr/local/etc`:** đó là chỗ macOS dành cho dữ liệu hệ thống của ứng dụng (cùng cấu trúc `config/actions.d/state` với `%ProgramData%\PocketBridge` trên Windows),
không phụ thuộc Homebrew, và không đi qua symlink `/etc → /private/etc` (agent từ chối đường dẫn chứa symlink). Dấu cách trong đường dẫn an toàn: agent không có shell, script bộ cài trích dẫn mọi biến; plist truyền đường dẫn
như một phần tử mảng. Binary đặt dưới `/usr/local/pocketbridge/` (không phải `/usr/local/bin`) để cả cây root-owned, tách khỏi mọi thứ người dùng/Homebrew có thể ghi.

### User hệ thống `_pocketbridge`

Bộ cài tạo bằng `dscl` (không dùng `sysadminctl`: nó dành cho tài khoản người dùng tương tác): UID = GID = số trống đầu tiên trong **200..399** (dải Apple dành cho daemon bên thứ ba, < 500), `UserShell /usr/bin/false`,
`NFSHomeDirectory /var/empty`, `Password '*'` (không đăng nhập được), `IsHidden 1` (ẩn khỏi màn hình đăng nhập), không thuộc nhóm `admin`. Đã có sẵn user thì giữ nguyên và cảnh báo nếu shell/UID lệch.

```sh
dscl . -read /Users/_pocketbridge UniqueID UserShell NFSHomeDirectory IsHidden   # kiểm
id _pocketbridge
```

Cách khác nếu bạn không muốn tạo user: chạy agent bằng một tài khoản chuẩn riêng có sẵn (sửa `UserName`/`GroupName` trong plist **và** chủ sở hữu thư mục); **không** chạy bằng root (không có ranh giới giữa agent và action)
và **không** bằng tài khoản `admin` (nhóm `admin` ghi được `/Library` nên `SystemChecker` sẽ từ chối executable ở đó).

### LaunchDaemon

`RunAtLoad`; `KeepAlive{SuccessfulExit=false}` (chạy lại khi thoát bất thường, thoát mã 0 thì thôi); `ThrottleInterval=5` (cũng là nhịp thử lại nếu IP tailnet chưa sẵn lúc máy vừa khởi động — agent thoát vì không listen được);
`ExitTimeOut=20` (SIGTERM → chờ 20 giây → SIGKILL); `Umask=0077`; stdout/stderr nối `/dev/null` (agent tự ghi log có xoay vòng; trỏ stderr vào file sẽ phình mãi vì agent in lại mọi dòng log ra stderr).

```sh
sudo launchctl print system/com.pocketbridge.agent            # state = running, pid, last exit code
sudo launchctl kickstart -k system/com.pocketbridge.agent     # khởi động lại
sudo launchctl bootout system/com.pocketbridge.agent          # dừng + gỡ khỏi launchd (SIGTERM, agent thoát êm)
sudo launchctl bootstrap system /Library/LaunchDaemons/com.pocketbridge.agent.plist   # nạp lại
tail -f /Library/Logs/PocketBridge/agent.jsonl
```

Lỗi ngay khi khởi động (config hỏng) không để lại dòng log nào (stderr → `/dev/null`): chạy tay để thấy lỗi — `sudo -u _pocketbridge /usr/local/pocketbridge/bin/pocketbridge-agent run --config "/Library/Application Support/PocketBridge/config.yaml"`.

## 4. Gatekeeper, quarantine và chữ ký (đọc kỹ)

- Binary **chưa có Developer ID và chưa notarize**. Linker của Go tự ký **ad-hoc** cho `darwin/arm64` (Apple Silicon không chạy binary không có chữ ký; ad-hoc đủ để chạy). `codesign --verify /usr/local/pocketbridge/bin/pocketbridge-agent` phải đạt.
- Tải binary bằng trình duyệt thì macOS gắn cờ `com.apple.quarantine`; Gatekeeper có thể chặn hoặc hỏi khi chạy mà không ai bấm được (daemon). Bộ cài gỡ cờ (`xattr -d com.apple.quarantine`) **chỉ sau khi checksum đã khớp**;
  với `--skip-checksum` thì **không** tự gỡ và in lệnh để bạn tự quyết định. Rủi ro: gỡ cờ là tắt hàng rào của hệ điều hành cho đúng file này; vì chưa có chữ ký nguồn gốc, hàng rào còn lại là checksum — lấy `SHA256SUMS` qua kênh tin cậy
  (tự build, hoặc artifact CI của repo), như rủi ro 17 và 21 trong `docs/threat-model.md`.
- Khi có Developer ID: ký `codesign --options runtime --timestamp`, notarize `xcrun notarytool`, đính `stapler` — việc của bản 1.0.

## 5. Tường lửa, Tailscale, bind

- Agent chỉ nghe đúng IP bạn đưa vào `--listen` (mặc định loopback; từ chối wildcard). Tailscale phải chạy và đã có IP lúc daemon khởi động; chưa có thì agent thoát và launchd thử lại sau mỗi 5 giây.
- **Application Firewall:** bộ cài chỉ **đọc** trạng thái (`socketfilterfw --getglobalstate`) và **in** lệnh, không đổi gì. Nếu firewall bật và chặn kết nối đến, cho phép binary bằng tay:
  `sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /usr/local/pocketbridge/bin/pocketbridge-agent` rồi `--unblockapp` cùng đường dẫn. Mỗi lần thay binary (nâng cấp) macOS có thể coi là app mới và hỏi lại.
- macOS 15 có quyền riêng tư **Local Network**: có thể ảnh hưởng UDP broadcast của relay WoL tới LAN từ binary không-bundle chạy dưới launchd. Chưa kiểm; nếu WoL relay không ra LAN, đây là nghi phạm đầu tiên.
- **Tailscale ACL:** chỉ cho thiết bị Pocket 1 vào `tcp:4268` của Mac này (cùng mẫu `docs/install-linux.md`, mục Tailscale ACL).

## 6. Pairing bundle

Như Linux (`docs/install-linux.md`, mục "Ghép đôi"): bundle chứa token bootstrap dùng một lần, hết hạn sau `--ttl` (mặc định 15 phút, tối đa 1 giờ). Hết hạn:
`sudo /usr/local/pocketbridge/bin/pocketbridge-agent pair --config "/Library/Application Support/PocketBridge/config.yaml" --id <id mới> --scopes … --host <IP>`. `pair` chạy bằng root và từ chối nếu chưa có `key.pem`
(bộ cài chạy `fingerprint` dưới `_pocketbridge` trước).

## 7. Action

- **process:** executable phải ở `/usr/local/pocketbridge/actions/` (thêm `actions.executable_dirs` vào config nếu muốn ép), thuộc root, không group/other-writable, đường dẫn **không** chứa symlink (đừng dùng `/tmp`, `/var`, `/etc`: chúng là symlink sang `/private/...`).
  Ví dụ trong `packaging/examples/actions.d/`. Env của tiến trình con tối thiểu: `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`, `LANG=C.UTF-8`.
- **launchd** (khởi động/dừng/khởi động lại một LaunchDaemon): `packaging/examples/actions.d/restart-sunshine-macos.yaml` (`executor: {type: launchd, label: …, operation: …}`, `platforms: [darwin]`); nhãn phải có trong `services.allow`.
  Giới hạn MVP: **chỉ domain `system/`** (LaunchDaemon), không LaunchAgent của người dùng.
  - Agent không phải root nên thao tác ghi chạy `sudo -n /bin/launchctl …` và **chỉ thành công khi bạn thêm luật sudoers** khớp từng lệnh: mẫu ở `packaging/macos/pocketbridge-sudoers.example` (không cài tự động; kiểm bằng `visudo -cf`). Không có luật → job `failed`, nội dung lỗi nằm trong log agent.
  - `status`/`GET /services`: `launchctl print system/<label>` (không qua sudo).
  - `stop` = `launchctl kill SIGTERM` + chờ, **không** `bootout` (xem `docs/architecture.md`). Job `KeepAlive=true` được launchd chạy lại ngay nên `stop` báo lỗi "vẫn đang chạy" — đúng thực tế, không giả vờ thành công.

## 8. Số đo trên macOS

| Số đo | Nguồn (không cgo) | Ghi chú |
|---|---|---|
| `memory.total_bytes` | `sysctl hw.memsize` | |
| `memory.used_bytes` | `/usr/bin/vm_stat`: `(Anonymous pages − Pages purgeable + Pages wired down + Pages occupied by compressor) × cỡ trang` | = App Memory + Wired + Compressed của Activity Monitor. Cỡ trang lấy từ dòng đầu của `vm_stat` (16384 trên Apple Silicon). Thiếu một dòng → `memory = null`. `host_statistics64` cần Mach (cgo) nên không dùng |
| `cpu_percent` | `/usr/bin/top -l 2 -n 0 -s 1`: 100 − `idle` của dòng `CPU usage` **thứ hai** | Mẫu đầu của `top` là số tích luỹ từ lúc boot nên bị bỏ. macOS không có `kern.cp_time`; `host_processor_info` cần cgo. Mỗi lần đo ~1–2 giây ở sampler nền (2 giây/lần) |
| `uptime_seconds` | `sysctl kern.boottime` | như `uptime(1)`: tính cả thời gian máy ngủ |
| `filesystems[]` | `statfs`: used = (blocks − bfree) × bsize, avail = bavail × bsize | Mặc định `/` (volume hệ thống chỉ-đọc, sealed) **và** `/System/Volumes/Data` (nơi nằm dữ liệu người dùng/ứng dụng trên APFS). Các volume cùng container APFS dùng chung chỗ trống |
| `temperature_c` | — | **luôn `null`**, `capabilities.temperature = false`: Apple Silicon không có nguồn không-root đáng tin; agent không ước lượng |

Mọi lệnh ngoài chạy bằng đường dẫn tuyệt đối, argv hằng, env tối thiểu (`PATH`, `LANG=C`, `LC_ALL=C`), có timeout, không shell. Không đo được hoặc định dạng lạ → `null` + capability `false`.
CI so với `sysctl hw.memsize`, `vm_stat`/`top` và `df -k` ở `packaging/macos/test-install.sh`.

## 9. Kiểm tra từ máy khác

```sh
curl -sk https://100.64.0.5:4268/api/v1/health
curl -sk -H "Authorization: Bearer $TOKEN" https://100.64.0.5:4268/api/v1/info             # os: darwin, arch: arm64
curl -sk -H "Authorization: Bearer $TOKEN" https://100.64.0.5:4268/api/v1/metrics/summary  # temperature_c: null
```

## 10. Gỡ

```sh
sudo sh packaging/macos/uninstall.sh --dry-run
sudo sh packaging/macos/uninstall.sh            # bootout, xoá plist và binary; GIỮ config/state/log và user
sudo sh packaging/macos/uninstall.sh --purge    # xoá luôn dữ liệu và user + nhóm _pocketbridge (fingerprint sẽ đổi nếu cài lại)
```

Script không sửa Application Firewall; nếu đã `--add` binary, gỡ bằng `socketfilterfw --remove` (script in lệnh). Xoá pairing bundle nếu còn.

## Giới hạn và điều chưa kiểm

Chưa chạy trên Mac thật (CI `test-macos` là lần chạy đầu tiên). Các giả định cần xác nhận:

1. **Định dạng `vm_stat`/`top`/`launchctl print`:** fixture trong repo là bản dựng lại theo định dạng đã biết, không phải đầu ra thu từ máy thật. Job CI tải đầu ra thật lên artifact `macos-real-outputs`; hãy thay fixture bằng bản thật khi có.
2. **`launchctl print` không root:** chưa biết user `_pocketbridge` có đọc được `print system/<label>` không. Nếu không, `GET /services` báo `unknown` + "không đọc được trạng thái".
3. **Ngữ nghĩa `kickstart`/`kill` qua `sudo -n`:** chỉ kiểm bằng `launchctl`/`sudo` giả; chưa kiểm sudoers thật.
4. **Số `used` của `df` trên APFS:** `statfs` của volume APFS có thể khác trực giác (total là cả container); CI so từng số với `df -k` và sẽ đỏ nếu lệch quá ngưỡng.
5. **Dọn tiến trình con khi agent chết cứng, Local Network privacy, Application Firewall với daemon không ký:** theo tài liệu, chưa kiểm.
6. **Chi phí CPU của `top` mỗi 2 giây:** chưa đo.
7. **`top`/`vm_stat` chạy từ daemon không có tty, dưới user không-root:** chưa kiểm (CI chạy test Go dưới user runner, `test-install.sh` kiểm agent chạy dưới `_pocketbridge` qua launchd nên sẽ lộ ra nếu `cpu_percent`/`memory` là null).
8. Mac Intel, macOS ≤ 13, Tailscale chạy bằng `tailscaled` (không phải app) chưa kiểm.

## Gỡ lỗi nhanh

| Triệu chứng | Kiểm |
|---|---|
| `launchctl bootstrap` lỗi `5: Input/output error` | plist không phải root:wheel 0644, hoặc user `_pocketbridge` không tồn tại; `plutil -lint` plist; `ls -l /Library/LaunchDaemons/com.pocketbridge.agent.plist` |
| `state = not running`, `last exit code` ≠ 0 | chạy tay (xem mục LaunchDaemon) để thấy lỗi; thường là config hỏng hoặc IP `--listen` chưa có |
| Agent bị hệ điều hành giết ngay (`Killed: 9`) | `codesign --verify`; binary mất chữ ký (đã sửa/cắt) |
| `/health` 200 trên máy mà từ máy khác không vào được | Application Firewall, Tailscale ACL, `--listen` sai IP |
| `memory`/`cpu_percent` là `null` | chạy tay `/usr/bin/vm_stat`, `/usr/bin/top -l 2 -n 0 -s 1 \| grep 'CPU usage'` dưới `_pocketbridge` và so định dạng với `agent/internal/platform/darwin/testdata/` |
