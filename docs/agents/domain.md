# Domain Docs

Các skill engineering nên đọc tài liệu domain của repo này thế nào khi khảo sát codebase.

## Đọc trước khi khảo sát

- **`CONTEXT-MAP.md`** ở root — repo này là **multi-context**. File chứa glossary (*ubiquitous language*),
  danh sách **8 bounded context**, quan hệ giữa chúng, và mục *Flagged ambiguities*.
- **`docs/adr/`** — đọc các ADR chạm tới vùng sắp làm. Hiện có **34 ADR** (0001–0033).

Nếu file nào không tồn tại thì **đi tiếp trong im lặng** — đừng nêu ra, đừng đề nghị tạo trước. Skill
`/domain-modeling` (đi vào qua `/grill-with-docs` và `/improve-codebase-architecture`) tạo chúng **lười**,
đúng lúc một thuật ngữ hay một quyết định thực sự được chốt.

## Cấu trúc file — LỆCH khỏi template gốc, đọc kỹ

```
/
├── CONTEXT-MAP.md          ← glossary + 8 bounded context     [GITIGNORE]
├── docs/
│   ├── adr/                ← quyết định toàn hệ                [TRACK ✅]
│   └── agents/             ← cấu hình skill                    [GITIGNORE]
├── apps/                   ← MỖI APP LÀ MỘT BOUNDED CONTEXT
│   ├── auth/  configuring/  organizing/  processing/
│   ├── coordinating/  gateway/  wrapper/  file-handling/
└── libs/                   ← core · utils · nats · grpc (dùng chung, không phải context)
```

Hai chỗ khác template của skill:

1. **Context nằm ở `apps/<app>/`, không phải `src/<context>/`.** Đây là NestJS monorepo. Nếu sau này sinh
   `CONTEXT.md` riêng cho từng context thì đặt ở `apps/<app>/CONTEXT.md`.
2. **Không có `pnpm-workspace.yaml`, không có `packages/`** — nên bộ dò monorepo mặc định của skill sẽ
   kết luận nhầm là *single-context*. **Không phải.** Mỗi app là một service độc lập, **DB riêng**
   (`docs/adr/0001` database-per-service), deploy VM riêng, và **cấm import chéo giữa các app**.

## ⚠️ Cái gì được đẩy lên, cái gì không

**Luật công ty: mọi file agent-facing là LOCAL, không commit.** `CLAUDE.md`, `CONTEXT-MAP.md`,
`docs/agents/`, `.tmp/`, `.claude/` — tất cả đều gitignore.

**Ngoại lệ duy nhất được track: `docs/adr/`.**

⇒ **Hệ quả cho mọi skill sinh tài liệu:** thứ gì đồng đội cần đọc thì phải nằm **trong ADR**. Đừng viết nó
vào `CONTEXT-MAP.md` rồi trỏ sang — người đọc trên máy khác **không có file đó**.

Tiền lệ đã áp: `docs/adr/0033` có mục **"Ngôn ngữ dùng trong ADR này"** chép lại bốn thuật ngữ từ glossary,
kèm giải thích vì sao phải chép.

## Dùng đúng vốn từ của glossary

Khi output gọi tên một khái niệm domain (tiêu đề issue, đề xuất refactor, giả thuyết, tên test), dùng
**đúng thuật ngữ trong `CONTEXT-MAP.md`**, đừng trôi sang từ đồng nghĩa mà glossary ghi rõ là *tránh*.

Khái niệm cần dùng mà chưa có trong glossary là **một tín hiệu**: hoặc đang bịa ra ngôn ngữ dự án không
dùng (nên nghĩ lại), hoặc có khoảng trống thật (ghi lại cho `/domain-modeling`).

⚠️ **Biết trước một vênh đã ghi nhận:** glossary quy định **Flow / Officer / Unit**, nhưng code dùng
`Workflow` / `Employee` / `Department` ở khắp nơi (thừa hưởng từ tên nguồn CSDL lúc sync). Xem mục
*Flagged ambiguities* trong `CONTEXT-MAP.md` — chưa chốt bên nào, và **đừng trộn hai phương ngữ trong cùng
một service**.

## Nêu rõ khi mâu thuẫn với ADR

Nếu output mâu thuẫn với một ADR đang có, **nói thẳng ra** thay vì lặng lẽ ghi đè:

> *Mâu thuẫn với ADR-0026 (ABAC scope policies) — nhưng đáng mở lại vì…*

Tiền lệ: `docs/adr/0033` **thay §3 của ADR 0025** và ghi rõ ngay ở dòng Status.
