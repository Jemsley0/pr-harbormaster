# PRflow — GitHub PR Dashboard

A single local HTML file that shows all your GitHub PRs in four action-oriented columns.

## Setup

1. **Create a Personal Access Token** at https://github.com/settings/tokens
   - Select **Classic token**
   - Required scopes: `repo` and `read:user`
   - Set an expiration date (90 days recommended)

2. **Open `dashboard.html`** in your browser — just double-click it or run:
   ```
   open ~/github-pr-dashboard/dashboard.html
   ```

3. **Paste your token** into the setup screen and click Save.

That's it. The dashboard refreshes every 5 minutes automatically.

## Columns

| Column | What's in it |
|---|---|
| 💤 Stale | Your PRs (draft or submitted) with no activity for 14+ days |
| 🔨 In Progress | Your active draft PRs |
| 🔍 Reviews | Your PRs waiting on reviewers (top) + PRs assigned to you (bottom) |
| 🚀 Ready to Deploy | Your approved PRs with no outstanding change requests |

## Resetting your token

Click **Reset token** in the header. Your token is only stored in your browser's `localStorage` — it never leaves your machine.
