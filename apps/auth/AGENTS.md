# auth — internal officer authentication (SSO thuần)

**Bounded context:** authenticates **internal officers only**. Từ ADR 0028 auth là SSO
thuần **không DB**: login/refresh/logout + JWT + Redis session. Toàn bộ authz (catalog quyền,
nhóm quyền, gán người, SYSTEM_ACCOUNT) đã move sang `organizing` (Internal IAM).
Port 8000, prefix `/auth/api/v1`. Read the root `AGENTS.md` for shared conventions.

## Owns

- Phiên đăng nhập officer: JWT sign/verify, Redis session (ghi lúc login), refresh token.
- `sso/{login,refresh-token,logout}` + `me/permissions`.

## Trạng thái chuyển tiếp (TTHCDC-329)

- Issue #33 (DONE): 13 endpoint authz + shared/authz + EffectivePermissionsService đã có bản
  sống ở `organizing`.
- Issue #34 (DONE, commit `05d18bc`): auth đã KHÔNG-DB — xóa DatabaseModule/database/oracle/,
  3 module admin, shared/authz; login gọi `organizing.officer.get-login-context.v1` fail-closed
  (`LOGIN_CONTEXT_UNAVAILABLE` 503; citizenId lạ vẫn `EMPLOYEE_NOT_FOUND`) + configuring
  group-context best-effort; `/me/permissions` đã là POST. DB vật lý `DPGQ_TTHC_AUTH` còn đó
  làm đường lùi — dọn (kèm config `database.auth` trong libs/core) ở MR riêng sau land.

## Boundaries (important)

- **Citizens do NOT authenticate here.** Citizen identity is external (VNeID/DVCQG) and
  enters via SSO through `wrapper` (`docs/adr/0002`). Do not add citizen login/accounts.
- Dữ liệu authz + org placement của officer là của `organizing` (ADR 0028) — auth chỉ đọc
  qua NATS lúc login và cache vào session.

## Don'ts

- ❌ No citizen accounts.
- ❌ Don't model Units/org hierarchy here — that's `organizing`.
- ❌ Đừng thêm bảng/entity authz mới vào auth — authz data là của `organizing` (ADR 0028).
