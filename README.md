# PR Harbormaster — GitHub PR Dashboard

A single local HTML file that shows all your GitHub PRs in four action-oriented columns.

It runs entirely in your own browser. There are no shared secrets and no server: each
person authenticates with their own GitHub token, which is stored only in their own
browser and is sent only to `api.github.com`. Sharing the file with a teammate is safe.

## Setup

1. **Create a token** at https://github.com/settings/personal-access-tokens/new

   Use a **fine-grained token** with read-only access (preferred — it limits the blast
   radius if the token ever leaks):
   - **Resource owner:** the org or account whose PRs you want to see
   - **Repository access:** all repositories (or just the ones you care about)
   - **Repository permissions** (all **Read-only**):
     - Metadata
     - Pull requests
     - Commit statuses
     - Checks
   - **Expiration:** 90 days

   > If your org hasn't enabled fine-grained tokens, you can fall back to a **Classic
   > token** with the `repo` and `read:user` scopes — but note `repo` grants full
   > read/write to all your private repos, so prefer the fine-grained option.

2. **Open `dashboard.html`** in your browser — just double-click it or run:
   ```
   open dashboard.html
   ```

3. **Paste your token** into the setup screen and click Save.

That's it. The dashboard refreshes every 5 minutes automatically.

## Columns

| Column | What's in it |
|---|---|
| 💤 Stale | Your PRs (draft or submitted) with no activity for 14+ days |
| 🔨 In Progress | Your active draft PRs |
| 🔍 Reviews | Your PRs waiting on reviewers (top) + PRs assigned to you (bottom) |
| 🚢 Ready to Ship | Your approved PRs with no outstanding change requests |

## Resetting your token

Click **Reset token** in the header. Your token is only stored in your browser's `localStorage` — it never leaves your machine.
