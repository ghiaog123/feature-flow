---
name: ff-discover
description: "Cửa vào của bundle feature-flow — dùng KHI user chưa phát biểu nổi vấn đề, mới chỉ có một cảm giác mơ hồ. Biến một ý định chưa định hình (\"thấy chỗ này sai sai\", \"muốn cải thiện X nhưng chưa rõ cái gì\", \"nên thêm Y mà chưa biết chính xác\", \"không biết bắt đầu từ đâu\", \"giúp tôi tìm ra vấn đề thật là gì\", \"khám phá vùng này trước khi quyết\") thành một PHÁT BIỂU VẤN ĐỀ sắc + bản đồ ẩn số (known-unknowns) + các hướng can thiệp để chốt phạm vi. Kết hợp phỏng vấn có cấu trúc, thang từ vựng, blindspot pass (audit vùng code để lôi unknown-unknowns) và brainstorm hướng theo timescale. Chạy Claude-solo + phỏng vấn user (KHÔNG gọi Codex — chưa có gì để tranh luận). Output `discovery_brief.html` sinh động (TL;DR ngôn ngữ thường, sơ đồ Mermaid bản đồ vùng + coupling, card ẩn số tô màu theo độ tin cậy, option-space theo timescale) feed thẳng ff-problem-solver. Hãy dùng skill này bất cứ khi nào yêu cầu còn mơ hồ tới mức chưa lập được vấn đề rõ ràng — kể cả khi user không nói chữ \"discover\". KHÔNG dùng cho: vấn đề đã phát biểu rõ (ff-problem-solver), lập plan (ff-planning), review code (codex-review), tracking tiến độ (ff-impl-status)."
---

# ff-discover — Cửa vào: lôi ẩn số trước khi đặt vấn đề

Phần còn lại của bundle (`ff-problem-solver` → `ff-planning` → `ff-implement` → …) đều **giả định đã có một phát biểu vấn đề rõ**. Nhưng thực tế thường bắt đầu sớm hơn thế: user chỉ có một *cảm giác* — "chỗ này sai sai", "muốn nó tốt hơn", "hình như nên đổi cái gì đó" — chưa đủ sắc để hai chuyên gia debate hay để lập plan. Debate/plan trên nền mơ hồ = xây nhà trên cát.

`ff-discover` lấp đúng khoảng đó: **biến ý định chưa định hình → phát biểu vấn đề sắc + bản đồ ẩn số + option space**, để downstream có nền vững. Câu chốt của phương pháp: *"The map is not the territory — the gap between them is your unknowns."* Lôi ẩn số ra **lúc còn rẻ** (trước khi code) tốt hơn phát hiện lúc đã build sai.

```
Ý định mơ hồ (user: "chỗ này sai sai / muốn tốt hơn / chưa biết bắt đầu đâu")
   │
   ▼
[1] Phỏng vấn có cấu trúc (sắp theo tác động kiến trúc)   ← lôi ý định tiềm ẩn
   │
   ▼
[2] Thang từ vựng — từ mờ → nghĩa kỹ thuật đo được
   │
   ▼
[3] Blindspot Pass — audit vùng code, lôi unknown-unknowns (adaptive: inline / tỏa sub-agent)
   │
   ▼
[4] Brainstorm hướng can thiệp theo timescale — ĐỂ CHỐT SCOPE, không để quyết
   │
   ▼
[5] Gộp discovery_brief.html — sinh động (phát biểu sắc + sơ đồ vùng + card ẩn số + hướng + skill kế)
   │
   ▼
[Compact có kiểm soát] → chain sang ff-problem-solver (hoặc thẳng ff-planning nếu hướng đã rõ)
```

## Ranh giới skill (đọc trước khi làm bất cứ gì)

**Discover, KHÔNG decide, KHÔNG solve.** Output của skill này là một vấn đề *đã sắc* + option space, **không phải** chẩn đoán và **không phải** giải pháp đã chọn:
- Chẩn đoán bản chất/nguyên nhân → việc của `ff-problem-solver`.
- Chọn giải pháp / cách hiện thực → việc của `ff-problem-solver` (Stage 2) và `ff-planning`.

Nếu trong lúc discover bạn thấy mình đang *chốt* một nguyên nhân hay *chọn* một giải pháp → dừng lại, đó là dấu hiệu vượt ranh giới. Việc của bạn là làm cho vấn đề **hỏi được cho đúng**, không phải trả lời nó.

## Nguyên tắc cốt lõi

- **Claude-solo + phỏng vấn user.** Ở stage này chưa có giả thuyết/quyết định nào để hai AI phản biện — giá trị nằm ở **moi ý định từ user** và **audit code thật**, không phải debate. Vì vậy KHÔNG gọi Codex ở đây; debate bắt đầu ở `ff-problem-solver` downstream.
- **Ẩn số là sản phẩm chính, không phải phụ.** Mục tiêu không phải "trả lời nhanh" mà là *làm lộ ra cái user (và cả bạn) chưa biết mình chưa biết*. Một discovery brief tốt thường để lại nhiều ẩn số đã-đặt-tên hơn là câu trả lời.
- **Tương tác nhưng bền khi thiếu người.** Skill này sống nhờ hội thoại với user (AskUserQuestion). Nhưng nếu chạy trong ngữ cảnh **không có người trả lời** (batch/eval/CI), đừng treo: **nêu rõ giả định đang dùng, gắn cờ chúng là ẩn số, rồi tiếp tục** — brief vẫn ra được, chỉ là phần "chưa xác nhận" dày hơn.
- **Bám code thật, không lý thuyết.** Blindspot pass phải trỏ tới `file:line` thật. Ẩn số suy diễn từ đọc code > ẩn số tưởng tượng.
- **Không kéo dài vô hạn.** Discover để *đủ sắc để đi tiếp*, không phải để hiểu hết mọi thứ. Khi phát biểu vấn đề đã đo được và các ẩn số lớn đã đặt tên → chốt brief, bàn giao. Đừng biến discover thành một cuộc điều tra không hồi kết.
- **Trình bày sinh động, dễ hiểu.** Brief là để user nhìn phát nắm được, không phải wall-of-text: TL;DR ngôn ngữ thường, sơ đồ làm coupling ngầm hiện ra, card ẩn số tô màu theo độ tin cậy. Dùng đúng house style bundle (`assets/discovery_brief_template.html`) — đừng chế HTML riêng.
- **Artifact ra đĩa, main giữ gọn.** Report audit của sub-agent là trung gian; kết quả sống trong `discovery_brief.html`.

## Workflow

### Bước 0 — Nhận ý định + xác định vùng + output

1. **Bắt lấy ý định thô** đúng như user nói, chưa vội diễn giải.
2. **Xác định vùng (area)**: subsystem / module / feature / luồng mà cảm giác đang trỏ tới. Nếu user không chỉ rõ → hỏi 1 câu để khoanh vùng (đừng đoán vùng rồi audit nhầm chỗ).
3. **Suy `feature`/slug** (snake_case/kebab-case) từ ý định hoặc git branch → chọn thư mục output `docs/features/<feature>/discovery_brief.html`. Chưa rõ tên → hỏi hoặc dùng slug tạm rồi xác nhận sau.

### Bước 1 — Phỏng vấn có cấu trúc (sắp theo tác động kiến trúc)

Moi ý định tiềm ẩn ra ánh sáng. **Hỏi theo thứ tự tác động — nặng & khó đảo ngược hỏi TRƯỚC**, vì trả lời sai một câu impact cao sẽ đầu độc cả brief:

1. **Impact cao** — Đau ở đâu, cụ thể (triệu chứng quan sát được, không phải suy diễn)? Điều gì *kích hoạt* cảm giác này lúc này (bug mới, khiếu nại, deadline, thay đổi upstream)? Ai/cái gì chịu ảnh hưởng?
2. **Impact vừa** — "Xong / tốt hơn" nghĩa là gì với user? Ràng buộc cứng (stack, không được đổi gì, thời hạn)? Đã thử gì chưa?
3. **Impact thấp** — Sở thích cosmetic, để sau cùng.

Gộp tối đa 3–5 câu mỗi vòng, đừng tra tấn. Dùng `AskUserQuestion` khi phương án rời rạc; open-ended khi cần mô tả. **Không đưa ý kiến giải pháp** ở bước này — giữ trung lập, đang moi vấn đề chứ chưa trả lời.

Nếu **không có người trả lời** (xem nguyên tắc "bền khi thiếu người"): rút câu trả lời khả dĩ từ chính prompt + code, viết ra dưới dạng **giả định tường minh** và đánh dấu là ẩn số cần user xác nhận.

### Bước 2 — Thang từ vựng (từ mờ → nghĩa đo được)

Ý định mơ hồ luôn đầy từ chưa đo được: "nhanh", "sạch", "tốt hơn", "gọn", "ổn định". Neo từng chữ vào nghĩa kỹ thuật cụ thể, bám code/domain thật:

- **Nêu từ mờ** — liệt kê đúng những chữ chưa đo được trong ý định.
- **Neo mỗi từ** vào một mệnh đề đo được: "nhanh" → *p95 latency endpoint X < 200ms*; "sạch" → *bỏ vòng import A↔B*; "hay hỏng khi sửa" → *sửa module M thường kéo theo lỗi ở N vì …*.
- **Xác nhận với user** (nếu có mặt) hoặc ghi thành giả định (nếu vắng).

Đây là bước biến "muốn nó tốt hơn" thành thứ mà `ff-problem-solver` có thể chẩn đoán. Không neo được vì thiếu thông tin → chính đó là một ẩn số, ghi vào bản đồ ẩn số (Bước 3).

### Bước 3 — Blindspot Pass (audit vùng, lôi unknown-unknowns)

Đây là trái tim của skill: Claude tự đọc vùng code user trỏ tới và **soát cái mà cả hai bên chưa biết mình chưa biết** — không chỉ xác nhận cái đã nghĩ. **Chọn cách theo quy mô (adaptive-by-size — cổng chống over-engineer):**

- **Vùng nhỏ / đã biết chỗ** → audit **inline** bằng Glob/Grep/Read ngay trên main.
- **Vùng lớn / chưa biết touchpoint** → **tỏa parallel, main giữ lean**:
  1. **Index rẻ trước**: có công cụ index (vd `graphify`) → chạy để có bản đồ nhanh; không thì một lượt Glob/Grep định vị.
  2. **Fan-out trong 1 message**: nhiều sub-agent `Explore`/`general-purpose` song song, mỗi con một góc (điểm vào, data model, phụ thuộc ngầm, coupling, giả định kiến trúc, test hiện có, prior art/tài liệu). Yêu cầu mỗi con trả **report gọn `file:line` + phát hiện + "cái gì bất ngờ / không như kỳ vọng"**, KHÔNG dump code.
  3. **Synthesis trên main** — main không nuốt nguyên file.

Xuất **danh sách ẩn số có cấu trúc** (cards), mỗi ẩn số kèm:
- Mô tả ẩn số / giả định chưa kiểm.
- **Độ tin cậy** (Cao / TB / Thấp).
- **Tác động nếu sai** (vì sao đáng quan tâm).
- **Cách đóng** (đọc file nào / hỏi ai / chạy thử gì).

Loại ẩn số cần chủ động săn: **phụ thuộc ngầm** (chỗ tưởng không liên quan nhưng có), **giả định kiến trúc** ngầm định, **edge case** chưa ai nói tới, **khoảng trống hiểu biết domain**, **chênh giữa cái user tin và cái code thật làm**.

**Point at a Reference (nếu user viện dẫn prior art)** — nếu ý định kiểu "làm như cách X/repo Y/thư viện Z làm" → đọc tham chiếu thật, ghi **cái map 1:1**, **khác biệt ngữ nghĩa**, và **gotchas platform-/stack-specific** không mang qua nguyên vẹn. Đừng để brief mặc định hành vi tham chiếu tự chuyển sang stack đích.

### Bước 4 — Brainstorm hướng can thiệp (ĐỂ CHỐT SCOPE, không để quyết)

Với đau đã làm rõ, phác **option space** các hướng can thiệp theo **timescale** — để user thấy dải lựa chọn và **chốt phạm vi/tham vọng**, KHÔNG phải để chọn hướng ở đây (chọn là việc của `ff-problem-solver`/`ff-planning`):

| Timescale | Kiểu can thiệp | Ví dụ |
|---|---|---|
| Ngay / trong ngày | Vá triệu chứng | workaround, guard, log thêm |
| Ngắn (ngày–tuần) | Sửa nguyên nhân cục bộ | fix logic, thêm test, refactor nhỏ |
| Trung (tuần–tháng) | Đổi cấu trúc cục bộ | tách module, đổi data model |
| Dài (quý) | Đổi kiến trúc | thay approach/công nghệ |

Đánh dấu hướng nào chạm **triệu chứng** vs **gốc rễ**. Rồi hỏi user (hoặc ghi giả định): *bạn muốn giải tới tầng nào lần này?* Câu trả lời chốt **scope** cho phát biểu vấn đề — không phải chốt giải pháp.

Nếu ý định thực chất là một feature mới (không phải "sửa đau"), thay bảng timescale bằng câu hỏi scope: *phiên bản tối thiểu là gì, cái gì để sau, cái gì dứt khoát không làm.*

### Bước 5 — Gộp discovery_brief.html (sinh động)

Tổng hợp thành brief **dễ hiểu, nhìn phát nắm được**, ghi `docs/features/<feature>/discovery_brief.html` từ template `assets/discovery_brief_template.html`. **Không viết HTML mới từ đầu — luôn dựa template.** Ngôn ngữ theo user: VI prompt → VI HTML, EN prompt → EN HTML; **không rõ → mặc định tiếng Việt**. **Viết ký tự UTF-8 THẬT** (dấu —, ngoặc " thật, xuống dòng thật) — KHÔNG nhét escape sequence (`—`, `\"`, `\n` literal) vào HTML.

Bố cục **3 tầng đọc** (khớp template) — gộp để mắt biết nhìn đâu trước, không phải 8 khối trọng số ngang nhau:

**Header** — `<h1>` chỉ tên feature (bỏ jargon); dưới là dòng `.sub` (Discovery Brief · vùng · ngày). Điền **đếm nhanh** `.glance` cho khớp nội dung thật: số ẩn số + số ưu tiên cao (thêm class `.hot` nếu có), số điểm coupling ngầm, thời gian đọc ước lượng. Sửa các `<a href>` trong `.toc` cho khớp id section thực có.

**Tầng 1 — Hiểu nhanh** (đọc 30 giây là nắm):
1. **TL;DR — vấn đề THẬT** (box `.tldr`, `#tldr`): 1–2 câu **ngôn ngữ thường, KHÔNG jargon** — cảm giác mơ hồ của user thực chất là gì. Phần quan trọng nhất cho người đọc nhanh; viết cụ thể (vd "auth mong manh" → "refresh dùng chung state trong bộ nhớ với auth, restart là mất hết").
2. **Vấn đề & phạm vi** (`#scope`) — một câu sắc "cái cần hiểu/quyết" + đau/trigger/ai ảnh hưởng, rồi tín hiệu thành công đo được, rồi card ✅ Trong phạm vi / 🚫 Không làm lần này. Kèm `.lane-note`: chưa chẩn đoán/chưa chọn giải pháp.

**Tầng 2 — Bức tranh** (muốn đào sâu thì đọc):
3. **Bản đồ vùng & coupling ngầm** (`#map`) — **sơ đồ Mermaid** (xem "Vẽ diagram" bên dưới): vẽ vùng code + phụ thuộc, coupling **ngầm/rủi ro = mũi tên nét đứt đỏ** (`linkStyle ... stroke:#DC2626`). Thêm 1–2 câu "đọc bản đồ giúp".
4. **Từ vựng đã neo** — đặt trong `<details class="fold">` (mặc định gấp): bảng từ mờ → nghĩa đo được → nguồn code. Là tra cứu, không phải nội dung đọc-liền → gấp cho brief gọn.

**Tầng 3 — Ẩn số & bước tiếp** (phần quan trọng nhất):
5. **Ẩn số (known-unknowns)** — `<section class="hero">` `#unknowns`, **SẢN PHẨM CHÍNH**, khối này viền accent nổi bật hơn hẳn. Card tô màu = **MỨC ƯU TIÊN XEM** (khớp trực giác đỏ=gấp), KHÔNG phải "độ tin cậy": `.card.pri-hi` 🔴 xem trước tiên · `.card.pri-mid` 🟡 kế tiếp · `.card.pri-lo` 🟢 để sau/phỏng đoán. Giữ khối `.legend` ở đầu để khỏi đoán màu. Mỗi card: tên ẩn số + **điều đã thấy** + **tác động nếu bỏ qua** + **cách đóng**. Kèm tag `t-del`/`t-warn`/`t-add`.
6. **Hướng can thiệp (option space)** (`#directions`) — 4 cột timescale (`.grid.cols-4`) + đánh dấu `ts-symptom`/`ts-root` + tầng scope user chốt. `.lane-note`: *chưa quyết hướng nào*. Nếu là feature mới → thay bằng Trong phạm vi / Sau này / Không làm.
7. **Bước tiếp — skill kế tiếp** (`#next`) — `ff-problem-solver` (còn cần chẩn đoán/quyết) hoặc `ff-planning` (hướng đã rõ). Nêu đường dẫn brief để chain.

**Phụ lục (tùy chọn):** Tham chiếu & gotchas — `<details class="fold">` cuối trang; **XOÁ cả `<details>` nếu không port từ prior art** (đừng để bảng rỗng).

**Vẽ diagram bản đồ vùng (`#map`) — 3 lớp tín hiệu thị giác, nhìn phát hiểu:**
1. **Màu theo vai trò** (`classDef`, `stroke-width:2px`): 🟣 tím = module/xử lý · ⚪ xám = data store · 🔴 đỏ = điểm rủi ro/ẩn số. Coupling ngầm/rủi ro = **cạnh nét đứt đỏ** (`-.->`  + `linkStyle`).
2. **Hình theo loại node**: `[/.../]` = module · `[(...)]` = store · `{{...}}` = cảnh báo/rủi ro.
3. **Icon emoji đầu label** gợi nghĩa: 🔑 auth · ⚙️ xử lý · 🗄️ DB/store · ♻️ refresh/cycle · ⚠️ rủi ro.

Quy tắc giữ diagram dễ hiểu: **≤7 node** (nhiều hơn → tách 2 diagram); label ngắn (≤4–5 từ); chọn loại đơn giản nhất đủ diễn đạt (mặc định `flowchart`). Mục đích diagram = làm coupling ngầm **hiện ra**, không trang trí.

### Bước cuối — Bàn giao + chain

- Trình tóm tắt ≤10 dòng: phát biểu vấn đề sắc, 2–3 ẩn số lớn nhất còn mở, tầng scope user chốt, skill kế.
- **Chain**: `discovery_brief.html` là đầu vào tự nhiên cho `ff-problem-solver` (dùng làm "phát biểu vấn đề" + mang theo bản đồ ẩn số để debate probe vào đó; downstream đọc HTML được, như bundle đang làm với `proposal.html`). Nếu hướng đã rõ và chỉ là "làm thế nào" → có thể thẳng `ff-planning`.

## Vệ sinh context (compact có kiểm soát)

Đừng `/compact` trần. Sau khi gộp brief ra đĩa, đề xuất block compact điền path thật, rõ GIỮ/BỎ:

```
/compact GIỮ: nội dung docs/features/<feature>/discovery_brief.html (phát biểu sắc, bản đồ ẩn số,
hướng + tầng scope đã chốt, skill kế). BỎ: report audit của sub-agent, transcript phỏng vấn từng vòng
(đã tổng hợp vào brief).
```

Ẩn số/quyết định scope chưa kịp vào brief → gắn cờ must-keep.

## Anti-patterns

- ❌ Vượt ranh giới: chốt nguyên nhân hoặc chọn giải pháp trong discover (đó là việc của `ff-problem-solver`/`ff-planning`).
- ❌ Bỏ Blindspot Pass, chỉ hỏi user rồi viết brief → mất đúng phần "unknown-unknowns" mà skill sinh ra để lôi.
- ❌ Audit chỉ để xác nhận cái đã nghĩ, không săn cái bất ngờ → blindspot pass thành echo chamber.
- ❌ Debate/gọi Codex ở stage này khi chưa có giả thuyết để phản biện (over-engineer; để dành cho downstream).
- ❌ Treo vì user không trả lời, thay vì nêu giả định tường minh + gắn cờ ẩn số rồi tiếp tục.
- ❌ Neo từ vựng qua loa ("nhanh = nhanh hơn") — phải ra mệnh đề đo được hoặc thừa nhận là ẩn số.
- ❌ Điều tra vô tận: cố hiểu hết mọi thứ thay vì "đủ sắc để đi tiếp".
- ❌ Đoán vùng code rồi audit nhầm chỗ thay vì hỏi 1 câu khoanh vùng.
- ❌ Nuốt nguyên file vào main khi vùng lớn (nên tỏa sub-agent trả report gọn); hoặc tỏa sub-agent cho vùng nhỏ đã biết chỗ.
- ❌ TL;DR đầy jargon — mất luôn tác dụng "đọc 10 giây hiểu". Viết ngôn ngữ thường.
- ❌ Diagram >7 node / vẽ cho có — làm rối thay vì làm coupling ngầm hiện ra. Tách nhỏ, hoặc bỏ nếu không thêm hiểu biết.
- ❌ Viết HTML mới từ đầu thay vì dựa `assets/discovery_brief_template.html` (lệch house style bundle).

## Đọc thêm

- `assets/discovery_brief_template.html` — template HTML cho `discovery_brief.html` (TL;DR box, card ẩn số theo màu, Mermaid, cột timescale).
