# CI/CD Pipeline Demo với GitHub Actions, Docker, TruffleHog và Trivy

## Giới thiệu

Project này minh họa một **CI/CD pipeline hoàn chỉnh theo mô hình DevSecOps** sử dụng:

* **GitHub Actions** để chạy pipeline
* **TruffleHog** để phát hiện secrets trong source code và lịch sử Git
* **Trivy** để scan dependency và container image
* **Docker** để build container
* **Telegram Bot** để gửi thông báo pipeline
* **Self-hosted Runner** để chạy pipeline trên server riêng

Pipeline sẽ chạy tự động khi:

```bash
git push origin main
```

Workflow nằm trong:

```bash
.github/workflows/cicd.yml
```

---

# Kiến trúc Pipeline

Pipeline được thiết kế theo luồng:

```text
Security Scan (TruffleHog)
        │
        ▼
Dependency Scan (Trivy) ── Build & Test
        │
        ▼
Docker Build & Push
        │
        ▼
Container Image Scan (Trivy)
        │
        ▼
Deploy Application
        │
        ▼
Success Notification (Telegram)
```

Chi tiết:

| Stage | Công cụ | Mục đích |
| --- | --- | --- |
| Security Scan | TruffleHog | Detect hardcoded secrets trong source code và Git history |
| Dependency Scan | Trivy | Scan thư viện, package và CVE mức HIGH/CRITICAL |
| Build & Test | Node.js | Cài dependency, test ứng dụng |
| Docker Build & Push | Docker | Build image và push lên GitHub Container Registry |
| Container Scan | Trivy | Scan lỗ hổng image Docker |
| Deploy | Docker | Chạy hoặc cập nhật container trên máy deploy |
| Notification | Telegram | Thông báo trạng thái pipeline |

---

# 1. Tạo Telegram Bot

Pipeline sẽ gửi thông báo qua Telegram.

## Bước 1: tạo bot

Mở Telegram và chat với:

```text
@BotFather
```

Gửi lệnh:

```text
/newbot
```

Sau đó BotFather sẽ yêu cầu:

```text
Bot name
Bot username
```

Sau khi tạo xong bạn sẽ nhận được:

```text
Bot Token
```

Ví dụ:

```text
7981513078:AAFxxxxxxxxxxxxxxxx
```

Lưu token này lại.

---

# 2. Lấy Chat ID

Gửi một tin nhắn bất kỳ cho bot.

Sau đó mở:

```text
https://api.telegram.org/bot<BOT_TOKEN>/getUpdates
```

Ví dụ:

```text
https://api.telegram.org/bot7981513078:AAFxxx/getUpdates
```

Response sẽ chứa:

```json
"chat": {
  "id": 123456789,
  "type": "private"
}
```

Giá trị:

```text
123456789
```

chính là **Chat ID**.

---

# 3. Tạo GitHub Repository

Tạo repo mới trên GitHub.

Clone repo về máy:

```bash
git clone https://github.com/your-username/cicd-demo.git
cd cicd-demo
```

Khởi tạo project:

```bash
git init
git add .
git commit -m "initial commit"
git branch -M main
git push origin main
```

---

# 4. Set GitHub Secrets

Pipeline cần Telegram credentials.

Vào:

```text
Repository → Settings → Secrets and variables → Actions
```

Tạo 2 secrets:

```text
TELEGRAM_BOT_TOKEN
TELEGRAM_CHAT_ID
```

Ví dụ:

```text
TELEGRAM_BOT_TOKEN = 7981513078:AAFxxxx
TELEGRAM_CHAT_ID = 123456789
```

Ngoài ra, pipeline sử dụng sẵn `GITHUB_TOKEN` do GitHub Actions cấp để đăng nhập GitHub Container Registry.

---

# 5. Tạo Self-Hosted Runner

Self-hosted runner giúp chạy các job Docker build, image scan và deploy trên server của bạn thay vì hoàn toàn dùng runner của GitHub.

Vào:

```text
Repository
Settings
Actions
Runners
New self-hosted runner
```

Chọn hệ điều hành:

```text
Linux / MacOS / Windows
```

Ví dụ với Linux:

Download runner:

```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64.tar.gz
tar xzf actions-runner.tar.gz
```

Config runner:

```bash
./config.sh --url https://github.com/your-username/cicd-demo --token YOUR_TOKEN
```

Start runner:

```bash
./run.sh
```

Runner sẽ online trong GitHub.

---

# 6. Cấu trúc Project

```text
cicd-demo
│
├── .github
│   └── workflows
│       └── cicd.yml
│
├── app.js
├── package.json
├── Dockerfile
└── README.md
```

---

# 7. Workflow GitHub Actions

File:

```bash
.github/workflows/cicd.yml
```

Luồng thực tế của workflow hiện tại:

1. **Security Scan (TruffleHog)**
   - Checkout toàn bộ Git history với `fetch-depth: 0`
   - Chạy `trufflesecurity/trufflehog:latest`
   - Xuất báo cáo JSON ra `reports/trufflehog-report.json`
   - Upload artifact để phục vụ kiểm tra sau này
   - Nếu phát hiện secret thì fail pipeline và gửi cảnh báo Telegram

2. **Dependency Scan (Trivy)**
   - Dùng Trivy scan filesystem/project source
   - Lưu báo cáo vào `reports/trivy-deps.json`
   - Fail nếu có lỗ hổng mức `CRITICAL` hoặc `HIGH`

3. **Build & Test**
   - Setup Node.js 18
   - Chạy `npm ci`
   - Chạy `npm test`

4. **Docker Build & Push**
   - Chạy trên `self-hosted runner`
   - Login vào `ghcr.io`
   - Build image với 2 tag:
     - `latest`
     - `${{ github.sha }}`
   - Push image lên GitHub Container Registry

5. **Container Image Scan (Trivy)**
   - Pull image vừa build
   - Dùng Trivy scan image Docker
   - Upload báo cáo `trivy-image.json`
   - Fail nếu có lỗ hổng `CRITICAL` hoặc `HIGH`

6. **Deploy Application**
   - Pull image mới nhất từ GHCR
   - Stop và remove container cũ
   - Run container mới với cổng `3000:3000`

7. **Success Notification**
   - Nếu toàn bộ pipeline pass thì gửi thông báo thành công qua Telegram

---

# 8. TruffleHog dùng để làm gì?

TruffleHog là công cụ dùng để phát hiện các thông tin nhạy cảm bị hardcode trong source code hoặc còn sót lại trong lịch sử Git, ví dụ:

* API Key
* Access Token
* Password
* Private Key
* Credential nội bộ

Điểm mạnh của TruffleHog trong pipeline này:

* Có thể quét cả **working tree** lẫn **Git history**
* Phù hợp để phát hiện secret đã từng commit rồi xóa
* Có thể xuất báo cáo JSON để lưu artifact
* Có thể chặn pipeline ngay từ đầu nếu phát hiện rủi ro

---

# 9. Trigger Pipeline

Sau khi setup xong, chỉ cần:

```bash
git add .
git commit -m "update"
git push origin main
```

GitHub Actions sẽ tự động chạy pipeline.

---

# 10. Kết quả Pipeline

Pipeline sẽ:

1. Scan secrets bằng **TruffleHog**
2. Scan dependencies bằng **Trivy**
3. Build và test ứng dụng
4. Build Docker image và push lên **GHCR**
5. Scan container image bằng **Trivy**
6. Deploy container
7. Gửi notification Telegram

Nếu pipeline thành công bạn sẽ nhận được thông báo thành công trên Telegram.

---

# 11. Demo Scenario

## Success Case

```text
git push
↓
Security Scan PASS (TruffleHog)
↓
Dependency Scan PASS (Trivy)
↓
Build & Test PASS
↓
Docker Build & Push PASS
↓
Image Scan PASS
↓
Deploy PASS
↓
Telegram Success Message
```

## Failure Case

Ví dụ trong source code có:

```text
AWS_SECRET_ACCESS_KEY=xxxx
```

TruffleHog có thể detect và pipeline sẽ:

```text
Fail
↓
Upload báo cáo scan
↓
Gửi Telegram alert
```

---

# 12. Best Practices

Một số best practices được áp dụng:

* Scan secrets trước khi build
* Dùng **TruffleHog** để quét secret trong source code và Git history
* Scan dependencies để phát hiện CVE
* Scan container image trước khi deploy
* Fail pipeline nếu có security issue nghiêm trọng
* Tách rõ CI phase và CD phase
* Dùng Telegram để alert realtime
* Dùng self-hosted runner cho build, push image và deploy

---

# 13. Gợi ý cải thiện thêm

Bạn có thể nâng cấp pipeline sau bằng cách:

* Tách riêng môi trường `dev`, `staging`, `production`
* Gắn thêm badge trạng thái workflow vào README
* Bổ sung `npm audit` hoặc SAST tool nếu muốn mở rộng DevSecOps
* Chỉ deploy khi image scan pass hoàn toàn
* Bổ sung rollback strategy khi deploy lỗi
