---
name: plan-implementer
description: Hiện thực một implementation plan rồi để Claude Code và Codex cùng review code vừa viết qua /codex-impl-review — đảm bảo code chất lượng VÀ bám đúng plan. Claude đọc plan, implement theo từng bước, sau đó invoke /codex-impl-review để hai bên review adversarial trên thay đổi chưa commit (working-tree) hoặc branch diff, sửa tới khi APPROVE hoặc stalemate, kiểm cả độ khớp với plan. Dùng khi user nói "đọc plan rồi implement", "implement xong review code", "code theo plan rồi cho codex review", "hiện thực plan này", "implement và review lại code base vừa viết", "đảm bảo code follow plan". Thường chạy SAU implementation-planner (đã có plan APPROVE). KHÔNG dùng cho: lập plan (implementation-planner), tranh luận quyết định (problem-solver), review plan trước khi code (codex-plan-review), chỉ review code không implement (codex-impl-review trực tiếp).
---

# Plan Implementer (Claude implement → Claude ⇄ Codex review)

Từ **implementation plan đã chốt** → **code đã viết + đã review đạt chất lượng + bám plan**. Một pha implement (Claude) nối một pha review ngang hàng (Claude ⇄ Codex):

```
implementation_plan.md (thường từ implementation-planner)
   │
   ▼
[1] Đọc plan, xác nhận scope
   │
   ▼
[2] Implement theo từng bước   ← Claude viết code
   │
   ▼
[3] Self-check vs plan + sanity (build/test)
   │
   ▼
[4] Review code: Claude ⇄ Codex   ← delegate /codex-impl-review
   │   (kiểm chất lượng + độ khớp plan, sửa loop tới APPROVE/stalemate)
   ▼
[5] Hoàn tất + bàn giao
```

Skill này **không tự chế giao thức codex** — phần review giao cho `/codex-impl-review`. Giá trị thêm: đọc & bám plan khi implement, và đưa **độ khớp với plan** thành một tiêu chí review tường minh.

## Nguyên tắc cốt lõi

- **Bám plan**: mọi bước implement trỏ về một mục trong plan. Lệch plan → hoặc cập nhật plan có lý do, hoặc không lệch. Không âm thầm trôi khỏi plan.
- **Claude viết code; Codex chỉ review** — Codex KHÔNG sửa file. Claude áp fix sau khi đánh giá từng issue.
- **Cần sửa code → phải thoát plan mode** (skill này chỉnh sửa file). `/codex-impl-review` yêu cầu thoát plan mode trước.
- **Adversarial review tới cùng**: chỉ dừng khi `verdict === APPROVE` hoặc stalemate. Không nhận APPROVE giả.
- **Giữ ý định chức năng**: fix theo review không đổi hành vi trừ khi issue đòi đổi hành vi.

## Workflow

### Bước 1 — Đọc plan + xác nhận scope

- Tìm plan: user chỉ đường dẫn → dùng; nếu không → `docs/features/<feature>/implementation_plan.md` (output của `implementation-planner`). Nhiều/không có → hỏi.
- Đọc plan: nắm Mục tiêu/Outcomes, các bước, vùng tác động (file:line), test & verify, ngoài phạm vi.
- Nếu plan chưa qua review (`implementation-planner`/`codex-plan-review`) → báo user, gợi ý review plan trước; vẫn cho phép tiếp nếu user muốn.
- Chốt scope review sau này: **working-tree** (thay đổi chưa commit, mặc định) hay **branch** (diff vs base). Nếu repo đang sạch → sẽ tạo thay đổi ở Bước 2.

### Bước 2 — Implement theo từng bước

- Thoát plan mode (skill này sửa file).
- Theo thứ tự bước trong plan, tôn trọng phụ thuộc. Bước song song được → có thể gộp, nhưng giữ thay đổi mạch lạc.
- Mỗi bước: sửa đúng file plan chỉ ra; theo convention/pattern sẵn có trong code (đọc quanh trước khi viết).
- Lệch plan khi cần (plan sai/thiếu) → ghi lại lý do, cập nhật plan tương ứng để plan vẫn là nguồn sự thật.
- Viết/cập nhật test theo mục "Test & verify" của plan khi áp dụng.

### Bước 3 — Self-check trước khi gọi Codex

- Đối chiếu lại từng Outcome trong plan: đã đạt chưa? Bước nào chưa làm → làm nốt hoặc ghi rõ lý do hoãn.
- Sanity: chạy build/lint/test liên quan nếu có. Lỗi rõ ràng → sửa trước, đừng đẩy rác sang review.
- Xác nhận có thay đổi để review: working-tree phải có staged/unstaged (`git status --short`), hoặc branch phải khác base. Không có thay đổi → không gọi review.

### Bước 4 — Review code: invoke /codex-impl-review

Giao phần review adversarial cho `/codex-impl-review` (engine lo init→start→poll→verdict→apply fix/rebut→resume tới APPROVE hoặc stalemate, auto-detect scope + effort theo số file).

Invoke Skill `codex-impl-review`, truyền:
- **USER_REQUEST** = mục tiêu feature + "code này hiện thực plan tại `<đường dẫn plan>`".
- **SESSION_CONTEXT** = **độ khớp với plan là tiêu chí review tường minh** — liệt kê Outcomes + các bước plan để Codex kiểm code có bám không, kèm convention/ràng buộc. Nếu có lệch plan có chủ đích → nêu rõ lý do để Codex không báo nhầm.
- Scope (working-tree/branch) + base branch nếu là branch.
- Effort: để engine tự suy theo số file (`<10`=medium, `10-50`=high, `>50`=xhigh) trừ khi user override.

Mỗi issue Codex nêu: hợp lệ → Claude **sửa code** rồi verify rồi resume; không hợp lệ → rebut kèm bằng chứng cụ thể. Lặp tới `verdict === APPROVE` hoặc stalemate. Codex KHÔNG sửa file. Luôn finalize+stop.

Đặc biệt soi: **chỗ code không khớp plan** (thiếu bước, làm khác hướng đã chốt) — đây là lý do chính chạy skill này, ngoài bug/chất lượng thông thường.

### Bước 5 — Hoàn tất + bàn giao

- **APPROVE** → báo: file đã đụng, số vòng review, issue tìm/sửa/bác, defect đã fix theo mức nghiêm trọng, rủi ro còn lại, độ khớp plan cuối cùng.
- **Stalemate** → liệt kê điểm bế tắc (`Điểm | Claude | Codex`), khuyến nghị, hỏi user chốt.
- Đối chiếu lần cuối: mọi Outcome trong plan ✅ hay còn nợ gì.
- Gợi ý bước kế của bundle: `impl-status` (cập nhật tiến độ), `api-contract-writer` (API docs), `test-case-writer`/`service-test-runner` (test plan + chạy), `feature-brief` (bàn giao PO/QC). Commit/PR chỉ khi user yêu cầu.

## Anti-patterns

- ❌ Implement không đọc plan, hoặc trôi khỏi plan mà không ghi lý do/không cập nhật plan.
- ❌ Gọi `/codex-impl-review` khi chưa có thay đổi (repo sạch) → engine pre-flight fail.
- ❌ Đẩy code lỗi build/test sang review thay vì sửa sanity trước.
- ❌ Bỏ tiêu chí "khớp plan" trong SESSION_CONTEXT → review thành review code chung chung, mất mục đích.
- ❌ Để Codex sửa file (Codex chỉ review; Claude áp fix).
- ❌ Nhận APPROVE giả khi engine chưa ra `verdict: APPROVE`.
- ❌ Tự commit/PR khi user chưa yêu cầu.
