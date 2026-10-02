---
share_link: https://share.note.sx/k8rf5fzi#VDMshf7lGIqx2fnTAIZYlgsHye0rRufm+2YqCIO2bYo
share_updated: 2026-09-23T05:17:20+03:00
---
# App Version Control & Maintenance Mode

> [!NOTE]
> These three controls let you **gate app access** without a new release — block bad versions, enforce minimums, and pause the app during maintenance. All applied client-side from a single config endpoint.

---

## 💡 The Three Controls

| Control                       | Purpose                                                                 | Trigger                                   |
| ----------------------------- | ----------------------------------------------------------------------- | ----------------------------------------- |
| **Exact Blocked Version**     | Force a *specific* version to update — useful for yanking a bad release | `client_version == exact_blocked_version` |
| **Minimum Supported Version** | Force *any version below this* to update                                | `client_version < min_supported_version`  |
| **Maintenance Mode**          | Temporarily block all usage and show a message                          | `maintenance_mode == true`                |

---

## 🔄 Data Flow

```
Admin / Dashboard
  └─ POST / PATCH  →  backend admin API  →  stores config

Mobile Client (on launch / periodic poll)
  └─ GET  →  public config endpoint  →  parses & applies rules locally
```

> [!IMPORTANT]
> The client **never writes** to this config — it only reads and enforces it. All changes go through the admin API.

---

## ⚙️ Config Schema

```jsonc
{
  // ── Audit metadata ──────────────────────────────────────────────────────
  // Ignored by clients; used by admins for change tracking
  "meta": {
    "updated_at": "2025-08-08T10:30:00Z",
    "updated_by": "[email protected]",
    "change_reason": "Hotfix: block 5.2.7 and raise minimums"
  },

  // ── Android rules ────────────────────────────────────────────────────────
  "android": {
    // Force THIS exact version to update. Set "" or null to disable.
    "exact_blocked_version": "5.2.7",

    // Any version below this must update. Set "" or null to disable.
    "min_supported_version": "5.3.0",

    // Block all usage and show maintenance_message when true.
    "maintenance_mode": false,
    "maintenance_message": "Scheduled maintenance. We'll be back soon."
  },

  // ── iOS rules ────────────────────────────────────────────────────────────
  "ios": {
    "exact_blocked_version": "",        // disabled — no version blocked
    "min_supported_version": "5.4.0",
    "maintenance_mode": false,
    "maintenance_message": "Maintenance in progress. Please try again later."
  }
}
```

---

## 🧠 Client Decision Logic

The client evaluates rules in **priority order** on every launch:

```
1. maintenance_mode == true?
       └─ YES → show maintenance screen, block app          ⛔ STOP
       └─ NO  → continue

2. client_version == exact_blocked_version?
       └─ YES → force update screen                         ⛔ STOP
       └─ NO  → continue

3. client_version < min_supported_version?
       └─ YES → force update screen                         ⛔ STOP
       └─ NO  → continue

4. ✅ Allow app to proceed normally
```

> [!IMPORTANT]
> **Maintenance mode is checked first** — it overrides everything, including valid versions.

---

## 📝 Rules & Edge Cases

| Rule                   | Detail                                                                          |
| ---------------------- | ------------------------------------------------------------------------------- |
| **Version format**     | Semantic versioning `x.y.z` (numeric only) — e.g., `5.4.0`                      |
| **Disable a control**  | Set `exact_blocked_version` or `min_supported_version` to `""` or `null`        |
| **Per-platform**       | Android and iOS configs are independent — block one without affecting the other |
| **`meta` block**       | Optional; clients should ignore it. Useful for admin audit trails               |
| **Version comparison** | Compare numerically per segment, not lexicographically (`5.10.0 > 5.9.0`)       |

> [!CAUTION]
> Do **not** compare version strings lexicographically — `"5.10.0" < "5.9.0"` alphabetically, which is wrong. Use a semver-aware comparison.
