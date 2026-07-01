---
name: plan-implementer
description: Hiện thực một implementation plan rồi để Claude Code và Codex cùng review code vừa viết qua /codex-impl-review — đảm bảo code chất lượng VÀ bám đúng plan. Claude đọc plan, implement theo từng bước (inline khi plan nhỏ, hoặc orchestrate sub-agent song song khi plan lớn), sau đó invoke /codex-impl-review để hai bên review adversarial trên thay đổi chưa commit (working-tree) hoặc branch diff, sửa tới khi APPROVE hoặc stalemate, kiểm cả độ khớp với plan. Dùng khi user nói "đọc plan rồi implement", "implement xong review code", "code theo plan rồi cho codex review", "hiện thực plan này", "implement và review lại code base vừa viết", "đảm bảo code follow plan". Thường chạy SAU implementation-planner (đã có plan APPROVE). KHÔNG dùng cho: lập plan (implementation-planner), tranh luận quyết định (problem-solver), review plan trước khi code (codex-plan-review), chỉ review code không implement (codex-impl-review trực tiếp).
---

# Plan Implementer (Claude implement → Claude ⇄ Codex review)

Từ **implementation plan đã chốt** → **code đã viết + đã review đạt chất lượng + bám plan**. Cách implement **thích nghi theo quy mô plan**; phần review là một pha ngang hàng (Claude ⇄ Codex):

```
implementation_plan.md (thường từ implementation-planner)
   │
   ▼
[1] Đọc plan, xác nhận scope + CHỌN CHẾ ĐỘ (inline / orchestrated)
   │
   ├─ NHỎ  → [2a] Implement inline (Claude viết trực tiếp)
   │
   └─ LỚN  → [2b] Orchestrate: Phase 0 contract-first → tỏa sub-agent theo Owns files
   │              (main = nhạc trưởng, giữ dependency map + conformance trong FILE)
   ▼
[3] Integration join + self-check vs plan (build/test trên main)
   │
   ▼
[4] Review code: Claude ⇄ Codex   ← delegate /codex-impl-review (1 join trên main, không fan out)
   │
   ▼
[5] Hoàn tất + bàn giao + compact có kiểm soát
```

Skill này **không tự chế giao thức codex** — phần review giao cho `/codex-impl-review`. Giá trị thêm: đọc & bám plan khi implement, orchestrate an toàn khi plan lớn, và đưa **độ khớp với plan** thành một tiêu chí review tường minh.

## Nguyên tắc cốt lõi

- **Bám plan**: mọi bước implement trỏ về một mục trong plan. Lệch plan → hoặc cập nhật plan có lý do, hoặc không lệch. Không âm thầm trôi khỏi plan.
- **Adaptive-by-size (cổng chống over-engineer)**: chỉ orchestrate khi plan **lớn/vượt ngưỡng**; nhỏ thì implement inline. Đừng dựng dàn nhạc cho việc một người làm.
- **Orchestrator giữ lean**: khi orchestrate, main = nhạc trưởng — giữ **dependency map + conformance report trong FILE**, không trong bộ nhớ chat. Ruột từng phase nằm ở sub-agent; main không nuốt code.
- **Song song chỉ khi disjoint + no-dep + contract-first**: hai bước chạy song song ⟺ `Owns files` disjoint VÀ không phụ thuộc (theo dependency contract của plan), VÀ seam chung đã khóa ở Phase 0.
- **Brief tự đủ + report gọn**: mỗi sub-agent nhận brief tự đủ, sửa **CHỈ file nó sở hữu**, trả report gọn (`plan item → status → file:line → deviation`), **KHÔNG dump code**.
- **Review = 1 join trên main, không fan out**: `/codex-impl-review` đọc **git diff**, không đọc chat. Claude viết code; Codex chỉ review; Claude áp fix sau khi đánh giá từng issue.
- **Cần sửa code → phải thoát plan mode** (skill này chỉnh sửa file). `/codex-impl-review` yêu cầu thoát plan mode trước.
- **Adversarial review tới cùng**: chỉ dừng khi `verdict === APPROVE` hoặc stalemate. Không nhận APPROVE giả.
- **Giữ ý định chức năng**: fix theo review không đổi hành vi trừ khi issue đòi đổi hành vi.

## Workflow

### Bước 1 — Đọc plan + xác nhận scope + chọn chế độ

- Tìm plan: user chỉ đường dẫn → dùng; nếu không → `docs/features/<feature>/implementation_plan.md` (output của `implementation-planner`). Nhiều/không có → hỏi.
- Đọc plan: nắm Mục tiêu/Outcomes, các bước, **dependency contract** (`Owns files`/`Depends on`/song song?), vùng tác động (file:line), test & verify, ngoài phạm vi.
- Nếu plan chưa qua review (`implementation-planner`/`codex-plan-review`) → báo user, gợi ý review plan trước; vẫn cho phép tiếp nếu user muốn.
- **Chọn chế độ implement (adaptive-by-size)**:
  - **Inline (2a)** — mặc định — khi plan nhỏ: ít bước, ít file, một mạch code làm được mà không phình context main.
  - **Orchestrated (2b)** khi plan **lớn/vượt ngưỡng**: nhiều phase với `Owns files` disjoint, hoặc dự kiến context main sẽ phình quá mức làm tụt chất lượng code (ngưỡng heuristic ~150k tokens — **tunable**, chỉ là mốc, không phải luật cứng). Nghi ngờ / phase ít nhưng dài → vẫn có thể orchestrate để giữ main gọn.
- Chốt scope review sau này: **working-tree** (thay đổi chưa commit, mặc định) hay **branch** (diff vs base). Repo đang sạch → sẽ tạo thay đổi ở Bước 2.

### Bước 2a — Implement inline (plan nhỏ)

- Thoát plan mode (skill này sửa file).
- Theo thứ tự bước trong plan, tôn trọng `Depends on`. Bước song song được → có thể gộp, nhưng giữ thay đổi mạch lạc.
- Mỗi bước: sửa đúng file plan chỉ ra; theo convention/pattern sẵn có trong code (đọc quanh trước khi viết).
- Lệch plan khi cần (plan sai/thiếu) → ghi lại lý do, cập nhật plan tương ứng để plan vẫn là nguồn sự thật.
- Viết/cập nhật test theo mục "Test & verify" của plan khi áp dụng.

→ sang Bước 3.

### Bước 2b — Orchestrate (plan lớn)

Main = **nhạc trưởng**, ruột phase ở sub-agent. Trình tự:

1. **Dựng dependency map ra FILE**: từ dependency contract của plan, dựng DAG các phase (node = phase, cạnh = `Depends on`) ghi vào file làm việc (vd `docs/features/<feature>/impl_progress.md`) — KHÔNG giữ trong chat memory. File này cũng là **conformance report** (mỗi phase: status / file:line đụng / deviation) cập nhật dần.
2. **Phase 0 — contract-first (trên main)**: khóa **seam chung** (kiểu dữ liệu, chữ ký hàm, interface, stub) TRƯỚC trên main. Các phase song song sau chỉ điền vào contract cố định này → không giẫm lên seam chung.
3. **Tỏa sub-agent theo DAG**: các phase có `Owns files` disjoint + không phụ thuộc + Phase 0 đã xong → chạy **song song** (nhiều sub-agent trong 1 message); phần còn lại tuần tự theo cạnh phụ thuộc. Mỗi sub-agent nhận **brief tự đủ**:
   - Mục tiêu phase + các Outcome liên quan + trích đoạn plan của phase đó.
   - **Owns files** (chỉ được sửa các file này), convention cần theo, contract từ Phase 0.
   - Yêu cầu trả **report gọn**: mỗi plan item → `status` (done/partial/blocked) → `file:line` đã đụng → `deviation` (nếu lệch plan, kèm lý do). **KHÔNG dump code** (diff nằm ở git).
   - Nếu hai phase buộc phải đụng file chung → **không** song song; nối tuần tự hoặc gộp vào một sub-agent.
4. **Main gom report** vào conformance file sau mỗi lượt; không nuốt code của sub-agent vào context.

→ sang Bước 3.

### Bước 3 — Integration join + self-check vs plan (trên main)

- **Integration join**: sau khi gom hết phase → chạy **full build/lint/test suite trên main** (không phải trong sub-agent). Lỗi tích hợp rõ ràng → sửa trên main trước, đừng đẩy rác sang review.
- **Self-check vs plan**: đối chiếu lại từng Outcome trong plan — đã đạt chưa? Bước nào chưa làm → làm nốt hoặc ghi rõ lý do hoãn vào conformance file.
- Xác nhận có thay đổi để review: working-tree phải có staged/unstaged (`git status --short`), hoặc branch phải khác base. Không có thay đổi → không gọi review.

### Bước 4 — Review code: invoke /codex-impl-review

Giao phần review adversarial cho `/codex-impl-review` — **một join duy nhất trên main, không fan out**; engine đọc **git diff** (không đọc chat, không đọc report sub-agent), lo init→start→poll→verdict→apply fix/rebut→resume tới APPROVE hoặc stalemate, auto-detect scope + effort theo số file.

Invoke Skill `codex-impl-review`, truyền:
- **USER_REQUEST** = mục tiêu feature + "code này hiện thực plan tại `<đường dẫn plan>`".
- **SESSION_CONTEXT** = **độ khớp với plan là tiêu chí review tường minh** — liệt kê Outcomes + các bước plan để Codex kiểm code có bám không, kèm convention/ràng buộc. Nếu có lệch plan có chủ đích (ghi ở conformance file) → nêu rõ lý do để Codex không báo nhầm. Không dán report sub-agent — chỉ trỏ plan + để engine đọc diff.
- Scope (working-tree/branch) + base branch nếu là branch.
- Effort: để engine tự suy theo số file (`<10`=medium, `10-50`=high, `>50`=xhigh) trừ khi user override.

Mỗi issue Codex nêu: hợp lệ → Claude **sửa code** rồi verify rồi resume; không hợp lệ → rebut kèm bằng chứng cụ thể. Lặp tới `verdict === APPROVE` hoặc stalemate. Codex KHÔNG sửa file. Luôn finalize+stop.

Đặc biệt soi: **chỗ code không khớp plan** (thiếu bước, làm khác hướng đã chốt) — đây là lý do chính chạy skill này, ngoài bug/chất lượng thông thường.

### Bước 5 — Hoàn tất + bàn giao

- **APPROVE** → báo: file đã đụng, số vòng review, issue tìm/sửa/bác, defect đã fix theo mức nghiêm trọng, rủi ro còn lại, độ khớp plan cuối cùng (đối chiếu conformance file).
- **Stalemate** → liệt kê điểm bế tắc (`Điểm | Claude | Codex`), khuyến nghị, hỏi user chốt.
- Đối chiếu lần cuối: mọi Outcome trong plan ✅ hay còn nợ gì.
- Gợi ý bước kế của bundle: `impl-status` (cập nhật tiến độ), `api-contract-writer` (API docs), `test-case-writer`/`service-test-runner` (test plan + chạy), `feature-brief` (bàn giao PO/QC). Commit/PR chỉ khi user yêu cầu.

## Vệ sinh context (compact có kiểm soát)

Đừng `/compact` trần — nhất là sau khi orchestrate (context dễ phình vì nhiều lượt gom). Sau khi review xong, đề xuất block compact điền path thật, rõ GIỮ/BỎ:

```
/compact GIỮ: conformance report docs/features/<feature>/impl_progress.md (status từng Outcome + deviation),
plan docs/features/<feature>/implementation_plan.md, kết quả review (verdict, issue đã fix, rủi ro còn lại).
BỎ: edit history từng bước, report chi tiết của sub-agent, transcript debate của /codex-impl-review
(đã tổng hợp vào conformance report; diff đầy đủ ở git).
```

Cái gì chưa vào conformance file/plan → gắn cờ must-keep trước khi compact.

## Anti-patterns

- ❌ Implement không đọc plan, hoặc trôi khỏi plan mà không ghi lý do/không cập nhật plan.
- ❌ Orchestrate cho plan nhỏ (over-engineer) — inline là đủ.
- ❌ Cho sub-agent chạy song song khi `Owns files` chồng nhau hoặc chưa khóa Phase 0 → giẫm chân, xung đột file.
- ❌ Giữ dependency map/conformance trong chat memory thay vì FILE → main phình, mất dấu khi context bị cắt.
- ❌ Sub-agent dump code về main thay vì report gọn → phá mục tiêu giữ main lean.
- ❌ Fan out cho pha review, hoặc để `/codex-impl-review` đọc chat/report thay vì git diff.
- ❌ Gọi `/codex-impl-review` khi chưa có thay đổi (repo sạch) → engine pre-flight fail.
- ❌ Đẩy code lỗi build/test sang review thay vì integration join sửa sanity trước.
- ❌ Bỏ tiêu chí "khớp plan" trong SESSION_CONTEXT → review thành review code chung chung, mất mục đích.
- ❌ Để Codex sửa file (Codex chỉ review; Claude áp fix).
- ❌ Nhận APPROVE giả khi engine chưa ra `verdict: APPROVE`.
- ❌ Tự commit/PR khi user chưa yêu cầu.
- ❌ `/compact` trần làm mất conformance report chưa kịp ghi ra đĩa.
