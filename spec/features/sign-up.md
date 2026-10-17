# Sign-up (đăng ký tài khoản)

Trạng thái: `done`
Người/phiên tạo: 2026-10-17, branch `feature/create-api-sign-up`
Mockup nguồn: `C:\Users\huy\Downloads\.stitch_extracts\latest_3\stitch_ai_interview_platform_ui_ux\interviewai_create_account\`

## 1. Mục tiêu

Người mới tạo tài khoản (form hoặc OAuth Google/GitHub) và vào được app ngay sau khi đăng ký; mọi API luồng đăng ký trả đúng status + thông điệp, có rate-limit chống spam và có test.

## 2. Phạm vi / Không làm

- Làm: `POST /register` (validate, chống trùng email 409, rate-limit theo cửa sổ thời gian), `GET /verify`, `POST /resend-verification`, link email trỏ về trang `/verify` của frontend; RegisterPage dựng lại đúng mockup (bỏ phone + confirm password theo mockup); OAuth Google/GitHub từ trang register (GitHub callback về `/login`).
- Không làm: bắt xác minh email trước khi đăng nhập (đáp án đề xuất ở mục 9 — hiện chưa có SMTP thật, chặn thì chết luồng đăng ký ở mọi môi trường); 2FA/magic link; API "check email tồn tại" realtime (mockup không có khối đó — luật "không gọi thêm API mới để làm cho đầy").

## 3. API thật dùng được

| Endpoint | Method | Response liên quan | Ghi chú |
|---|---|---|---|
| `/api/v1/auth/register` | POST | 201 `ApiResponse{message}`; 400 validation; 401 password yếu/rate-limit; 409 email trùng | đã đo bằng curl sau khi sửa |
| `/api/v1/auth/verify?token=` | GET | 200 `ApiResponse{message}`; 401 token hết hạn/sai | route công khai |
| `/api/v1/auth/resend-verification` | POST | 200; 401 email chưa đăng ký | giữ nguyên hành vi |
| `/api/v1/auth/google` + `/google/client-id` | POST/GET | `AuthResponse` | đã đo trên staging |
| `/api/v1/auth/github` + `/github/client-id` | POST/GET | `AuthResponse` | OAuth app callback chỉ nhận `/login` -> từ trang register phải đặt `redirect_uri=/login` |

## 4. Ánh xạ số liệu -> UI

| Thành phần mockup | Nguồn | Xử lý khi trống |
|---|---|---|
| "Continue with Google/GitHub" | client-id endpoints | nút hiện nhưng báo notice "coming soon" (như LoginPage) |
| Field Full Name / Work/Personal Email / Password | người dùng nhập | — |
| Terms checkbox | client-only, không submit | chặn nút Create Account khi chưa tick |
| Panel gradient phải (headline, badges, quote "Elena R.") | chữ nguyên văn mockup (STATIC, không số liệu động) | — |

## 5. Khối mockup BỎ + lý do

- Không có field phone/confirm password trong mockup -> bỏ khỏi form (backend vẫn nhận `phone` optional cho API cũ).
- Header có nút Language/Theme: giữ y như LoginPage (chrome sẵn có, không tự chế thêm).

## 6. Việc backend cần làm

- `RegisterAttempt`: mỗi lần đăng ký thất bại ghi 1 dòng (address=ip, account=email, timestamp=now) như `LoginAttempt`; rule đếm theo cửa sổ 5 phút (IP ≥10, email ≥5) thay vì counter/lock flags cũ (các flag này tính nhầm vì row email và row IP thường là cùng một dòng).
- Email trùng: exception riêng map 409 (đang trả 401 qua `InvalidAccountException`).
- `EmailServiceImpl.sendVerificationEmail/sendResendVerificationEmail`: link `app.frontend-url + /verify?token=` thay vì hardcode `http://localhost:8080/api/...`.
- Message register: "Đăng ký thành công! Bạn có thể đăng nhập ngay." (tài khoản auto-enabled, message cũ hứa "kiểm tra email để kích hoạt" là sai thực tế).

## 7. Test cases

| # | Tình huống | Kết quả mong đợi |
|---|---|---|
| 1 | Đăng ký hợp lệ | 201, user `ROLE_USER` enabled, không có dòng `register_attempts` |
| 2 | Email đã tồn tại | 409, message "Email đã tồn tại trong hệ thống" |
| 3 | Password yếu (thiếu ký tự đặc biệt) | 401 + 1 dòng attempt |
| 4 | 5 lần fail cùng email trong 5 phút | bị chặn, message "Vui lòng thử lại sau 5 phút" |
| 5 | 10 lần fail cùng IP trong 5 phút | bị chặn IP |
| 6 | Missing fields / sai format | 400 `Validation failed` + `detail.<field>` |
| 7 | GET /verify token thật | 200, token bị xoá khỏi user |
| 8 | GET /verify token sai | 401 |

## 8. Cách đã verify

Đã đo 2026-10-17 trên code production (docker compose, bundle `main.7a646b3f.js` khớp image):

- `mvn` test trong container `maven:3.9.6`: `RegisterInterfaceImplTest` 6/6 pass, `SsoEmailDomainPolicyTest` 3/3 pass (mvn host thiếu, `mvnw.cmd` mất wrapper jar — chạy Docker).
- curl `POST /register` hợp lệ -> 201 `Đăng ký thành công! Bạn có thể đăng nhập ngay.`; email trùng -> 409; thiếu field/sai email/yếu mật khẩu -> 400 `Validation failed` + `detail.<field>` (rule password chặn ở DTO trước khi tới service, nên service-guard chỉ unit test mới bao được).
- `POST /login` với user vừa đăng ký -> 200 (phải cherry-pick thủ công bản sửa `LoginAttempt.java` từ `48f3899` vì branch này chưa có; thiếu nó login 500 `Field 'created_at' doesn't have a default value`).
- `GET /verify?token=` token thật -> 200 và token bị xoá; dùng lại -> 401. `POST /resend-verification` khi user đã enabled -> 401 đúng hành vi cũ.
- UI: `/register` DOM khớp mockup (h1 "Create your account", 4 input full-name/work-email/password-input/terms-consent, 2 nút OAuth, panel gradient, footer); submit thật bằng browser-use -> chuyển `/login?registered=true` + banner "Registration successful! Please sign in to continue."; `/verify?token=sai` -> card lỗi hiển thị message API + link Sign in.
- `register_attempts` sau lô test vẫn 3 dòng cũ -> luồng thành công không ghi dòng rác.
- Đã xoá user probe `ui.probe.reg.1017@`/`ui.probe.ui.1017@` + dòng `login_attempts` tương ứng.
- Route `/verify` công khai trong `App.tsx`; `npx tsc --noEmit` sạch; 0 `style={{` ở 2 file mới.

Chưa verify bằng mắt ở desktop >=1024px (browser-use hidden viewport — xem theo `verification.md` mục 4).

## 9. Câu hỏi mở cho anh Huy

- **Chặn login khi chưa verify email?** Đề xuất: chưa (SMTP chưa cấu hình thật — `MAIL_PASSWORD` rỗng ở mọi env; chặn thì không ai đăng ký được). Khi nào bật SMTP: `enabled=false` lúc register + trang `/verify` đã có sẵn đường đi.
