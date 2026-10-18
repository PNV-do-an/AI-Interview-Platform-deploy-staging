# Spec

Thư mục kế hoạch (plan) cho AI và người. Khi nhận một yêu cầu tính năng, đọc ở đây trước khi mở code.

## Cấu trúc

```
spec/
  README.md                      file này — cách dùng thư mục
  srs-core-baseline.md           SRS Core Baseline (3 vai trò) -> đối chiếu từng FR/NFR với code thật
  INDEX.md                       29 màn Stitch -> route -> file -> trạng thái -> thiếu gì
  conventions/frontend-ui.md     luật giao diện bắt buộc tuân theo (design system Stitch)
  verification.md                cách build + soát DOM thật, kèm các bẫy đã gặp
  known-gaps.md                  lỗ hổng backend/API/số liệu đã kiểm chứng, chưa sửa
  templates/feature-plan.md      mẫu kế hoạch cho tính năng mới
  features/<ten-tinh-nang>.md    kế hoạch chi tiết từng tính năng (một file một feature)
```

## Bản cắt theo mảng

Hai repo code mang bản sao để đọc ngay cạnh code; **bản gốc vẫn là thư mục `spec/` ở đây**, và vẫn là nơi để sửa.

| Repo | `spec/` gồm |
|---|---|
| `AI-Interview-Platform-FE` | `README.md` (riêng), `INDEX.md`, `conventions/frontend-ui.md`, `verification.md`, `features/mockup-parity-next.md`, `templates/feature-plan.md` |
| `AI-Interview-Platform-BE` | `README.md` (riêng), `srs-core-baseline.md`, `known-gaps.md`, `features/sign-up.md`, `features/password-reset.md`, `templates/feature-plan.md` |

Sửa xong ở đây thì cập nhật bản sao tương ứng. Một luồng chạm cả hai mảng phải ghi ở cả hai nơi — đó là giá phải trả của cách chia này.

## Cách dùng khi code một thứ mới

1. Tìm màn/tính năng tương ứng trong `INDEX.md`. Nếu chưa có dòng nào -> tạo `features/<ten>.md` từ `templates/feature-plan.md` và điền trước phần Mục tiêu, Mockup nguồn, Phạm vi, Không làm.
1b. Nếu yêu cầu đến từ phạm vi nghiệp vụ (không phải một màn), chiếu mã FR/NFR trong `srs-core-baseline.md` trước — file đó đã ghi yêu cầu nào code thật đang đáp ứng, yêu cầu nào chưa, và quyết định nào còn thuộc chủ dự án.
2. Đọc `conventions/frontend-ui.md` (nếu chạm UI) và `known-gaps.md` (nếu chạm số liệu/API) — hai file này trả lời sẵn những câu sẽ bị hỏi lại.
3. Ghi vào file plan: ánh xạ field thật -> thành phần UI, và danh sách khối mockup **bỏ** kèm lý do (thiếu endpoint/thiếu column). Đây là phần giá trị nhất của plan.
4. Làm theo plan, rồi cập nhật lại trạng thái trong `INDEX.md` và mục Kết quả trong file plan.
5. chạy các bước trong `verification.md` trước khi báo xong.

## Quy ước

- Một file plan = một tính năng, tên ke-case: `features/job-approval-workflow.md`.
- Trạng thái screen: `TODO` (chưa có file) / `UI` (đã dựng form, chưa nối số liệu) / `DATA` (dùng số liệu thật) / `STATIC` (màn tĩnh theo mockup, cố ý không có API) / `BLOCKED` (chờ backend).
- Plan ghi **thứ đã kiểm chứng** kèm cách kiểm chứng (curl, log container, DOM), không ghi suy đoán.
- Số liệu ước lượng chi phí/thời gian: không ghi vào spec.
- Spec là tài liệu làm việc, không commit chung một lô với code tính năng nếu chưa xong; xoá các file tạm (`*.test.tsx` chạy một lần, script probe) sau khi dùng.

## Định nghĩa "xong" cho một màn

- `npx tsc --noEmit` sạch.
- Render thật trong container nginx :3000 (đã `docker compose build frontend`), không còn bundle cache.
- Không có `style={{...}}` ngoài width/height % động; không có hex literal không giải thích.
- Khối nào bỏ phải có dòng ghi lý do trong plan (thiếu endpoint nào, column nào).
- Không có chữ bịa số liệu ở màn thuộc loại `DATA`/`UI`.
