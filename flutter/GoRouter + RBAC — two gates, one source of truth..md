

Last Post: JWT + RBAC inside the app — the full session pipeline. (https://surl.li/ycmwao)
Today: How routing becomes a security layer — not just navigation.

GoRouter + RBAC — two gates, one source of truth.

Most apps protect the UI.
Few protect the URL.
This architecture does both — from the same permission matrix.

1️⃣ The permission table — one file rules all routes

   routePermissionMap ties every path to a List<AppPermission>:

      RoutePaths.soldiersView → [canViewSoldiers, unitRepresentativeOnly]
      RoutePaths.vacationsView → [canManageVacations, unitRepresentativeOnly]
      RoutePaths.ncosReportsView → [canViewNCOReport, unitRepresentativeOnly]

   One change here. Every gate updates automatically.
   No scattered if-checks. No duplicated role logic.

2️⃣ GoRouter redirect — the URL can't lie

   Every navigation passes through _redirect():

   - Not authenticated → /login
   - Authenticated + hits / or /login → firstAuthorizedBranch()
   - Deep link to a guarded route → permissionForMatchedLocation()
     * Has permission → let through
     * No permission → /no-access

   Deep links are fully covered.
   A user can't paste /soldiers into the browser and bypass the gate.

3️⃣ Nested routes inherit — not duplicate

   routeNestedInheritPermissionRules handles detail screens:

      /soldiers/:mid → inherits canViewSoldiers
      /ncos/:mid    → inherits canViewNCO

   Parent permission = child permission.
   No need to re-declare guards on every nested route.

4️⃣ firstAuthorizedBranch() — smart post-login redirect

   After login, the app doesn't dump everyone on the same screen.
   It walks orderedBranchRoutes and opens the first branch the user can access.

      soldiersAffairs role → lands on /soldiers
      vacationsAffairs role → lands on /vacations
      readOnly role → lands on /soldiers-reports

   No hardcoded home screen. Roles decide the entry point.

5️⃣ GoRouterRefresh — session drives routing

   class GoRouterRefresh extends ChangeNotifier {
     void notify() => notifyListeners();
   }

   SessionCubit calls .notify() on login and logout.
   GoRouter re-runs redirect immediately.
   No manual navigation calls. No stale screens.

6️⃣ PermissionGate — widget-level enforcement

   Routes block unauthorized navigation.
   PermissionGate hides what unauthorized users must not even see:

      PermissionGate(
        requirement: PermissionRequirement.permission(
          AppPermission.canManageSoldiers,
        ),
        child: SaveButton(),
      )

   Same AppPermission enum. Same session. Same matrix.
   UI can't show a button the route would block anyway.

The architecture in 6 steps:

🗺️ routePermissionMap → single source of truth for all route guards
🔀 GoRouter redirect → URL-level enforcement on every navigation
🧬 Nested inheritance → detail routes covered without duplication
🎯 firstAuthorizedBranch() → role-aware post-login landing
🔔 GoRouterRefresh → session changes trigger instant re-evaluation
🎨 PermissionGate → widget layer mirrors route layer exactly

Two gates. One matrix. Zero gaps.

#Flutter #Security #MobileDev #CleanArchitecture #RBAC #JWT #GoRouter