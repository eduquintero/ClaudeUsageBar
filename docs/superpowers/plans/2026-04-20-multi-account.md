# Multi-Account Switch — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add multi-account support to ClaudeUsageBar with a tile-based switcher in the popover, where the active account's usage is shown in the menu bar.

**Architecture:** Replace the single `sessionCookie` string in `UsageManager` with a `[Account]` array and `activeAccountId`; `sessionCookie` becomes a computed property. Move the 5-min timer into `UsageManager` so `switchAccount` can cancel and restart it. Three new SwiftUI views (`AccountTile`, `AddAccountTile`, `AccountTileRow`) replace the old cookie-input section in `UsageView`.

**Tech Stack:** Swift 5.9+, SwiftUI, AppKit, JSONEncoder/JSONDecoder, NSUserDefaults

---

## File Map

All changes are in a single file:

| File | Change |
|------|--------|
| `app/ClaudeUsageBar.swift` | All model, logic, and UI changes |

---

### Task 1: Account data model + UsageManager logic refactor

**Files:**
- Modify: `app/ClaudeUsageBar.swift`

This task wires up the full model layer. After this task the app compiles and behaves identically to before (migration runs transparently for existing users).

- [ ] **Step 1: Add `Account` struct**

Insert the following block at line 285 (immediately after the closing `}` of the `NSColor` extension, before the `@main` struct):

```swift
struct Account: Codable, Identifiable {
    let id: UUID
    var name: String
    var cookie: String
    var orgId: String?

    init(name: String, cookie: String) {
        self.id = UUID()
        self.name = name
        self.cookie = cookie
    }
}
```

- [ ] **Step 2: Replace `sessionCookie` stored property with a computed property and add new published properties**

In `UsageManager`, find and replace (around line 319):

**Remove:**
```swift
    private var sessionCookie: String = ""
```

**Add in its place:**
```swift
    @Published var accounts: [Account] = []
    @Published var activeAccountId: UUID?

    var sessionCookie: String {
        accounts.first(where: { $0.id == activeAccountId })?.cookie ?? ""
    }

    private var currentTask: URLSessionDataTask?
    private var refreshTimer: Timer?
```

- [ ] **Step 3: Add `loadAccounts()`, `saveAccounts()`, and account management methods**

Insert the following block immediately after the closing brace of `clearSessionCookie()` (around line 396), before `fetchOrganizationId`:

```swift
    func loadAccounts() {
        // Migration: convert legacy single cookie to accounts array
        if let legacy = UserDefaults.standard.string(forKey: "claude_session_cookie"), !legacy.isEmpty {
            let account = Account(name: "Principal", cookie: legacy)
            accounts = [account]
            activeAccountId = account.id
            saveAccounts()
            UserDefaults.standard.removeObject(forKey: "claude_session_cookie")
            UserDefaults.standard.synchronize()
            return
        }

        if let data = UserDefaults.standard.data(forKey: "accounts"),
           let decoded = try? JSONDecoder().decode([Account].self, from: data) {
            accounts = decoded
        }
        if let uuidString = UserDefaults.standard.string(forKey: "active_account_id"),
           let uuid = UUID(uuidString: uuidString) {
            activeAccountId = uuid
        } else {
            activeAccountId = accounts.first?.id
        }
    }

    func saveAccounts() {
        if let data = try? JSONEncoder().encode(accounts) {
            UserDefaults.standard.set(data, forKey: "accounts")
        }
        UserDefaults.standard.set(activeAccountId?.uuidString, forKey: "active_account_id")
        UserDefaults.standard.synchronize()
    }

    func addAccount(name: String, cookie: String) {
        let account = Account(name: name, cookie: cookie)
        accounts.append(account)
        saveAccounts()
        if accounts.count == 1 {
            switchAccount(to: account.id)
        }
    }

    func removeAccount(id: UUID) {
        accounts.removeAll(where: { $0.id == id })
        saveAccounts()
        if activeAccountId == id {
            if let first = accounts.first {
                switchAccount(to: first.id)
            } else {
                activeAccountId = nil
                currentTask?.cancel()
                stopTimer()
                sessionUsage = 0
                weeklyUsage = 0
                weeklySonnetUsage = 0
                sessionResetsAt = nil
                weeklyResetsAt = nil
                weeklySonnetResetsAt = nil
                hasFetchedData = false
                hasWeeklySonnet = false
                errorMessage = nil
                updatePercentages()
                delegate?.updateStatusIcon(percentage: 0)
            }
        }
    }

    func switchAccount(to id: UUID) {
        guard accounts.contains(where: { $0.id == id }) else { return }
        activeAccountId = id
        UserDefaults.standard.set(id.uuidString, forKey: "active_account_id")
        UserDefaults.standard.synchronize()
        sessionUsage = 0
        weeklyUsage = 0
        weeklySonnetUsage = 0
        sessionResetsAt = nil
        weeklyResetsAt = nil
        weeklySonnetResetsAt = nil
        hasFetchedData = false
        hasWeeklySonnet = false
        errorMessage = nil
        updatePercentages()
        currentTask?.cancel()
        stopTimer()
        fetchUsage()
        startTimer()
    }

    func cacheOrgId(_ orgId: String, for accountId: UUID) {
        if let idx = accounts.firstIndex(where: { $0.id == accountId }) {
            accounts[idx].orgId = orgId
            saveAccounts()
        }
    }

    private func startTimer() {
        refreshTimer?.invalidate()
        refreshTimer = Timer.scheduledTimer(withTimeInterval: 300, repeats: true) { [weak self] _ in
            self?.fetchUsage()
        }
    }

    private func stopTimer() {
        refreshTimer?.invalidate()
        refreshTimer = nil
    }
```

- [ ] **Step 4: Update `init()` to call `loadAccounts()` and `startTimer()`**

Find the `init` method (around line 323) and replace its body:

**Before:**
```swift
    init(statusItem: NSStatusItem?, delegate: AppDelegate? = nil) {
        self.statusItem = statusItem
        self.delegate = delegate
        loadSessionCookie()
        loadSettings()
        checkAccessibilityStatus()
    }
```

**After:**
```swift
    init(statusItem: NSStatusItem?, delegate: AppDelegate? = nil) {
        self.statusItem = statusItem
        self.delegate = delegate
        loadAccounts()
        loadSettings()
        checkAccessibilityStatus()
        DispatchQueue.main.async { self.startTimer() }
    }
```

- [ ] **Step 5: Update `saveSessionCookie()` and `clearSessionCookie()` to use the new model**

These methods are still called by the existing UI (will be removed in Task 5). Update them to delegate to the new model.

**Replace `saveSessionCookie`:**
```swift
    func saveSessionCookie(_ cookie: String) {
        if let id = activeAccountId, let idx = accounts.firstIndex(where: { $0.id == id }) {
            accounts[idx].cookie = cookie
            accounts[idx].orgId = nil
            saveAccounts()
        } else {
            addAccount(name: "Principal", cookie: cookie)
            return
        }
        fetchUsage()
    }
```

**Replace `clearSessionCookie`:**
```swift
    func clearSessionCookie() {
        if let id = activeAccountId {
            removeAccount(id: id)
        }
    }
```

- [ ] **Step 6: Update `fetchOrganizationId()` to use cached orgId and call `cacheOrgId`**

Replace the entire `fetchOrganizationId` method:

```swift
    func fetchOrganizationId(completion: @escaping (String?) -> Void) {
        // Use cached orgId if available
        if let id = activeAccountId,
           let account = accounts.first(where: { $0.id == id }),
           let cached = account.orgId {
            NSLog("📋 Using cached org ID: \(cached)")
            completion(cached)
            return
        }

        // Parse from cookie
        let cookieParts = sessionCookie.components(separatedBy: ";")
        for part in cookieParts {
            let trimmed = part.trimmingCharacters(in: .whitespaces)
            if trimmed.hasPrefix("lastActiveOrg=") {
                let orgId = trimmed.replacingOccurrences(of: "lastActiveOrg=", with: "")
                NSLog("📋 Found org ID in cookie: \(orgId)")
                if let id = activeAccountId { cacheOrgId(orgId, for: id) }
                completion(orgId)
                return
            }
        }

        // Fallback: fetch from bootstrap
        guard let url = URL(string: "https://claude.ai/api/bootstrap") else {
            completion(nil)
            return
        }

        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.setValue("sessionKey=\(sessionCookie)", forHTTPHeaderField: "Cookie")

        NSLog("📡 Fetching bootstrap to get org ID...")

        URLSession.shared.dataTask(with: request) { [weak self] data, response, error in
            guard let self = self,
                  let data = data,
                  let json = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
                  let account = json["account"] as? [String: Any],
                  let lastActiveOrgId = account["lastActiveOrgId"] as? String else {
                NSLog("❌ Could not parse org ID from bootstrap")
                completion(nil)
                return
            }
            NSLog("✅ Got org ID from bootstrap: \(lastActiveOrgId)")
            if let id = self.activeAccountId {
                DispatchQueue.main.async { self.cacheOrgId(lastActiveOrgId, for: id) }
            }
            completion(lastActiveOrgId)
        }.resume()
    }
```

- [ ] **Step 7: Update `fetchUsageWithOrgId()` to track the in-flight task**

In `fetchUsageWithOrgId`, replace the `URLSession.shared.dataTask` call so it stores and can cancel the task:

Find the line `URLSession.shared.dataTask(with: request) { [weak self] data, response, error in` (around line 488) and the `.resume()` call. Replace them:

**Before:**
```swift
        URLSession.shared.dataTask(with: request) { [weak self] data, response, error in
            DispatchQueue.main.async {
                self?.isLoading = false
                ...
            }
        }.resume()
```

**After:**
```swift
        currentTask = URLSession.shared.dataTask(with: request) { [weak self] data, response, error in
            DispatchQueue.main.async {
                self?.currentTask = nil
                self?.isLoading = false
                ...
            }
        }
        currentTask?.resume()
```

- [ ] **Step 8: Remove the Timer from `AppDelegate.applicationDidFinishLaunching`**

Find and remove this block (around lines 45–48 of `AppDelegate`):

```swift
        // Set up timer to refresh every 5 minutes
        Timer.scheduledTimer(withTimeInterval: 300, repeats: true) { _ in
            self.usageManager.fetchUsage()
        }
```

The timer now lives inside `UsageManager.init`.

- [ ] **Step 9: Typecheck**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && swiftc -typecheck app/ClaudeUsageBar.swift -sdk $(xcrun --show-sdk-path --sdk macosx) 2>&1
```

Expected: no output (no errors).

- [ ] **Step 10: Commit**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && git add app/ClaudeUsageBar.swift && git commit -m "refactor: replace single cookie with Account model + multi-account logic"
```

---

### Task 2: `AccountTile` SwiftUI view

**Files:**
- Modify: `app/ClaudeUsageBar.swift`

- [ ] **Step 1: Insert `AccountTile` view**

Insert the following block immediately after the closing `}` of `PasteableTextField` (around line 807), before `struct UsageView`:

```swift
struct AccountTile: View {
    @ObservedObject var usageManager: UsageManager
    let account: Account
    let onRemove: () -> Void

    private var isActive: Bool {
        usageManager.activeAccountId == account.id
    }

    private var displayPercentage: Double {
        isActive ? usageManager.sessionPercentage : 0
    }

    private var displayUsage: Int {
        isActive ? usageManager.sessionUsage : 0
    }

    private var usageColor: Color {
        if displayPercentage < 0.7 { return .green }
        else if displayPercentage < 0.9 { return .orange }
        else { return .red }
    }

    var body: some View {
        ZStack(alignment: .topTrailing) {
            Button(action: { usageManager.switchAccount(to: account.id) }) {
                VStack(alignment: .leading, spacing: 4) {
                    Text(account.name.uppercased())
                        .font(.system(size: 10))
                        .foregroundColor(isActive ? .secondary : Color.secondary.opacity(0.5))

                    Text(isActive && usageManager.hasFetchedData ? "\(displayUsage)%" : "—")
                        .font(.system(size: 20, weight: .bold))
                        .foregroundColor(isActive ? usageColor : Color.secondary.opacity(0.5))

                    GeometryReader { geo in
                        ZStack(alignment: .leading) {
                            RoundedRectangle(cornerRadius: 2)
                                .fill(Color.secondary.opacity(0.2))
                                .frame(height: 4)
                            RoundedRectangle(cornerRadius: 2)
                                .fill(isActive ? usageColor : Color.secondary.opacity(0.3))
                                .frame(width: geo.size.width * CGFloat(displayPercentage), height: 4)
                        }
                    }
                    .frame(height: 4)
                }
                .padding(10)
                .frame(maxWidth: .infinity, alignment: .leading)
                .background(isActive ? Color.secondary.opacity(0.12) : Color.secondary.opacity(0.04))
                .overlay(
                    RoundedRectangle(cornerRadius: 8)
                        .stroke(isActive ? usageColor : Color.secondary.opacity(0.3), lineWidth: isActive ? 2 : 1)
                )
                .cornerRadius(8)
                .opacity(isActive ? 1.0 : 0.6)
            }
            .buttonStyle(.plain)

            // × remove button
            Button(action: onRemove) {
                Image(systemName: "xmark")
                    .font(.system(size: 7, weight: .bold))
                    .foregroundColor(.secondary)
                    .padding(5)
            }
            .buttonStyle(.plain)
        }
    }
}
```

- [ ] **Step 2: Typecheck**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && swiftc -typecheck app/ClaudeUsageBar.swift -sdk $(xcrun --show-sdk-path --sdk macosx) 2>&1
```

Expected: no output.

- [ ] **Step 3: Commit**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && git add app/ClaudeUsageBar.swift && git commit -m "feat: add AccountTile SwiftUI view"
```

---

### Task 3: `AddAccountTile` SwiftUI view

**Files:**
- Modify: `app/ClaudeUsageBar.swift`

- [ ] **Step 1: Insert `AddAccountTile` view**

Insert the following block immediately after the closing `}` of `AccountTile`, before `struct UsageView`:

```swift
struct AddAccountTile: View {
    @ObservedObject var usageManager: UsageManager
    @State private var isExpanded: Bool
    @State private var nameInput: String = ""
    @State private var cookieInput: String = ""

    init(usageManager: UsageManager, initiallyExpanded: Bool = false) {
        self.usageManager = usageManager
        _isExpanded = State(initialValue: initiallyExpanded)
    }

    var body: some View {
        if isExpanded {
            expandedForm
        } else {
            collapsedTile
        }
    }

    private var collapsedTile: some View {
        Button(action: { isExpanded = true }) {
            VStack {
                Spacer()
                Image(systemName: "plus")
                    .font(.system(size: 16, weight: .medium))
                    .foregroundColor(.secondary)
                Spacer()
            }
            .frame(maxWidth: .infinity)
            .frame(height: 72)
            .overlay(
                RoundedRectangle(cornerRadius: 8)
                    .stroke(Color.secondary.opacity(0.3), style: StrokeStyle(lineWidth: 1, dash: [4]))
            )
            .cornerRadius(8)
        }
        .buttonStyle(.plain)
    }

    private var expandedForm: some View {
        VStack(alignment: .leading, spacing: 6) {
            Text("Nueva cuenta")
                .font(.system(size: 10))
                .foregroundColor(.secondary)
                .textCase(.uppercase)

            TextField("Nombre", text: $nameInput)
                .textFieldStyle(.roundedBorder)
                .font(.system(size: 11))

            PasteableTextField(text: $cookieInput, placeholder: "Pegar cookie...")
                .frame(height: 44)
                .cornerRadius(4)

            HStack(spacing: 6) {
                Button(action: confirmAdd) {
                    Image(systemName: "checkmark")
                        .font(.system(size: 11, weight: .semibold))
                }
                .buttonStyle(.borderedProminent)
                .controlSize(.small)
                .disabled(nameInput.trimmingCharacters(in: .whitespaces).isEmpty || cookieInput.isEmpty)

                Button(action: cancelAdd) {
                    Image(systemName: "xmark")
                        .font(.system(size: 11))
                }
                .buttonStyle(.bordered)
                .controlSize(.small)
            }
        }
        .padding(10)
        .overlay(
            RoundedRectangle(cornerRadius: 8)
                .stroke(Color.secondary.opacity(0.3), lineWidth: 1)
        )
        .cornerRadius(8)
    }

    private func confirmAdd() {
        let name = nameInput.trimmingCharacters(in: .whitespaces)
        guard !name.isEmpty, !cookieInput.isEmpty else { return }
        usageManager.addAccount(name: name, cookie: cookieInput)
        nameInput = ""
        cookieInput = ""
        isExpanded = false
    }

    private func cancelAdd() {
        nameInput = ""
        cookieInput = ""
        isExpanded = false
    }
}
```

- [ ] **Step 2: Typecheck**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && swiftc -typecheck app/ClaudeUsageBar.swift -sdk $(xcrun --show-sdk-path --sdk macosx) 2>&1
```

Expected: no output.

- [ ] **Step 3: Commit**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && git add app/ClaudeUsageBar.swift && git commit -m "feat: add AddAccountTile SwiftUI view"
```

---

### Task 4: `AccountTileRow` SwiftUI view

**Files:**
- Modify: `app/ClaudeUsageBar.swift`

- [ ] **Step 1: Insert `AccountTileRow` view**

Insert the following block immediately after the closing `}` of `AddAccountTile`, before `struct UsageView`:

```swift
struct AccountTileRow: View {
    @ObservedObject var usageManager: UsageManager

    var body: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(usageManager.accounts) { account in
                    AccountTile(usageManager: usageManager, account: account) {
                        usageManager.removeAccount(id: account.id)
                    }
                    .frame(width: 120)
                }
                AddAccountTile(usageManager: usageManager, initiallyExpanded: usageManager.accounts.isEmpty)
                    .frame(width: usageManager.accounts.isEmpty ? .infinity : 80)
            }
            .padding(.horizontal, 1)
        }
        .frame(height: 88)
    }
}
```

- [ ] **Step 2: Typecheck**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && swiftc -typecheck app/ClaudeUsageBar.swift -sdk $(xcrun --show-sdk-path --sdk macosx) 2>&1
```

Expected: no output.

- [ ] **Step 3: Commit**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && git add app/ClaudeUsageBar.swift && git commit -m "feat: add AccountTileRow SwiftUI view"
```

---

### Task 5: Integrate `AccountTileRow` into `UsageView` and update popover size

**Files:**
- Modify: `app/ClaudeUsageBar.swift`

- [ ] **Step 1: Update popover height in `AppDelegate`**

Find (around line 38):
```swift
        popover.contentSize = NSSize(width: 320, height: 260)
```

Replace with:
```swift
        popover.contentSize = NSSize(width: 320, height: 320)
```

- [ ] **Step 2: Replace `@State` properties in `UsageView`**

Find at the top of `UsageView.body` (around line 811):
```swift
    @State private var sessionCookieInput: String = ""
    @State private var showingCookieInput: Bool = false
    @State private var showingSettings: Bool = false
```

Replace with:
```swift
    @State private var showingSettings: Bool = false
```

- [ ] **Step 3: Add `AccountTileRow` at the top of `UsageView.body`**

Find inside `body`, after the opening `VStack(alignment: .leading, spacing: 16) {`:
```swift
            Text("Claude Usage")
                .font(.headline)
                .padding(.bottom, 4)
```

Replace with:
```swift
            Text("Claude Usage")
                .font(.headline)
                .padding(.bottom, 4)

            AccountTileRow(usageManager: usageManager)
```

- [ ] **Step 4: Remove old cookie button and cookie input section from `UsageView`**

Find and remove the following block entirely (around lines 919–980):

```swift
            Button(showingCookieInput ? "Hide Cookie" : "Set Session Cookie") {
                showingCookieInput.toggle()
            }
            .buttonStyle(.borderless)
            .font(.caption)

            if showingCookieInput {
                VStack(alignment: .leading, spacing: 8) {
                    ...
                }
                .padding(8)
                .background(Color.secondary.opacity(0.1))
                .cornerRadius(6)
            }
```

- [ ] **Step 5: Remove the `.onAppear` cookie-load block from `UsageView`**

Find and remove (around line 1092):
```swift
        .onAppear {
            // Load saved cookie when view appears
            if let savedCookie = UserDefaults.standard.string(forKey: "claude_session_cookie") {
                sessionCookieInput = String(savedCookie.prefix(20)) + "..."
            }
            // Force refresh to ensure progress bars show colors
            usageManager.updatePercentages()
        }
```

Replace with:
```swift
        .onAppear {
            usageManager.updatePercentages()
        }
```

- [ ] **Step 6: Update the welcome message for empty state**

Find:
```swift
                Text("👋 Welcome! Set your session cookie below to get started.")
                    .font(.subheadline)
                    .foregroundColor(.secondary)
                    .padding(.vertical, 8)
```

Replace with:
```swift
                Text("👋 Welcome! Add an account above to get started.")
                    .font(.subheadline)
                    .foregroundColor(.secondary)
                    .padding(.vertical, 8)
```

- [ ] **Step 7: Typecheck**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && swiftc -typecheck app/ClaudeUsageBar.swift -sdk $(xcrun --show-sdk-path --sdk macosx) 2>&1
```

Expected: no output.

- [ ] **Step 8: Full build**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && bash build.sh 2>&1 | tail -30
```

Expected: `ClaudeUsageBar.app` built successfully, no errors.

- [ ] **Step 9: Manual smoke test**

Launch the built app and verify:
1. The popover shows the `AccountTileRow` at the top with a `+` tile (no accounts yet, or migrated account if cookie existed)
2. Tap `+` → form expands with Name + Cookie fields
3. Fill in name and paste a cookie → tap ✓ → tile appears, usage fetches, menu bar updates
4. Add a second account → second tile appears
5. Tap the second tile → it becomes active, menu bar updates, usage fetches
6. Tap `×` on a tile → it is removed; if it was active, the other becomes active
7. Remove all accounts → row shows only the `+` tile and the welcome message appears

- [ ] **Step 10: Commit**

```bash
cd "/Users/Edqu/Documents/Trabajo/Edqu - Dev/ClaudeUsageBar" && git add app/ClaudeUsageBar.swift && git commit -m "feat: integrate AccountTileRow into UsageView, replace cookie input"
```
