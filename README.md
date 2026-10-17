# AI Interview Platform — Deploy (staging)

Repo điều phối triển khai staging + kho tài liệu đặc tả. Không chứa code ứng dụng.

## Luồng CI/CD

```
feature branch → PR → merge dev
        │
        ├── AI-Interview-Platform-FE ── Frontend CI (npm ci + build)
        │                                     │ workflow_run (chỉ khi xanh)
        │                                     ▼
        │                            repo nầy: Staging Deploy
        │                                     │ vercel build --prod + deploy
        │                                     ▼
        │                        https://aiinterview-vn.vercel.app
        │
        └── AI-Interview-Platform-BE ── Staging CI (mvn -B clean test)
                                              │ check "Backend tests" xanh
                                              ▼
                                    Render (autoDeployTrigger: checksPass)
                                              ▼
                        https://ai-interview-backend-staging.onrender.com
```

Backend không đi qua repo nầy vì Render build trực tiếp từ `render.yaml` trong
repo BE; repo nầy chỉ nhận kết quả CI để làm bảng theo dõi chung
(context `backend / staging`).

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

## Secrets cần có trong repo nầy

| Tên | Lấy ở đâu |
|---|---|
| `VERCEL_TOKEN` | vercel.com → Account → Settings → Tokens |
| `VERCEL_ORG_ID` | `team_...`, xem `vercel project inspect` hoặc repo cũ |
| `VERCEL_PROJECT_ID` | `prj_...`, hoặc `AI-Interview-Platform-FE/.vercel/project.json` |

## Runbook

Deploy frontend thủ công theo một SHA cụ thể (dùng khi rollback):
Actions → **Staging Deploy** → Run workflow → điền `frontend_sha`.

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
