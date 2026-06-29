# Implementation Plan — template

Ghi ra `docs/features/<feature>/implementation_plan.md`. Giữ heading `#`/`##` để `/codex-plan-review` bám cấu trúc.

```markdown
# Implementation Plan — <tên feature/vấn đề>

## Mục tiêu / Outcomes
<Cái gì coi là xong. Đây là nguồn để derive acceptance criteria — viết đo được.>
- [ ] <outcome 1>
- [ ] <outcome 2>

## Bối cảnh & quyết định
<Tóm tắt quyết định đã chốt + lý do. Link decision record nếu có: docs/decisions/...>

## Vùng tác động
| File | Vai trò trong thay đổi này |
|---|---|
| `path/to/file.py:120` | <sửa gì> |
| `path/to/other.py` | <thêm gì> |

## Các bước hiện thực
| # | Bước | File | Phụ thuộc | Song song? |
|---|---|---|---|---|
| 1 | <làm gì cụ thể> | `...` | — | — |
| 2 | <...> | `...` | 1 | với 3 |
| 3 | <...> | `...` | 1 | với 2 |

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
