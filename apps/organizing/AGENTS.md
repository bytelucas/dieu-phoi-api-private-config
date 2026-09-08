# organizing — internal org structure + Internal IAM

**Bounded context:** the internal organization — **Units** and **Officers** — VÀ từ ADR 0028
là **Internal IAM**: toàn bộ authz (catalog quyền, nhóm quyền, tick, gán người, SYSTEM_ACCOUNT).
Port 8003, prefix `/organizing/api/v1`. Owns its own DB. Read the root `AGENTS.md` for
shared conventions.

## Owns

- **Unit** (Đơn vị / Ban ngành) — an organizational body that owns Steps and Officers.
- **Officer** (Cán bộ) — an internal person who processes Steps; belongs to a Unit.
- **Authz (Internal IAM, ADR 0028):** `PERMISSIONS` / `ROLE_GROUPS` / `ROLE_GROUP_PERMISSIONS` /
  `EMPLOYEE_ROLE_GROUPS` / `SYSTEM_ACCOUNT` + 3 module admin (13 endpoint:
  `permission-catalog/*`, `role-groups/*`, `employee-role-groups/*` — nhóm employee-role-groups
  là TẠM, flag `authz.employeeAssignmentEnabled`) + kit `shared/authz/`
  (`PermissionTree`, `deriveMenuCodes`, `SubsetRule`, `EffectivePermissionsService`,
  `RolePermissionTablePublisher`, `SessionRoleGroupsService`).
- **Bảng quyền-theo-nhóm (`authz:role-permissions`, ADR 0033)** — hash Redis dùng chung do
  organizing SỞ HỮU, là đường ĐỌC quyền của mọi request ở mọi app. **Mọi mutation phân quyền** (đổi
  tick nhóm, đổi state/trần nhóm, hạ trần nhóm tối thiểu, CRUD catalog) PHẢI gọi
  `RolePermissionTablePublisher.rebuildAfterCommit()` **sau khi transaction commit** — đó là đường
  duy nhất; quên là thao tác admin không có hiệu lực. Dựng lại **toàn bộ** và idempotent, O(1) theo
  số cán bộ, **không chạm phiên của ai**.
- **Menu-level RBAC (`menuCodes`)** — mảng CODE phẳng node cấp 0–2, **không có endpoint**: nằm trên
  chính bảng đó (field `menu:<roleGroupId>` + hằng `menu:__ALL__` cho nhánh bypass). FE nhận qua
  `/me`. Sửa cấu trúc catalog (create/update-move/delete/restore node cấp 0–2) đổi mục menu của
  **mọi** nhóm nên cũng đi qua `rebuildAfterCommit()`, không có đường riêng.
- **Responder login-context:** `organizing.officer.get-login-context.v1` (module `officer`,
  NATS-only) trả employee + permissions + roleGroupCodes + menuCodes + orgContext + isSystemAccount
  trong MỘT lượt cho `auth` lúc login (design doc authz-organizing-migration §5).
- **Con trỏ nhóm trong phiên:** organizing own field `roleGroupIds` (và `officerContext`) trong
  Redis session của auth, ghi qua seam `AuthCachingService` (`authSessionSource: 'auth-service'`
  trong app.module) — `SessionRoleGroupsService.refreshRoleGroupIds` cho **đúng một** cán bộ khi
  gán/gỡ nhóm hoặc xoá cán bộ. Ranh giới ADR 0033: **phiên = *anh là ai*, bảng = *anh được làm gì***
  ⇒ thao tác trên trục quyền KHÔNG bao giờ chạm phiên. Đường "tính lại quyền rồi ghi đè phiên hàng
  loạt" đã bị xoá ở #69 (vỡ ở 1.000 và ~10.000 cán bộ, cả hai fail-open) — **đừng dựng lại**.

## Boundaries (important)

- This context is **internal-only**. **Citizens do not belong here** (`docs/adr/0002`) —
  a Citizen is an external applicant, not part of the org structure. Never add a citizen
  table or treat citizens as a kind of Officer.
- `auth` chỉ còn authentication/SSO thuần (JWT + session, không DB — ADR 0028); mọi dữ liệu
  và API authz nằm ở đây. Đừng thêm ngược logic sign/verify token vào organizing.
- `processing` assigns Steps to Units/Officers by referencing ids owned here.

## Don'ts

- ❌ No Citizen data.
- ❌ Don't sign/verify JWT hay ghi session lúc login — that's `auth` (organizing chỉ ghi
  `roleGroupIds`/`officerContext` cho một cán bộ, qua seam AuthCachingService).
- ❌ Đừng thêm đường ghi quyền vào phiên, kể cả "chỉ một người cho nhanh" — quyền chỉ có một nguồn
  là bảng `authz:role-permissions` (ADR 0033).
