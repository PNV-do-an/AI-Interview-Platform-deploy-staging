# AI Interview Platform — Deploy (superrepo)

Superrepo của môi trường staging: `frontend/` và `backend/` là **git submodule**
trỏ về hai repo tách riêng, cộng thêm `docker-compose.yml`, `keycloak/` và toàn
bộ đặc tả trong `spec/`. Đây là **nơi duy nhất chạy bước deploy**.

## Luồng CI/CD

```
feature branch → PR → merge dev
        │
        ├── AI-Interview-Platform-FE ── Frontend CI: npm ci + build
        │                                      │ build xanh tren dev
        │                                      └─ bump submodule `frontend` → repo nầy
        │
        └── AI-Interview-Platform-BE ── Staging CI: mvn -B clean test
                                             │ test xanh tren dev
                                             └─ bump submodule `backend`  → repo nầy

        repo nầy (Staging Deploy, chay theo push dev)
          ├─ diff commit truoc/sau → biet submodule nao doi
          ├─ frontend doi → checkout submodule → vercel pull/build --prod/deploy
          │                 → https://aiinterview-vn.vercel.app
          └─ backend  doi → POST Render deploy hook
                            → https://ai-interview-backend-staging.onrender.com
```

Hai repo con bump submodule bằng secret `DEPLOY_PAT`. `workflow_run` không dùng
được vì nó **chỉ phát trong cùng một repository** (đo 2026-10-17: run
`Frontend CI` #2 kết thúc 16:55:16, repo deploy không tạo run nào).

## Chạy local

```bash
git clone --recurse-submodules \
  https://github.com/PNV-do-an/AI-Interview-Platform-deploy-staging.git
cd AI-Interview-Platform-deploy-staging
cp .env.example .env
docker compose up -d --build          # db + backend + frontend (3000)
docker compose --profile sso up -d    # thêm Keycloak ở 8180
```

Clone rồi mà chưa lấy submodule: `git submodule update --init --recursive`.
Lấy code mới nhất của một repo con: `git submodule update --remote backend`.

## Secrets

Trong repo nầy:

| Tên | Lấy ở đâu |
|---|---|
| `VERCEL_TOKEN` | vercel.com → Account → Settings → Tokens |
| `VERCEL_ORG_ID` | `team_bgWCYiMMgj6WD2svhdjU5Kpg` |
| `VERCEL_PROJECT_ID` | `prj_Y3gCKVlIjThmzhqziNQcQzTrdGJm` |
| `RENDER_DEPLOY_HOOK_URL` | Render → service `ai-interview-backend-staging` → Settings → **Deploy hooks** |
| `DEPLOY_PAT` | xem hàng dưới — job `frontend` còn dùng token này để checkout submodule |

Trong repo FE **và** repo BE:

| Tên | Lấy ở đâu |
|---|---|
| `DEPLOY_PAT` | GitHub fine-grained token, quyền **Contents: read and write**; bắt buộc nhất là repo nầy (bot push commit bump submodule), thêm FE/BE để checkout khi repo chuyển private |

## Runbook

Deploy lại cả hai (khi Render free tier ngủ, hoặc deploy hỏng giữa chừng):
Actions → **Staging Deploy** → Run workflow → tích `force`.

Rollback về một cặp SHA cũ: `git -C backend checkout <sha>` (hoặc `frontend`),
`git add backend && git commit && git push dev` — workflow thấy submodule đổi sẽ
deploy đúng SHA đó. Superrepo luôn ghi lại cặp frontend/backend đang chạy, nên
nó là lịch sử deploy.

Kiểm tra nhanh sau deploy:

```bash
curl -sI https://aiinterview-vn.vercel.app | head -1
curl -s  https://ai-interview-backend-staging.onrender.com/api/v1/packages -o /dev/null -w '%{http_code}\n'
```

Lưu ý: Render build từ **branch `dev` của repo BE**, không từ SHA mà superrepo
đang pin. Nếu ai push thẳng vào repo BE sau lần bump gần nhất, bản đang chạy sẽ
đi trước con trỏ trong superrepo một commit.

## Tài liệu

`spec/` là nguồn kế hoạch của đồ án: đọc `spec/README.md`, `spec/INDEX.md`,
`spec/conventions/frontend-ui.md`, `spec/known-gaps.md`, `spec/srs-core-baseline.md`
trước khi code.
