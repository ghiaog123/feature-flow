# Assessment — template (Stage 1)

Chẩn đoán vấn đề, CHƯA bàn giải pháp. Ghi ra `docs/features/<feature>/assessment.md`. Đây là output khi request chỉ đòi phân tích/đánh giá, và là đầu vào cho Stage 2 khi đòi giải pháp.

```markdown
# Assessment — <phát biểu vấn đề một câu>

- **Ngày**: <YYYY-MM-DD>
- **Người tham gia**: Claude Code ⇄ Codex
- **Trạng thái debate**: Consensus | Stalemate
- **Ý định request**: Chỉ chẩn đoán | Chẩn đoán + giải pháp (→ Stage 2)

## Phát biểu vấn đề (đã sắc hóa)
<Một câu, rõ cái cần hiểu/quyết.>

## Bối cảnh & ràng buộc
<Tiêu chí thành công, ràng buộc (stack/perf/deadline/không được đổi gì).>

## Chẩn đoán
<Bản chất vấn đề là gì. Nguyên nhân (root cause) nếu tìm được — kèm bằng chứng file:line.>

## Bằng chứng
| Claim | Nguồn (file:line / URL + ngày) | Dữ kiện hay ý kiến |
|---|---|---|
| | | |

## Phân loại điểm debate
- **Đồng thuận thật**: <...>
- **Bất đồng thật**: <...>
- **Insight riêng Claude**: <...>
- **Insight riêng Codex**: <...>
- **Cùng hướng khác độ sâu**: <...>

## Blindspots / Ẩn số chưa đóng
Ẩn số nổi lên từ Blindspot Pass (Bước 2) — cái Claude CHƯA kiểm, giả định đang dựa vào — + trạng thái sau debate.
| Ẩn số / giả định chưa kiểm | Độ tin cậy (Cao/TB/Thấp) | Cách đóng (đọc file / hỏi ai / test gì) | Trạng thái (đã đóng / còn mở) |
|---|---|---|---|
| | | | |

## Bất đồng còn lại (nếu stalemate)
| Điểm | Claude | Codex | Khuyến nghị nghiêng bên nào |
|---|---|---|---|

## Rủi ro / vùng chưa chắc
- <rủi ro> → kiểm bằng <cách>

## Độ tin cậy
<Cao | Trung bình | Thấp> — <vì sao>
```
