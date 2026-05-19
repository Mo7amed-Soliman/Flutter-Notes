Last Post: GoRouter + RBAC — two gates, one permission matrix. (https://surl.li/ycmwao)
Today: What actually lives inside the session — and how it gets built.

SessionCubit + SessionState — the runtime source of truth.

The JWT is decoded once.
The permissions are merged once.
Everything else just reads SessionState.

1️⃣ SessionState — an immutable snapshot

   @immutable
   class SessionState {
     final bool isAuthenticated;
     final Set<AppPermission> permissions; // unmodifiable
     final List<UserRoles> roles;          // unmodifiable
   }

   Not a stream. Not a live token reader.
   A frozen snapshot computed once per session lifecycle.

   - permissions is an unmodifiable Set — nothing mutates it mid-session.
   - roles is an unmodifiable List — role checks are explicit, not ambient.
   - @immutable → Bloc equality works correctly, BlocSelector rebuilds only when state truly changes.

2️⃣ SessionState.anonymous — the safe default

   static const SessionState anonymous = SessionState(
     isAuthenticated: false,
     permissions: {},
     roles: [],
   );

   App starts here. Logout returns here.
   No null checks. No nullable session.
   The cubit is always in a valid state — even before login.

3️⃣ hydrate() — the only place session is built

   Future<void> hydrate() async {
     final data = _accessTokenService.cachedData;
     if (data == null || data.isExpired) {
       emit(SessionState.anonymous);
       _routerRefresh.notify();
       return;
     }
     final perms = _permissionMapper.permissionsForRoles(data.roles);
     emit(SessionState(
       isAuthenticated: true,
       permissions: Set<AppPermission>.unmodifiable(perms),
       roles: List<UserRoles>.unmodifiable(data.roles),
     ));
     _routerRefresh.notify();
   }

   Called in 3 moments:
   ✅ After login — fresh token just stored
   ✅ After silent token refresh — new token, same session feel
   ✅ App cold start — restore session from cached token

   Token expired? → anonymous. Token valid? → full session.
   One code path. No branching across the codebase.

4️⃣ clear() — logout in 2 lines

   void clear() {
     emit(SessionState.anonymous);
     _routerRefresh.notify();
   }

   No token deletion here — that's AccessTokenService's job.
   SessionCubit only owns session state.
   Single responsibility, cleanly enforced.

5️⃣ The query API — what the rest of the app calls

   session.can(AppPermission.canManageSoldiers)
   session.canAny([canViewSoldiers, canViewNCO])
   session.canExcept(AppPermission.adminOnly)
   session.hasAnyRole([UserRoles.administrator])

   UI, routes, and gates never touch the JWT.
   They never read roles directly.
   They ask the session — and the session answers from its frozen permission set.

6️⃣ Equality — precision rebuilds

   @override
   bool operator ==(Object other) =>
     other.isAuthenticated == isAuthenticated &&
     setEquals(permissions, other.permissions) &&
     listEquals(roles, other.roles);

   setEquals + listEquals → deep structural comparison.
   BlocSelector rebuilds only when permissions actually change.
   Not on every cubit event. Not on every hydrate() call.

The full picture:

📍 SessionState.anonymous → safe starting point, always valid
🔧 hydrate() → one entry point for login, refresh, cold start
🔒 Unmodifiable collections → immutability enforced at runtime
🧹 clear() → logout resets to anonymous instantly
❓ can() / canAny() → clean query API, no JWT leakage
⚖️ setEquals + listEquals → precise Bloc rebuilds

One cubit. One state. The whole app reads from it.

#Flutter #Security #MobileDev #CleanArchitecture #RBAC #JWT #Bloc