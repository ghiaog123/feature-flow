---
name: ff-feature-brief
description: Tạo "Feature Brief" — 1 trang HTML phi kỹ thuật giúp PO/QC hiểu feature mới (thay đổi gì, vì sao, luồng hoạt động, tiêu chí nghiệm thu, kết quả test, gaps). Tổng hợp từ docs/features/<feature>/ (proposal.html, test_plan_*.html, implementation_status.html) khi có, đọc code hoặc hỏi user khi thiếu. Dịch khái niệm kỹ thuật sang ẩn dụ đời thường (JWT → "thẻ thông hành") kèm ví dụ nhân vật cụ thể. Dùng skill này khi user nói "feature brief", "tạo brief", "one-pager", "tài liệu cho PO/QC xem", "tóm tắt feature cho PO", "doc bàn giao QC", "giải thích feature cho người không kỹ thuật", "trình bày feature dễ hiểu", "doc demo feature cho stakeholder" — kể cả khi không nói rõ chữ "brief". KHÔNG dùng cho: test plan chi tiết (ff-test-case-writer), tracking tiến độ code (ff-impl-status), API docs (ff-api-contract-writer).
---

# Feature Brief

Sinh **1 trang HTML** tóm tắt feature cho **PO** (hiểu nghiệp vụ, quyết định) và **QC** (biết test gì). Nguyên tắc cốt lõi: người đọc **không phải dev** — mọi khái niệm kỹ thuật phải dịch sang ngôn ngữ đời thường, mỗi quy tắc trừu tượng phải có ví dụ mô phỏng cụ thể.

```
docs/features/<feature>/          (đọc cái nào có)
├── proposal.html             → What changed, Why, Flow, AC
├── test_plan_*.html          → Test status, lý do SKIP
├── implementation_status.html → trạng thái code, việc còn lại
└── feature_brief.html        ← OUTPUT của skill này
```

## Workflow

### Bước 1 — Xác định feature + gom nguồn

1. User nêu tên feature → tìm `docs/features/<feature>/`. Nhiều folder ứng viên → hỏi.
2. Glob các nguồn: `proposal.html`, `test_plan_*.html`, `implementation_status.html`, `implementation_plan.md`.
3. **Linh hoạt, không chặn:** thiếu nguồn nào thì bù bằng cách đọc code liên quan hoặc hỏi user 2-3 câu. Thiếu test plan → mục Test status ghi rõ "chưa test" (không bịa số liệu).

Extract text từ HTML nguồn (file thường rất dài, đừng Read nguyên file):

```bash
python3 -c "
import re, html
src = open('<file>').read()
text = re.sub(r'<style.*?</style>|<script.*?</script>', '', src, flags=re.S)
text = re.sub(r'<[^>]+>', '\n', text)
print(chr(10).join(l.strip() for l in html.unescape(text).split('\n') if l.strip()))
"
```

Test plan của ff-test-case-writer/ff-service-test-runner có sẵn box tóm tắt (headline PASS/SKIP/FAIL + lý do SKIP) — ưu tiên lấy từ đó.

### Bước 2 — Chưng cất nội dung

Từ nguồn, rút ra (đây là **dữ kiện thật**, không suy diễn):

| Cần | Lấy từ |
|---|---|
| Before/after (3-5 khác biệt lớn nhất) | proposal mục kiến trúc/mục tiêu |
| Lý do business (2-3 gạch đầu dòng) | proposal mục tiêu + hỏi user nếu mơ hồ |
| Luồng chính + các nhánh từ chối/lỗi | proposal luồng chi tiết / sequence diagrams |
| Hành vi nghiệm thu được (→ AC) | proposal expect + test plan P0 cases |
| Số liệu test + lý do chưa test | test plan box tóm tắt |
| Việc còn lại trước release | implementation_status / test plan |

### Bước 3 — Dịch sang ngôn ngữ phi kỹ thuật

**Ngôn ngữ output theo ngôn ngữ user dùng** (user chat tiếng Việt → brief tiếng Việt). Giọng văn: như giải thích cho đồng nghiệp phòng kinh doanh.

Quy tắc dịch — đây là phần quyết định chất lượng brief:

1. **Mỗi khái niệm kỹ thuật → 1 ẩn dụ đời thường, dùng nhất quán cả trang.** Chọn 1 ẩn dụ khung (tòa nhà/thẻ ra vào, bưu điện, siêu thị…) rồi map mọi thứ vào đó. Ví dụ đã dùng tốt:
   - JWT/token → "thẻ thông hành có chữ ký"
   - service nội bộ → "kho công cụ", "chốt kiểm soát"
   - fail-closed → "thẻ giả = không mở được tầng nào"
   - intersection quyền → "vừa nằm trong bộ được giao, vừa được app thuê"
   - cache invalidation → "có hiệu lực ngay, không đợi 5 phút"
2. **Đặt nhân vật cụ thể** (vd "chị An dùng app Meeting Note") và kể mọi ví dụ qua nhân vật đó — người đọc nhớ câu chuyện, không nhớ thuật ngữ.
3. **Mỗi AC kèm 1 dòng `💡 Ví dụ:`** mô phỏng tình huống thật với thời gian/số liệu cụ thể ("10h00 admin gỡ quyền → 10h00 chị An hỏi lại → AI từ chối ngay").
4. **Quy tắc trừu tượng → hình vẽ phép tính.** Công thức quyền/điều kiện nhiều vế: vẽ thành dãy ô chips (① cái này ∩ ② cái kia − ③ trừ cái này = ✅ kết quả) thay vì viết công thức.
5. Tên kỹ thuật thật (endpoint, bảng DB, tên class) chỉ xuất hiện trong phần dành cho QC (sequence diagram, code ref) — phần PO tuyệt đối không.

### Bước 4 — Sinh HTML

Copy `assets/template.html` (cùng thư mục skill này) → `docs/features/<feature>/feature_brief.html`, điền nội dung. Template có sẵn CSS đầy đủ + Mermaid CDN; xem chú thích `<!-- FILL: ... -->` trong template cho từng vùng.

**Cấu trúc chuẩn 7 khối — linh hoạt bỏ khối không áp dụng** (feature chưa test → bỏ khối 5 hoặc ghi "chưa test"; không có gaps → bỏ khối 6):

| # | Khối | Audience | Nội dung |
|---|---|---|---|
| 0 | 📌 Tóm tắt 30 giây | Cả hai | 2-3 câu + box "Hình dung nhanh" bằng ẩn dụ khung |
| 1 | Có gì thay đổi | Cả hai | Bảng trước/sau 3-5 dòng + hình vẽ phép tính nếu có công thức |
| 2 | Vì sao cần | PO | 2-3 gạch đầu dòng business |
| 3 | Luồng hoạt động | Cả hai | 2 diagram Mermaid: **3a swimlane kể chuyện nhân vật** (cho PO, không thuật ngữ) + **3b sequence diagram** (cho QC, label tiếng dễ đọc nhưng giữ chi tiết kỹ thuật: token, fail-closed, log) |
| 4 | Tiêu chí nghiệm thu | QC | 5-8 AC dạng Nếu/Khi/Thì, mỗi cái 1 dòng 💡 ví dụ. Phủ: happy path, từ chối khi sai, ranh giới quyền, thu hồi, lỗi giữa chừng |
| 5 | Kết quả test | Cả hai | 3 card số (ĐẠT/CHƯA TEST/LỖI) + verdict + bug tìm được nhờ test (giá trị!) + link test plan |
| 6 | Ngoài phạm vi & việc còn lại | Cả hai | Chủ đích làm sau / lý do case chưa test (1 dòng/nhóm) / checklist trước khi bật |

Mỗi khối gắn badge audience (`PO` / `QC` / `PO + QC`) — người đọc lướt đúng phần mình.

**Diagram:** Mermaid qua CDN (`mermaid@10` esm) — cần internet khi mở file; chấp nhận được với mạng công ty. Label node bằng ngôn ngữ user, swimlane PO kể đúng 1 câu chuyện của nhân vật (input câu hỏi thật → các chốt → ✅/⛔).

### Bước 5 — Kiểm tra + bàn giao

1. Mọi con số (PASS/FAIL, số case) phải truy được về nguồn — **không bịa, không làm tròn đẹp**. Nguồn không có → ghi "chưa có dữ liệu".
2. Link tương đối tới docs nguồn ở khối Test status (`test_plan_*.html`, `proposal.html`, `implementation_status.html`).
3. `open <file>` cho user xem, hỏi cần chỉnh gì.
4. Nhắc nếu `docs/` bị gitignore: file chỉ local — gửi trực tiếp hoặc đưa lên Confluence/wiki.

## Anti-patterns

- ❌ Bê nguyên đoạn proposal kỹ thuật vào phần PO ("backend ký RS256 JWT aud=mcp-server…")
- ❌ AC không có ví dụ, hoặc ví dụ chung chung ("user làm gì đó sai → hệ thống từ chối")
- ❌ Bịa số test / verdict khi không có test plan
- ❌ Nhiều ẩn dụ lẫn lộn (vừa "thẻ ra vào" vừa "chìa khóa két" vừa "vé xem phim") — chọn 1 khung, theo đến cùng
- ❌ Brief dài quá ~2 màn hình scroll/khối — đây là one-pager, chi tiết để link sang doc nguồn
