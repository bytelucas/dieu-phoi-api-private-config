# Triage Labels

Các skill nói theo **năm vai triage chuẩn**. Bảng này ánh xạ chúng sang **chuỗi nhãn thật** dùng trên
GitLab của repo này.

| Nhãn trong mattpocock/skills | Nhãn trong tracker của mình | Nghĩa |
|---|---|---|
| `needs-triage` | `needs-triage` | Cần người đánh giá issue này |
| `needs-info` | `needs-info` | Đang chờ người báo cáo bổ sung thông tin |
| `ready-for-agent` | **`ready-for-dev`** | Đã đặc tả đủ, sẵn sàng cho người/agent thi công |
| `ready-for-human` | **`ready-for-review`** | Đã làm xong, chờ review |
| `wontfix` | `wontfix` | Sẽ không xử lý |

Skill nhắc tới vai nào thì dùng **chuỗi ở cột giữa**.

## ⚠️ Vì sao hai dòng bị đổi tên — đừng đổi ngược lại

**`ready-for-agent` → `ready-for-dev`: bắt buộc.** Tổ chức **hạn chế dùng AI**, nên **không đặt nhãn chứa
chữ `agent`** ở bất kỳ đâu trên tracker. Đây là ràng buộc từ bên ngoài, không phải sở thích.

**`ready-for-human` → `ready-for-review`:** repo đã dùng `ready-for-review` từ trước cho đúng nghĩa đó
(làm xong, chờ review). Thêm nhãn mới sẽ đẻ ra hai nhãn cùng chức năng.

## Trạng thái nhãn trên GitLab (18/08/2026)

| Nhãn | Đã tồn tại? |
|---|---|
| `ready-for-dev` | ✅ đang dùng |
| `ready-for-review` | ✅ đang dùng |
| `backend` | ✅ đang dùng (nhãn phân loại, không thuộc triage) |
| `needs-triage` · `needs-info` · `wontfix` | ❌ **chưa tạo** — tạo lúc `/triage` cần tới |

Tạo nhãn: `POST $API/labels` với `{name, color}` (xem `docs/agents/issue-tracker.md` để lấy `$API`/token).
