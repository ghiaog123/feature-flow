---
name: problem-solver
description: Giải một vấn đề kỹ thuật khó bằng cách để Claude Code và Codex tranh luận như hai chuyên gia ngang hàng, rồi chốt kết luận có thể hành động. Chạy 2 stage tùy request gốc — Stage 1 ASSESSMENT (chẩn đoán, luôn chạy), Stage 2 SOLUTION (đề xuất giải pháp, chỉ khi user đòi). Skill nhận vấn đề (quyết định kiến trúc, chọn công nghệ, gỡ bug khó, đánh đổi thiết kế, root-cause mơ hồ), làm rõ phạm vi + ý định, để Claude phân tích độc lập, sau đó invoke /codex-think-about cho Claude và Codex phản biện qua nhiều vòng, cuối cùng gộp thành assessment.md và (nếu đòi) analysis_brief.md + decision record. Dùng khi user nói "giải quyết vấn đề này", "cho hai bên tranh luận", "claude với codex bàn xem", "phản biện rồi chốt", "vấn đề này nên làm thế nào", "phân tích/đánh giá cái này", "đưa ra kết luận cuối cùng cho...", "debate rồi quyết". KHÔNG dùng cho: review code/PR đã viết (dùng codex-review), tạo doc/test/status (các skill khác trong bundle), hay câu hỏi đơn giản trả lời ngay được mà không cần tranh luận.
---

# Problem Solver (Claude ⇄ Codex)

Giải vấn đề kỹ thuật khó bằng **tranh luận ngang hàng** giữa Claude và Codex, rồi **chốt kết luận hành động được**. Skill này là lớp bọc quanh `/codex-think-about`: nó thêm phần **đóng khung vấn đề** ở đầu, **hai stage debate** ở giữa, và **gộp artifact ra đĩa** ở cuối — `/codex-think-about` lo phần debate engine trong mỗi stage.

Skill ra **một trong hai deliverable** tùy request gốc:
- **Chỉ chẩn đoán** ("phân tích/đánh giá cái này") → dừng ở `assessment.md`.
- **Chẩn đoán + giải pháp** ("nên làm gì/fix thế nào") → chạy tiếp Stage 2 → `analysis_brief.md`.

```
Vấn đề (user)
   │
   ▼
[1] Đóng khung + phát hiện ý định (chẩn đoán? / chẩn đoán+giải pháp?)   ← skill này
   │
   ▼
[2] Deep-dive code (adaptive: inline hoặc tỏa sub-agent) — Claude độc lập, rào thông tin
   │
   ▼
╔═══ STAGE 1 — ASSESSMENT (luôn chạy) ═══════════════════════╗
║  mỗi bên phân tích độc lập → md riêng                       ║
║  → peer debate  ← delegate /codex-think-about (framing: chẩn đoán) ║
║  → gộp assessment.md                                        ║
╚═════════════════════════════════════════════════════════════╝
   │
   ├─ request KHÔNG đòi giải pháp → trả assessment, DỪNG
   │
   └─ request ĐÒI giải pháp
        │
        ▼
╔═══ STAGE 2 — SOLUTION (chỉ khi đòi) ═══════════════════════╗
║  từ assessment → mỗi bên đề xuất giải pháp độc lập → md riêng║
║  → peer debate  ← delegate /codex-think-about (framing: giải pháp)║
║  → gộp analysis_brief.md                                    ║
╚═════════════════════════════════════════════════════════════╝
   │
   ▼
[Compact có kiểm soát] → chain sang implementation-planner
```

## Nguyên tắc cốt lõi

- **Claude và Codex ngang hàng** — không ai là reviewer, không ai là người thực thi. Hai chuyên gia phản biện nhau.
- **Rào thông tin**: Claude PHẢI hoàn tất phân tích độc lập của mình TRƯỚC khi đọc output của Codex. Không xem trộm để tránh thiên kiến mỏ neo.
- **Tranh luận để hội tụ, không để thắng**: mục tiêu là kết luận đúng, không phải bảo vệ quan điểm ban đầu. Đổi ý khi đối phương có bằng chứng tốt hơn là kết quả tốt.
- **Tách dữ kiện khỏi ý kiến**: claim tra cứu được phải có nguồn; suy luận/đánh giá ghi rõ là quan điểm.
- **Codex KHÔNG sửa file project** — chỉ tư duy + (nếu cần) web research. Nếu phát hiện Codex sửa file, dừng ngay theo File Modification Guard của `/codex-think-about`.
- **Tái dùng debate engine, compose nhiều lần**: mỗi stage gọi `/codex-think-about` một lần với framing khác nhau (chẩn đoán ≠ giải pháp). Không tự chế giao thức codex riêng.
- **Artifact ra đĩa, main giữ gọn**: kết quả cuối mỗi stage ghi thành file `.md`; transcript debate + md từng bên + report deep-dive chỉ là trung gian, gộp xong thì bỏ (xem "Vệ sinh context").

## Workflow

### Bước 1 — Đóng khung vấn đề + phát hiện ý định

Trước khi tranh luận, làm rõ để cả hai bên cùng giải đúng một bài. Hỏi user (gộp tối đa 2-3 câu, đừng tra tấn):

| Cần | Vì sao |
|---|---|
| **Phát biểu vấn đề** một câu, rõ "cái cần quyết/hiểu" | Tránh hai bên debate lệch đề |
| **Tiêu chí thành công** / ràng buộc (perf, deadline, stack, không được đổi gì) | Để kết luận đo được, không chung chung |
| **Bối cảnh project** + file/đoạn code liên quan | Cho debate bám thực tế, không lý thuyết suông |
| **Các phương án đang cân nhắc** (nếu user đã có) | Mở rộng/phản biện thay vì bắt đầu từ 0 |
| **Mức effort** cho Codex (mặc định `high`) | Vấn đề khó → effort cao |

**Phát hiện ý định (quyết định deliverable):**
- Tín hiệu **chỉ chẩn đoán**: "phân tích", "đánh giá", "hiểu vì sao", "cái gì đang xảy ra", "root cause là gì" → dừng ở Stage 1.
- Tín hiệu **chẩn đoán + giải pháp**: "nên làm gì", "fix thế nào", "giải pháp", "cách xử lý", "quyết định phương án nào" → chạy tiếp Stage 2.
- **Mơ hồ** → dùng `AskUserQuestion` hỏi user muốn dừng ở chẩn đoán hay đi tiếp tới giải pháp. Đừng đoán.

Nếu phát biểu vấn đề mơ hồ → đề xuất một bản viết lại sắc hơn, xác nhận với user rồi mới chạy. (Tham khảo `references/question-sharpening.md` của codex-think-about nếu cần khung làm rõ.)

**Không đưa ý kiến giải pháp ở bước này** — giữ trung lập tới khi vào tranh luận.

### Bước 2 — Deep-dive code (RÀO THÔNG TIN, adaptive)

Trước khi gọi Codex/đọc bất cứ output nào của Codex, Claude tự hiểu địa hình vấn đề. **Chọn cách theo quy mô (adaptive-by-size — cổng chống over-engineer):**

- **Vấn đề nhỏ / phạm vi hẹp** (vài file, đã biết chỗ) → deep-dive **inline** bằng Glob/Grep/Read ngay trên main. Không tỏa sub-agent.
- **Vấn đề lớn / phải quét rộng** (nhiều module, chưa biết touchpoint, hoặc cần cả web research) → **tỏa parallel**:
  1. **Index rẻ trước**: nếu có sẵn công cụ index code (vd `graphify`), chạy để có bản đồ nhanh; nếu không → một lượt Glob/Grep định vị vùng.
  2. **Fan-out trong 1 message**: nhiều sub-agent `Explore`/`general-purpose` chạy song song, mỗi con một góc (module, luồng dữ liệu, test hiện có, tài liệu/web). Yêu cầu mỗi con trả **report gọn `file:line` + phát hiện**, KHÔNG dump code.
  3. **Main chỉ giữ report gộp** — không nuốt toàn bộ file vào context main.

Từ hiểu biết đó, Claude viết **phân tích độc lập** của mình: các khả năng/giả thuyết, đánh đổi, rủi ro, khuyến nghị sơ bộ + độ tin cậy + giả định đang dựa vào. MAY dùng MCP tools (web_search, context7) — ghi nguồn.

Phân tích này phải **hoàn chỉnh và cuối cùng** trước khi vào debate. Đây là "lá phiếu độc lập" của Claude.

### Stage 1 — ASSESSMENT (luôn chạy)

**Mục tiêu**: chẩn đoán đúng vấn đề (bản chất, nguyên nhân, ràng buộc, rủi ro) — CHƯA bàn giải pháp.

1. **Mỗi bên độc lập → md riêng**: Claude ghi chẩn đoán của mình; Codex (qua debate engine) đưa chẩn đoán của Codex. Giữ rào thông tin.
2. **Peer debate**: invoke Skill `codex-think-about`, framing về **chẩn đoán**:
   - **QUESTION** = "Chẩn đoán vấn đề: `<phát biểu>` — bản chất/nguyên nhân/ràng buộc/rủi ro là gì?"
   - **PROJECT_CONTEXT** = bối cảnh + ràng buộc + tiêu chí thành công + report deep-dive gộp ở Bước 2.
   - **RELEVANT_FILES** = file/đoạn code liên quan (đường dẫn, không dán nguyên khối).
   - **CONSTRAINTS** = điều không được vi phạm.
   - **EFFORT** = mức đã chốt (mặc định `high`).
   Mang **phân tích độc lập ở Bước 2** làm lập trường mở màn của Claude — không vứt đi nghĩ lại từ đầu.
3. **Gộp `assessment.md`**: ghi `docs/features/<feature>/assessment.md` (hoặc nơi user chỉ định) theo `references/assessment-template.md`. Trong lúc debate, phân loại từng điểm: **Đồng thuận thật / Bất đồng thật / Insight riêng Claude / Insight riêng Codex / Cùng hướng khác độ sâu**.

Tôn trọng mọi luật của codex-think-about (rào thông tin, không cap số vòng, chỉ thoát khi consensus hoặc stalemate, File Modification Guard, luôn finalize+stop). Nếu `/codex-think-about` không khả dụng → báo user cài đặt, KHÔNG tự chế giao thức codex riêng.

### Nhánh — dừng hay đi tiếp?

- **Request KHÔNG đòi giải pháp** → trình `assessment.md`, tóm tắt chẩn đoán + bất đồng còn lại (nếu stalemate), **DỪNG**. Gợi ý: nếu sau này muốn giải pháp, chạy lại skill này với yêu cầu "đề xuất giải pháp" (dùng lại assessment.md).
- **Request ĐÒI giải pháp** → sang Stage 2.

### Stage 2 — SOLUTION (chỉ khi đòi)

**Mục tiêu**: từ chẩn đoán đã chốt → đề xuất giải pháp hành động được.

1. **Mỗi bên đề xuất độc lập → md riêng**: dựa trên `assessment.md`, Claude đề xuất phương án của mình; Codex đề xuất của Codex. Rào thông tin lần nữa.
2. **Peer debate**: invoke lại Skill `codex-think-about`, framing về **giải pháp**:
   - **QUESTION** = "Dựa trên chẩn đoán đã chốt, giải pháp tốt nhất cho `<phát biểu>` là gì?"
   - **PROJECT_CONTEXT** = tóm tắt `assessment.md` + ràng buộc + tiêu chí thành công.
   - **RELEVANT_FILES**, **CONSTRAINTS**, **EFFORT** như trên.
3. **Gộp `analysis_brief.md`**: chốt thành brief hành động được, ghi `docs/features/<feature>/analysis_brief.md` theo `references/brief-template.md`:
   1. **Quyết định**: chọn phương án nào (hoặc kết hợp), dứt khoát.
   2. **Lý do**: 2-4 lý do chính, dựa trên đồng thuận + bằng chứng mạnh nhất từ debate.
   3. **Đánh đổi đã chấp nhận**: thẳng thắn cái gì phải hy sinh.
   4. **Bất đồng còn lại** (nếu stalemate): bảng `Điểm | Claude | Codex` + khuyến nghị nên nghiêng bên nào; nếu thật sự cần user quyết → hỏi.
   5. **Bước tiếp theo**: checklist hành động cụ thể để hiện thực quyết định.
   6. **Rủi ro / cần kiểm chứng thêm**: cái gì chưa chắc, kiểm thế nào.
   7. **Nguồn hợp nhất** + **độ tin cậy** của kết luận.

### Bước cuối — Decision record (tùy chọn) + chain

- **Decision record (tùy chọn, hỏi user)**: nếu là quyết định kiến trúc cần truy vết → ghi `docs/decisions/<slug>-<YYYY-MM-DD>.md` theo `references/decision-record.md`.
- **Chain**: `analysis_brief.md` là đầu vào tự nhiên cho `implementation-planner` (lập plan có phản biện). Nhắc user bước kế tiếp; nêu rõ đường dẫn brief.

## Vệ sinh context (compact có kiểm soát)

Đừng bao giờ `/compact` trần. Sau khi gộp xong artifact ra đĩa, đề xuất một block compact **điền path thật, nói rõ GIỮ gì / BỎ gì**:

```
/compact GIỮ: nội dung docs/features/<feature>/assessment.md (+ analysis_brief.md nếu có Stage 2),
phát biểu vấn đề đã sắc hóa, ý định (chẩn đoán/giải pháp), quyết định + bất đồng còn lại.
BỎ: 2 transcript debate của /codex-think-about, md phân tích từng bên, report deep-dive của sub-agent
(đã tổng hợp vào artifact; diff/chi tiết còn ở file trên đĩa).
```

Cái gì **chưa** kịp ghi vào artifact → gắn cờ must-keep, đừng để compact nuốt mất.

## Anti-patterns

- ❌ Bỏ Bước 2, gọi thẳng Codex rồi ăn theo ý Codex (mất rào thông tin → mất giá trị phản biện).
- ❌ Nhảy thẳng sang giải pháp khi request chỉ đòi chẩn đoán (bỏ nhánh dừng ở Stage 1).
- ❌ Đoán ý định khi mơ hồ thay vì `AskUserQuestion`.
- ❌ Dừng ở "đây là những gì hai bên nói" mà không gộp thành artifact hành động được.
- ❌ Nuốt toàn bộ file vào context main khi lẽ ra nên tỏa sub-agent trả report gọn (vấn đề lớn).
- ❌ Tỏa sub-agent cho vấn đề nhỏ đã biết chỗ (over-engineer) — inline là đủ.
- ❌ Tự viết lại giao thức gọi codex thay vì delegate `/codex-think-about`.
- ❌ Ép consensus giả khi thật ra còn bất đồng — stalemate trung thực tốt hơn đồng thuận gượng.
- ❌ Để Codex sửa file project mà không dừng theo File Modification Guard.
- ❌ `/compact` trần làm mất assessment/brief chưa kịp ghi ra đĩa.
