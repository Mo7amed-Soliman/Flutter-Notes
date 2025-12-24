---
share_link: https://share.note.sx/k8rf5fzi#VDMshf7lGIqx2fnTAIZYlgsHye0rRufm+2YqCIO2bYo
share_updated: 2025-12-05T13:27:31+02:00
---

This document describes the controls for **Exact Blocked Version**, **Minimum Supported Version**, and **Maintenance Mode** for mobile apps.

---
## 💡 Concept

- **Exact blocked version**  
    Force only a specific app version to update (useful for blocking a bad release).
- **Minimum supported version**  
    Any version lower than this must update.
- **Maintenance mode**  
    Temporarily block app usage during maintenance.

---
## 🔄 Data Flow

- **Admin/Dashboard**:  
    `POST`/`UPDATE` these values in backend admin APIs.
- **App/Client**:  
    `GET`s the current config from a public endpoint and applies it.

---
## ⚙️ Configuration Example

```json
{
// Metadata for auditing/admin; clients can ignore
"meta": {
  "updated_at": "2025-08-08T10:30:00Z",
  // Who updated this configuration
  "updated_by": "[email protected]",
  // Summary of why this change was made
  "change_reason": "Hotfix: block 5.2.7 and raise minimums"
},


// Rules that apply to Android clients
"android": {
  // If the client's exact app version equals this value, force an update
  "exact_blocked_version": "5.2.7",
  // Minimum supported app version; any version lower than this must update
  "min_supported_version": "5.3.0",
  // When true, show a maintenance message and block usage
  "maintenance_mode": false,
  // Optional message shown when maintenance_mode is true
  "maintenance_message": "Scheduled maintenance. We'll be back soon."
},


// Rules that apply to iOS clients
"ios": {
  "exact_blocked_version": "",
  "min_supported_version": "5.4.0",
  "maintenance_mode": false,
  "maintenance_message": "Maintenance in progress. Please try again later." },
}
```

---
## 📝 Notes

- **Version format**: Use semantic versioning `x.y.z` (numeric only), e.g., `5.4.0`.
- **Disable a rule**: Set `exact_blocked_version` or `min_supported_version` to an empty string `""` or  `NULL`.
- **Optional metadata**: A top-level `meta` block (`updated_at`, `updated_by`, `change_reason`) is useful for auditing but can be ignored by clients.

