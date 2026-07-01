# Implementation Plan — template

Ghi ra `docs/features/<feature>/implementation_plan.md`. Giữ heading `#`/`##` để `/codex-plan-review` bám cấu trúc.

**Dependency contract** = các cột `Owns files` + `Depends on` + `Song song?` ở bảng "Các bước hiện thực". Đây là hợp đồng để `plan-implementer` parallelize an toàn. Khai **honest**: chỉ đánh song song khi file disjoint VÀ không phụ thuộc; nghi thì để tuần tự.

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

## Các bước hiện thực
> `Owns files` = file bước này sở hữu (được sửa). `Depends on` = bước phải xong trước.
> Song song an toàn ⟺ Owns files disjoint VÀ không phụ thuộc lẫn nhau.

| # | Bước | Owns files | Depends on | Song song? |
|---|---|---|---|---|
| 0 | <contract-first: khóa type/signature/stub chung nếu có seam> | `path/shared.py` (types, stubs) | — | — |
| 1 | <làm gì cụ thể> | `path/a.py` | 0 | với 2 |
| 2 | <...> | `path/b.py` | 0 | với 1 |
| 3 | <bước gom/integration> | `path/wire.py` | 1, 2 | — |

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

Nếu nhiều bước cần chung một seam (kiểu dữ liệu, chữ ký hàm, interface, stub), thêm **Bước 0** khóa seam đó trước. Khi ấy các bước sau điền vào contract cố định → `plan-implementer` cho chạy song song an toàn hơn (mỗi bước sở hữu file riêng, không giẫm lên seam chung).
