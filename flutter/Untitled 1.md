Today: what happens **inside the app** after login — step by step.

---

**The pipeline (Flutter + JWT + RBAC)**

**1️⃣ Authentication — JWT in**

User logs in → backend returns a JWT.

We decode the payload once and read:
- `roles` (who they are)
- `exp` (is the token still valid?)

`AccessTokenService` maps role strings → `UserRoles` enum.

That’s authentication: **identity is proven once at the door.**

---

**2️⃣ Roles → permissions — merge once**

JWT says *roles*. The app needs *capabilities*.

`rolePermissions` is the single matrix:

`UserRoles.soldiersAffairs` → `{ canViewSoldiers, canManageSoldiers, … }`

`PermissionMapper` merges all roles into one `Set<AppPermission>`.  // If the user has multiple roles 

Example: two roles → union of both permission sets.

`SessionCubit.hydrate()` builds:

```
SessionState {
  isAuthenticated: true,
  roles: [...],
  permissions: { ... }  // frozen for this session
}
```

Authorization is **computed once per session**, not re-decoded on every tap.

---

**3️⃣ SessionState — the source of truth**

UI and routing never read the JWT again.

They ask one question:

`session.can(AppPermission.canManageVacations)`  
or  
`session.canAny([...])`

Clean, testable, no stringly-typed role checks scattered in widgets.

---

**4️⃣ Two gates — same rules, two layers**

**UI — `PermissionGate`**

```dart
PermissionGate(
  requirement: PermissionRequirement.permission(
    AppPermission.canManageVacations,
  ),
  child: SaveButton(),
)
```

Hide buttons, filters, sidebars the user must not see.

**Routes — GoRouter `redirect`**

- Not logged in → login screen  
- Deep link to `/vacations` without permission → no-access screen  
- After login → first branch they’re allowed to open  

`routePermissionMap` ties each path to the same `AppPermission` list.

Same matrix. Same session. **UI can’t lie; URLs can’t bypass.**

---

**Why this shape?**

| Step                    | Responsibility                     |
| ----------------------- | ---------------------------------- |
| JWT decode              | Authentication (who)               |
| Role → permission merge | Authorization policy (what)        |
| `SessionState`          | Runtime snapshot                   |
| `PermissionGate`        | Widget-level enforcement           |
| GoRouter redirect       | Navigation & deep-link enforcement |

Authentication is the **front door**.  
Authorization is **every door inside the building** — and we use the same keyring everywhere.

#Flutter #Security #MobileDev #CleanArchitecture #RBAC #JWT #GoRouter