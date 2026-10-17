# AI Interview Platform — Deploy (staging)

Repo điều phối triển khai staging + kho tài liệu đặc tả. Không chứa code ứng dụng.

## Luồng CI/CD

```
feature branch → PR → merge dev
        │
        ├── AI-Interview-Platform-FE ── Frontend CI (npm ci + build)
        │                                     │ build xanh tren dev
        │                                     │ → workflow_dispatch (DEPLOY_PAT)
        │                                     ▼
        │                            repo nầy: Staging Deploy
        │                                     │ vercel pull + build --prod + deploy
        │                                     ▼
        │                        https://aiinterview-vn.vercel.app
        │
        └── AI-Interview-Platform-BE ── Staging CI (mvn -B clean test)
                                              │ check "Backend tests" xanh
                                              ├─▶ Render (checksPass) build & deploy
                                              │        https://ai-interview-backend-staging.onrender.com
                                              └─▶ workflow_dispatch (DEPLOY_PAT) → repo nầy
                                                    ghi status "backend / staging" + probe Render
```

`workflow_run` không dùng được vì nó **chỉ phát trong cùng một repository**
(đo ngày 2026-10-17: run `Frontend CI` #2 kết thúc 16:55:16, repo deploy
không tạo run nào). Nên hai repo FE/BE chủ động gọi `workflow_dispatch` sang
repo nầy bằng PAT.

## Chạy local

Clone ba repo nằm cạnh nhau:

```
dev/
├── AI-Interview-Platform-deploy-staging/
├── AI-Interview-Platform-BE/
└── AI-Interview-Platform-FE/
```

```bash
cp .env.example .env
docker compose up -d --build          # db + backend + frontend (3000)
docker compose --profile sso up -d    # thêm Keycloak ở 8180
```

## Secrets

Trong repo nầy:

| Tên | Lấy ở đâu |
|---|---|
| `VERCEL_TOKEN` | vercel.com → Account → Settings → Tokens |
| `VERCEL_ORG_ID` | `team_bgWCYiMMgj6WD2svhdjU5Kpg` |
| `VERCEL_PROJECT_ID` | `prj_Y3gCKVlIjThmzhqziNQcQzTrdGJm` (xem `AI-Interview-Platform-FE/.vercel/project.json`) |

Trong repo FE **và** repo BE:

| Tên | Lấy ở đâu |
|---|---|
| `DEPLOY_PAT` | GitHub → Settings → Developer settings → Fine-grained tokens, chỉ cấp repo nầy với quyền **Actions: read and write** |

## Runbook

Deploy frontend thủ công theo một SHA cụ thể (dùng khi rollback):
Actions → **Staging Deploy** → Run workflow → điền `frontend_sha` để nguyên
nếu muốn deploy đúng đầu branch `dev`.

Ghi lại kết quả backend: Run workflow → điền `backend_sha`.

Kiểm tra nhanh sau deploy:

```bash
curl -sI https://aiinterview-vn.vercel.app | head -1
curl -s  https://ai-interview-backend-staging.onrender.com/api/v1/packages -o /dev/null -w '%{http_code}\n'
```

Lưu ý: Render free tier ngủ khi không có traffic. Nếu merge backend mà không
thấy deploy, kiểm tra check `Backend tests` trên commit `dev` của repo BE —
Render bỏ qua deploy khi commit đó không có check nào.

## Tài liệu

`spec/` là nguồn kế hoạch của đồ án: đọc `spec/README.md`, `spec/INDEX.md`,
`spec/conventions/frontend-ui.md`, `spec/known-gaps.md`, `spec/srs-core-baseline.md`
trước khi code.
