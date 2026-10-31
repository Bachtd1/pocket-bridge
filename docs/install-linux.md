# Cài agent trên Linux (0.4.0; thư viện ROM chưa phát hành)

Dành cho Linux amd64/arm64 có systemd (Debian/Ubuntu, Raspberry Pi OS...). Agent cung cấp `/health`, `/info`, `/metrics/summary`, **action/job** (`/actions`, `/jobs`),
`/services`, relay **Wake-on-LAN** (`/wol`) và **ghép đôi** (`/pair/exchange`). Cài đầy đủ bằng `install.sh` chỉ mất vài giây (đo ~1 giây trên container Debian 12, chưa kể tải binary; mục "Cài nhanh"); các mục 1–9 bên dưới là cách làm tay, giải thích từng bước.
Windows: xem `docs/install-windows.md`.

> **Quy tắc vàng:** agent chạy bằng user `pocketbridge` **không phải root**; mọi file mà agent chạy hoặc tin (config action, script, thư mục chứa chúng) thuộc **root** và agent **không ghi được**.
> Agent tự kiểm điều này và **từ chối** action vi phạm (nó bị bỏ riêng, agent vẫn chạy). Nếu agent chạy bằng root, không còn ranh giới nào giữa agent và action.

## Cài nhanh bằng `install.sh` (khuyến nghị)

```sh
sh packaging/release.sh                 # build 3 binary + SHA256SUMS bằng Docker (hoặc lấy artifact CI); chưa có chữ ký, xem threat-model rủi ro 17
sudo sh packaging/linux/install.sh --dry-run --listen 100.64.0.5      # xem trước mọi thay đổi (không cần root)
sudo sh packaging/linux/install.sh --listen 100.64.0.5 --id pocket1 \
        --scopes metrics:read,actions:list,jobs:read,action:sunshine
```

`install.sh` làm đúng 8 bước của spec, và **dừng ngay** khi một bước hỏng:

1. Kiểm `SHA256SUMS` của binary (`sha256sum -c` trên đúng dòng của file; thiếu/lệch → không cài).
2. Tạo user hệ thống `pocketbridge`, **không shell** (`nologin`), không home.
3. Cài `/usr/local/bin/pocketbridge-agent` (root:root 0755); thư mục `/etc/pocketbridge` (root:pocketbridge 0750), `actions.d`, `/opt/pocketbridge/actions` (root), `/var/lib/pocketbridge` + `/var/log/pocketbridge` (pocketbridge).
4. Viết `/etc/pocketbridge/config.yaml` (root:pocketbridge **0640**: root ghi, agent đọc qua nhóm) với `listen` = `--listen`; không ghi đè config có sẵn; rồi `config check` dưới user dịch vụ.
5. Cài systemd unit có hardening (`packaging/systemd/pocketbridge-agent.service`).
6. Bind đúng `--listen` (agent từ chối `0.0.0.0`). Không đưa `--listen` thì dùng `tailscale ip -4` nếu có, không thì dừng.
7. Firewall: **chỉ in lệnh** `ufw`/`nftables`/`firewalld` theo công cụ đang có. Chỉ khi thêm `--configure-firewall` mới áp luật **ufw** (cần ufw đang bật và giao diện `tailscale0` hoặc `--firewall-from CIDR`); nftables/firewalld không bao giờ bị sửa tự động.
8. Sinh chứng chỉ (dưới user dịch vụ), `pair` → **pairing bundle** (in ra màn hình, hoặc `--pair-out FILE` 0600), rồi `systemctl enable --now` và thử `/health`.

Tham số chính: `--listen`, `--id`/`--scopes` (token dài hạn cấp cho client), `--host`/`--name` (trong bundle), `--ttl` (hạn bootstrap, mặc định 15m, tối đa 1h), `--pair-out`, `--binary`/`--sums`, `--configure-firewall`/`--firewall-from`, `--no-start`, `--skip-pair` (cài lại/nâng cấp), `--dry-run`. `sh install.sh --help` để xem đủ.
Gỡ: `sudo sh packaging/linux/uninstall.sh [--dry-run] [--purge]` (mặc định giữ config/state/log và user; `--purge` xoá hết).

Đã chạy thật trên Debian 12 (systemd 252, container `--privileged`) bằng `packaging/linux/test-install.sh`: user/quyền/unit đúng, hardening bật (kể cả `MemoryDenyWriteExecute`), exchange 200 rồi 409, thu hồi không cần restart, gỡ sạch.

### Ghép đôi (pairing bundle) với Pocket 1

Bundle là JSON theo `schemas/pairing_bundle.schema.json` với `credential.type = "bootstrap"`: token **dùng một lần**, hết hạn sau `--ttl`, chỉ có quyền đổi lấy token dài hạn qua `POST /api/v1/pair/exchange`
(hợp đồng đầy đủ: `docs/api.md`). Chuyển bundle sang Pocket 1 bằng kênh riêng (không để trong thư mục chia sẻ/đám mây), import, rồi **xoá**. Client đổi token → nhận token dài hạn đúng `--scopes`, bootstrap vô hiệu ngay.

- Hết hạn mà chưa đổi → chạy lại: `sudo pocketbridge-agent pair --config /etc/pocketbridge/config.yaml --id <id> --scopes … --host <IP>`.
- Mất mạng đúng lúc đổi → client nhận 409 `already_exchanged` và **không** lấy lại được token (chủ ý, `docs/threat-model.md`): tạo bundle mới với `--id` mới, thu hồi token cũ bằng `token revoke --id <id cũ>`.
- `pair` chạy bằng **root** và **từ chối** khi chưa có `key.pem` (nó sẽ thuộc root, agent không đọc được): `install.sh` chạy `fingerprint` bằng user dịch vụ trước; làm tay thì cũng vậy (mục 5).
- Trạng thái ghép đôi do agent ghi nằm ở `/var/lib/pocketbridge/pairing.json` (0600); config vẫn root-owned. `token list` hiện cả token cấp qua ghép đôi; `token revoke --id` thu hồi được chúng.

## 0. Chạy thử nhanh (foreground, không cần root)

```sh
mkdir -p ~/pb && cd ~/pb
cat > config.yaml <<'EOF'
schema: 1
listen: ["127.0.0.1:4268"]
state_dir: /home/USER/pb/state
log: {path: /home/USER/pb/logs/agent.jsonl}
EOF
chmod 600 config.yaml
./pocketbridge-agent config check --config config.yaml
./pocketbridge-agent token create --config config.yaml --id test --scopes metrics:read   # in token đúng 1 lần
./pocketbridge-agent run --config config.yaml &
curl -sk https://127.0.0.1:4268/api/v1/health
```

Chạy thử kiểu này chưa có action nào (action cần file thuộc root, xem bước 7). Muốn thử action trên máy cá nhân, làm bước 2–7 đầy đủ.

## 1. Tải binary và kiểm checksum

Lấy `pocketbridge-agent-linux-<amd64|arm64>` và `SHA256SUMS` từ artifact CI, hoặc tự build bằng `sh packaging/release.sh` (Docker; build lặp lại được: cùng toolchain + mã nguồn → cùng hash).
Chưa có chữ ký phát hành (bản 1.0): `SHA256SUMS` chỉ chống file hỏng/tải lỗi, không chứng minh nguồn gốc.

```sh
sha256sum -c --ignore-missing SHA256SUMS      # kiểm file bạn có
sudo install -m 0755 -o root -g root pocketbridge-agent-linux-arm64 /usr/local/bin/pocketbridge-agent
pocketbridge-agent version
```

## 2. Tạo user và thư mục

```sh
sudo useradd --system --no-create-home --shell /usr/sbin/nologin pocketbridge
sudo install -d -o root -g pocketbridge -m 0750 /etc/pocketbridge
sudo install -d -o root -g pocketbridge -m 0750 /etc/pocketbridge/actions.d     # định nghĩa action: root ghi, agent chỉ đọc
sudo install -d -o root -g root -m 0755 /opt/pocketbridge /opt/pocketbridge/actions   # script/executable của action: root sở hữu
sudo install -d -o pocketbridge -g pocketbridge -m 0750 /var/lib/pocketbridge /var/log/pocketbridge
```

`/var/lib/pocketbridge` chứa chứng chỉ TLS và `jobs/` (trạng thái job, output, khoá idempotency); chỉ nơi này và log là agent ghi được.

## 3. Config

Địa chỉ `listen` phải là IP LAN hoặc IP Tailscale của máy (`tailscale ip -4`) — agent từ chối `0.0.0.0`/`::`
trừ khi bạn đặt `allow_wildcard_listen: true` (không khuyến nghị). Mẫu đầy đủ có chú thích: `packaging/examples/config.example.yaml`.

```sh
sudo tee /etc/pocketbridge/config.yaml >/dev/null <<'EOF'
schema: 1
listen:
  - 100.64.0.5:4268          # IP tailnet (ví dụ)
  # - 192.168.1.20:4268      # IP LAN nếu cần
node_id: workstation         # hiện trong job.node_id; mặc định là hostname
metrics:
  filesystems:
    - { name: root, mount: / }
    # - { name: data, mount: /mnt/data }

actions:
  dir: /etc/pocketbridge/actions.d
  # executable_dirs: [/opt/pocketbridge/actions]   # (tuỳ chọn) executable phải nằm trong các thư mục này

jobs:
  max_concurrent: 2          # job chạy song song
  max_queue: 100             # job chờ; đầy → 429
  idempotency_ttl: 24h       # giữ khoá request_id: retry trong khoảng này không chạy lại
  retention: 168h            # giữ job đã xong 7 ngày (>= idempotency_ttl)
  min_free_bytes: 104857600  # dưới 100 MiB trống: từ chối job mới (503); /health vẫn ok
  kill_grace: 5s             # chờ giữa SIGTERM và SIGKILL khi huỷ/timeout

# Allowlist unit systemd: action systemd và GET /services chỉ làm việc với các unit ở đây.
services:
  allow:
    - sunshine.service

# Relay Wake-on-LAN (tuỳ chọn): chỉ MAC liệt kê; broadcast/cổng do bạn quyết định, client không chọn được.
# wol:
#   rate_limit_per_minute: 6
#   allow:
#     - { mac: "AA:BB:CC:DD:EE:FF", broadcast: 192.168.1.255, port: 9 }
EOF
sudo chown root:pocketbridge /etc/pocketbridge/config.yaml
sudo chmod 0640 /etc/pocketbridge/config.yaml
sudo -u pocketbridge pocketbridge-agent config check --config /etc/pocketbridge/config.yaml
```

Config chỉ root ghi; user `pocketbridge` chỉ đọc. Mặc định `state_dir` là `/var/lib/pocketbridge` và log là
`/var/log/pocketbridge/agent.jsonl` (audit: `audit.jsonl` cùng thư mục). Config (trừ danh sách token) chỉ được nạp lúc khởi động.

### 3b. Thư viện ROM (tuỳ chọn)

Cho Pocket 1 duyệt và tải **ROM của chính bạn** (bản dump riêng, homebrew) đặt trên máy này, qua cùng cổng TLS + token. Agent chỉ **đọc** các thư mục bạn khai báo; không tải gì từ Internet, không ghi vào thư viện.
Thêm vào `config.yaml` (cần khởi động lại agent):

```yaml
library:
  max_file_bytes: 4294967296   # mặc định 4 GiB; file lớn hơn không được liệt kê, tải bị 403
  hash_workers: 1              # sha256 tính nền cho ETag/Range; 0 = tắt (sha256 luôn null)
  roots:
    - id: roms                 # [a-z0-9-]{1,32}, dùng trong URL /api/v1/library/roms/...
      name: ROM của tôi        # tên hiển thị trên Pocket 1
      path: /srv/roms          # tuyệt đối, đã chuẩn hoá (không '..', không '/' cuối)
```

Quy tắc và cạm bẫy:

- **Quyền đọc là việc của bạn.** Agent chạy bằng user `pocketbridge`, **không tự chmod/chown**. Thư mục (và mọi thư mục cha) phải cho user đó duyệt (`x`) và đọc (`r`); file phải đọc được:
  `sudo setfacl -R -m u:pocketbridge:rX /srv/roms` (hoặc thêm user vào nhóm sở hữu). Kiểm đúng như agent sẽ thấy:
  `sudo -u pocketbridge pocketbridge-agent config check --config /etc/pocketbridge/config.yaml` — thư mục không đọc được báo `LỖI` kèm cách xử lý và mã thoát 1.
  Lúc chạy, root không đọc được **không làm agent dừng** (ví dụ ổ USB chưa gắn): agent ghi log lỗi, index rỗng, quét lại mỗi 10 phút.
- **Đừng đặt thư viện dưới `/home`, `/root`, `/run/user`**: unit có `ProtectHome=true` nên agent không thấy chúng. Dùng `/srv`, `/mnt`, `/media`; nếu bắt buộc, tạo drop-in
  `sudo systemctl edit pocketbridge-agent` với `[Service]` / `ProtectHome=read-only` (chỉ-đọc, đủ cho thư viện).
- **Root không được chứa hoặc nằm trong** `state_dir` (có `key.pem`), thư mục log, `actions.dir` hay thư mục chứa `config.yaml` (vd `path: /` bị từ chối): `config check` và lúc khởi động đều từ chối.
- Chỉ phục vụ **file thường không ẩn**: bỏ file/thư mục bắt đầu bằng `.`, symlink (kể cả symlink trong thư viện trỏ vào trong), thiết bị. Chính `path` của root được phép là symlink; nội dung bên trong thì không.
  Thư viện nên chỉ chứa ROM: token `library:read` đọc được mọi file thường trong root.
- Tên thư mục cấp 1 trùng tên hệ máy chuẩn của Leaf (`GBA`, `SNES`, `PSX`…, không phân biệt hoa/thường) được gắn `system_hint`; thư mục khác thì client tự phân loại.
- Hash nền rất nhẹ (1 worker, nice 19, ~5% một nhân, nhường khi có lượt tải) nhưng lần đầu với thư viện lớn mất lâu (bội số của thời gian đọc đĩa); kết quả lưu ở `/var/lib/pocketbridge/library/<id>.hashes`. File mới sửa trong 30 giây chưa được hash.

Token riêng cho việc này (scope tối thiểu), rồi thử từ máy khác:

```sh
sudo pocketbridge-agent token create --config /etc/pocketbridge/config.yaml --id pocket1-library --scopes library:read --expires 4320h
curl -sk -H "Authorization: Bearer $TOKEN" https://NODE:4268/api/v1/library
curl -sk -H "Authorization: Bearer $TOKEN" "https://NODE:4268/api/v1/library/roms/index?limit=50"
# tải (đường dẫn mã hoá URL); -C - tiếp tục tải dở nhờ Range; ETag = "sha256:<hex>" khi đã băm xong
curl -sk -H "Authorization: Bearer $TOKEN" -OJ "https://NODE:4268/api/v1/library/roms/file?path=GBA/Advance%20Wars%20(USA).gba"
```

Mỗi lần tải ghi một dòng `library_download` trong `audit.jsonl` (đường dẫn, số byte, token). Chi tiết hợp đồng: `docs/api.md`, rủi ro: `docs/threat-model.md`.

## 4. Token (mỗi node một token riêng, scope tối thiểu)

```sh
# Pocket 1: xem số đo + liệt kê/chạy action nhóm sunshine + xem job + bật máy qua relay
sudo pocketbridge-agent token create --config /etc/pocketbridge/config.yaml \
     --id pocket1 --scopes metrics:read,actions:list,jobs:read,action:sunshine,wol:send --expires 4320h
```

| Scope | Cho phép |
|---|---|
| `metrics:read` | `/info`, `/metrics/summary`, `/services` |
| `actions:list` | `GET /actions` (chỉ thấy action thuộc nhóm token có) |
| `action:<nhóm>` | chạy và huỷ action có `permissions.scope: action:<nhóm>` |
| `jobs:read` | xem job/output/sự kiện (cộng scope nhóm của action) |
| `wol:send` | `POST /wol` |
| `library:read` | duyệt và tải thư viện ROM (`GET /library`, `/library/{root}/index`, `/library/{root}/file`) — xem 3b |

Token **chỉ hiện một lần** trên stdout; config chỉ giữ SHA-256. Lệnh giữ nguyên chủ/nhóm/quyền file config.
Xem danh sách: `token list`. Thu hồi: `token revoke --id pocket1` (agent đang chạy nạp lại trong ≤5 giây).

Cách tốt hơn cho thiết bị thật là **ghép đôi** (`pocketbridge-agent pair`, xem "Ghép đôi" ở đầu tài liệu): token dài hạn không bao giờ đi qua màn hình/clipboard, chỉ qua kênh TLS đã ghim fingerprint.

## 5. Fingerprint

```sh
sudo -u pocketbridge pocketbridge-agent fingerprint --config /etc/pocketbridge/config.yaml
# sha256:2506caddd9...   → nhập vào client (tls_mode: pinned)
```

Chạy bằng user `pocketbridge` để chứng chỉ sinh ra thuộc đúng user dịch vụ. Chứng chỉ được tạo một lần và
dùng lại; chỉ đổi khi bạn xoá `cert.pem`/`key.pem` trong `state_dir`.

## 6. systemd

```sh
sudo cp packaging/systemd/pocketbridge-agent.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now pocketbridge-agent
systemctl status pocketbridge-agent
sudo journalctl -u pocketbridge-agent -n 20
```

Unit có hardening: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem=strict`, `ProtectHome`,
`RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`, bỏ mọi capability; chỉ `/var/lib/pocketbridge` và
`/var/log/pocketbridge` ghi được (qua `StateDirectory`/`LogsDirectory`). Ghi chú:

- `AF_UNIX` cần để `systemctl` nói chuyện với systemd/polkit. Không dùng action systemd thì bỏ được.
- Action process cần ghi vào đâu đó (ví dụ `/srv/backup`) thì mở **đúng đường dẫn đó** bằng `ReadWritePaths=` trong unit (dòng đã có sẵn dạng chú thích).
- `KillMode=control-group`: khi dịch vụ dừng, mọi tiến trình con của job cũng bị giết. Đây là lưới an toàn khi agent chết cứng (xem threat model).
- `PrivateTmp=true`: action process thấy `/tmp` riêng.

## 7. Định nghĩa action

Mỗi action là một file YAML trong `/etc/pocketbridge/actions.d/` (**root sở hữu, `0644`, thư mục root ghi**). Mẫu thật nằm ở `packaging/examples/actions.d/`.

### 7a. Bật/tắt/khởi động lại một dịch vụ (systemd)

```sh
sudo install -o root -g root -m 0644 packaging/examples/actions.d/start-sunshine.yaml   /etc/pocketbridge/actions.d/
sudo install -o root -g root -m 0644 packaging/examples/actions.d/stop-sunshine.yaml    /etc/pocketbridge/actions.d/
sudo install -o root -g root -m 0644 packaging/examples/actions.d/restart-sunshine.yaml /etc/pocketbridge/actions.d/
sudo install -o root -g root -m 0644 packaging/examples/actions.d/status-sunshine.yaml  /etc/pocketbridge/actions.d/
```

```yaml
schema: 1
id: restart-sunshine
name: Restart Sunshine
risk: medium            # client bắt giữ nút A khi xác nhận
timeout: 60s
platforms: [linux]
singleton: true         # không cho hai lần restart chạy chồng nhau (409 job_conflict)
executor: {type: systemd, unit: sunshine.service, operation: restart}   # operation: status|start|stop|restart
parameters: []
permissions: {scope: action:sunshine}
```

`unit` **phải có** trong `services.allow` của config, nếu không action bị bỏ. User `pocketbridge` không có quyền `systemctl start/stop` mặc định: cài **quy tắc polkit giới hạn theo từng unit**:

```sh
sudo install -o root -g root -m 0644 packaging/polkit/50-pocketbridge.rules /etc/polkit-1/rules.d/50-pocketbridge.rules
sudo $EDITOR /etc/polkit-1/rules.d/50-pocketbridge.rules     # sửa mảng `allowed` cho đúng unit của bạn
sudo systemctl restart polkit
```

`status` (và `GET /services`) chỉ dùng `systemctl show`, không cần polkit.

### 7b. Chạy một script (process)

```sh
sudo install -o root -g root -m 0755 packaging/examples/actions/backup-nas.sh /opt/pocketbridge/actions/backup-nas.sh
sudo install -o root -g root -m 0644 packaging/examples/actions.d/backup-nas.yaml /etc/pocketbridge/actions.d/
```

```yaml
schema: 1
id: backup-nas
name: Backup NAS
risk: medium
timeout: 2h
singleton: true
executor:
  type: process
  executable: /opt/pocketbridge/actions/backup-nas.sh   # tuyệt đối, đã chuẩn hoá, KHÔNG symlink
  args: ["--mode", "{{ mode }}"]                         # mỗi phần tử: literal HOẶC đúng "{{ id }}"
parameters:
  - {id: mode, type: enum, values: [quick, full], default: quick}
output:
  max_bytes: 1048576                          # vượt thì giữ phần ĐUÔI
  progress_regex: '^PROGRESS ([0-9]{1,3})$'   # đúng một nhóm bắt, số 0-100
  redact: ['password=\S+']                    # đoạn khớp bị thay bằng *** trước khi lưu/trả
permissions: {scope: action:backup}
```

Quy tắc tham số (đủ ví dụ ở `packaging/examples/actions.d/ai-upscale.yaml`): `enum`, `bool`, `integer` (bắt buộc `min`/`max`), `string` (bắt buộc `regex` **neo `^…$`** và `max_length`; không được bắt đầu bằng `-`),
`path` (bắt buộc `root` tuyệt đối; client gửi đường dẫn **tương đối** trong root; `..` và symlink thoát root bị từ chối). `secret: true` che giá trị khỏi log/audit/output.
Giá trị chứa ký tự điều khiển/newline/NUL/Unicode đảo chiều luôn bị từ chối. **Không có cách nào truyền "một đoạn lệnh".**

Script của bạn vẫn phải tự an toàn: đừng `eval "$1"` hay ghép tham số vào lệnh shell khác.

### 7c. Kiểm trước khi chạy

```sh
sudo -u pocketbridge pocketbridge-agent actions check --config /etc/pocketbridge/config.yaml
```

Lệnh nạp `actions.d` đúng như agent (cùng bộ kiểm quyền/symlink/tham số), in từng action `OK` (kèm hash) và từng file bị `BỎ` kèm lý do. Mã thoát 1 nếu có file bị bỏ. Ví dụ:

```text
  OK    restart-sunshine         medium   scope=action:sunshine  executor=systemd         sha256=1c445b9a88fb  /etc/pocketbridge/actions.d/restart-sunshine.yaml
  BỎ    /etc/pocketbridge/actions.d/evil.yaml
        lý do: executable không an toàn: /opt/pocketbridge/actions/evil.sh: nhóm của agent ghi được (group-writable)
```

Đổi action phải `sudo systemctl restart pocketbridge-agent` (action chỉ nạp lúc khởi động). `/info` hiện `actions.loaded/rejected` và hash tổng hợp để phát hiện cấu hình bị sửa.

## 8. Tường lửa

Chỉ mở cổng 4268 cho Pocket 1 / tailnet. Ví dụ ufw:

```sh
sudo ufw allow in on tailscale0 to any port 4268 proto tcp      # chỉ qua giao diện tailnet
# hoặc theo nguồn LAN:  sudo ufw allow from 192.168.1.0/24 to any port 4268 proto tcp
```

Relay WoL gửi UDP broadcast ra LAN của **máy chạy agent** (không cần mở cổng vào).

### Tailscale ACL: chỉ Pocket 1 vào cổng agent

Firewall máy chỉ lọc theo giao diện; **ACL của Tailscale** mới giới hạn *thiết bị nào* trong tailnet được tới cổng agent. Trong tailnet policy (admin console → Access controls):

```jsonc
{
  "tagOwners": { "tag:pocket1": ["autogroup:admin"], "tag:pbnode": ["autogroup:admin"] },
  "grants": [
    { "src": ["tag:pocket1"], "dst": ["tag:pbnode"], "ip": ["tcp:4268"] }
  ]
  // Không có grant "*:*" hay "autogroup:member -> tag:pbnode": thiết bị khác trong tailnet bị chặn mặc định.
}
```

Gắn thẻ: Pocket 1 `tailscale up --advertise-tags=tag:pocket1`; node Linux `sudo tailscale up --advertise-tags=tag:pbnode`. Kiểm: từ một thiết bị tailnet **khác**, `nc -vz <ip-node> 4268` phải thất bại; từ Pocket 1 phải thành công.
(Cú pháp `acls` cũ: `{"action":"accept","src":["tag:pocket1"],"dst":["tag:pbnode:4268"]}`.) Token vẫn bắt buộc: Tailscale là lớp mạng, không thay cho xác thực ứng dụng.

## 9. Kiểm tra từ máy khác

```sh
FP=sha256:...   # từ bước 5
echo | openssl s_client -connect NODE:4268 2>/dev/null | openssl x509 -outform DER | sha256sum   # phải khớp FP
curl -sk https://NODE:4268/api/v1/health
curl -sk -H "Authorization: Bearer $TOKEN" https://NODE:4268/api/v1/metrics/summary
curl -sk -H "Authorization: Bearer $TOKEN" https://NODE:4268/api/v1/actions
curl -sk -H "Authorization: Bearer $TOKEN" https://NODE:4268/api/v1/services
```

Chạy một action (nhớ `request_id` là UUID v4 mới; **retry phải dùng lại đúng request_id**):

```sh
RID=$(cat /proc/sys/kernel/random/uuid)
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"request_id\":\"$RID\",\"parameters\":{\"mode\":\"quick\"}}" https://NODE:4268/api/v1/actions/backup-nas/run
# → 202 {"id":"01J...","state":"queued",...}      (gửi lại cùng $RID → 200 + Idempotent-Replayed: true, KHÔNG chạy lần hai)
JOB=01J...
curl -sk -N -H "Authorization: Bearer $TOKEN" https://NODE:4268/api/v1/jobs/$JOB/events     # luồng SSE: state/progress/output/heartbeat
curl -sk -H "Authorization: Bearer $TOKEN" "https://NODE:4268/api/v1/jobs/$JOB/output?offset=0"
curl -sk -X POST -H "Authorization: Bearer $TOKEN" https://NODE:4268/api/v1/jobs/$JOB/cancel
```

Bật máy qua relay (MAC phải có trong `wol.allow`):

```sh
curl -sk -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"request_id\":\"$(cat /proc/sys/kernel/random/uuid)\",\"mac\":\"AA:BB:CC:DD:EE:FF\"}" https://NODE:4268/api/v1/wol
```

(`-k` chỉ để curl bỏ qua kiểm CA; việc xác minh node là bước so fingerprint ở dòng trên.)

## Windows

Xem `docs/install-windows.md` (Windows Service + `install.ps1`, ACL, firewall, pairing).

## macOS

Xem `docs/install-macos.md` (Apple Silicon, LaunchDaemon `com.pocketbridge.agent`, user hệ thống `_pocketbridge`, `install.sh` cho macOS).

## Gỡ lỗi nhanh

| Triệu chứng | Nguyên nhân thường gặp |
|---|---|
| `listen ... mọi giao diện mạng; từ chối` | Dùng IP cụ thể thay cho `0.0.0.0` |
| `không listen được` | IP không thuộc máy này hoặc cổng bận |
| 401 liên tục rồi 429 | Token sai/hết hạn/đã thu hồi; chờ ≤60 giây sau khi sửa |
| `temperature: null` | Máy không có cảm biến đọc được (VM, một số NAS); xem `thermal.zone` |
| `cpu_percent: null` | Trong ~2 giây đầu sau khi agent khởi động |
| Action không xuất hiện trong `/actions` | `actions check` xem lý do bị bỏ; token có `actions:list` **và** scope nhóm không? |
| `actions check`: "nhóm của agent ghi được" / "thuộc sở hữu của chính user chạy agent" | File/thư mục do user `pocketbridge` sở hữu hoặc group-writable: `chown root:root`, `chmod go-w` cả file lẫn thư mục cha |
| `actions check`: "chứa symlink" | Dùng đường dẫn thật (`readlink -f`), ví dụ `/usr/bin/…` thay vì `/bin/…` nếu `/bin` là symlink |
| `actions check`: "không nằm trong services.allow" | Thêm unit vào `services.allow` rồi khởi động lại |
| Job `failed`, log agent: "executable không còn an toàn" | File bị đổi quyền/thay thế sau khi nạp; sửa lại rồi khởi động lại |
| `systemctl start` lỗi "Access denied" trong output job | Chưa cài quy tắc polkit cho unit đó (bước 7a) |
| `/services` toàn `unknown` | Máy không chạy systemd (container) hoặc thiếu `/usr/bin/systemctl` |
| `POST /actions/x/run` 503 | Đĩa dưới `jobs.min_free_bytes`; `/health` vẫn 200 |
| Job `interrupted` sau khi agent khởi động lại | Đúng thiết kế: agent không tự chạy lại; tạo `request_id` mới để chạy lại |
| `install.sh`: "checksum KHÔNG khớp" / "SHA256SUMS phải có đúng một dòng" | Binary không phải bản đã build (hoặc `SHA256SUMS` cũ). Build lại bằng `packaging/release.sh`; đừng `--skip-checksum` trừ khi bạn tự chịu trách nhiệm |
| `pair`: "Chưa có chứng chỉ TLS ... Chạy `fingerprint` bằng user dịch vụ trước" | Chạy `sudo -u pocketbridge pocketbridge-agent fingerprint --config /etc/pocketbridge/config.yaml` rồi `pair` lại |
| `pair`: "Id ... đã được dùng" | Id token không bao giờ cấp lại (kể cả đã thu hồi). Dùng `--id` khác (vd `pocket1-2`) |
| `pair`: "listen đầu tiên ... là loopback/wildcard: cần --host" | Bundle cần địa chỉ client tới được: thêm `--host <IP tailnet/LAN>` |
| Client nhận 409 `already_exchanged` khi ghép đôi | Bootstrap đã dùng (kể cả khi mất mạng sau POST). Tạo bundle mới; xem `token list` để thu hồi token cấp ra mà client không nhận được |
| Client nhận 401 ở `pair/exchange` | Bootstrap hết hạn (`--ttl`) hoặc sai. Chạy lại `pair` |
| Agent không khởi động: "sổ ghép đôi ... hỏng" | `/var/lib/pocketbridge/pairing.json` bị sửa/hỏng. Không tự bỏ qua (bootstrap đã dùng sẽ sống lại). Khôi phục từ bản sao lưu, hoặc xoá file **và** thu hồi/tạo lại mọi token liên quan |
