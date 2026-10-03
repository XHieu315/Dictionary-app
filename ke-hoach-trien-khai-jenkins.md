# BẢN KẾ HOẠCH TRIỂN KHAI CI/CD VỚI JENKINS & GHCR (DOCKER LOCAL)

> **Mục tiêu**: Xây dựng hệ thống CI/CD khép kín trên máy tính cá nhân (Windows / Docker Desktop):  
> `Commit Code` → `Jenkins tự động kích hoạt` → `Build & Test` → `Đẩy image lên GHCR (GitHub Container Registry)` → `Pull ngược image về Docker local` → `Deploy ứng dụng production ở cổng 3000`.

---

## 1. Kiến Trúc Hoạt Động (Mô hình DooD - Docker outside of Docker)

Jenkins chạy dưới dạng một container, nhưng **không chạy Docker daemon riêng bên trong**. Thay vào đó, nó mount trực tiếp socket `/var/run/docker.sock` của máy host (Docker Desktop).

```
 ┌────────────────────── MÁY HOST (WINDOWS / LAPTOP) ─────────────────────┐
 │                                                                         │
 │   ┌─── Container: jenkins ────┐                                         │
 │   │  Jenkins Core + Docker CLI│                                         │
 │   │                           │                                         │
 │   │  docker build ────────────┼──┐                                      │
 │   │  docker compose up ───────┼──┤                                      │
 │   └───────────────────────────┘  │                                      │
 │                 │                │ qua /var/run/docker.sock             │
 │                 │                ▼                                      │
 │                 │      ┌─── Docker Desktop Daemon ───┐                  │
 │                 │      │                             │                  │
 │                 │      │  dictionary-prod-web ───────┼──► Cổng :3000    │
 │                 │      │  dictionary-prod-db  ───────┼──► Cổng :5432    │
 │                 │      └─────────────────────────────┘                  │
 │                 │                     ▲                                 │
 │                 └─────────────────────┘                                 │
 │              http://host.docker.internal:3000 (Stage 6 Verify)          │
 └─────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼  Push / Pull
                     ghcr.io/<github-user>/<repo-name>
```

### Điểm mấu chốt của quy trình:
1. **Build 1 lần & Test bản native**: Build bản native cho kiến trúc máy hiện tại, chạy bộ 8 smoke test.
2. **Multi-arch Image**: Đẩy lên GHCR hỗ trợ cả `linux/amd64` và `linux/arm64`.
3. **Xoá local rồi mới PULL**: Ở Stage 4, Jenkins cố tình xoá image local (`docker rmi`) trước khi `docker pull`. Điều này chứng minh rằng image trên GitHub Container Registry thực sự tải về và hoạt động được, tránh rủi ro "chỉ chạy được trên máy người build".
4. **Deploy production từ image đã pull**: Khởi chạy stack độc lập bằng file `docker-compose.prod.yml` không build lại mã nguồn.

---

## 2. Checklist Chuẩn Bị

- [ ] Docker Desktop đang chạy bình thường trên Windows.
- [ ] Tài khoản GitHub có quyền truy cập repo `dictionary-app-5`.
- [ ] Đã tạo GitHub Personal Access Token (Classic) có quyền `write:packages`.
- [ ] Tên tài khoản GitHub trong `Jenkinsfile` đã được đổi sang chữ thường (lowercase).
- [ ] File script `scripts/smoke-test.sh` giữ định dạng dòng `LF` (tránh lỗi Windows `CRLF`).

---

## 3. Các Bước Thực Hiện Chi Tiết

### Bước 1: Tạo GitHub Personal Access Token (PAT)
1. Truy cập: **GitHub** → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**.
2. Chọn **Generate new token (classic)**.
3. Điền thông tin:
   - **Note**: `jenkins-ghcr`
   - **Expiration**: 90 days (hoặc tuỳ chọn)
   - **Scopes**: Tích chọn 2 quyền bắt buộc:
     - `write:packages` (hệ thống tự động chọn kèm `read:packages`)
4. Nhấn **Generate token** và **sao chép ngay mã token** (token này chỉ hiển thị 1 lần).

---

### Bước 2: Cập Nhật Cấu Hình trong `Jenkinsfile`
Mở file [Jenkinsfile](file:///d:/DevOps-test/dictionary-app-5/Jenkinsfile), tìm dòng:
```groovy
IMAGE_NAME = 'yourgithubusername/test-devops'
```
Đổi thành tài khoản GitHub và tên repository của bạn (bắt buộc **chữ thường toàn bộ**):
```groovy
IMAGE_NAME = 'dinhluc/dictionary-app-5'  // Ví dụ: thay bằng username/repo thật của bạn
```

> **Lưu ý định dạng dòng (Line Endings) trên Windows**:  
> Đảm bảo file [scripts/smoke-test.sh](file:///d:/DevOps-test/dictionary-app-5/scripts/smoke-test.sh) có định dạng dòng là **LF** (trong VS Code / IDE nhìn góc dưới bên phải chuyển `CRLF` sang `LF`).

Commit và Push các thay đổi lên GitHub:
```powershell
git add Jenkinsfile
git commit -m "ci: update github username for ghcr"
git push origin main
```

---

### Bước 3: Khởi Động Container Jenkins Trên Docker Local
1. Mở PowerShell tại thư mục dự án và chuyển vào thư mục `jenkins`:
   ```powershell
   cd d:\DevOps-test\dictionary-app-5\jenkins
   docker compose up -d --build
   ```
   *(Lần đầu tiên sẽ mất khoảng 2–4 phút để tải image Jenkins, cài Docker CLI, Docker Compose và các plugin CI/CD).*

2. **Kiểm tra Jenkins đã điều khiển được Docker Host**:
   ```powershell
   docker exec jenkins docker version --format '{{.Server.Version}} / {{.Server.Arch}}'
   ```
   *Kết quả hiển thị phiên bản Docker (ví dụ: `27.x.x / amd64` hoặc `arm64`) là thành công.*

3. Truy cập vào giao diện Web của Jenkins:
   - Địa chỉ: [http://localhost:8080](http://localhost:8080)
   - Do đã cấu hình `-Djenkins.install.runSetupWizard=false`, bạn sẽ vào thẳng trang chủ Jenkins mà không cần cấu hình tài khoản ban đầu.

---

### Bước 4: Tạo Credential Cho GHCR Trên Jenkins
1. Trên giao diện Jenkins, vào menu:  
   **Manage Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)** → chọn **Add Credentials**.
2. Điền chính xác các mục sau:
   - **Kind**: `Username with password`
   - **Username**: Tên tài khoản GitHub của bạn (viết thường)
   - **Password**: Dán mã **Personal Access Token** vừa tạo ở Bước 1
   - **ID**: `ghcr-credentials` *(Bắt buộc phải khớp đúng chuỗi này với Jenkinsfile)*
   - **Description**: `GitHub Container Registry Token`
3. Nhấn **Create**.

---

### Bước 5: Tạo Pipeline Job Trên Jenkins
1. Trở về Dashboard Jenkins → chọn **New Item**.
2. Đặt tên: `dictionary-cicd` → chọn kiểu **Pipeline** → nhấn **OK**.
3. Cuộn xuống mục **Build Triggers**:
   - Tích chọn **Poll SCM** (Jenkins sẽ kiểm tra commit mới từ GitHub mỗi 2 phút).
4. Cuộn xuống mục **Pipeline**:
   - **Definition**: Chọn `Pipeline script from SCM`
   - **SCM**: Chọn `Git`
   - **Repository URL**: `https://github.com/<tên_github>/dictionary-app-5.git`
   - **Credentials**: Nếu repo công khai (Public) thì để trống `none`. Nếu repo riêng tư (Private), tạo thêm 1 credential Git riêng.
   - **Branch Specifier**: `*/main` (hoặc `*/master` tuỳ nhánh mặc định của bạn)
   - **Script Path**: `Jenkinsfile`
5. Nhấn **Save**.

---

### Bước 6: Chạy Pipeline & Theo Dõi Tiến Trình
1. Nhấn nút **Build Now** trên thanh menu trái của Job.
2. Bấm vào lượt build vừa tạo (ví dụ `#1`) → chọn **Console Output** để xem log chi tiết qua 6 stages:
   - **Chuẩn bị**: Login GHCR bằng token an toàn (`--password-stdin`), khởi tạo Buildx Multi-arch.
   - **1. BUILD**: Build bản native, tận dụng buildcache từ GHCR.
   - **2. TEST**: Chạy `scripts/smoke-test.sh` kiểm thử 8 kịch bản API với Postgres.
   - **3. PUSH**: Build đa kiến trúc (amd64 + arm64) và đẩy lên GHCR (`:sha` và `:latest`).
   - **4. PULL**: Xoá local cache và kéo image trực tiếp từ GHCR về Docker local.
   - **5. DEPLOY**: Chạy `docker compose up -d` với file [docker-compose.prod.yml](file:///d:/DevOps-test/dictionary-app-5/docker-compose.prod.yml).
   - **6. Kiểm tra sau deploy**: Ping kiểm tra ứng dụng qua `host.docker.internal:3000`.

---

### Bước 7: Nghiệm Thu & Kiểm Chứng Kết Quả

1. **Kiểm tra ứng dụng chạy thật**:
   - Mở trình duyệt truy cập: [http://localhost:3000](http://localhost:3000)
   - Kiểm tra API trực tiếp: [http://localhost:3000/api/health](http://localhost:3000/api/health)
   - Kết quả mong đợi: `{"db":"connected","words":10}`
2. **Kiểm tra package trên GitHub**:
   - Vào trang GitHub cá nhân → tab **Packages** → thấy package `dictionary-app-5` chứa cả 2 kiến trúc `linux/amd64` và `linux/arm64`.
3. **Kiểm tra container đang chạy trên máy**:
   ```powershell
   docker ps --filter "name=dictionary-prod"
   ```

---

## 4. Các Thử Nghiệm Kiểm Chứng Nâng Cao

### 4.1. Thử nghiệm CI chặn code lỗi (Bảo vệ Production)
1. Mở file [db/init.sql](file:///d:/DevOps-test/dictionary-app-5/db/init.sql), xoá bớt 1 dòng `INSERT`.
2. Commit và push lên GitHub.
3. Jenkins tự động kích hoạt hoặc bấm **Build Now**.
4. Pipeline sẽ dừng ngay tại **Stage 2 (TEST)** vì không đủ 10 từ.
5. Truy cập [http://localhost:3000](http://localhost:3000): Ứng dụng production **vẫn hoạt động nguyên vẹn** bản cũ.
6. Khôi phục lại:
   ```powershell
   git revert HEAD
   git push origin main
   ```

### 4.2. Thử nghiệm Rollback nhanh về bản cũ
Khi cần quay về phiên bản trước mà không cần build lại:
```powershell
cd d:\DevOps-test\dictionary-app-5
$IMAGE_TAG="ghcr.io/<tên_github>/dictionary-app-5:<sha_cũ>"
docker compose -p dictionary-prod -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 5. Bảng Khắc Phục Lỗi Thường Gặp Trên Windows

| Hiện tượng | Nguyên nhân | Cách khắc phục |
| :--- | :--- | :--- |
| `Cannot connect to the Docker daemon` | Chưa mount socket hoặc Docker Desktop chưa chạy | Đảm bảo Docker Desktop đang chạy. Kiểm tra dòng volume `- /var/run/docker.sock:/var/run/docker.sock` trong `jenkins/docker-compose.yml`. |
| `scripts/smoke-test.sh: \r: command not found` | File script bị dính định dạng dòng Windows (CRLF) | Chuyển định dạng file `scripts/smoke-test.sh` sang **LF** bằng VS Code / Notepad++ rồi commit lại. |
| `denied` khi push lên GHCR | Token thiếu quyền hoặc sai tên tài khoản | Đảm bảo Token có quyền `write:packages`. Tên GitHub trong `Jenkinsfile` và credential phải **viết thường**. |
| Stage 6 Timeout không kết nối được | Jenkins không phân giải được host | Trên Docker Desktop Windows, DNS `host.docker.internal` được hỗ trợ sẵn. Đảm bảo cổng 3000 không bị firewall chặn. |
| Cổng 8080 hoặc 3000 bị xung đột | Tiến trình khác đang chiếm cổng | Đổi cổng host trong `jenkins/docker-compose.yml` (ví dụ `"8090:8080"`), hoặc tắt service đang chiếm cổng. |

---

## 6. Lệnh Dọn Dẹp Môi Trường Sau Khi Hoàn Thành

Khi muốn tắt ứng dụng và Jenkins:
```powershell
# 1. Dừng stack ứng dụng production
cd d:\DevOps-test\dictionary-app-5
docker compose -p dictionary-prod -f docker-compose.yml -f docker-compose.prod.yml down

# 2. Dừng Jenkins (vẫn giữ dữ liệu cấu hình)
cd jenkins
docker compose down

# 3. Dừng Jenkins và xoá toàn bộ dữ liệu cấu hình (nếu muốn làm lại từ đầu)
docker compose down -v
```
