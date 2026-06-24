# GitHub PR Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `dashboard.html` file that shows all of a user's GitHub PRs in four action-oriented columns, sorted by recency, auto-refreshing every 5 minutes.

**Architecture:** One self-contained HTML file — no build step, no dependencies, no server. Vanilla HTML + CSS + JS. Calls GitHub GraphQL API v4 directly from the browser using a PAT stored in `localStorage`. Column assignment and sorting happen client-side after each fetch.

**Tech Stack:** HTML5, CSS3 (custom properties), vanilla JS (ES2020+), GitHub GraphQL API v4 (`https://api.github.com/graphql`)

---

## File Structure

```
~/github-pr-dashboard/
  dashboard.html    ← entire app (HTML + CSS + JS in one file)
  README.md         ← PAT setup instructions
```

Everything lives in `dashboard.html`. The script section is organized into these logical sections (in order):
1. Constants
2. Auth utilities
3. GitHub API
4. Data processing (pure functions)
5. Rendering (pure functions)
6. UI state management
7. Refresh loop
8. Init

---

## Task 1: Static shell and CSS

**Files:**
- Create: `~/github-pr-dashboard/dashboard.html`

- [ ] **Step 1: Create `dashboard.html` with the full static layout**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>PR Harbormaster</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --bg:        #0d1117;
      --surface-1: #161b22;
      --surface-2: #21262d;
      --surface-3: #30363d;
      --border:    #30363d;
      --text:      #e6edf3;
      --text-muted:#7d8590;
      --accent:    #58a6ff;
      --green:     #3fb950;
      --red:       #f85149;
      --yellow:    #d29922;
      --amber:     #f59e0b;
      --purple:    #a371f7;
    }

    body {
      background: var(--bg);
      color: var(--text);
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      font-size: 13px;
      min-height: 100vh;
    }

    /* ── Setup screen ── */
    .setup-screen {
      display: flex; align-items: center; justify-content: center;
      min-height: 100vh; padding: 24px;
    }
    .setup-card {
      background: var(--surface-1); border: 1px solid var(--border);
      border-radius: 12px; padding: 32px; max-width: 440px; width: 100%;
    }
    .setup-card h1 { font-size: 20px; margin-bottom: 6px; }
    .setup-card h1 span { color: var(--accent); }
    .setup-card p { color: var(--text-muted); line-height: 1.6; margin-bottom: 16px; font-size: 13px; }
    .setup-card .scopes {
      background: var(--surface-2); border: 1px solid var(--border);
      border-radius: 6px; padding: 10px 12px; margin-bottom: 16px; font-size: 12px;
    }
    .setup-card .scopes code { color: var(--accent); }
    .setup-input {
      width: 100%; background: var(--surface-2); border: 1px solid var(--border);
      border-radius: 6px; padding: 8px 12px; color: var(--text); font-size: 13px;
      font-family: monospace; margin-bottom: 10px; outline: none;
    }
    .setup-input:focus { border-color: var(--accent); }
    .setup-btn {
      width: 100%; background: var(--accent); color: #0d1117; border: none;
      border-radius: 6px; padding: 9px; font-size: 13px; font-weight: 600;
      cursor: pointer;
    }
    .setup-btn:hover { opacity: .9; }
    .setup-error { color: var(--red); font-size: 12px; margin-top: 8px; min-height: 18px; }

    /* ── Dashboard shell ── */
    .db-shell { display: flex; flex-direction: column; height: 100vh; overflow: hidden; }

    .db-header {
      display: flex; align-items: center; justify-content: space-between;
      padding: 10px 16px; border-bottom: 1px solid var(--border);
      background: var(--surface-1); flex-shrink: 0;
    }
    .db-header-left { display: flex; align-items: center; gap: 10px; }
    .db-logo { font-weight: 700; font-size: 14px; letter-spacing: -.3px; }
    .db-logo span { color: var(--accent); }
    .db-user {
      font-size: 11px; color: var(--text-muted);
      background: var(--surface-3); padding: 2px 8px; border-radius: 20px;
    }
    .db-header-right { display: flex; align-items: center; gap: 10px; }
    .db-updated { font-size: 11px; color: var(--text-muted); }
    .db-btn {
      font-size: 11px; background: var(--surface-2); border: 1px solid var(--border);
      border-radius: 6px; padding: 4px 10px; cursor: pointer; color: var(--text);
    }
    .db-btn:hover { border-color: var(--accent); color: var(--accent); }

    /* ── Error banner ── */
    .error-banner {
      background: rgba(248, 81, 73, .1); border-bottom: 1px solid rgba(248,81,73,.3);
      padding: 8px 16px; font-size: 12px; color: var(--red);
      display: flex; align-items: center; justify-content: space-between;
    }
    .retry-btn {
      background: none; border: 1px solid var(--red); border-radius: 4px;
      padding: 2px 8px; color: var(--red); cursor: pointer; font-size: 11px;
    }

    /* ── Column layout ── */
    .db-cols {
      display: grid; grid-template-columns: repeat(4, 1fr);
      flex: 1; overflow: hidden;
    }
    .db-col {
      border-right: 1px solid var(--border);
      display: flex; flex-direction: column; overflow: hidden;
    }
    .db-col:last-child { border-right: none; }
    .db-col-stale  { background: rgba(245,158,11,.02); }
    .db-col-deploy { background: rgba(63,185,80,.02); }

    .db-col-header {
      padding: 10px 10px 8px; border-bottom: 1px solid var(--border);
      flex-shrink: 0;
    }
    .db-col-title {
      font-weight: 600; font-size: 10.5px; text-transform: uppercase;
      letter-spacing: .5px; color: var(--text-muted);
      display: flex; align-items: center; gap: 6px;
    }
    .db-col-stale  .db-col-title { color: var(--amber); }
    .db-col-deploy .db-col-title { color: var(--green); }
    .db-col-reviews .db-col-title { color: var(--accent); }

    .db-count {
      font-size: 10px; background: var(--surface-3);
      border-radius: 10px; padding: 1px 6px; font-weight: 600;
      color: var(--text-muted);
    }
    .db-stale-note {
      font-size: 10px; color: var(--text-muted); text-align: center;
      padding: 4px 0 2px; font-style: italic; opacity: .6;
    }

    .db-cards { padding: 8px 8px; overflow-y: auto; flex: 1; }

    /* ── Cards ── */
    .db-card {
      background: var(--surface-1); border: 1px solid var(--border);
      border-radius: 7px; padding: 9px 10px; margin-bottom: 7px;
      cursor: pointer; transition: border-color .12s, box-shadow .12s;
      text-decoration: none; display: block;
    }
    .db-card:hover { border-color: var(--accent); box-shadow: 0 0 0 1px var(--accent); }
    .db-card-stale  { border-color: rgba(245,158,11,.25); background: rgba(245,158,11,.03); }
    .db-card-stale:hover  { border-color: var(--amber); box-shadow: 0 0 0 1px var(--amber); }
    .db-card-deploy { border-color: rgba(63,185,80,.3); background: rgba(63,185,80,.04); }
    .db-card-deploy:hover { border-color: var(--green); box-shadow: 0 0 0 1px var(--green); }

    .db-card-title { font-weight: 600; font-size: 11.5px; line-height: 1.4; margin-bottom: 3px; color: var(--text); }
    .db-card-meta  { font-size: 10px; color: var(--text-muted); margin-bottom: 7px; }
    .db-card-tags  { display: flex; gap: 4px; flex-wrap: wrap; align-items: center; }

    /* ── Tags & badges ── */
    .tag, .badge {
      font-size: 10px; padding: 2px 6px; border-radius: 10px; font-weight: 500;
    }
    .tag-green  { background: rgba(63,185,80,.15);  color: var(--green); }
    .tag-red    { background: rgba(248,81,73,.15);  color: var(--red); }
    .tag-yellow { background: rgba(210,153,34,.15); color: var(--yellow); }
    .tag-gray   { background: var(--surface-3);     color: var(--text-muted); }
    .badge-stale  { background: rgba(245,158,11,.2);  color: var(--amber); font-weight: 600; }
    .badge-draft  { background: var(--surface-3);     color: var(--text-muted); font-weight: 600; }
    .badge-me     { background: rgba(163,113,247,.15); color: var(--purple); font-weight: 600; }
    .badge-them   { background: rgba(245,158,11,.12); color: #d97706; font-weight: 600; }
    .badge-deploy { background: rgba(63,185,80,.2);   color: var(--green); font-weight: 600; }

    /* ── Reviews column sections ── */
    .reviews-divider {
      font-size: 9.5px; font-weight: 700; text-transform: uppercase;
      letter-spacing: .6px; color: var(--text-muted); opacity: .5;
      padding: 8px 0 5px; border-bottom: 1px solid var(--border);
      margin-bottom: 7px;
    }
    .reviews-divider:not(:first-child) { margin-top: 12px; }

    /* ── Empty states ── */
    .db-empty {
      color: var(--text-muted); font-size: 11px;
      text-align: center; padding: 24px 0; opacity: .5;
    }

    /* ── Hidden utility ── */
    .hidden { display: none !important; }
  </style>
</head>
<body>

  <!-- Setup screen -->
  <div id="setup-screen" class="setup-screen hidden">
    <div class="setup-card">
      <h1>PR<span>flow</span></h1>
      <p>Enter a GitHub Personal Access Token to load your pull requests. The token is stored locally in your browser and never leaves your machine.</p>
      <div class="scopes">
        Required scopes: <code>repo</code> and <code>read:user</code>
      </div>
      <input id="pat-input" class="setup-input" type="password"
             placeholder="ghp_xxxxxxxxxxxxxxxxxxxx" autocomplete="off" />
      <button class="setup-btn" onclick="saveAndInit()">Save token</button>
      <div id="setup-error" class="setup-error"></div>
    </div>
  </div>

  <!-- Dashboard -->
  <div id="dashboard" class="db-shell hidden">
    <div class="db-header">
      <div class="db-header-left">
        <div class="db-logo">PR<span>flow</span></div>
        <div id="db-user" class="db-user"></div>
      </div>
      <div class="db-header-right">
        <span id="db-updated" class="db-updated"></span>
        <button class="db-btn" onclick="manualRefresh()">⟳ Refresh</button>
        <button class="db-btn" onclick="resetToken()">Reset token</button>
      </div>
    </div>

    <div id="error-banner" class="error-banner hidden">
      <span id="error-msg"></span>
      <button class="retry-btn" onclick="manualRefresh()">Retry</button>
    </div>

    <div class="db-cols">
      <div class="db-col db-col-stale">
        <div class="db-col-header">
          <div class="db-col-title">
            💤 Stale <span id="count-stale" class="db-count">0</span>
          </div>
        </div>
        <div class="db-stale-note">No activity for 14+ days</div>
        <div id="cards-stale" class="db-cards"></div>
      </div>

      <div class="db-col">
        <div class="db-col-header">
          <div class="db-col-title">
            🔨 In Progress <span id="count-inprogress" class="db-count">0</span>
          </div>
        </div>
        <div id="cards-inprogress" class="db-cards"></div>
      </div>

      <div class="db-col db-col-reviews">
        <div class="db-col-header">
          <div class="db-col-title">
            🔍 Reviews <span id="count-reviews" class="db-count">0</span>
          </div>
        </div>
        <div id="cards-reviews" class="db-cards"></div>
      </div>

      <div class="db-col db-col-deploy">
        <div class="db-col-header">
          <div class="db-col-title">
            🚀 Ready to Deploy <span id="count-ready" class="db-count">0</span>
          </div>
        </div>
        <div id="cards-ready" class="db-cards"></div>
      </div>
    </div>
  </div>

  <script>
    // placeholder — JS added in later tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: Open the file in a browser and verify the layout**

```
open ~/github-pr-dashboard/dashboard.html
```

Expected: blank dark page (both panels are `.hidden`). No console errors.

- [ ] **Step 3: Commit**

```bash
cd ~/github-pr-dashboard && git init && git add dashboard.html
git commit -m "feat: static HTML shell and CSS for PR dashboard"
```

---

## Task 2: Auth flow and UI state

**Files:**
- Modify: `~/github-pr-dashboard/dashboard.html` (replace `<script>` placeholder)

- [ ] **Step 1: Replace the `<script>` placeholder with the constants, auth utilities, and UI state functions**

Replace the `<script>// placeholder — JS added in later tasks</script>` block with:

```html
<script>
  // ── Constants ──────────────────────────────────────────────────────────
  const PAT_KEY = 'gh_prd_pat';
  const STALE_DAYS = 14;
  const REFRESH_MS = 5 * 60 * 1000;

  // ── Auth utilities ─────────────────────────────────────────────────────
  function getPAT()        { return localStorage.getItem(PAT_KEY); }
  function savePAT(token)  { localStorage.setItem(PAT_KEY, token.trim()); }
  function clearPAT()      { localStorage.removeItem(PAT_KEY); }

  // ── UI state ───────────────────────────────────────────────────────────
  function showSetup(errorMsg = '') {
    document.getElementById('setup-screen').classList.remove('hidden');
    document.getElementById('dashboard').classList.add('hidden');
    document.getElementById('setup-error').textContent = errorMsg;
    document.getElementById('pat-input').value = '';
    setTimeout(() => document.getElementById('pat-input').focus(), 50);
  }

  function showDashboard(username) {
    document.getElementById('setup-screen').classList.add('hidden');
    document.getElementById('dashboard').classList.remove('hidden');
    document.getElementById('db-user').textContent = `@${username}`;
  }

  function showError(msg) {
    const banner = document.getElementById('error-banner');
    document.getElementById('error-msg').textContent = msg;
    banner.classList.remove('hidden');
  }

  function hideError() {
    document.getElementById('error-banner').classList.add('hidden');
  }

  function updateTimestamp() {
    document.getElementById('db-updated').textContent =
      `Updated ${new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}`;
  }

  // ── PAT setup handlers ─────────────────────────────────────────────────
  function saveAndInit() {
    const token = document.getElementById('pat-input').value.trim();
    if (!token) {
      document.getElementById('setup-error').textContent = 'Please enter a token.';
      return;
    }
    savePAT(token);
    init();
  }

  document.getElementById('pat-input').addEventListener('keydown', e => {
    if (e.key === 'Enter') saveAndInit();
  });

  function resetToken() {
    clearPAT();
    if (refreshTimer) clearInterval(refreshTimer);
    showSetup();
  }

  // ── Refresh placeholders (filled in later tasks) ──────────────────────
  let refreshTimer = null;
  function manualRefresh() { /* filled in Task 6 */ }

  // ── Init ───────────────────────────────────────────────────────────────
  function init() {
    const pat = getPAT();
    if (!pat) { showSetup(); return; }
    showDashboard('loading…');
    // fetch will be wired up in Task 6
  }

  init();
</script>
```

- [ ] **Step 2: Verify auth flow in the browser**

Reload the file. Expected:
- Setup screen is visible (no PAT in localStorage yet)
- Type any string into the input and press Enter or click Save → setup screen hides, dashboard shows with "@loading…"
- Click "Reset token" → setup screen reappears

Open DevTools → Application → Local Storage → `file://` — confirm `gh_prd_pat` is set then cleared on reset.

- [ ] **Step 3: Commit**

```bash
git add dashboard.html
git commit -m "feat: auth flow — PAT setup screen, localStorage get/save/clear"
```

---

## Task 3: GitHub GraphQL fetch function

**Files:**
- Modify: `~/github-pr-dashboard/dashboard.html`

- [ ] **Step 1: Add the `fetchDashboard` function after the `updateTimestamp` function**

Add this block directly before the `// ── PAT setup handlers` comment:

```javascript
  // ── GitHub API ─────────────────────────────────────────────────────────
  const GQL_QUERY = `
    query Dashboard {
      viewer { login }
      myPRs: search(query: "is:open is:pr author:@me", type: ISSUE, first: 50) {
        nodes {
          ... on PullRequest {
            title number url updatedAt isDraft
            comments { totalCount }
            reviews(last: 10) { nodes { state } }
            reviewRequests { totalCount }
            repository { nameWithOwner }
            headRef { target { ... on Commit { statusCheckRollup { state } } } }
          }
        }
      }
      reviewRequests: search(query: "is:open is:pr review-requested:@me", type: ISSUE, first: 50) {
        nodes {
          ... on PullRequest {
            title number url updatedAt
            author { login }
            comments { totalCount }
            repository { nameWithOwner }
            headRef { target { ... on Commit { statusCheckRollup { state } } } }
          }
        }
      }
    }
  `;

  async function fetchDashboard(pat) {
    const res = await fetch('https://api.github.com/graphql', {
      method: 'POST',
      headers: {
        Authorization: `bearer ${pat}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ query: GQL_QUERY }),
    });

    if (res.status === 401) throw Object.assign(new Error('Invalid token'), { code: 'UNAUTHORIZED' });
    if (res.status === 403 || res.status === 429) throw Object.assign(new Error('Rate limited'), { code: 'RATE_LIMITED' });
    if (!res.ok) throw Object.assign(new Error(`HTTP ${res.status}`), { code: 'NETWORK_ERROR' });

    const json = await res.json();
    if (json.errors?.length) throw Object.assign(new Error(json.errors[0].message), { code: 'GRAPHQL_ERROR' });

    const { viewer, myPRs, reviewRequests } = json.data;
    return {
      login: viewer.login,
      myPRs: myPRs.nodes.filter(n => n?.title),
      reviewRequestedPRs: reviewRequests.nodes.filter(n => n?.title),
    };
  }
```

- [ ] **Step 2: Test the fetch function from the browser console**

Open the file, complete the PAT setup with a real token. Then in DevTools console:

```javascript
fetchDashboard(localStorage.getItem('gh_prd_pat')).then(d => console.log(d))
```

Expected: object with `{ login: 'Jemsley0', myPRs: [...], reviewRequestedPRs: [...] }`

Test error handling:

```javascript
fetchDashboard('bad_token').catch(e => console.log(e.code))
// Expected: "UNAUTHORIZED"
```

- [ ] **Step 3: Commit**

```bash
git add dashboard.html
git commit -m "feat: GitHub GraphQL fetch function with error codes"
```

---

## Task 4: Column assignment and sorting logic

**Files:**
- Modify: `~/github-pr-dashboard/dashboard.html`

- [ ] **Step 1: Add data processing functions after the `fetchDashboard` function**

Add this block directly before the `// ── PAT setup handlers` comment:

```javascript
  // ── Data processing ────────────────────────────────────────────────────
  const STALE_MS = STALE_DAYS * 24 * 60 * 60 * 1000;

  function isStale(updatedAt) {
    return Date.now() - new Date(updatedAt).getTime() > STALE_MS;
  }

  function isApproved(reviews) {
    const nodes = reviews?.nodes ?? [];
    return nodes.some(r => r.state === 'APPROVED') &&
          !nodes.some(r => r.state === 'CHANGES_REQUESTED');
  }

  function byUpdatedDesc(a, b) {
    return new Date(b.updatedAt) - new Date(a.updatedAt);
  }

  function processData(myPRs, reviewRequestedPRs) {
    const columns = { stale: [], inProgress: [], reviewsTop: [], reviewsBottom: [], ready: [] };
    const myPrKeys = new Set();

    for (const pr of myPRs) {
      myPrKeys.add(`${pr.number}/${pr.repository.nameWithOwner}`);
      if (isStale(pr.updatedAt))      columns.stale.push(pr);
      else if (isApproved(pr.reviews)) columns.ready.push(pr);
      else if (pr.isDraft)             columns.inProgress.push(pr);
      else                             columns.reviewsTop.push(pr);
    }

    for (const pr of reviewRequestedPRs) {
      if (!myPrKeys.has(`${pr.number}/${pr.repository.nameWithOwner}`)) {
        columns.reviewsBottom.push(pr);
      }
    }

    for (const arr of Object.values(columns)) arr.sort(byUpdatedDesc);
    return columns;
  }
```

- [ ] **Step 2: Verify logic with mock data in the browser console**

Reload the file (with PAT already saved). In DevTools console:

```javascript
const now = new Date().toISOString();
const old = new Date(Date.now() - 15 * 86400000).toISOString(); // 15 days ago

const mockMyPRs = [
  { number: 1, repository: { nameWithOwner: 'org/repo' }, updatedAt: now,  isDraft: true,  reviews: { nodes: [] } },
  { number: 2, repository: { nameWithOwner: 'org/repo' }, updatedAt: old,  isDraft: false, reviews: { nodes: [] } },
  { number: 3, repository: { nameWithOwner: 'org/repo' }, updatedAt: now,  isDraft: false, reviews: { nodes: [{ state: 'APPROVED' }] } },
  { number: 4, repository: { nameWithOwner: 'org/repo' }, updatedAt: now,  isDraft: false, reviews: { nodes: [] } },
];
const mockReview = [
  { number: 5, repository: { nameWithOwner: 'org/other' }, updatedAt: now, author: { login: 'alice' } },
  { number: 1, repository: { nameWithOwner: 'org/repo' }, updatedAt: now, author: { login: 'me' } }, // dup — should be skipped
];

const cols = processData(mockMyPRs, mockReview);
console.assert(cols.inProgress.length === 1 && cols.inProgress[0].number === 1, 'active draft → inProgress');
console.assert(cols.stale.length === 1      && cols.stale[0].number === 2,      'old PR → stale');
console.assert(cols.ready.length === 1      && cols.ready[0].number === 3,      'approved → ready');
console.assert(cols.reviewsTop.length === 1 && cols.reviewsTop[0].number === 4, 'submitted → reviewsTop');
console.assert(cols.reviewsBottom.length === 1 && cols.reviewsBottom[0].number === 5, 'others → reviewsBottom');
console.log('All assertions passed ✓');
```

Expected: `All assertions passed ✓` (no assertion failures)

- [ ] **Step 3: Commit**

```bash
git add dashboard.html
git commit -m "feat: column assignment and sorting logic with stale/approved/draft detection"
```

---

## Task 5: Card rendering functions

**Files:**
- Modify: `~/github-pr-dashboard/dashboard.html`

- [ ] **Step 1: Add rendering functions after the `processData` function**

Add this block directly before the `// ── PAT setup handlers` comment:

```javascript
  // ── Rendering ──────────────────────────────────────────────────────────
  function escapeHtml(s) {
    return s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
  }

  function relativeTime(iso) {
    const s = Math.floor((Date.now() - new Date(iso)) / 1000);
    if (s < 60)  return 'just now';
    if (s < 3600) return `${Math.floor(s/60)}m ago`;
    if (s < 86400) return `${Math.floor(s/3600)}h ago`;
    return `${Math.floor(s/86400)}d ago`;
  }

  function ciTag(pr) {
    const state = pr.headRef?.target?.statusCheckRollup?.state ?? null;
    if (!state)                              return '<span class="tag tag-gray">⊘ no CI</span>';
    if (state === 'SUCCESS')                 return '<span class="tag tag-green">✓ CI passing</span>';
    if (state === 'FAILURE' || state === 'ERROR') return '<span class="tag tag-red">✗ CI failing</span>';
    return '<span class="tag tag-yellow">⟳ CI running</span>';
  }

  function renderCard(pr, variant) {
    const repo = escapeHtml(pr.repository.nameWithOwner.split('/').pop());
    const age  = relativeTime(pr.updatedAt);
    const comments = pr.comments.totalCount;

    let badges = '';
    if (variant === 'stale') {
      const days = Math.floor((Date.now() - new Date(pr.updatedAt)) / 86400000);
      badges += `<span class="badge badge-stale">💤 ${days}d idle</span>`;
      if (pr.isDraft) badges += '<span class="badge badge-draft">draft</span>';
    } else if (variant === 'inProgress') {
      badges += '<span class="badge badge-draft">draft</span>';
    } else if (variant === 'reviewsTop') {
      badges += '<span class="badge badge-me">authored by me</span>';
    } else if (variant === 'reviewsBottom') {
      badges += `<span class="badge badge-them">by ${escapeHtml(pr.author.login)}</span>`;
    } else if (variant === 'ready') {
      badges += '<span class="badge badge-deploy">✓ approved</span>';
    }

    if (variant === 'reviewsTop' && pr.reviewRequests?.totalCount > 0) {
      badges += `<span class="tag tag-gray">👤 ${pr.reviewRequests.totalCount}</span>`;
    }

    const cardClass = variant === 'stale'  ? 'db-card db-card-stale'  :
                      variant === 'ready'  ? 'db-card db-card-deploy' : 'db-card';

    return `<a class="${cardClass}" href="${escapeHtml(pr.url)}" target="_blank" rel="noopener">
      <div class="db-card-title">${escapeHtml(pr.title)}</div>
      <div class="db-card-meta">${repo} · #${pr.number} · ${age}</div>
      <div class="db-card-tags">${badges}${ciTag(pr)}<span class="tag tag-gray">💬 ${comments}</span></div>
    </a>`;
  }

  function renderReviews(topCards, bottomCards) {
    let html = '';
    if (topCards.length) {
      html += '<div class="reviews-divider">My PRs awaiting review</div>';
      html += topCards.map(pr => renderCard(pr, 'reviewsTop')).join('');
    }
    if (bottomCards.length) {
      html += '<div class="reviews-divider">PRs I need to review</div>';
      html += bottomCards.map(pr => renderCard(pr, 'reviewsBottom')).join('');
    }
    if (!topCards.length && !bottomCards.length) {
      html = '<div class="db-empty">No review activity</div>';
    }
    return html;
  }

  function render(columns, login) {
    showDashboard(login);

    const { stale, inProgress, reviewsTop, reviewsBottom, ready } = columns;

    document.getElementById('cards-stale').innerHTML =
      stale.length ? stale.map(pr => renderCard(pr, 'stale')).join('') :
      '<div class="db-empty">No stale PRs 🎉</div>';

    document.getElementById('cards-inprogress').innerHTML =
      inProgress.length ? inProgress.map(pr => renderCard(pr, 'inProgress')).join('') :
      '<div class="db-empty">No drafts in progress</div>';

    document.getElementById('cards-reviews').innerHTML = renderReviews(reviewsTop, reviewsBottom);

    document.getElementById('cards-ready').innerHTML =
      ready.length ? ready.map(pr => renderCard(pr, 'ready')).join('') :
      '<div class="db-empty">Merge it! 🎉</div>';

    document.getElementById('count-stale').textContent      = stale.length;
    document.getElementById('count-inprogress').textContent = inProgress.length;
    document.getElementById('count-reviews').textContent    = reviewsTop.length + reviewsBottom.length;
    document.getElementById('count-ready').textContent      = ready.length;
  }
```

- [ ] **Step 2: Verify card rendering in the browser console**

In DevTools console:

```javascript
const mockPR = {
  title: 'Add fee-calculation waterfall',
  number: 191,
  url: 'https://github.com/org/repo/pull/191',
  updatedAt: new Date(Date.now() - 86400000).toISOString(),
  isDraft: true,
  comments: { totalCount: 0 },
  reviewRequests: { totalCount: 0 },
  reviews: { nodes: [] },
  repository: { nameWithOwner: 'octocat/hello-world' },
  headRef: null,
};
console.log(renderCard(mockPR, 'inProgress'));
```

Expected: an HTML string containing `draft` badge, `⊘ no CI`, `💬 0`, and the PR title. No `undefined` or `null` visible in the output.

- [ ] **Step 3: Commit**

```bash
git add dashboard.html
git commit -m "feat: card rendering functions — badges, CI tags, relative time, XSS-safe escaping"
```

---

## Task 6: Wire up fetch, render, and auto-refresh

**Files:**
- Modify: `~/github-pr-dashboard/dashboard.html`

- [ ] **Step 1: Replace the refresh placeholder and `init` function**

Replace the entire `// ── Refresh placeholders` and `// ── Init` sections with:

```javascript
  // ── Refresh loop ───────────────────────────────────────────────────────
  let refreshTimer = null;

  async function refresh() {
    const pat = getPAT();
    if (!pat) { showSetup(); return; }

    try {
      hideError();
      const { login, myPRs, reviewRequestedPRs } = await fetchDashboard(pat);
      const columns = processData(myPRs, reviewRequestedPRs);
      render(columns, login);
      updateTimestamp();
    } catch (err) {
      if (err.code === 'UNAUTHORIZED') {
        clearPAT();
        if (refreshTimer) clearInterval(refreshTimer);
        showSetup('Token is invalid or expired. Please enter a new one.');
        return;
      }
      if (err.code === 'RATE_LIMITED') {
        showError('GitHub API rate limit reached — will retry on next refresh.');
        return;
      }
      showError(`Failed to load PRs: ${err.message}`);
    }
  }

  function manualRefresh() {
    if (refreshTimer) clearInterval(refreshTimer);
    refresh();
    refreshTimer = setInterval(refresh, REFRESH_MS);
  }

  function resetToken() {
    clearPAT();
    if (refreshTimer) clearInterval(refreshTimer);
    showSetup();
  }

  // ── Init ───────────────────────────────────────────────────────────────
  async function init() {
    const pat = getPAT();
    if (!pat) { showSetup(); return; }
    await refresh();
    refreshTimer = setInterval(refresh, REFRESH_MS);
  }

  init();
```

- [ ] **Step 2: Remove the earlier duplicate `let refreshTimer = null;` and stub `manualRefresh`**

Search for and delete these two lines that were added as stubs in Task 2 (they are now replaced by the full implementations above):

```javascript
  let refreshTimer = null;
  function manualRefresh() { /* filled in Task 6 */ }
```

- [ ] **Step 3: Test full end-to-end in browser**

Reload the file. Expected:
- Setup screen shows if no PAT
- After saving a valid PAT, dashboard loads with real PRs in the correct columns
- Cards show accurate CI status, comment counts, relative timestamps
- Click any card → PR opens in new tab
- Click "⟳ Refresh" → data reloads
- Click "Reset token" → returns to setup screen
- Wait 5 minutes (or set `REFRESH_MS = 10000` temporarily) → auto-refresh fires

Test error handling:
- Open DevTools → Network → set throttling to Offline → click Refresh
- Expected: red error banner appears with Retry button
- Set throttling back to Online → click Retry → banner disappears, data loads

- [ ] **Step 4: Commit**

```bash
git add dashboard.html
git commit -m "feat: wire up fetch/render/auto-refresh — full end-to-end dashboard working"
```

---

## Task 7: README

**Files:**
- Create: `~/github-pr-dashboard/README.md`

- [ ] **Step 1: Create README.md**

```markdown
# PR Harbormaster — GitHub PR Dashboard

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
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with PAT setup instructions and column reference"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Single HTML file, no server | Task 1 |
| PAT setup screen, localStorage, reset link | Task 2 |
| GitHub GraphQL API, two queries in one request | Task 3 |
| Column assignment: Stale > Ready > InProgress > ReviewsTop | Task 4 |
| Stale = 14+ days, includes drafts | Task 4 |
| Deduplication (my PRs win over review-requested) | Task 4 |
| Cards sorted by `updatedAt` descending per column | Task 4 |
| Standard card density: title, repo, age, CI, comments | Task 5 |
| Author badges in Reviews column (me vs them) | Task 5 |
| XSS-safe rendering (escapeHtml on all user content) | Task 5 |
| Click card → open PR in new tab | Task 5 |
| Auto-refresh every 5 min + manual refresh button | Task 6 |
| Last-updated timestamp | Task 6 |
| 401 → clear token, show setup screen | Task 6 |
| Rate limit → show banner, keep data | Task 6 |
| Network error → show banner with retry | Task 6 |
| README with PAT instructions | Task 7 |

No gaps found.
