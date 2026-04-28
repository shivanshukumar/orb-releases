# Orb

**Ambient GitHub awareness for AI-native development orchestration.**

Orb lives in your menu bar and watches your GitHub PRs. It categorizes them by what needs your attention, shows CI status and review state at a glance, and emits structured events that AI agents can act on. Review feedback lands, your agent addresses it. CI fails, your agent debugs it. PR approved, it merges automatically. You stay in flow.

### Why not just use GitHub webhooks / Actions / notifications?

| Approach | Limitation |
|----------|-----------|
| **Webhooks** | Require a server you host, per-repo setup, fire for all activity (not just yours) |
| **Actions triggers** | Run in the cloud, can't trigger local AI agents, per-repo workflow files |
| **Notifications API** | Unstructured feed, no diffing, no priority categorization |

Orb is **user-centric and local-first**: one view across all repos, events delivered locally where your agents run, smart priority buckets, granular state diffing, zero infrastructure.

---

## Download

Head to the [Releases](https://github.com/shivanshukumar/orb-releases/releases) page and download the installer for your platform:

| Platform | File | Notes |
|----------|------|-------|
| macOS (Apple Silicon) | `Orb_x.x.x_Apple_Silicon.dmg` | M1 and later |
| macOS (Intel) | `Orb_x.x.x_Intel.dmg` | 2020 and earlier Macs |
| Linux (x86_64) | `Orb_x.x.x_amd64.deb` | Debian-based distros |
| Linux (x86_64) | `Orb-x.x.x-1.x86_64.rpm` | Fedora / RHEL / openSUSE |
| Linux (x86_64) | `Orb_x.x.x_amd64.AppImage` | Any x86_64 distro |
| Linux (ARM64) | `Orb_x.x.x_arm64.deb` | ARM64 Debian-based distros |
| Linux (ARM64) | `Orb-x.x.x-1.aarch64.rpm` | ARM64 Fedora / RHEL / openSUSE |
| Linux (ARM64) | `Orb_x.x.x_aarch64.AppImage` | Raspberry Pi, ARM servers, Docker on Apple Silicon |

**Linux `.deb` compatibility:** Ubuntu 22.04+, Debian 12+, Linux Mint 21+, Pop!_OS 22.04+, elementary OS 7+, Raspberry Pi OS (ARM64), and other Debian-based distros.

**Linux `.rpm` compatibility:** Fedora 38+, RHEL 9+, AlmaLinux/Rocky 9+, openSUSE Leap 15.5+ / Tumbleweed.

**Linux `.AppImage` compatibility:** Runs on any Linux distro — Arch, Manjaro, NixOS, etc. No installation needed, just make it executable and run.

**Linux tray icon note:** Works natively on KDE, XFCE, MATE, Cinnamon, and most non-GNOME desktops. On GNOME (Ubuntu's default), install the [AppIndicator extension](https://extensions.gnome.org/extension/615/appindicator-support/) to see the tray icon — takes 30 seconds.

**Headless use:** No desktop required. Run Orb as a background process and consume events via the JSONL stream or Unix socket — pipe them into scripts and AI agents without a tray icon.

---

## Setup

### 1. Install the GitHub CLI

Orb uses the `gh` CLI to talk to GitHub. You need it installed and authenticated.

**macOS:**
```bash
brew install gh
```

**Linux (Debian/Ubuntu):**
```bash
sudo apt install gh
```

Or download from https://cli.github.com/

### 2. Authenticate with GitHub

```bash
gh auth login
```

Follow the prompts. Choose HTTPS and authenticate via browser. Orb needs this to read your PRs — it never touches your credentials directly.

To verify it worked:
```bash
gh auth status
```

You should see "Logged in to github.com."

### 3. Install Orb

- **macOS:** Open the `.dmg`, drag Orb to Applications.

  > **⚠️ macOS will block the app on first launch.** Orb is not notarized by Apple. You'll see this warning — it does NOT mean the app is damaged. This is normal for open-source apps distributed outside the Mac App Store.

  <p align="center"><img src="docs/macos-gatekeeper-warning.png" width="400" alt="macOS Gatekeeper warning: Orb.app is damaged and can't be opened"></p>

  **Click Cancel**, then run this one-time command to allow Orb to launch:
  ```bash
  xattr -cr /Applications/Orb.app
  ```

- **Linux (.deb):** `sudo dpkg -i Orb_*.deb`
- **Linux (.rpm):** `sudo rpm -i Orb-*.rpm` (or `sudo dnf install ./Orb-*.rpm`)
- **Linux (.AppImage):** `chmod +x Orb_*.AppImage && ./Orb_*.AppImage`

### 4. Launch

Orb appears as an icon in your menu bar (macOS) or system tray (Linux). Click it to see your PRs. That's it.

On first launch, Orb will poll GitHub and populate your PR list. This takes a few seconds.

---

## Software updates

Orb checks for updates automatically — once on startup and every few minutes afterward. When a new version is available:

- **Popup**: a green banner appears at the top with an "Update" button
- **Settings → General**: shows current version, update status, and an "Update & restart" button
- **Headless mode**: an `update_available` event is emitted via JSONL and Unix socket

Clicking "Update & restart" downloads the correct binary for your platform, installs it, and relaunches Orb. On macOS this downloads the DMG, mounts it, copies the `.app` to `/Applications`, and strips the quarantine flag. On Linux it replaces the `.deb` / `.rpm` / `.AppImage` in place.

---

## What you get

**The popup** — click the tray icon to see all your PRs organized by what needs your attention. Each PR shows CI status, review state, author avatar, and age. Sections are collapsible and reorderable. Right-click any PR for quick actions: approve, merge (with strategy picker), request changes, re-request review, mark ready, convert to draft, or copy link.

**7 PR sections:**

| Section | What it means |
|---------|---------------|
| Needs Your Review | Someone requested your review on this PR |
| Action Needed | Your PR needs attention — changes requested, CI failing, or unresolved comments |
| Waiting for Reviewers | Your PR is open and waiting for reviewers |
| Waiting for Author | You reviewed this PR and the author hasn't acted yet |
| Approved | Your PR is approved and ready to merge |
| Drafts | Your draft PRs |
| Recently Merged | PRs you authored or reviewed that merged recently |

**Events for AI agents** — Orb emits structured events (JSONL log, Unix socket, shell hooks) on every PR state change. Wire up your AI agents to respond automatically. See the [agent integration guide](#agent-integration) below.

**Settings** — poll interval, notification preferences, section visibility and order, badge configuration, event hooks with safety rails, repo filtering, software updates. The settings window is resizable — drag any edge if you need more room.

---

## Agent integration

Orb fires events whenever a PR changes state. You can consume them three ways:

**Shell hooks** — configure commands in Settings → Events & Hooks that run when specific events fire. Each hook receives `$ORB_EVENT` (event type), `$ORB_EVENT_JSON` (path to the full event payload), and `$ORB_WORK_DIR` (a per-PR scratch folder that persists across events).

The recommended default is to send the event into a Claude Code [Remote Control](https://code.claude.com/docs/en/remote-control) session, so you can review and steer it from claude.ai/code or the Claude mobile app instead of giving an unattended hook full machine access:

```json
{
  "hooks": {
    "on_ci_status_change": "cd ~/code/$(jq -r '.data.pr.repo_name' \"$ORB_EVENT_JSON\") && claude --remote-control \"Orb: CI failed on #$(jq -r '.data.pr.number' \"$ORB_EVENT_JSON\")\" -p \"$(cat \"$ORB_EVENT_JSON\") — diagnose the CI failure on this PR.\"",
    "on_pr_approved": "gh pr merge $(jq -r '.data.pr.number' \"$ORB_EVENT_JSON\") --squash --auto --repo $(jq -r '.data.pr.repo' \"$ORB_EVENT_JSON\")",
    "on_new_pr": "cd ~/code/$(jq -r '.data.pr.repo_name' \"$ORB_EVENT_JSON\") && claude --remote-control \"Orb: review #$(jq -r '.data.pr.number' \"$ORB_EVENT_JSON\")\" -p \"$(cat \"$ORB_EVENT_JSON\") — do a first-pass review.\""
  }
}
```

For fully unattended hooks (CI, batch jobs), use `--permission-mode dontAsk` with a scoped allowlist instead of `--dangerously-skip-permissions`.

**JSONL log** — every event is appended to a local file. Pipe it into anything:

```bash
# macOS
tail -f ~/Library/Application\ Support/orb/events.jsonl | jq 'select(.event == "bucket_change")'

# Linux
tail -f ~/.local/share/orb/events.jsonl | jq 'select(.event == "bucket_change")'
```

**Unix socket** (macOS/Linux) — real-time streaming with optional event filtering:

```bash
# macOS
echo '{"subscribe": {"events": ["pr_approved", "ci_status_change"]}}' \
  | socat - UNIX-CONNECT:"$HOME/Library/Application Support/orb/orb.sock"

# Linux
echo '{"subscribe": {"events": ["pr_approved", "ci_status_change"]}}' \
  | socat - UNIX-CONNECT:"$HOME/.local/share/orb/orb.sock"
```

### Safety rails for shell hooks

Shell hooks can call paid APIs, take a while, or only matter for specific PRs. Settings → Events & Hooks has four knobs to throttle them — all four affect only the spawned shell command. The JSONL log and Unix socket always emit every event.

| Setting | Default | What it does |
|---|---|---|
| **Trigger labels** | empty (every PR fires) | Comma-separated GitHub labels. Hooks only fire on PRs carrying any of these labels. Global across all hooks |
| **Max fires per hour** | `0` (no limit) | Cap how many times any hook can run in a rolling hour |
| **Max fires per day** | `0` (no limit) | Cap how many times any hook can run in a rolling day |
| **Skip repeats** | on | Fingerprints each run by `(event, PR, state)` and skips identical re-fires. Real changes (new commits, reviews, comments) still fire |

Live counters in the settings show usage. The dedup ledger persists for 7 days at `~/Library/Application Support/orb/delivery-log.jsonl` (`~/.local/share/orb/...` on Linux).

### Per-PR scratch folder — `$ORB_WORK_DIR`

Every PR event includes a stable scratch folder Orb keeps around between events. Multi-step automations can drop notes during one event and pick them back up later for the same PR — without keeping a long-running process alive.

You'll see the path in two places:

- **In the event JSON** at `.data.pr.work_dir` — anyone reading the JSONL log or socket sees it
- **As an environment variable** `$ORB_WORK_DIR` in shell hooks — easier to use in bash

Folder lives at `~/Library/Application Support/orb/work/<pr_node_id>/` (`~/.local/share/orb/work/<pr_node_id>/` on Linux). The PR node ID is GitHub's globally-unique identifier, so the path doesn't break if a repo is renamed or transferred.

```bash
# Hook on on_pr_needs_review:
echo "First-pass review at $(date)" > "$ORB_WORK_DIR/notes.md"
gh pr diff "$(jq -r '.data.pr.number' "$ORB_EVENT_JSON")" > "$ORB_WORK_DIR/diff.patch"

# Hook on on_review_received (later, same PR):
cat "$ORB_WORK_DIR"/*.md \
  | claude --remote-control "Orb #$(jq -r '.data.pr.number' "$ORB_EVENT_JSON")" \
    -p "Continue review with prior notes; address new feedback at $(jq -r '.data.pr.url' "$ORB_EVENT_JSON")"
```

Orb never auto-cleans these folders — they're yours to manage.

### Event types

**Primitive events** — each covers one state dimension:

| Event | When it fires |
|---|---|
| `new_pr` | A PR appears in your view for the first time |
| `pr_removed` | A PR disappears from your view |
| `pr_updated` | PR metadata changed (title, description, labels, base branch) |
| `bucket_change` | A PR moves from one section to another |
| `ci_status_change` | CI status flips (e.g. pending to failure) |
| `commits_pushed` | New commits land on the PR branch |
| `merge_status_change` | PR mergeability changed (e.g. clean to conflicting) |
| `review_received` | A new review is submitted on your PR |
| `review_re_requested` | Review re-requested from a reviewer |
| `comment_added` | New comments on a PR |
| `poll_complete` | A polling cycle finished |
| `action_taken` | You used a quick action from the popup |

**Convenience events** — emitted alongside primitives for common workflows:

| Event | Equivalent to |
|---|---|
| `pr_approved` | `bucket_change` where new section is Approved |
| `pr_merged` | `bucket_change` where new section is Recently Merged |
| `attention_needed` | Summary of all PRs needing your action |
| `update_available` | A new version of Orb is available (includes current/latest version and release URL) |

### Hook interface

Every hook receives three environment variables:

| Variable | Description |
|---|---|
| `ORB_EVENT` | Event type for quick filtering (e.g. `bucket_change`, `new_pr`) |
| `ORB_EVENT_JSON` | Path to a temp JSON file containing the complete event payload |
| `ORB_WORK_DIR` | Per-PR scratch folder that persists across events |

All event data lives in the JSON file. Extract any field with `jq`:

```bash
jq -r '.data.pr.title' "$ORB_EVENT_JSON"     # PR title
jq -r '.data.pr.number' "$ORB_EVENT_JSON"     # PR number
jq -r '.data.pr.repo' "$ORB_EVENT_JSON"       # Repository (owner/name)
jq -r '.data.pr.url' "$ORB_EVENT_JSON"        # Full GitHub URL
jq -r '.data.pr.author' "$ORB_EVENT_JSON"     # PR author
jq -r '.data.pr.bucket' "$ORB_EVENT_JSON"     # Current section
jq -r '.data.pr.work_dir' "$ORB_EVENT_JSON"   # Per-PR scratch folder (same as $ORB_WORK_DIR)
jq -r '.data.trigger' "$ORB_EVENT_JSON"       # What triggered the event
```

This design is intentional — all data flows through the JSON, so the interface is extensible (new fields appear without env var changes) and safe (no shell injection risk from attacker-controlled PR titles or branch names).

### Event envelope

Every event has a consistent top-level structure:

```json
{
  "schema_version": 1,
  "event_id": "evt_68234abc_1f3a",
  "event": "bucket_change",
  "version": 1,
  "timestamp": "2026-04-06T12:00:00Z",
  "viewer": "your-github-username",
  "data": { ... }
}
```

| Field | Purpose |
|---|---|
| `schema_version` | Format version — bump signals breaking changes to the envelope shape |
| `event_id` | Unique per event instance — use for deduplication if replaying or reconnecting |
| `event` | Event type (matches hook names) |
| `version` | Per-event-type version — bump signals changes to that event's `data` shape |
| `timestamp` | ISO 8601 UTC |
| `viewer` | Your GitHub username |
| `data` | Event-specific payload |

---

## Settings reference

| Setting | Default | Description |
|---|---|---|
| Software updates | Check + install | Checks every few minutes and on startup; one-click update & restart |
| Poll interval | 120 seconds | How often Orb checks GitHub (minimum 30s) |
| Merged PR window | 7 days | How far back to show recently merged PRs |
| Max PR age | No limit | Hide PRs older than N days |
| Blocked repos | None | Repos to exclude (new repos visible by default) |
| Badge sections | Needs Your Review, Action Needed, Waiting for Reviewers | Which sections count toward the tray badge |
| Notifications | On | Desktop notifications for state changes |
| Notification sound | On | Play a sound with notifications |
| Events | On | Enable the event system (JSONL + socket + hooks) |
| Hook timeout | 15 seconds | Kill hook commands that exceed this |
| Trigger labels | empty | Hooks only fire on PRs carrying these labels (global, OR-list) |
| Max fires per hour | `0` (no limit) | Rolling hour cap on hook fires |
| Max fires per day | `0` (no limit) | Rolling day cap on hook fires |
| Skip repeats | On | Skip hook re-fires when the underlying state hasn't changed |

---

## Changelog

### v0.1.4

- **Safety rails for shell hooks** — trigger labels, per-hour and per-day caps, and a skip-repeats fingerprint so hooks don't run more than they should
- **Per-PR scratch folder** — `$ORB_WORK_DIR` and `.data.pr.work_dir` give every hook a stable per-PR folder that survives across events
- **Auto-update install no longer hangs** — installer used to stall waiting for the old process to release the app bundle
- **Auto-update restart works** — the new version launches automatically after install completes
- **Update button shows a spinner** — visual feedback while the install is running
- **Safer Claude hook examples** — in-app examples now use Claude's Remote Control mode by default instead of unattended full-machine access
- **Resizable settings window** — drag any edge to expand. The current 720×520 is the new minimum

### v0.1.3

- **Re-request review** — right-click any PR where someone requested changes to re-request their review
- **Mark as ready** — right-click draft PRs to mark them ready for review
- **Convert to draft** — right-click in-progress PRs to convert back to draft
- **SVG toast icons** — replaced emoji icons in toasts with crisp SVGs
- **Undo countdown ring** — visual countdown shows exactly when an action will fire

### v0.1.2

- **Richer event payloads** — event JSON now includes CI status, review details, file stats, and more
- **Hi-res icons** — app icon, tray icon, and animation frames upscaled for Retina/HiDPI
- **Refresh indicator** — tray header pulses during background refresh

### v0.1.1

- **Auto-update** — checks for updates on startup and hourly; one-click "Update & restart"
- **Event schema hardening** — `schema_version` and unique `event_id` on every event for forward compatibility and deduplication
- **Secure hook interface** — hooks receive only `ORB_EVENT` and `ORB_EVENT_JSON` (data via `jq`), eliminating shell injection from attacker-controlled PR metadata
- **`update_available` event** — headless consumers are notified when a new version is available

### v0.1.0

Initial release.

- Command center popup with quick actions (approve, merge with strategy picker, request changes, copy link)
- 7 PR sections with smart categorization (Action Needed detects changes requested, CI failing, unresolved comments)
- Event system with 12 primitive events + 3 convenience wrappers, delivered via JSONL log, Unix socket, and shell hooks
- Configurable settings: poll interval, notifications, section visibility/order/badge, repo filtering, event hooks
- Soft dark theme with light mode support
- macOS and Linux support

---

## Troubleshooting

**Orb shows an error icon (X) in the tray**
- Run `gh auth status` to check if the GitHub CLI is authenticated
- Open Settings and check the GitHub CLI status section
- Click "Try Again" after fixing the issue

**"Orb.app is damaged" / "Unidentified developer" on macOS**
```bash
xattr -cr /Applications/Orb.app
```
See [Install step 3](#3-install-orb) for details.

**No PRs showing up**
- Make sure you have open PRs or recent review requests on GitHub
- Check that the repos aren't in your blocked list (Settings)
- Wait for one poll cycle (check the timestamp in the popup header)

**My hook isn't firing**
- If you set Trigger Labels, only PRs carrying one of those labels will fire hooks. Clear the field to fire on every PR
- If you hit a per-hour or per-day cap, hooks are silently skipped until older runs roll off — the live counter in Settings shows usage
- If "Skip repeats" is on (default), hooks won't re-fire for an unchanged state — a real change (new commit, review, comment) produces a different fingerprint and will fire

**Notifications keep repeating**
- Orb has a 30-minute cooldown per PR per section. If you're still seeing repeats, file an issue.

---

## Feedback and issues

This is an early release. If you run into problems or have feature requests, please open an issue on this repository.

---

## Disclaimer

Orb is an independent project and is not affiliated with, endorsed by, or supported by GitHub, Anthropic, or any other company. It is provided as-is with no warranty. By using Orb, you accept responsibility for its operation in your environment. See the full [LICENSE](LICENSE) for details.
