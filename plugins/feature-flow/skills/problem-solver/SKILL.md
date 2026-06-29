---
name: problem-solver
description: Giải một vấn đề kỹ thuật khó bằng cách để Claude Code và Codex tranh luận như hai chuyên gia ngang hàng, rồi chốt một kết luận cuối cùng có thể hành động. Skill nhận vấn đề (quyết định kiến trúc, chọn công nghệ, gỡ bug khó, đánh đổi thiết kế, root-cause mơ hồ), làm rõ phạm vi, để Claude phân tích độc lập, sau đó invoke /codex-think-about cho Claude và Codex phản biện lẫn nhau qua nhiều vòng, cuối cùng tổng hợp thành quyết định + lý do + các bước tiếp theo + decision record. Dùng khi user nói "giải quyết vấn đề này", "cho hai bên tranh luận", "claude với codex bàn xem", "phản biện rồi chốt", "vấn đề này nên làm thế nào", "đưa ra kết luận cuối cùng cho...", "debate rồi quyết". KHÔNG dùng cho: review code/PR đã viết (dùng codex-review), tạo doc/test/status (các skill khác trong bundle), hay câu hỏi đơn giản trả lời ngay được mà không cần tranh luận.
---

# Problem Solver (Claude ⇄ Codex)

Giải vấn đề kỹ thuật khó bằng **tranh luận ngang hàng** giữa Claude và Codex, rồi **chốt một kết luận hành động được**. Skill này là lớp bọc quanh `/codex-think-about`: nó thêm phần **đóng khung vấn đề** ở đầu và **tổng hợp quyết định + decision record** ở cuối — `/codex-think-about` lo phần debate engine ở giữa.

```
Vấn đề (user)
   │
   ▼
[1] Đóng khung vấn đề   ← skill này
   │
   ▼
[2] Claude phân tích độc lập   ← skill này (rào thông tin)
   │
   ▼
[3] Peer debate Claude ⇄ Codex   ← delegate /codex-think-about
   │
   ▼
[4] Tổng hợp quyết định cuối   ← skill này
   │
   ▼
[5] Decision record (tùy chọn)
```

## Nguyên tắc cốt lõi

- **Claude và Codex ngang hàng** — không ai là reviewer, không ai là người thực thi. Hai chuyên gia phản biện nhau.
- **Rào thông tin**: Claude PHẢI hoàn tất phân tích độc lập của mình TRƯỚC khi đọc output của Codex. Không xem trộm để tránh thiên kiến mỏ neo.
- **Tranh luận để hội tụ, không để thắng**: mục tiêu là kết luận đúng, không phải bảo vệ quan điểm ban đầu. Đổi ý khi đối phương có bằng chứng tốt hơn là kết quả tốt.
- **Tách dữ kiện khỏi ý kiến**: claim tra cứu được phải có nguồn; suy luận/đánh giá ghi rõ là quan điểm.
- **Codex KHÔNG sửa file project** — chỉ tư duy + (nếu cần) web research. Nếu phát hiện Codex sửa file, dừng ngay theo File Modification Guard của `/codex-think-about`.

## Workflow

### Bước 1 — Đóng khung vấn đề

Trước khi tranh luận, làm rõ để cả hai bên cùng giải đúng một bài. Hỏi user (gộp tối đa 2-3 câu, đừng tra tấn):

| Cần | Vì sao |
|---|---|
| **Phát biểu vấn đề** một câu, rõ "cái cần quyết" | Tránh hai bên debate lệch đề |
| **Tiêu chí thành công** / ràng buộc (perf, deadline, stack, không được đổi gì) | Để kết luận đo được, không chung chung |
| **Bối cảnh project** + file/đoạn code liên quan | Cho debate bám thực tế, không lý thuyết suông |
| **Các phương án đang cân nhắc** (nếu user đã có) | Mở rộng/phản biện thay vì bắt đầu từ 0 |
| **Mức effort** cho Codex (mặc định `high`) | Vấn đề khó → effort cao |

Nếu phát biểu vấn đề mơ hồ → đề xuất một bản viết lại sắc hơn, xác nhận với user rồi mới chạy. (Tham khảo `references/question-sharpening.md` của codex-think-about nếu cần khung làm rõ.)

**Không đưa ý kiến giải pháp ở bước này** — giữ trung lập tới khi vào tranh luận.

### Bước 2 — Claude phân tích độc lập (RÀO THÔNG TIN)

Trước khi gọi Codex/đọc bất cứ output nào của Codex, Claude tự giải vấn đề:

- Liệt kê các phương án khả dĩ (kể cả ngoài danh sách user đưa).
- Với mỗi phương án: đánh đổi, rủi ro, điều kiện phù hợp.
- Đưa khuyến nghị sơ bộ + độ tin cậy + giả định đang dựa vào.
- MAY dùng MCP tools (web_search, context7) để lấy dữ kiện — ghi nguồn.

Phân tích này phải **hoàn chỉnh và cuối cùng** trước khi sang Bước 3. Đây là "lá phiếu độc lập" của Claude.

### Bước 3 — Peer debate: invoke /codex-think-about

Giao phần tranh luận đa vòng cho skill `/codex-think-about` (engine đã có sẵn toàn bộ giao thức: init → start Codex → poll → cross-analysis → resume loop → consensus/stalemate → cleanup).

**Cách gọi**: invoke Skill `codex-think-about`, truyền vào:
- **QUESTION** = phát biểu vấn đề đã sắc hóa ở Bước 1.
- **PROJECT_CONTEXT** = bối cảnh + ràng buộc + tiêu chí thành công.
- **RELEVANT_FILES** = file/đoạn code liên quan.
- **CONSTRAINTS** = điều không được vi phạm.
- **EFFORT** = mức đã chốt (mặc định `high`).

Mang theo **phân tích độc lập ở Bước 2** làm lập trường mở màn của Claude trong debate — không vứt đi rồi nghĩ lại từ đầu.

Tôn trọng mọi luật của codex-think-about (rào thông tin, không cap số vòng, chỉ thoát khi consensus hoặc stalemate, File Modification Guard, luôn finalize+stop). Nếu `/codex-think-about` không khả dụng trên máy → báo user cài đặt, KHÔNG tự chế giao thức codex riêng.

Trong lúc debate, theo dõi và phân loại từng điểm: **Đồng thuận thật / Bất đồng thật / Insight riêng của Claude / Insight riêng của Codex / Cùng hướng khác độ sâu**.

### Bước 4 — Tổng hợp quyết định cuối

Khi debate kết thúc (consensus hoặc stalemate), KHÔNG dừng ở bản tóm tắt tranh luận. Chốt thành **quyết định hành động được**:

1. **Quyết định**: chọn phương án nào (hoặc kết hợp), phát biểu dứt khoát.
2. **Lý do**: 2-4 lý do chính, dựa trên điểm đồng thuận + bằng chứng mạnh nhất từ debate.
3. **Đánh đổi đã chấp nhận**: thẳng thắn cái gì phải hy sinh.
4. **Bất đồng còn lại** (nếu stalemate): bảng `Điểm | Claude | Codex`, kèm khuyến nghị nên nghiêng về bên nào và vì sao; nếu thật sự cần user quyết → hỏi.
5. **Bước tiếp theo**: checklist hành động cụ thể để hiện thực quyết định.
6. **Rủi ro / điều cần kiểm chứng thêm**: cái gì còn chưa chắc, kiểm thế nào.
7. **Nguồn hợp nhất** + **độ tin cậy** của kết luận.

### Bước 5 — Decision record (tùy chọn, hỏi user)

Hỏi user có muốn lưu lại không. Nếu có, ghi `docs/decisions/<slug>-<ngày>.md` (hoặc nơi user chỉ định) theo `references/decision-record.md`. Hữu ích khi vấn đề là quyết định kiến trúc cần truy vết về sau.

## Anti-patterns

- ❌ Bỏ Bước 2, gọi thẳng Codex rồi ăn theo ý Codex (mất rào thông tin → mất giá trị phản biện).
- ❌ Dừng ở "đây là những gì hai bên nói" mà không chốt quyết định hành động được.
- ❌ Tự viết lại giao thức gọi codex thay vì delegate `/codex-think-about`.
- ❌ Ép consensus giả khi thật ra còn bất đồng — stalemate trung thực tốt hơn đồng thuận gượng.
- ❌ Để Codex sửa file project mà không dừng theo File Modification Guard.
- ❌ Trộn dữ kiện và ý kiến không phân biệt; claim web không nguồn.
