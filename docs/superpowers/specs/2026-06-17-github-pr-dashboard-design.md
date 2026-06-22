# GitHub PR Dashboard — Design Spec

**Date:** 2026-06-17  
**Status:** Approved  

---

## Overview

A single local HTML file that serves as a personal PR launching point. Opens in any browser — no server, no install. Shows all GitHub PRs involving the user in four action-oriented columns, refreshes automatically every 5 minutes, and opens any PR in a new tab on click.

---

## Columns

Four columns in this left-to-right order:

| # | Column | Emoji | Logic |
|---|---|---|---|
| 1 | **Stale** | 💤 | PRs I authored (draft or submitted) with no activity for **14+ days** |
| 2 | **In Progress** | 🔨 | PRs I authored, `isDraft: true`, activity within 14 days |
| 3 | **Reviews** | 🔍 | Two sections — my submitted PRs awaiting review (top) + open PRs where I'm a requested reviewer and not the author (bottom) |
| 4 | **Ready to Deploy** | 🚀 | PRs I authored, `isDraft: false`, at least one `APPROVED` review, no `CHANGES_REQUESTED` outstanding |

A PR appears in exactly one column. Priority when a PR could match multiple:

**Stale > Ready to Deploy > Reviews (awaiting) > In Progress**

"Activity" is defined as the PR's `updatedAt` timestamp from the GitHub API. Stale threshold is 14 days.

Within each column, cards are sorted by `updatedAt` descending — most recently active at the top.

---

## Reviews Column Detail

The Reviews column has two sub-sections separated by a labeled divider:

**Section 1 — My PRs awaiting review**  
My submitted (non-draft) PRs that are not approved and not stale. Cards show a purple `authored by me` badge.

**Section 2 — PRs I need to review**  
Open PRs where I'm a requested reviewer and I'm not the author. Cards show an amber `by {username}` badge with the PR author's login.

Each section is sorted by `updatedAt` descending independently.

---

## Card Design (Standard density)

Each card shows:
- **Title** — PR title, bold, full text (wraps)
- **Meta line** — `{repo-name} · #{number} · {relative age}` (e.g. "4d ago")
- **Tags row:**
  - CI status badge: `✓ CI passing` (green) / `✗ CI failing` (red) / `⟳ CI running` (yellow) / `⊘ no CI` (gray)
  - Reviewer count: `👤 N` (gray) — shown when > 0
  - Comment count: `💬 N` (gray)
  - **Stale column:** `💤 Nd idle` amber badge showing days since last activity; `draft` gray badge if still a draft
  - **In Progress column:** `draft` gray badge
  - **Reviews column:** `authored by me` purple badge (top section) or `by {login}` amber badge (bottom section)
  - **Ready to Deploy column:** `✓ approved` green badge replaces reviewer count

Clicking anywhere on a card opens the PR's GitHub URL in a new tab.

---

## Architecture

**File:** `~/github-pr-dashboard/dashboard.html`  
**Runtime:** Open directly in any browser (`file://` URL)  
**Dependencies:** None — vanilla HTML/CSS/JS only  
**API:** GitHub GraphQL API v4 (`https://api.github.com/graphql`)  

### Authentication

On first load, if no PAT is found in `localStorage`, a setup screen is shown:
- Explains that a GitHub Personal Access Token is required
- Lists required scopes: `repo`, `read:user`
- Text input + Save button
- PAT stored under the key `gh_prd_pat` in `localStorage`
- A small "Reset token" link in the header clears it and re-shows the setup screen

### Data fetching

Two GraphQL queries sent in a single batched request on each refresh:

**Query 1 — My open PRs:**
```graphql
search(query: "is:open is:pr author:@me", type: ISSUE, first: 50) {
  nodes {
    ... on PullRequest {
      title, number, url, createdAt, updatedAt, isDraft
      comments { totalCount }
      reviews(last: 10) { nodes { state } }
      reviewRequests { totalCount }
      repository { nameWithOwner }
      headRef {
        target {
          ... on Commit { statusCheckRollup { state } }
        }
      }
    }
  }
}
```

**Query 2 — Review requests:**
```graphql
search(query: "is:open is:pr review-requested:@me", type: ISSUE, first: 50) {
  nodes {
    ... on PullRequest {
      title, number, url, updatedAt
      author { login }
      comments { totalCount }
      repository { nameWithOwner }
      headRef {
        target {
          ... on Commit { statusCheckRollup { state } }
        }
      }
    }
  }
}
```

### Column assignment logic (applied client-side)

```
staleThreshold = 14 days

# Track my PR IDs to deduplicate against Query 2
myPrIds = new Set()

for each PR in Query 1 results (sorted by updatedAt desc):
  myPrIds.add(pr.number + "/" + pr.repository.nameWithOwner)

  if updatedAt < now - staleThreshold:
    → Stale                                      # drafts and submitted both go here when idle
  else if reviews contain APPROVED and no CHANGES_REQUESTED:
    → Ready to Deploy
  else if isDraft:
    → In Progress
  else:
    → Reviews (top section: "My PRs awaiting review")

for each PR in Query 2 results (sorted by updatedAt desc):
  if pr.id is in myPrIds → skip                  # dedup: my PRs win
  else → Reviews (bottom section: "PRs I need to review")
```

### Refresh

- Auto-refresh via `setInterval` every **5 minutes**
- Manual refresh button in header triggers the same fetch
- Header shows "Updated N min ago" timestamp, updated after each successful fetch

---

## Visual Design

- Dark-themed, CSS custom properties for all colors
- **Stale:** amber tint column background + amber border on cards
- **Ready to Deploy:** green tint column background + green border on cards
- **Reviews:** accent-color column title; cards use purple or amber author badge to distinguish ownership
- Hover state on all cards: accent-color border + subtle ring
- Each column section (in Reviews) has a small uppercase divider label
- Empty state message per column / section when no PRs match

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Network error | Inline error banner with retry button; last-known data stays visible |
| 401 Unauthorized | Clear PAT, show setup screen with "token invalid" message |
| 403 / 429 rate limit | "Rate limited — retrying in Xm" banner; auto-refresh paused until reset |
| Empty column | Per-column empty state (e.g. "Merge it! 🎉" for Ready to Deploy) |

---

## File Structure

```
~/github-pr-dashboard/
  dashboard.html          ← the entire app
  README.md               ← how to get a PAT and open the file
  docs/
    superpowers/
      specs/
        2026-06-17-github-pr-dashboard-design.md
```

No build step, no package.json, no node_modules.

---

## Out of Scope (v1)

- Filtering by repo or org
- Configurable staleness threshold (hardcoded at 14 days)
- Closed / merged PR history
- Notifications or desktop alerts
- Multi-account GitHub support
