# Checklist còn lại

> Toàn bộ phần bắt buộc đã hoàn tất. Kết quả mới nhất:
>
> - `pytest tests/ -q`: **92 passed, 4 skipped, 0 failed**; bốn test bị skip là
>   nhóm local fallback vì repo đang kiểm tra cloud thật.
> - `python grade.py --no-bonus`: **100/100**.
> - Railway: agent và Redis cùng Online; `/health`, `/ready`, auth và `/ask` có
>   key đều đã kiểm tra thành công.
> - Bonus CI/CD không tiếp tục theo yêu cầu.

## Đã xác nhận

- [x] CP1–CP5 đạt toàn bộ điểm bắt buộc.
- [x] Public URL HTTPS hoạt động.
- [x] `/health` trả 200 và `/ready` chứng minh kết nối Redis.
- [x] `/ask` không key trả 401; có key thật trả 200.
- [x] `DEPLOYMENT.md` chứa URL, cấu hình và output cloud thật.
- [x] `screenshots/health.png` được chụp từ domain Railway public.
- [x] `screenshots/dashboard.png` ghi lại trạng thái Railway live và các public
  checks; không còn ảnh local fallback cũ.
- [x] `.env`, API key, token và private key không bị Git theo dõi.
- [x] Không còn `TODO`, placeholder hoặc `NotImplementedError` bắt buộc.
- [x] Repo có lịch sử commit theo từng checkpoint.

## Việc cuối của học viên

- [ ] Nộp link repo public lên Codelab/LMS:
  `https://github.com/KuanqXol/K4-L3A-DAY12-DamQuangSon-2A202602868-CloudServicesAndDeployment`
