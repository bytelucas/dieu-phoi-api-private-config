# Issue tracker: GitLab tự host (qua REST API)

Issue và PRD của repo này nằm ở **GitLab tự host** `git-ttcpdt.mbfs.vn`, project
`dieu-phoi-giai-quyet-tthc/dieu-phoi-api`, **project id `64`**.

> ## ⚠️ KHÔNG dùng `glab` ở repo này
>
> `glab` **có cài** trên máy (v1.93.0) nhưng **chưa từng đăng ký host này** — nó chỉ biết `gitlab.com`
> (không token) và `gitlab.itel.vn` (dự án khác). Chạy trong repo sẽ fail ngay:
>
> ```
> ERROR  None of the git remotes configured for this repository point to a known GitLab host.
>        Configured remotes: git-ttcpdt.mbfs.vn
> ```
>
> Đường **đang chạy thật** là REST API + token lấy từ **Git Credential Manager**. Mọi issue của repo
> (#52–#60) đều tạo bằng cách này.
>
> Muốn bật `glab` về sau thì chạy một lần:
> `glab auth login --hostname git-ttcpdt.mbfs.vn` (cần Personal Access Token, scope `api`).
> Bật xong thì mọi lệnh trong template gốc của skill dùng lại được.

## Lấy token

Token nằm trong GCM (`credential.helper = git-credential-manager`, provider `generic`, user `hieuna`).
Lấy ra bằng chính git, **không hard-code, không echo ra output**:

```bash
TOKEN=$(printf 'protocol=https\nhost=git-ttcpdt.mbfs.vn\n\n' | git credential fill | sed -n 's/^password=//p')
API="https://git-ttcpdt.mbfs.vn/api/v4/projects/64"
```

Kiểm tra kết nối/token nhanh: `curl -s -o /dev/null -w '%{http_code}\n' "$API"` — `200` là ổn,
`401` là token hỏng, `000` là **VPN rớt** (host chỉ tới được qua VPN công ty).

## Quy ước thao tác

| Việc | Lệnh |
|---|---|
| **Tạo issue** | `POST $API/issues` · body JSON `{title, description, labels}` (labels là chuỗi phân tách bởi dấu phẩy) |
| **Đọc issue** | `GET $API/issues/<iid>` · comment: `GET $API/issues/<iid>/notes` |
| **Liệt kê** | `GET $API/issues?state=opened&per_page=50` · lọc nhãn: `&labels=ready-for-dev` |
| **Comment** | `POST $API/issues/<iid>/notes` · body `{body}` (GitLab gọi comment là **note**) |
| **Đổi nhãn** | `PUT $API/issues/<iid>` · `{labels: "a,b"}` (ghi đè cả tập) hoặc `{add_labels}` / `{remove_labels}` |
| **Đóng** | `PUT $API/issues/<iid>` · `{state_event: "close"}` |
| **Merge request** | `$API/merge_requests` — GitLab gọi PR là **merge request**, và **đánh số riêng** với issue (`!315` ≠ `#315`) |

**Đóng issue thì comment TRƯỚC rồi mới đóng** — hai lệnh riêng, và lý do phải nằm lại trên issue.

Body dài thì dựng payload bằng `python3` + `json.dumps` thay vì nhét thẳng vào chuỗi shell — mô tả
issue hay có backtick, `$`, xuống dòng, dễ vỡ quoting.

## Nhãn

Xem `docs/agents/triage-labels.md`. ⚠️ **Không bao giờ đặt nhãn chứa chữ `agent`** — tổ chức hạn chế AI.

## MR như một bề mặt tiếp nhận yêu cầu

**MRs as a request surface: no.** *(Đổi thành `yes` nếu repo coi MR từ ngoài là feature request; `/triage`
đọc cờ này.)*

## Khi skill nói "publish to the issue tracker"

Tạo một GitLab issue theo bảng trên.

## Khi skill nói "fetch the relevant ticket"

`GET $API/issues/<iid>` + `GET $API/issues/<iid>/notes`.

## Thao tác cho `/wayfinder`

**Map** là một issue, **ticket** là các issue con.

- **Map**: issue gắn nhãn `wayfinder:map`, thân chứa Notes / Decisions-so-far / Fog.
- **Ticket con**: issue có dòng `Part of #<map>` ở đầu mô tả, nhãn `wayfinder:<type>`
  (`research`/`prototype`/`grilling`/`task`). Nhận việc thì assign cho người làm.
- **Blocking**: quick action `/blocked_by #<n>` gửi dưới dạng **note**
  (`POST $API/issues/<child>/notes` với `{body: "/blocked_by #<blocker>"}`). ⚠️ Native blocking link là
  tính năng **Premium/Ultimate** — **chưa xác minh tier của instance này**. Nếu không có, fallback là dòng
  `Blocked by: #<n>, #<n>` ở đầu mô tả. Ticket hết bị chặn khi **mọi** blocker đã đóng.
- **Frontier query**: `GET $API/issues?state=opened`, lọc theo con của map, bỏ cái nào còn blocker mở
  (`GET $API/issues/<iid>/links`, hoặc đọc dòng `Blocked by`) hoặc đã có assignee; thứ tự trong map thắng.
- **Claim**: `PUT $API/issues/<iid>` với `{assignee_id: <id>}` — thao tác ghi đầu tiên của phiên.
- **Resolve**: post note câu trả lời → đóng issue → nối một dòng trỏ ngược vào Decisions-so-far của map.
