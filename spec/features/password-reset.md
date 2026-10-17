# Quên mật khẩu (forgot / reset password)

Trạng thái: `done`
Người/phiên tạo: 2026-10-17, branch `feature/create-api-sign-up`
Mockup nguồn: `interviewai_reset_password` (màn nhập email) + `interviewai_set_new_password_confirmation` (màn đặt mật khẩu mới)

## 1. Mục tiêu

User quên mật khẩu nhận link qua email, mở link thấy được tài khoản của mình + thời gian còn hiệu lực, đặt mật khẩu mới (có xác nhận lại), tự chọn vô hiệu hoá các phiên khác; frontend gọi được hết các API đó và hiển thị đúng số liệu thật.

## 2. Phạm vi / Không làm

- Làm: `POST /forgot-password` (đã có, sửa TTL 15 -> 30 phút theo mockup), `GET /reset-password?token=` (mới: validate link + trả danh tính), `POST /reset-password` (hợp đồng mới: token + password + confirmPassword + signOutEverywhere), thu hồi phiên qua claim `tv` ở `POST /refresh`, 2 màn frontend dựng lại theo mockup.
- Không làm: "Verify via SMS code" (không có API SMS — nút hiển thị notice "not available yet"); đếm số phiên bị thu hồi ("COMPLETED (2 SESSIONS)") vì hệ thống không có bảng session, chỉ có `token_version`; revocation tức thời access token (access token vẫn hạn 24h — xem mục 9); email thật (SMTP chưa bật, link nằm trong log).

## 3. API thật dùng được

| Endpoint | Method | Response liên quan | Ghi chú |
|---|---|---|---|
| `/api/v1/auth/forgot-password` | POST | 200 `ApiResponse{message}` (message chung, không lộ email tồn tại hay không) | token UUID, TTL 30 phút, mỗi lần gọi ghi đè token cũ |
| `/api/v1/auth/reset-password?token=` | GET | 200 `{fullName, email, role, remainingSeconds}`; 401 "không hợp lệ" / "đã hết hạn" | route công khai, dùng cho màn đặt mật khẩu mới |
| `/api/v1/auth/reset-password` | POST | 200 `{eventId, signedOutEverywhere}`; 400 thiếu field; 401 sai confirm / mật khẩu yếu / token sai-hết hạn | token chỉ dùng 1 lần, thành công là xoá `reset_token` |
| `/api/v1/auth/refresh` | POST | 200 token mới; 401 "Phiên đăng nhập không còn hiệu lực" | so `tv` trong refresh token với `users.token_version` |
| `/api/v1/auth/login` | POST | `AuthResponse` | refresh token phát ra mang `tv` hiện tại |

## 4. Ánh xạ số liệu -> UI

| Thành phần mockup | Nguồn | Xử lý khi trống/lỗi |
|---|---|---|
| Pill "Verified Session • 12m remaining" | `remainingSeconds` từ GET, đồng hồ đếm ở client (tick 15s) | hết hạn giữa chừng -> đổi sang card "link đã hết hạn" + nút xin link lại |
| Identity chip (avatar tên, email, "Verified Account", role) | `fullName`/`email`/`role` từ GET | role hiển thị "User" vì hệ thống chưa có role CANDIDATE |
| 4 rule mật khẩu + thanh strength | rule client, tập ký tự đặc biệt `@#$%^&+=!` giống hệt `PASSWORD_PATTERN` backend | nút Save disable cho tới khi đủ 4 rule + confirm khớp |
| "Passwords match" badge | so sánh client `password === confirmPassword` | — |
| Modal `EVENT_ID` | `eventId` từ API (`evt_rot_<userId>_<8 ký tự>`) | — |
| Modal `REVOCATION_STATUS` | `signedOutEverywhere` -> "COMPLETED (ALL SESSIONS)" / "SKIPPED (THIS DEVICE ONLY)" | không bịa số phiên |
| "Check your inbox" + "Resend email" | 200 từ forgot-password | cooldown 30s ("Resend available in Ns") rồi mới cho gửi lại |

## 5. Khối mockup BỎ + lý do

- `COMPLETED (2 SESSIONS)`: backend không lưu danh sách session -> chỉ báo trạng thái, không báo số.
- CTA "Continue to Evaluation Console": user vừa mất phiên, vào console được ngay là sai -> "Continue to sign in".
- Chữ "CANDIDATE" trong chip danh tính: tài khoản thật có `ROLE_USER` -> hiển thị role thật.
- Mật khẩu ký tự đặc biệt `(!@#$%^&*)`: backend từ chối `*`, UI liệt kê đúng tập `@#$%^&+=!` để không hứa sai.
- Không tự thêm endpoint "kiểm tra email tồn tại" hay "danh sách thiết bị đang đăng nhập" (luật: không gọi thêm API để làm cho đầy).

## 6. Việc backend cần làm (đã xong)

- `RESET_TOKEN_TTL_MINUTES = 30`; user không tồn tại hoặc `deletedAt != null` -> trả message chung, không ném lỗi (chống dò email).
- `getResetTokenInfo(token)`: cùng luồng kiểm tra với reset (token hợp lệ -> còn hạn -> tài khoản còn sống), trả `remainingSeconds = Duration.between(now, expiry)`.
- `resetPassword(token, newPassword, confirmPassword, signOutEverywhere)`: thứ tự kiểm tra token -> confirm khớp -> pattern -> đổi mật khẩu; `signOutEverywhere` khác `false` (kể cả null) là bump `token_version`; kết quả trả `eventId` + cờ thực thi.
- Claim `tv` trong refresh token (`JwtUtil.generateRefreshToken(account, role, tokenVersion)`), `POST /refresh` từ chối khi `tv` lệch `users.token_version`; cả 4 chỗ phát refresh token (login, Google, GitHub, SSO) đều truyền version hiện tại.
- Thành công: `reset_token`/`expiry` = null, `login_attempts` = 0, `lock_until` = null.

## 7. Test cases

| # | Tình huống | Kết quả mong đợi |
|---|---|---|
| 1 | forgot với email hợp lệ | 200 message chung, token có TTL <= 30 phút |
| 2 | forgot với email không tồn tại | vẫn 200, message y hệt |
| 3 | GET reset-password token thật | 200 + `{fullName, email, role, remainingSeconds<=1800}` |
| 4 | GET/POST với token sai hoặc token hết hạn | 401 "không hợp lệ" / "đã hết hạn", mật khẩu giữ nguyên |
| 5 | POST thiếu confirmPassword | 400 `Validation failed` + `detail.confirmPassword` |
| 6 | POST password != confirmPassword | 401 "Mật khẩu xác nhận không khớp...", không gửi email xác nhận |
| 7 | POST mật khẩu yếu / chỉ có `*` | 401 kèm rule message |
| 8 | POST hợp lệ, signOutEverywhere true/thiếu | 200 `signedOutEverywhere: true`, `token_version` +1, refresh token cũ -> 401 |
| 9 | POST hợp lệ, signOutEverywhere false | 200 `signedOutEverywhere: false`, `token_version` giữ nguyên, refresh token hiện tại -> 200 |
| 10 | Dùng lại token sau khi reset | 401 "không hợp lệ" |
| 11 | Login mật khẩu cũ / mới sau reset | cũ 401 "Mật khẩu sai", mới 200 |

## 8. Cách đã verify

Đo 2026-10-17 trên code production (docker compose, backend image đang chạy, DB `mysqljava`/3307 — backend trỏ 3307 qua `.env`, không phải container `db` trong compose):

- `mvn -B clean test-compile` trong container `maven:3.9.6`: BUILD SUCCESS (main + 4 file test). `mvn clean test` có `PersistenceTest`/`AdminUserControllerIntegrationTest` cần Testcontainers -> chỉ xanh trên CI.
- curl: #1 200 + `timestampdiff` còn 1320s; #2 200 message giống hệt; #3 200 `{"fullName":"Probe Reset User","email":"probe-reset-...","role":"ROLE_USER","remainingSeconds":1206}`; #4 token lạ 401, token back-date 401 (mật khẩu + `token_version` không đổi); #5 400 `detail.confirmPassword`; #6 401 "không khớp"; #7 "Ab1@" và "Abcdef1*" đều 401; #8 200 `evt_rot_18_3002dcc8` + `signedOutEverywhere:true`, `tv` 0 -> 1, refresh token cũ 401; #9 200 `signedOutEverywhere:false`, refresh 200; #10 401; #11 401/200.
- `npx tsc --noEmit` exit 0; eslint 2 trang mới: 0 error, 9 warning `jsx-a11y/anchor-is-valid` (baseline cả thư mục pages 70 warning, không phải lỗi mới).
- `docker compose build frontend && up -d`: `/forgot-password` DOM đủ mockup (badge "Password Recovery", H1, "Work or Personal Email", "Send Reset Link", note 30 phút, "Check your inbox" + "Resend available in 13s" -> "Resend email", SSO + SMS, Help & Security 3 mục, footer). `/reset-password?token=<thật>` render pill "Verified Session • 25m remaining" + chip danh tính từ API; điền "Hello1234" -> Good + 3/4 rule, thiếu special; mismatch -> "Passwords do not match" + nút disable; đủ điều kiện -> Strong + "Passwords match" + nút enable; submit -> modal `COMPLETED (ALL SESSIONS)` / `evt_rot_18_86e30774`; bỏ tick -> `SKIPPED (THIS DEVICE ONLY)`.
- Token giả -> card lỗi "Link đặt lại mật khẩu không hợp lệ"; vào `/reset-password` không query -> "This link is missing or no longer valid."; "Request a new reset link" -> `/forgot-password`; "Verify via SMS code" -> notice; SSO -> `/login`.

Chưa verify ở viewport >=1024px (browser-use hidden viewport — xem `verification.md` mục 4).

## 9. Câu hỏi mở cho anh Huy

- **Access token vẫn sống 24h sau khi "Sign out of all other sessions".** Refresh token bị vô hiệu ngay (đã đo 401), nhưng request đang cầm access token cũ vẫn qua được `JwtFilter` tới hết hạn. Muốn thu hồi tức thời thì phải rút `EXPIRATION_TIME_ACCESS_TOKEN` xuống ngắn (15 phút) + dựa vào `/refresh`, hoặc thêm blacklist token. Đổi lại là mỗi 15 phút app phải refresh — cần anh chốt.
- **JWT secret ở local**: `docker-compose.yml` chỉ truyền `JWT_SECRET`, còn `JwtUtil` đọc `JWT_ACCESS_SECRET`/`JWT_REFRESH_SECRET` -> local đang dùng fallback commit trong repo (staging/Render sinh secret thật qua `render.yaml`). Có nên sửa compose truyền đủ 2 biến không?
- **Tài khoản probe** `probe-reset-1792242022@example.test` (id 18, DB local) đang để lại, cần xác nhận xoá hay giữ để test tiếp.
