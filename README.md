# Session 12 - Bài 3: Tách luồng CD đa môi trường (Staging & Production) với GitHub Environments

- Repository: <https://github.com/thang-260704/IT209_Ss12_Ex3>
- Workflow: [`.github/workflows/cd-multi-env.yml`](.github/workflows/cd-multi-env.yml)

## 1. Mục tiêu

| Nhánh | Job | Environment | Bảo vệ |
|---|---|---|---|
| `develop` | `deploy-staging` | `staging` | Không, tự động deploy |
| `main` | `deploy-prod` | `production` | **Required reviewers**: chờ `thang-260704` phê duyệt |

```text
push develop ─► deploy-staging ──────────────────────────► ✅ staging
push main    ─► deploy-prod ─► ⏸ Waiting for review ─► Approve and deploy ─► ✅ production
```

Đây là mô hình **Continuous Delivery**: code trên `main` luôn sẵn sàng phát hành, nhưng việc đưa lên production cần một người xác nhận.

## 2. Cấu hình GitHub Environments

Settings → Environments → **New environment**:

1. `staging`: không đặt quy tắc bảo vệ.
2. `production`: bật **Required reviewers**, thêm tài khoản `thang-260704` → **Save protection rules**.

Environment `production` sau khi lưu: Required reviewers = `thang-260704`, **không** bật *Prevent self-review* (để chủ repo tự duyệt được trong bài tập) và **bỏ** *Allow administrators to bypass configured protection rules* (để admin cũng phải chờ duyệt, không bỏ qua được).

![Environment production](images/01-env-production.png)

## 3. Workflow `cd-multi-env.yml`

```yaml
name: CD Multi Environment

on:
  push:
    branches: [ "develop", "main" ]

permissions:
  contents: read

jobs:
  deploy-staging:
    name: Deploy to Staging
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://api.staging.example.com
    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Deploy lên Staging (mô phỏng)
        run: |
          echo "Môi trường : ${{ github.job }} -> staging"
          echo "Nhánh      : ${{ github.ref_name }}"
          echo "Commit     : ${{ github.sha }}"
          echo "Image      : ghcr.io/thang-260704/it209_ss12_ex2:latest"
          echo "Deploy staging thành công, không cần phê duyệt."

  deploy-prod:
    name: Deploy to Production
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.prod.example.com
    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Deploy lên Production (mô phỏng)
        run: |
          echo "Môi trường : ${{ github.job }} -> production"
          echo "Nhánh      : ${{ github.ref_name }}"
          echo "Commit     : ${{ github.sha }}"
          echo "Người kích hoạt: ${{ github.actor }}"
          echo "Image      : ghcr.io/thang-260704/it209_ss12_ex2:v1.0.0"
          echo "Đã được reviewer phê duyệt, deploy production thành công."
```

| Thành phần | Ý nghĩa |
|---|---|
| `on.push.branches: [develop, main]` | Một workflow phục vụ cả hai nhánh |
| `if: github.ref == 'refs/heads/develop'` | Job staging chỉ chạy khi push `develop`; khi push `main` job này hiện trạng thái **skipped** |
| `if: github.ref == 'refs/heads/main'` | Job production chỉ chạy khi push `main` |
| `environment.name` | Gắn job với Environment trên GitHub; mọi protection rule và secret của environment đó áp dụng cho job |
| `environment.url` | Link hiển thị trên trang run và tab Deployments sau khi deploy xong |
| Required reviewers | Runner **chưa được cấp** cho job `deploy-prod` cho tới khi reviewer bấm Approve; không có bước nào chạy trước khi duyệt |

Bước deploy là **mô phỏng** (in thông tin phiên bản) vì bài tập tập trung vào cơ chế environment và phê duyệt; ở dự án thật bước này thay bằng SSH vào server rồi `docker pull` + `docker run` image từ GHCR (bài 2).

## 4. Kết quả

| Run | Nhánh | Commit | Deploy to Staging | Deploy to Production |
|---|---|---|---|---|
| #1 | `develop` | `d8caaff` | ✅ 5s, không cần duyệt | ⊘ skipped |
| #2 | `main` | `d8caaff` | ⊘ skipped | ✅ 4s, sau khi `thang-260704` approve |
| #3 | `main` | `332e9a3` | ⊘ skipped | ⏸ Waiting → Approve → ✅ |

### 4.1 Push `develop`: staging tự deploy

Job `Deploy to Staging` chạy ngay, hiển thị URL `https://api.staging.example.com`; `Deploy to Production` bị bỏ qua vì điều kiện `if`.

![Staging](images/02-staging-run.png)

### 4.2 Push `main`: production dừng chờ phê duyệt

Run ở trạng thái **Waiting**, ô `Deploy to Production` hiện *"production waiting for review"*; banner vàng *"thang-260704 requested your review to deploy to production"* kèm nút **Review deployments**. Lúc này job chưa được cấp runner, không có step nào chạy.

![Waiting for review](images/03-waiting-for-review.png)

### 4.3 Cửa sổ phê duyệt deployment Production

Hộp thoại **Review pending deployments**: tích môi trường `production` (*Review needed from thang-260704*), ô comment, hai nút **Reject** và **Approve and deploy**.

![Approval dialog](images/04-approval-dialog.png)

### 4.4 Sau khi Approve and deploy

Mục **Deployment protection rules** ghi nhận *thang-260704 approved – production*; job `Deploy to Production` ✅ và hiển thị URL `https://api.prod.example.com`.

![Production approved](images/05-prod-approved.png)

Log step deploy:

```text
Môi trường : deploy-prod -> production
Nhánh      : main
Commit     : d8caaffd3edeedf14cdd658e9ccdfe1c9316bff0
Người kích hoạt: thang-260704
Image      : ghcr.io/thang-260704/it209_ss12_ex2:v1.0.0
Đã được reviewer phê duyệt, deploy production thành công.
```

![Production log](images/06-prod-log.png)

## 5. Kết luận

- Hai GitHub Environment `staging` và `production` tách biệt luồng triển khai theo nhánh: `develop` → staging, `main` → production.
- Staging deploy tự động ngay khi push; production bị **chặn** ở trạng thái *Waiting for review* cho tới khi reviewer bấm **Approve and deploy**.
- Lịch sử phê duyệt (ai duyệt, môi trường nào, lúc nào) được GitHub lưu lại trên trang run, phục vụ truy vết.
- Đây là mô hình Continuous Delivery: mọi thay đổi trên `main` đều sẵn sàng phát hành, nhưng quyết định đưa lên production vẫn do con người kiểm soát. Muốn chuyển sang Continuous Deployment chỉ cần bỏ Required reviewers.
