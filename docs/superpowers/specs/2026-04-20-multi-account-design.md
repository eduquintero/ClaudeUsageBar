# Multi-Account Switch — Design Spec
Date: 2026-04-20

## Overview

Add support for multiple Claude.ai accounts in ClaudeUsageBar. Users can configure any number of accounts, each with a user-defined name and session cookie. A tile-based switcher in the popover lets users activate any account; the menu bar icon always reflects the active account's usage percentage.

---

## Data Model

### Account struct

```swift
struct Account: Codable {
    let id: UUID
    var name: String       // user-defined: "Trabajo", "Personal"
    var cookie: String     // claude.ai session cookie
    var orgId: String?     // cached on first successful fetch
}
```

### UserDefaults keys

| Key | Type | Description |
|-----|------|-------------|
| `accounts` | Data (JSON) | `[Account]` array |
| `active_account_id` | String | UUID of the active account |
| `notifications_enabled` | Bool | unchanged |
| `has_set_notifications` | Bool | unchanged |
| `open_at_login` | Bool | unchanged |
| `shortcut_enabled` | Bool | unchanged |
| `last_notified_threshold` | Int | unchanged |

Keys removed: `claude_session_cookie` (replaced by `accounts`).

### Migration

On first launch of the new version, if `claude_session_cookie` exists in UserDefaults:
1. Create an `Account` with `name = "Principal"`, `cookie = <existing value>`, `id = UUID()`
2. Save it as `accounts = [account]`, set `active_account_id = account.id`
3. Remove `claude_session_cookie` from UserDefaults

---

## Refresh Behavior

- Only the **active account** runs the 5-minute auto-refresh timer.
- On `switchAccount(to:)`: cancel the current timer, fetch immediately for the new account, restart the timer.
- Inactive accounts retain their last-fetched data in memory (not persisted between app launches).

---

## ClaudeUsageManager Changes

### New properties

```swift
@Published var accounts: [Account] = []
@Published var activeAccountId: UUID?
```

### Computed property (replaces sessionCookie stored property)

```swift
var sessionCookie: String {
    accounts.first(where: { $0.id == activeAccountId })?.cookie ?? ""
}
```

### New methods

| Method | Behavior |
|--------|----------|
| `loadAccounts()` | Replaces `loadSessionCookie()`. Decodes `accounts` from UserDefaults, sets `activeAccountId` from `active_account_id`. Triggers migration if legacy cookie found. |
| `saveAccounts()` | Replaces `saveSessionCookie()`. Encodes `accounts` array to UserDefaults. |
| `switchAccount(to id: UUID)` | Sets `activeAccountId`, saves to UserDefaults, cancels timer, calls `fetchUsage()`, restarts timer. |
| `addAccount(name:cookie:)` | Creates `Account`, appends to `accounts`, saves. If it's the first account, activates it immediately. |
| `removeAccount(id:)` | Removes account from array. If removed account was active: activate the first remaining account (or set `activeAccountId = nil` if none left). Saves. |
| `cacheOrgId(_:for:)` | Called by `fetchOrganizationId()` after resolving the org ID. Updates the `orgId` field on the matching Account in the array and saves. |

### Unchanged

- `fetchOrganizationId()` — still reads from `sessionCookie` computed property
- `fetchUsageWithOrgId()` — no changes
- JSON parsing logic
- Notification system
- Settings (notifications, open at login, shortcut)
- Keyboard shortcut (Cmd+U)

---

## UI Changes

### Popover size

Increases from `320×260` to `320×320` to accommodate the tile row.

### New views

#### `AccountTileRow`

- Horizontal `HStack` of `AccountTile` views + one `AddAccountTile` at the end
- If `accounts` is empty, renders only `AddAccountTile` in expanded form (the old `sessionCookie` text input is removed)
- Wrapped in `ScrollView(.horizontal, showsIndicators: false)` so tiles don't overflow if many accounts are added

#### `AccountTile`

Properties shown per tile:
- Account name (uppercase label, 10px)
- Session usage % (large, bold)
- Mini horizontal progress bar (4px height)
- `×` button (top-right corner, always visible, small)

Visual states:
- **Active**: `border: 2px solid <usage-color>`, full brightness, colored text
- **Inactive**: `border: 2px solid #374151`, 60% opacity, gray text

Usage color follows existing scheme: green < 70%, yellow < 90%, red ≥ 90%.

On tap: calls `usageManager.switchAccount(to: account.id)`.

#### `AddAccountTile`

- Shown as a `+` tile at the end of the row (same width as `AccountTile`, dashed border)
- On tap: expands the tile row into an inline form:

```
[ Nombre     ] [ Pegar cookie...       ] [✓] [×]
```

- `✓` calls `addAccount(name:cookie:)` and collapses the form
- `×` collapses without saving
- Cookie field supports paste (existing `PasteButton` pattern from codebase)

#### `DeleteAccount` behavior

- Tap `×` on an `AccountTile` → calls `removeAccount(id:)`
- If deleting the active account and others remain → first remaining account becomes active
- If deleting the last account → `activeAccountId = nil`, popover shows empty state with cookie input

### Menu bar icon

No format change. The icon continues to show `⚡ XX%` reflecting the active account's session usage. Account name is not shown in the menu bar to keep it uncluttered.

---

## Error & Edge Cases

| Case | Behavior |
|------|----------|
| Switch while fetch is in-flight | Store the active `URLSessionDataTask` as a property on `ClaudeUsageManager`; call `.cancel()` before starting new fetch |
| Invalid cookie on new account | Shows existing error state ("Cookie inválida") in the detail section |
| All accounts removed | Returns to empty state; shows cookie input as before |
| orgId not cached yet | Fetches from `/api/bootstrap` on first activation (existing fallback) |

---

## Out of Scope

- Showing aggregated usage across all accounts
- Background refresh of inactive accounts
- Automatic account name detection from cookie/API
- Account reordering
- Account limit enforcement (no hard cap)
