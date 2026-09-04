# Hướng dẫn cài RomM trên Synology DS723+ (Tiếng Việt)

> Dành cho DSM 7.2+ • Container Manager • image `rommapp/romm:latest`.
> RomM là app tự host để **quản lý + chơi** thư viện ROM. Repo này chỉ là mã nguồn,
> **không kèm ROM** — ROM do bạn tự cung cấp và gắn vào qua thư mục `library`.

---

## 1. Kiến trúc khi chạy trên NAS

Image production đã gói sẵn nginx + backend + frontend + **Redis/Valkey**. Bạn chỉ cần thêm
**1 database** (MariaDB). Tổng cộng **2 container**:

| Container | Việc | Ghi chú |
|---|---|---|
| `romm` | Web + API + worker + Redis (bundle) | nginx nội bộ cổng **8080**, chạy user UID/GID **1000** |
| `romm-db` | Cơ sở dữ liệu MariaDB | Bắt buộc, không có sẽ không chạy |

CPU/GPU của NAS **không phải vấn đề**: chơi game (EmulatorJS) chạy trên trình duyệt người chơi,
không đè lên NAS. Giới hạn thật của DS723+ là **2 khay đĩa** = dung lượng thư viện.

**RAM:** DS723+ mặc định 2GB — chạy được. Nếu thư viện lớn / quét nặng / nhiều người dùng chung,
nên nâng RAM (khe SO-DIMM DDR4 ECC).

---

## 2. Chuẩn bị trên DSM

**B1 — Cài Container Manager**
Package Center → tìm **Container Manager** → Install.

**B2 — Tạo shared folder chứa ROM**
Control Panel → Shared Folder → Create → tên `roms` (nằm ở `/volume1/roms`).
Đây là nơi bạn bỏ file ROM vào (quản lý qua File Station).

**B3 — Tạo cây thư mục thư viện** trong File Station, bên trong `roms`:

```
/volume1/roms/
├─ roms/
│  ├─ gbc/     ← bỏ file .gbc vào đây
│  ├─ snes/    ← bỏ file .sfc/.smc
│  ├─ n64/     ← .n64/.z64
│  └─ psx/     ← .bin/.cue/.chd ...
└─ bios/       ← (tùy chọn) bios theo từng hệ máy
   └─ psx/
```

- Tên thư mục hệ máy phải đúng **slug** của RomM (`gbc`, `gba`, `snes`, `nes`, `n64`, `psx`, `ps2`, `nds`, `gc`, `arcade`...).
- Danh sách slug: <https://docs.romm.app/latest/Platforms-and-Players/Supported-Platforms/>
- Cấu trúc thư mục (2 kiểu hợp lệ): <https://docs.romm.app/latest/Getting-Started/Folder-Structure/>

**B4 — Tạo khóa bí mật (ROMM_AUTH_SECRET_KEY)**
Khóa này giữ phiên đăng nhập ổn định giữa các lần khởi động lại. **Phải đặt cố định** (nếu để trống,
mỗi lần restart container sẽ sinh khóa mới → mọi người bị đăng xuất).

Cách tạo (chọn 1):
- Bật SSH trên NAS rồi chạy lệnh sau, copy chuỗi 64 ký tự hex nhận được:

  ```
  openssl rand -hex 32
  ```

- Hoặc dùng bất kỳ trình tạo chuỗi hex 64 ký tự nào bạn tin tưởng.

> ⚠️ **Không** đưa khóa/mật khẩu thật vào file dùng chung hay chỗ công khai. Điền trực tiếp lúc tạo Project.

---

## 3. File docker-compose cho DS723+

Thay các chỗ `<...>` bằng giá trị của bạn. `DB_PASSWD` (ở `romm`) phải **trùng** `MARIADB_PASSWORD` (ở `romm-db`).

```yaml
name: romm

volumes:
  romm_resources:   # ảnh bìa/screenshot tải về (Container Manager tự quản)
  romm_assets:      # save/state người dùng upload
  romm_config:      # config.yml (tùy chọn)
  romm_redis:       # dữ liệu Redis/Valkey bundle
  romm_mariadb:     # dữ liệu database

services:
  romm:
    image: rommapp/romm:latest
    container_name: romm
    restart: unless-stopped
    environment:
      # --- Database ---
      - DB_HOST=romm-db
      - DB_NAME=romm
      - DB_USER=romm-user
      - DB_PASSWD=<MAT_KHAU_DB>                 # trùng MARIADB_PASSWORD bên dưới
      # --- Bảo mật (bắt buộc) ---
      - ROMM_AUTH_SECRET_KEY=<KEY_64_HEX>       # từ: openssl rand -hex 32
      # --- Nhà cung cấp metadata (để trống vẫn chạy; điền để có ảnh bìa/thông tin) ---
      - IGDB_CLIENT_ID=
      - IGDB_CLIENT_SECRET=
      - SCREENSCRAPER_USER=
      - SCREENSCRAPER_PASSWORD=
      - STEAMGRIDDB_API_KEY=
      - RETROACHIEVEMENTS_API_KEY=
      - HASHEOUS_API_ENABLED=true               # nhận diện game theo mã hash, không cần key
    ports:
      - "8998:8080"                             # truy cập qua http://<IP-NAS>:8998
    volumes:
      - /volume1/roms:/romm/library             # thư viện ROM (bind mount để quản lý qua File Station)
      - romm_resources:/romm/resources
      - romm_assets:/romm/assets
      - romm_config:/romm/config
      - romm_redis:/redis-data
    depends_on:
      romm-db:
        condition: service_healthy

  romm-db:
    image: mariadb:11
    container_name: romm-db
    restart: unless-stopped
    environment:
      - MARIADB_ROOT_PASSWORD=<MAT_KHAU_ROOT_DB>
      - MARIADB_DATABASE=romm
      - MARIADB_USER=romm-user
      - MARIADB_PASSWORD=<MAT_KHAU_DB>          # trùng DB_PASSWD ở trên
    volumes:
      - romm_mariadb:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      start_period: 30s
      start_interval: 10s
      interval: 10s
      timeout: 5s
      retries: 5
```

**Vì sao cấu hình như trên:**
- Chỉ `library` dùng **bind mount** `/volume1/roms` để bạn thả ROM qua File Station.
- `resources / assets / config / redis / db` dùng **named volume** → Container Manager tự lo quyền,
  tránh lỗi phân quyền hay gặp trên Synology (container chạy UID 1000). Vẫn sao lưu được bằng Hyper Backup
  (backup thư mục `/volume1/docker`).
- Cổng host **8998** để tránh đụng cổng DSM (5000/5001) và 80/443.

---

## 4. Tạo Project trong Container Manager

**B5** — Mở **Container Manager** → tab **Project** → **Create**.

**B6** — Điền:
- **Project name:** `romm`
- **Path:** chọn/tạo `/volume1/docker/romm`
- **Source:** chọn **Create docker-compose.yml** → dán nội dung ở mục 3 (đã điền `<...>`).

**B7** — **Next** → bỏ qua bước Web Portal (Set up web portal) nếu được hỏi → **Done**.
Container Manager sẽ tải image và khởi động. Lần đầu tải image mất vài phút.

**B8** — Xem log: vào Project `romm` → tab **Log**, đợi tới khi `romm-db` healthy và `romm` chạy ổn định.

---

## 5. Cấu hình lần đầu

**B9** — Mở trình duyệt: `http://<IP-NAS>:8998`

**B10** — **Setup Wizard** hiện ra → tạo **tài khoản admin** (đặt mật khẩu mạnh).

**B11** — Vào **Settings** để kiểm tra/nhập API metadata (nếu chưa nhập ở compose):
- ScreenScraper (tài khoản miễn phí) hoặc IGDB (cần Twitch Client ID/Secret) cho ảnh bìa + thông tin.
- Hướng dẫn lấy key: <https://docs.romm.app/latest/Getting-Started/Metadata-Providers/>

**B12** — **Scan** thư viện: vào trang chính → nút **Scan** → chờ RomM quét `/volume1/roms` và khớp metadata.

---

## 6. Truy cập từ xa (an toàn)

**Không** mở thẳng (port-forward) cổng 8998 ra internet. Chọn 1 trong các cách:

| Cách | Ưu điểm | Ghi chú |
|---|---|---|
| **Tailscale** (khuyến nghị) | Cài package trên NAS, không mở cổng | Truy cập như trong mạng LAN từ mọi nơi |
| **Cloudflare Tunnel** | Có tên miền + HTTPS, không mở cổng | Chạy `cloudflared` dạng container |
| **Reverse Proxy của DSM + DDNS + HTTPS** | Dùng tên miền + chứng chỉ Let's Encrypt | Control Panel → Login Portal → Advanced → Reverse Proxy; bật Firewall |

Nếu dùng reverse proxy/HTTPS, thêm biến môi trường cho khớp tên miền, ví dụ `ROMM_BASE_URL` /
cấu hình theo docs: <https://docs.romm.app/latest/>

---

## 7. Kiểm thử (bạn tự làm trên UI — Tier C)

| ID | Bước | KQ mong đợi ✅ | FAIL ❌ |
|---|---|---|---|
| T1 | Container Manager → Project `romm` → xem **Log** | `romm-db` healthy, `romm` không báo lỗi khởi động | |
| T2 | Mở `http://<IP-NAS>:8998` | Hiện Setup Wizard hoặc trang đăng nhập | |
| T3 | Tạo tài khoản admin ở Setup Wizard → đăng nhập | Vào được giao diện chính | |
| T4 | Bỏ vài ROM vào `/volume1/roms/roms/<hệ máy>/` → bấm **Scan** | Game hiện ra trong thư viện | |
| T5 | Mở 1 game → bấm **Play** (hệ máy hỗ trợ EmulatorJS) | Game chạy trong trình duyệt | |
| T6 | (Nếu nhập API) Kiểm tra 1 game bất kỳ | Có ảnh bìa + mô tả | |

---

## 8. Xử lý sự cố thường gặp

| Triệu chứng | Nguyên nhân & cách xử lý |
|---|---|
| Scan không thấy game | Sai **slug** thư mục hệ máy, hoặc sai cấu trúc `roms/<hệ máy>/`. Đối chiếu docs Folder-Structure & Supported-Platforms. |
| Lỗi quyền đọc thư viện | RomM chạy UID 1000. Cấp quyền **đọc** cho thư mục `roms`: Control Panel → Shared Folder → `roms` → Edit → Permissions (cấp read cho user tương ứng hoặc `everyone`). |
| Container `romm` chờ mãi | `romm-db` chưa healthy → xem log DB; kiểm tra `DB_PASSWD` có **trùng** `MARIADB_PASSWORD` không. |
| Bị đăng xuất sau mỗi lần restart | Chưa đặt `ROMM_AUTH_SECRET_KEY` cố định (mục B4). |
| Cổng 8998 bị chiếm | Đổi cổng host, ví dụ `"8999:8080"`. |

---

## 9. Cập nhật & sao lưu

- **Cập nhật:** Container Manager → Project `romm` → **Action → Build/Pull** (kéo image mới) rồi khởi động lại.
  Nên tạo bản backup DB trước khi cập nhật lớn.
- **Sao lưu:** Hyper Backup thư mục `/volume1/docker` (chứa named volume: db, resources, assets, config)
  và thư mục `/volume1/roms` (thư viện ROM). Save/state nằm trong volume `romm_assets`.

---

_Tài liệu chính thức: <https://docs.romm.app/latest/> • Repo: rommapp/romm_
