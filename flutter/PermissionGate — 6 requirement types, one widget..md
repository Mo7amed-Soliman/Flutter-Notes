Last Post: SessionCubit internals — how the session is built & hydrated. (https://surl.li/ycmwao)
Today: How the UI enforces authorization — declaratively, without a single if-statement.

PermissionGate — 6 requirement types, one widget.

Routes block unauthorized navigation.
But routes can't hide a button inside a screen.
That's PermissionGate's job.

1️⃣ The widget — dead simple on purpose

   class PermissionGate extends StatelessWidget {
     const PermissionGate({
       required this.requirement,
       required this.child,
       this.fallback = const SizedBox.shrink(),
     });
   }

   One question: does the current session satisfy this requirement?
   Yes → show child. No → show fallback (invisible by default).

   No Bloc boilerplate in the widget tree.
   No context.read<SessionCubit>().state.can(...) scattered in build().
   The gate owns that logic entirely.

2️⃣ Under the hood — BlocSelector, not BlocBuilder

   BlocSelector<SessionCubit, SessionState, bool>(
     selector: requirement.isSatisfiedBy,
     builder: (_, allowed) => allowed ? child : fallback,
   )

   BlocSelector rebuilds only when the bool flips.
   Not on every SessionState emission.
   Precise, cheap, correct.

3️⃣ The 6 requirement types

   • Single permission (most common)
     PermissionRequirement.permission(AppPermission.canManageSoldiers)
     → show SaveButton only if session holds this exact permission.

   • Any of (OR)
     PermissionRequirement.anyOf([canViewSoldiers, canViewNCO])
     → show Reports tab if user has at least one of these.

   • All of (AND)
     PermissionRequirement.all([canManageSoldiers, canViewSoldierArchive])
     → show AdvancedPanel only if user holds every permission.

   • All except (negative single)
     PermissionRequirement.allExcept(AppPermission.adminOnly)
     → show content to everyone except admins.

   • None of (negative OR)
     PermissionRequirement.noneOf([canManageSoldiers, canManageNCO])
     → hide management toolbar from read-only users.

   • Role-based (escape hatch)
     PermissionRequirement.roles([UserRoles.administrator])
     → use only when role identity matters, not capability.
     Prefer permission-based requirements everywhere else.

4️⃣ Real usage — composable, readable

   // Hide a button
   PermissionGate(
     requirement: PermissionRequirement.permission(
       AppPermission.canManageSoldiers,
     ),
     child: SaveButton(),
   )

   // Show a tab if user can see anything
   PermissionGate(
     requirement: PermissionRequirement.anyOf([
       AppPermission.canViewSoldiers,
       AppPermission.canViewNCO,
     ]),
     child: ReportsTab(),
   )

   // Show a fallback instead of hiding
   PermissionGate(
     requirement: PermissionRequirement.permission(
       AppPermission.canManageVacations,
     ),
     child: EditForm(),
     fallback: ReadOnlyView(),
   )

   No if (session.can(...)) in any widget.
   The requirement is the documentation.

5️⃣ OCP in practice

   Adding a new requirement type = add a factory to PermissionRequirement.
   Zero changes to PermissionGate.
   Zero changes to existing consumers.

   The gate is closed for modification.
   The requirement is open for extension.

The full picture:

🎨 PermissionGate → one widget, all UI authorization
🔍 BlocSelector → rebuilds only when access flips
✅ .permission() → single capability check
∨ .anyOf() → OR across permissions
∧ .all() → AND across permissions
❌ .allExcept() / .noneOf() → negative guards
🏷️ .roles() → identity check, last resort

Same session. Same matrix. Routes and widgets enforcing the same rules.

#Flutter #Security #MobileDev #CleanArchitecture #RBAC #Bloc #GoRouter