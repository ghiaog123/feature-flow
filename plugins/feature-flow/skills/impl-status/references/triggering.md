# Triggering phrases

Skill `impl-status` chỉ active sau khi user opt-in rõ ràng. Tham chiếu nhanh.

## Activate (skill bật)

- `/impl-status`
- `impl-status`
- "dùng skill impl-status"
- "bật impl-status"
- "bật status file"
- "maintain status file"
- "theo dõi implementation status"
- "use impl-status skill"
- "enable status tracker"

## Deactivate / pause (giữ file, không update)

- "stop status file"
- "skip status"
- "khỏi cần status file"
- "đừng update status nữa"
- "pause impl-status"
- "tắt status tracker"

## Resume (sau pause trong cùng session, update lại)

- "resume impl-status"
- "bật lại status file"
- "update status đi"

## Load lại từ chat trước (cross-session)

Khi user mở chat mới và muốn tiếp tục feature đã có file trong `docs/features/`:

- "resume impl-status <feature>"
- "load status <feature>"
- "tiếp tục feature X"
- "tiếp tục từ session trước"
- "đọc lại status file của <feature>"
- "load lại proposal + status"
- "continue from <feature>"

Skill workflow khi gặp các phrase này:
1. Locate `docs/features/<feature>/`. Nếu user không nói tên → list các folder có sẵn cho user chọn.
2. Đọc `proposal.html` (nếu có) → `implementation_status.html` → `README.md`.
3. Tóm tắt context ≤10 dòng cho user (progress, decisions, bugs, rollout pending).
4. Verify file/symbol nhắc trong status còn tồn tại (grep). Drift → ghi vào status.
5. Đợi chỉ đạo của user, không auto-code.

## KHÔNG trigger (false positive thường gặp)

Đây là phrase **không** được trigger skill — user chỉ yêu cầu implement, không yêu cầu tracking:

- "implement feature X"
- "làm cho tôi task này"
- "thêm endpoint Y"
- "refactor module Z"
- "viết migration"
- "fix bug này"

Skill chỉ tạo file khi user opt-in. Một mình "implement" / "code đi" / "làm đi" KHÔNG đủ để trigger.
