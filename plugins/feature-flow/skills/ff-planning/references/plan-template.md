# Implementation Plan — template

Ghi ra `docs/features/<feature>/implementation_plan.md`. Giữ heading `#`/`##` để `/codex-plan-review` bám cấu trúc.

**Dependency contract** = các cột `Owns files` + `Depends on` + `Song song?` ở bảng "Các bước hiện thực". Đây là hợp đồng để `ff-implement` parallelize an toàn. Khai **honest**: chỉ đánh song song khi file disjoint VÀ không phụ thuộc; nghi thì để tuần tự.

```markdown
# Implementation Plan — <tên feature/vấn đề>

## Mục tiêu / Outcomes
<Cái gì coi là xong. Đây là nguồn để derive acceptance criteria — viết đo được.>
- [ ] <outcome 1>
- [ ] <outcome 2>

## Bối cảnh & quyết định
<Tóm tắt quyết định đã chốt + lý do. Link brief/decision record nếu có:
docs/features/<feature>/analysis_brief.md · docs/decisions/...>

## Vùng tác động
| File | Vai trò trong thay đổi này |
|---|---|
| `path/to/file.py:120` | <sửa gì> |
| `path/to/other.py` | <thêm gì> |

## Tham chiếu & gotchas
<Chỉ khi hiện thực port/phỏng theo một tham chiếu (ngôn ngữ/thư viện/repo/prior art khác). Bỏ mục này nếu không có tham chiếu.>
| Khía cạnh | Tham chiếu | Stack đích | Gotcha / khác biệt |
|---|---|---|---|
| <hành vi/API> | <đoạn khớp / cách tham chiếu làm> | <cách stack này làm> | <chỗ KHÔNG chuyển 1:1: concurrency, bộ nhớ, timezone/locale, lỗi ngầm, thứ tự init...> |

## Các bước hiện thực
> `Owns files` = file bước này sở hữu (được sửa). `Depends on` = bước phải xong trước.
> Song song an toàn ⟺ Owns files disjoint VÀ không phụ thuộc lẫn nhau.
> `Độ bất định` = cao/trung/thấp — bước dễ phải sửa lại nhất (kiến trúc, schema, giả định chưa kiểm chứng) để **cao**. Cờ này chỉ xếp **ưu tiên chú ý/review** (tweakable-first), KHÔNG đổi thứ tự chạy — thứ tự chạy vẫn theo `Depends on`. Nêu bước bất định cao lên đầu / cho nổi bật.

| # | Bước | Owns files | Depends on | Song song? | Độ bất định |
|---|---|---|---|---|---|
| 0 | <contract-first: khóa type/signature/stub chung nếu có seam> | `path/shared.py` (types, stubs) | — | — | cao |
| 1 | <làm gì cụ thể> | `path/a.py` | 0 | với 2 | trung |
| 2 | <...> | `path/b.py` | 0 | với 1 | thấp |
| 3 | <bước gom/integration> | `path/wire.py` | 1, 2 | — | thấp |

## Test & verify
| Phần | Cách kiểm chứng | Tiêu chí pass |
|---|---|---|
| <bước/feature> | <unit/integration/manual> | <điều kiện> |

## Rủi ro & rollback
- **Rủi ro**: <điều dễ vỡ> → **Giảm thiểu**: <cách>
- **Rollback**: <cách lùi nếu hỏng>

## Ngoài phạm vi
- <chủ đích không làm lần này>
```

## Ghi chú về Phase 0 contract-first (cho plan lớn)

Nếu nhiều bước cần chung một seam (kiểu dữ liệu, chữ ký hàm, interface, stub), thêm **Bước 0** khóa seam đó trước. Khi ấy các bước sau điền vào contract cố định → `ff-implement` cho chạy song song an toàn hơn (mỗi bước sở hữu file riêng, không giẫm lên seam chung).
