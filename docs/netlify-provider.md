# Netlify Provider Support

**Status: Planning**
**Goal: Add Netlify as a first-class provider alongside GitHub Actions**

---

## Overview

Add Netlify deploy monitoring to Octowatch via a top-level provider switcher. Each provider is independently optional — users can connect GitHub, Netlify, or both. The app stays additive: new files are created for Netlify; existing GitHub files are touched only where strictly necessary.

---

## UI/UX Design

### Provider Switcher

A segmented control (`GitHub | Netlify`) sits at the top of the authenticated popover, above the run/deploy list. It is:

- Visible whenever at least one provider is connected
- Automatically hidden (shows only the connected provider's content) when exactly one provider is connected
- State persisted to `UserDefaults` as `activeProvider` so the last-selected tab survives restarts

### Popover Layout (authenticated)

```
┌──────────────────────────────────┐
│  HeaderView (refresh / settings) │
├──────────────────────────────────┤
│  [ GitHub Actions ] [ Netlify ]  │  ← ProviderSwitcherView (new)
├──────────────────────────────────┤
│  Run / Deploy list for active    │
│  provider                        │
├──────────────────────────────────┤
│  FooterView (username / sign out)│
└──────────────────────────────────┘
```

### Sign-In Flow

- `SignInView` (existing) handles GitHub PAT entry — **unchanged**
- A new `NetlifySignInView` handles Netlify personal access token entry
- Both are independently reachable: if neither provider is connected, a landing screen lets you choose which to connect first
- Connecting a second provider does not replace the first

### Settings

A new **Netlify** section in `SettingsView` (below the existing GitHub repo picker) contains:

- Netlify token status + disconnect button
- Site picker — toggle list of available sites (same UX as the existing repo picker)

### Deploy Row (`NetlifyDeployRow`)

Mirrors `WorkflowRunRow`. Each row shows:

- Status dot (color-coded: green = ready, orange = building/enqueued, red = error)
- Deploy title (commit message or branch name)
- Site name · branch · duration
- Relative timestamp (right-aligned)
- Clicking opens the deploy URL in the browser

### Menu Bar Label

The `MenuBarLabel` shows bubbles for the **currently selected provider only**, keeping the label consistent with what's visible in the popover. Each Netlify bubble uses the site name's first letter as its initial, matching the GitHub repo bubble pattern.

---

## Architecture

### New Files

| File | Purpose |
|---|---|
| `Services/NetlifyAPIClient.swift` | Swift `actor` — wraps Netlify REST API. Mirrors `GitHubAPIClient`. |
| `Models/NetlifyDeploy.swift` | Data models: `NetlifySite`, `NetlifyDeploy`, `DeployState`, `DeployContext` |
| `ViewModels/NetlifyViewModel.swift` | `@Observable @MainActor` class owning Netlify auth, polling, site selection, notifications. Mirrors `WorkflowViewModel` structure. |
| `Views/NetlifySignInView.swift` | Token entry screen for Netlify. Mirrors `SignInView`. |
| `Views/NetlifyDeployRow.swift` | Single deploy row. Mirrors `WorkflowRunRow`. |
| `Views/NetlifyDeployListView.swift` | Scrollable list of deploys. Mirrors `WorkflowRunListView`. |
| `Views/ProviderSwitcherView.swift` | Segmented control + `activeProvider` state. |

### Modified Files (minimal, additive only)

| File | Change |
|---|---|
| `GitHubActionsBarApp.swift` | Add `@State private var netlifyViewModel = NetlifyViewModel()`. Pass to `MainPopoverView`. |
| `MainPopoverView.swift` | Add `NetlifyViewModel` binding. Inject `ProviderSwitcherView` + conditional list rendering based on `activeProvider`. |
| `SettingsView.swift` | Add Netlify section (site picker, token management). No changes to existing GitHub section. |
| `MenuBarLabel.swift` | Accept `activeProvider` + Netlify site statuses. Render bubbles for active provider only. |
| `KeychainService.swift` | Add `netlify-pat` account key + corresponding `saveNetlifyPAT` / `retrieveNetlifyPAT` / `deleteNetlifyPAT` static methods. |
| `NotificationService.swift` | Add `sendDeployCompleted(deploy: NetlifyDeploy, playSound: Bool)`. Existing `sendRunCompleted` untouched. |

### Netlify API

- Base URL: `https://api.netlify.com/api/v1`
- Auth: `Authorization: Bearer <token>`
- List sites: `GET /api/v1/sites?per_page=100`
- List deploys per site: `GET /api/v1/sites/{site_id}/deploys?per_page=15`

**`NetlifyDeploy` key fields:**

| Field | Notes |
|---|---|
| `id: String` | Deploy ID |
| `state: DeployState` | `ready`, `building`, `error`, `enqueued`, `processing`, `uploading`, `new` |
| `context: DeployContext` | `production`, `deploy-preview`, `branch-deploy` |
| `name: String` | Site name |
| `branch: String?` | Git branch |
| `title: String?` | Commit message (may be nil) |
| `deploy_url: String` | URL to open in browser |
| `created_at: Date` | Deploy start |
| `updated_at: Date` | Last state change |
| `published_at: Date?` | When it went live (ready deploys only) |

**`DeployState` → `AggregateStatus` mapping:**

| Netlify state | AggregateStatus |
|---|---|
| `ready` | `.allGreen` |
| `building`, `enqueued`, `processing`, `uploading` | `.inProgress` |
| `error` | `.failed` |
| `new` | `.idle` |

### `NetlifyViewModel` Structure

Mirrors `WorkflowViewModel` closely:

```
NetlifyViewModel
├── isAuthenticated: Bool
├── sites: [NetlifySite]
├── deploys: [NetlifyDeploy]
├── selectedSiteIds: Set<String>          ← UserDefaults "selectedNetlifySites"
├── aggregateStatus: AggregateStatus
├── siteStatuses: [SiteStatusItem]        ← drives MenuBarLabel bubbles
├── isLoading: Bool
├── errorMessage: String?
├── pollingInterval: TimeInterval         ← shared UserDefaults key with GitHub
├── notificationSoundEnabled: Bool        ← shared
│
├── signIn(token: String)
├── signOut()
├── loadInitialData()
├── fetchDeploys()
├── startPolling() / stopPolling()
└── detectCompletions(newDeploys:)
```

`SiteStatusItem` is a new struct identical in shape to `RepoStatusItem` — no changes to the existing struct needed.

---

## Provider Switcher State

```swift
enum Provider: String {
    case github
    case netlify
}
```

Stored in `UserDefaults` as `"activeProvider"`. Lives in `MainPopoverView` (or a thin coordinator) as `@State var activeProvider: Provider`.

The switcher is shown/hidden based on which providers are authenticated:
- Both connected → show segmented control
- Only GitHub connected → skip switcher, show GitHub content
- Only Netlify connected → skip switcher, show Netlify content
- Neither → show provider selection landing (new, or repurpose SignInView)

---

## File Inventory

| File | Status |
|---|---|
| `Services/NetlifyAPIClient.swift` | 🔲 Todo |
| `Models/NetlifyDeploy.swift` | 🔲 Todo |
| `ViewModels/NetlifyViewModel.swift` | 🔲 Todo |
| `Views/NetlifySignInView.swift` | 🔲 Todo |
| `Views/NetlifyDeployRow.swift` | 🔲 Todo |
| `Views/NetlifyDeployListView.swift` | 🔲 Todo |
| `Views/ProviderSwitcherView.swift` | 🔲 Todo |
| `GitHubActionsBarApp.swift` | 🔲 Todo |
| `MainPopoverView.swift` | 🔲 Todo |
| `SettingsView.swift` | 🔲 Todo |
| `MenuBarLabel.swift` | 🔲 Todo |
| `KeychainService.swift` | 🔲 Todo |
| `NotificationService.swift` | 🔲 Todo |

---

## Known Constraints

- **No file renames:** Existing files keep their GitHub-centric names (`WorkflowViewModel`, `WorkflowRun`, etc.) to keep the diff additive and PR-friendly.
- **Shared polling interval:** Both providers use the same `pollingInterval` UserDefaults key. This is intentional for simplicity — if independent intervals are needed later, it's a small refactor.
- **Deploy context filtering deferred:** The site picker lets users choose which sites to monitor, but does not yet filter by context (`production` only vs. all contexts). All contexts are shown, matching GitHub's approach of showing all branches.
- **Netlify rate limits:** The Netlify API doesn't expose rate limit headers the way GitHub does. The `rateLimitRemaining` warning UI is GitHub-only for now.
- **Single Netlify account per app instance:** One token stored in Keychain under `"netlify-pat"`. Multi-account support is on the roadmap.

---

## Phases

### Phase 1 — Data Layer (Models + API Client)

**Files:**
- `Models/NetlifyDeploy.swift` (new)
- `Services/NetlifyAPIClient.swift` (new)
- `Services/KeychainService.swift` (add Netlify PAT methods)

**Work:**
- Define `NetlifySite`, `NetlifyDeploy`, `DeployState`, `DeployContext` models
- Implement `NetlifyAPIClient` actor with `fetchAuthenticatedUser`, `fetchSites`, `fetchDeploys(siteId:)`, `fetchAllDeploys(sites:)`
- Add `saveNetlifyPAT` / `retrieveNetlifyPAT` / `deleteNetlifyPAT` to `KeychainService`

**Verification:**
- [ ] Project builds with zero errors
- [ ] `NetlifyAPIClient` can be instantiated in a scratch Swift file / REPL call
- [ ] `KeychainService.saveNetlifyPAT("test")` round-trips correctly (save → retrieve → delete) confirmed via a temporary `print` in app init

---

### Phase 2 — ViewModel

**Files:**
- `ViewModels/NetlifyViewModel.swift` (new)

**Work:**
- Implement full `NetlifyViewModel` mirroring `WorkflowViewModel`: auth, `loadInitialData`, `fetchDeploys`, polling loop, `detectCompletions`, `computeAggregateStatus`, `siteStatuses`
- Wire `selectedSiteIds` to `UserDefaults`

**Verification:**
- [ ] Project builds with zero errors
- [ ] `NetlifyViewModel` can be added as `@State` in `GitHubActionsBarApp` without conflicts
- [ ] With a valid Netlify token hardcoded temporarily, `loadInitialData` populates `sites` and `deploys` (confirmed via `print` or Xcode debugger)
- [ ] Polling loop starts and fires `fetchDeploys` on schedule
- [ ] `aggregateStatus` returns the correct value for a known site in a known state

---

### Phase 3 — Deploy List UI

**Files:**
- `Views/NetlifyDeployRow.swift` (new)
- `Views/NetlifyDeployListView.swift` (new)

**Work:**
- `NetlifyDeployRow`: status dot, deploy title, site name · branch · duration, relative timestamp, tap opens `deploy_url`
- `NetlifyDeployListView`: scrollable list, empty/loading states mirroring `WorkflowRunListView`

**Verification:**
- [ ] Project builds with zero errors
- [ ] `NetlifyDeployListView` renders correctly in a SwiftUI Preview with stubbed `NetlifyDeploy` data
- [ ] Status dot color matches `DeployState` correctly (green/orange/red/gray)
- [ ] Tapping a row opens the correct deploy URL in the browser
- [ ] Duration string formats correctly for sub-minute, multi-minute, and multi-hour deploys

---

### Phase 4 — Sign-In + Settings

**Files:**
- `Views/NetlifySignInView.swift` (new)
- `Views/SettingsView.swift` (add Netlify section)

**Work:**
- `NetlifySignInView`: token entry field, sign-in button, error display — mirrors `SignInView` for GitHub
- `SettingsView`: add Netlify section below GitHub repos — token status, disconnect button, site picker toggles

**Verification:**
- [ ] Project builds with zero errors
- [ ] Entering a valid Netlify token in `NetlifySignInView` authenticates and populates the site list
- [ ] Entering an invalid token shows an error message without crashing
- [ ] Site picker in Settings correctly persists selections across app restarts
- [ ] Disconnect button clears the Netlify token from Keychain and resets `NetlifyViewModel` state
- [ ] Existing GitHub Settings section is fully unaffected

---

### Phase 5 — Provider Switcher + Main Wiring

**Files:**
- `Views/ProviderSwitcherView.swift` (new)
- `Views/MainPopoverView.swift` (modify)
- `GitHubActionsBarApp.swift` (modify)

**Work:**
- `ProviderSwitcherView`: segmented `GitHub | Netlify` control, `activeProvider` binding, auto-hides when only one provider is connected
- `MainPopoverView`: accept `NetlifyViewModel`, inject `ProviderSwitcherView`, conditionally render GitHub or Netlify list based on `activeProvider`
- `GitHubActionsBarApp`: add `@State private var netlifyViewModel = NetlifyViewModel()`, pass to `MainPopoverView`
- Handle the unauthenticated landing state: if neither provider is connected, show a provider selection prompt

**Verification:**
- [ ] Project builds with zero errors
- [ ] With both providers connected, segmented control is visible and switching shows the correct list
- [ ] With only GitHub connected, switcher is hidden and GitHub list shows directly
- [ ] With only Netlify connected, switcher is hidden and Netlify list shows directly
- [ ] `activeProvider` persists across popover close/reopen
- [ ] Unauthenticated state with neither provider connected shows the provider selection prompt without crashing

---

### Phase 6 — Menu Bar + Notifications

**Files:**
- `Views/MenuBarLabel.swift` (modify)
- `Services/NotificationService.swift` (modify)

**Work:**
- `MenuBarLabel`: accept `activeProvider` + `siteStatuses: [SiteStatusItem]`, render bubbles for the active provider only
- `NotificationService`: add `sendDeployCompleted(deploy: NetlifyDeploy, playSound: Bool)` alongside the existing `sendRunCompleted` — no changes to existing method

**Verification:**
- [ ] Project builds with zero errors
- [ ] Menu bar bubbles switch to Netlify site initials when Netlify tab is active
- [ ] Menu bar bubbles switch back to GitHub repo initials when GitHub tab is active
- [ ] A deploy transitioning from `building` → `ready` fires a Mac notification with the correct site name and success sound
- [ ] A deploy transitioning to `error` fires a notification with the failure sound
- [ ] Existing GitHub workflow notifications still fire correctly and are unaffected

---

## Out of Scope (Future)

- **Multi-account Netlify support:** Would require a Keychain strategy for multiple tokens (array or per-team-slug account keys) and a team switcher in Settings.
- **Deploy context filter:** Filter site deploys by context (production-only vs. all). Low priority since the existing `per_page=15` already surfaces the most recent.
- **PR-back to upstream:** `hbourget/Octowatch` may want this eventually. The additive file strategy keeps a potential upstream PR clean, but that's a separate conversation.
