Post 2 — Roles vs permissions (with code)

---

📋 Roles → Who you are in the org

A role is a label from the backend — what the JWT carries after login.

enum UserRoles {

administrator('Administrator'),

unitRepresentative('UnitRepresentative'),

readOnly('ReadOnly'),

soldiersAffairs('SoldiersAffairs'),

financialAffairs('FinancialAffairs');

// ...

final String value;

const UserRoles(this.value);

}

Roles describe identity in the system, not every button you can press.

---

🔑 Permissions → What you can actually do

Permissions are finer-grained: view soldiers, manage financials, open the archive.

enum AppPermission {

adminOnly,

unitRepresentativeOnly,

canViewSoldiers,

canManageSoldiers,

canViewFinancial,

canManageFinancial,

// ...

}

The app never asks: _“Are you Soldiers Affairs?”_  
It asks: _“Do you have `canViewSoldiers`?”_

bool can(AppPermission permission) =>

permissions.contains(permission);

---

🗺️ One map. One place to change access.

Every role → set of permissions lives in a single const map:

/// Single source of truth for RBAC.

const Map<UserRoles, Set<AppPermission>> rolePermissions = {

UserRoles.readOnly: {

AppPermission.canViewSoldiers,

AppPermission.canViewNCO,

AppPermission.canViewSoldierReport,

AppPermission.canViewNCOReport,

AppPermission.canViewVacationReports,

AppPermission.canViewFaxes,

},

UserRoles.soldiersAffairs: {

AppPermission.canViewSoldiers,

AppPermission.canManageSoldiers,

AppPermission.canViewNCO,

AppPermission.canViewSoldierReport,

AppPermission.canViewFaxes,

},

UserRoles.unitRepresentative: {

AppPermission.unitRepresentativeOnly,

},

};

No scattered `if (role == …)` across screens.  
Change the matrix once — routes, drawer, and gates all follow.

---

🔀 Multiple roles? Merge them.

One user can hold more than one role.  
`PermissionMapper` unions every permission set into one `Set`:

static Set<AppPermission> mergeRolePermissions(

Iterable<UserRoles> roles, {

Map<UserRoles, Set<AppPermission>> map = rolePermissions,

}) {

final merged = <AppPermission>{};

for (final role in roles) {

merged.addAll(map[role] ?? const {});

}

return merged;

}

On login / refresh, roles become the live session:

final perms = _permissionMapper.permissionsForRoles(data.roles);

emit(SessionState(

isAuthenticated: true,

permissions: Set<AppPermission>.unmodifiable(perms),

roles: List<UserRoles>.unmodifiable(data.roles),

));

More roles → more capabilities.  
Never fewer than what any single role grants.

---

✅ Why this split matters

- Roles change when HR or the API changes job titles.
- Permissions change when a feature needs a new guard.
- UI & routing only ever check permissions — stable, testable, explicit.

Roles are the keys on the keyring.  
Permissions are which doors each key opens.

#Flutter #Security #MobileDev #CleanArchitecture #RBAC





