# سِلفة — MVP Prototype Plan: Invite Flow + Dashboard

> **Scope:** Features 3 (WhatsApp invite → join) and 4 (circle dashboard) as a no-backend prototype using localStorage. Everything stays in `index.html`. Goal: presentable, tap-through demo.

---

## Architecture Decision: localStorage + Hash Routing

No backend. All circle data lives in `localStorage`. Views are switched via hash routing (`#dashboard`, `#join?id=xxx`). This lets us:
- Share real links (e.g. `silfaa.netlify.app/#join?id=abc123`)
- Deep-link directly to a circle invite
- Navigate between dashboard and wizard without page reload
- Present a fully working flow in a demo

### Data Model (localStorage)

```js
// Key: "silfa_circles"
// Value: JSON array of circle objects
[
  {
    id: "xK9mQ2",           // 6-char random ID
    name: "سِلفة العائلة",
    amount: 50000,
    members: 5,
    createdAt: "2026-04-07",
    creatorName: "أحمد",
    contacts: [              // from selectedContacts
      { id: 1, name: "علي", phone: "07701234567", color: "#...", status: "pending" },
      { id: 2, name: "حسن", phone: "07709876543", color: "#...", status: "joined" }
    ],
    cycleOrder: ["أنت", "علي", "حسن", "فاطمة", "زينب"],
    payoutTracker: [         // index = month, value = status
      { month: 0, status: "paid" },    // أنت — done
      { month: 1, status: "current" }, // علي — this month
      { month: 2, status: "upcoming" } // حسن — future
    ]
  }
]
```

### Hash Router

```js
// Lightweight router — runs on load + hashchange
function router() {
  const hash = window.location.hash; // e.g. "#join?id=xK9mQ2" or "#dashboard"
  if (hash.startsWith('#join'))      showJoinView(parseId(hash));
  else if (hash === '#dashboard')    showDashboard();
  else                               showDefault(); // wizard or landing
}
window.addEventListener('hashchange', router);
```

---

## PHASE 1 — Data Persistence Layer (Foundation)

### TICKET P1-1: Circle Storage API

**What:** Create a small JS API for CRUD operations on circles in localStorage.

**Functions to add (in `<script>` block):**

```js
function generateId()        // 6-char alphanumeric
function saveCircle(circle)  // push to array in localStorage
function getCircles()        // return parsed array or []
function getCircle(id)       // find by id
function updateCircle(id, updates) // merge updates, re-save
function deleteCircle(id)    // filter out by id
```

**Integration point:** At the end of the wizard (Step 5 transition in `goToStep`), after confetti, call `saveCircle()` with data assembled from `circleData`, `selectedContacts`, and `cycleOrder`. Also generate a real `id` and update the `#inviteLink` input value to use it.

**Update invite link:** Change from hardcoded `silfa.app/join/xK9mQ2` to:
```js
const link = `${window.location.origin}${window.location.pathname}#join?id=${circle.id}`;
document.getElementById('inviteLink').value = link;
```

This makes the WhatsApp share button send a real working link.

---

### TICKET P1-2: Hash Router

**What:** Add a minimal hash-based router that intercepts navigation and shows the right view.

**Views to route:**

| Hash | View | Action |
|------|------|--------|
| (none) / `#` | Default | Show wizard overlay (current behavior) |
| `#landing` | Landing page | Call `goToLanding()` |
| `#dashboard` | Dashboard | Show dashboard view |
| `#join?id=xxx` | Join screen | Show circle invite for given ID |
| `#circle?id=xxx` | Circle detail | Show active circle management |

**Implementation:**
- Add `router()` function that reads `window.location.hash`
- Listen to `hashchange` event
- Call `router()` on DOMContentLoaded
- Update `goToLanding()` to set `location.hash = '#landing'`
- Update `.open-wizard` to set `location.hash = '#'`
- Each view function hides all other views and shows its own container

**HTML:** Add empty view containers (hidden by default) after the wizard overlay:
```html
<div class="view-container" id="dashboardView" style="display:none;"></div>
<div class="view-container" id="joinView" style="display:none;"></div>
<div class="view-container" id="circleDetailView" style="display:none;"></div>
```

---

## PHASE 2 — Join Circle Flow (Feature 3)

### TICKET P2-1: Join View — HTML + CSS

**What:** A full-screen mobile view that shows when someone opens an invite link. This is what the WhatsApp recipient sees.

**Layout (top to bottom):**

```
┌─────────────────────────────┐
│  سِلفة logo (small, centered)│
│                             │
│  ╭───────────────────────╮  │
│  │  🏦  سِلفة العائلة     │  │
│  │                       │  │
│  │  ٢٥٠,٠٠٠ د.ع         │  │  ← payout hero (teal, large)
│  │  ستستلمها في دورك      │  │
│  │                       │  │
│  │  ┌──────┬──────────┐  │  │
│  │  │شهري  │ مدة      │  │  │  ← meta row
│  │  │٥٠,٠٠٠│ ٥ أشهر   │  │  │
│  │  └──────┴──────────┘  │  │
│  │                       │  │
│  │  👤 أحمد (المنشئ)     │  │  ← creator badge
│  │  👥 ٣ من ٥ انضموا    │  │  ← progress
│  │  ████████░░  60%      │  │  ← progress bar
│  ╰───────────────────────╯  │
│                             │
│  ╭───────────────────────╮  │
│  │ الاسم                 │  │  ← name input
│  ╰───────────────────────╯  │
│  ╭───────────────────────╮  │
│  │ رقم الهاتف            │  │  ← phone input (07xx)
│  ╰───────────────────────╯  │
│                             │
│  ┌───────────────────────┐  │
│  │   ✅ انضم للسِلفة      │  │  ← join CTA (teal, full-width)
│  └───────────────────────┘  │
│                             │
│  ماهي السِلفة؟ (link)       │  ← scrolls to landing FAQ
└─────────────────────────────┘
```

**CSS:** Reuse existing design tokens (navy, teal, gold, radius, shadows). The card style should match `.wizard-fs` feeling — full height, centered content, white card on light-bg.

**States:**
- **Loading:** Skeleton shimmer while "fetching" circle (fake 400ms delay for realism)
- **Found:** The layout above
- **Not found:** "هذا الرابط غير صالح" with CTA to create your own circle
- **Already full:** "السِلفة مكتملة!" with same CTA
- **Joined:** "تم انضمامك! ✅" confirmation with circle detail link

---

### TICKET P2-2: Join View — JS Logic

**What:** Wire up the join view to read circle data from localStorage and handle the join action.

**Flow:**
1. `showJoinView(id)` called by router
2. `getCircle(id)` fetches from localStorage
3. If not found → show error state
4. If found → render card with circle data
5. Check if circle is full (`contacts.filter(c => c.status === 'joined').length + 1 >= circle.members`) → show full state
6. User fills name + phone → validate (reuse existing Iraqi phone regex)
7. On "انضم" click:
   - Find matching contact by phone in `circle.contacts`
   - If found: update their `status` to `"joined"`, update name if different
   - If not found (open invite): add new contact with `status: "joined"`
   - `updateCircle(id, circle)`
   - Animate to "joined" confirmation state
   - Show button: "شاهد السِلفة" → navigates to `#circle?id=xxx`

**Phone matching logic:**
```js
// Normalize: strip spaces, convert Arabic digits, ensure 07xx format
function normalizePhone(phone) {
  return phone.replace(/[٠-٩]/g, d => '٠١٢٣٤٥٦٧٨٩'.indexOf(d))
              .replace(/\D/g, '')
              .replace(/^964/, '0');
}
```

---

### TICKET P2-3: Post-Wizard Redirect

**What:** After wizard completion (Step 5), add a "شاهد لوحة التحكم" button that navigates to the dashboard. Also auto-set `location.hash = '#dashboard'` option.

**Changes:**
- In Step 5 HTML, add a new button below the WhatsApp share:
  ```html
  <button id="goToDashboardBtn" class="wizard-dashboard-btn">
    📊 شاهد لوحة التحكم
  </button>
  ```
- JS: `goToDashboardBtn.onclick = () => location.hash = '#dashboard'`
- This is the bridge from "I created a circle" to "I manage my circles"

---

## PHASE 3 — Dashboard (Feature 4)

### TICKET P3-1: Dashboard Shell — HTML + CSS

**What:** Full-screen dashboard view listing all circles the user has created or joined. This is the "home" after you've created your first circle.

**Layout:**

```
┌─────────────────────────────┐
│  سِلفة    [+ سِلفة جديدة]   │  ← top bar with create button
├─────────────────────────────┤
│                             │
│  سِلفاتي (٢)               │  ← section header
│                             │
│  ╭───────────────────────╮  │
│  │ 🏦 سِلفة العائلة       │  │
│  │ ٥٠,٠٠٠ د.ع / شهر     │  │
│  │ ████████░░ ٣/٥ أعضاء  │  │  ← member join progress
│  │ 🟢 نشطة               │  │  ← status badge
│  │ الدور القادم: علي (مايو)│  │  ← next payout info
│  ╰───────────────────────╯  │
│                             │
│  ╭───────────────────────╮  │
│  │ 💼 سِلفة الشغل         │  │
│  │ ١٠٠,٠٠٠ د.ع / شهر    │  │
│  │ ██░░░░░░░░ ١/٥ أعضاء  │  │
│  │ 🟡 بانتظار الأعضاء     │  │  ← waiting for members
│  │ شارك الرابط ←          │  │
│  ╰───────────────────────╯  │
│                             │
│  ╭─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─╮  │
│  │                       │  │
│  │  + أنشئ سِلفة جديدة    │  │  ← dashed card, opens wizard
│  │                       │  │
│  ╰─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─╯  │
└─────────────────────────────┘
```

**Circle card states:**
1. **بانتظار الأعضاء** (yellow) — not all members joined yet
2. **نشطة** (green) — all members joined, payouts in progress
3. **مكتملة** (grey) — all payouts done

**CSS approach:** Cards reuse `.wizard-card`-like styling. Progress bar is a simple div-in-div with teal fill. Status badges use colored pill (`.badge-active`, `.badge-pending`, `.badge-complete`).

---

### TICKET P3-2: Dashboard — JS Logic

**What:** Render the dashboard from localStorage data. Handle empty state.

**Functions:**
```js
function showDashboard() {
  // Hide other views
  // Get all circles from localStorage
  // If empty → show empty state with CTA
  // If circles exist → render cards
  // Attach click handlers: card → #circle?id=xxx
}

function renderCircleCard(circle) {
  const joinedCount = circle.contacts.filter(c => c.status === 'joined').length + 1; // +1 for creator
  const isFull = joinedCount >= circle.members;
  const status = isFull ? 'active' : 'pending';
  // ... build card HTML
}
```

**Empty state:**
```
┌─────────────────────────────┐
│                             │
│         🏦                  │
│   ما عندك سِلفات بعد        │
│   أنشئ أول سِلفة وادعِ ناسك │
│                             │
│   [أنشئ سِلفة جديدة]        │
│                             │
└─────────────────────────────┘
```

---

### TICKET P3-3: Circle Detail View — HTML + CSS

**What:** Tapping a circle card opens a detail view showing full circle status, member list, and payout timeline.

**Layout:**

```
┌─────────────────────────────┐
│  → رجوع     سِلفة العائلة   │  ← top bar with back
├─────────────────────────────┤
│                             │
│  ╭───────────────────────╮  │
│  │  الدور الحالي          │  │
│  │  علي — مايو ٢٠٢٦     │  │  ← highlighted current turn
│  │  ٢٥٠,٠٠٠ د.ع          │  │
│  │  [✅ تم الاستلام]      │  │  ← mark as received button
│  ╰───────────────────────╯  │
│                             │
│  ── الجدول ──               │
│                             │
│  🟢 أنت      أبريل  ✅ تم   │
│  🔵 علي      مايو   ⏳ الآن │  ← highlighted
│  ⚪ حسن      يونيو  ─       │
│  ⚪ فاطمة    يوليو  ─       │
│  ⚪ زينب     أغسطس  ─       │
│                             │
│  ── الأعضاء (٥/٥) ──       │
│                             │
│  👤 أنت (المنشئ)    ✅      │
│  👤 علي             ✅ انضم  │
│  👤 حسن             ✅ انضم  │
│  👤 فاطمة           ⏳ بانتظار│
│  👤 زينب            ⏳ بانتظار│
│                             │
│  ┌───────────────────────┐  │
│  │  📤 شارك رابط الدعوة   │  │  ← share invite link
│  └───────────────────────┘  │
│  ┌───────────────────────┐  │
│  │  🗑️ حذف السِلفة        │  │  ← delete (with confirm)
│  └───────────────────────┘  │
└─────────────────────────────┘
```

**Key interactions:**
- "تم الاستلام" button → updates `payoutTracker[currentMonth].status = 'paid'`, advances to next
- Share button → same WhatsApp share logic, reusing the invite link
- Delete → `confirm()` dialog → `deleteCircle(id)` → redirect to `#dashboard`
- Back arrow → `location.hash = '#dashboard'`

---

### TICKET P3-4: Circle Detail View — JS Logic

**What:** Render and manage the circle detail view.

**Functions:**
```js
function showCircleDetail(id) {
  const circle = getCircle(id);
  if (!circle) { location.hash = '#dashboard'; return; }
  
  renderCurrentTurn(circle);
  renderTimeline(circle);
  renderMemberList(circle);
  attachDetailActions(circle);
}

function getCurrentTurnIndex(circle) {
  // First unpaid month in payoutTracker
  return circle.payoutTracker.findIndex(p => p.status !== 'paid') || 0;
}

function markPaid(circleId, monthIndex) {
  const circle = getCircle(circleId);
  circle.payoutTracker[monthIndex].status = 'paid';
  if (monthIndex + 1 < circle.payoutTracker.length) {
    circle.payoutTracker[monthIndex + 1].status = 'current';
  }
  updateCircle(circleId, circle);
  showCircleDetail(circleId); // re-render
}
```

**Timeline rendering:** Reuse the duration strip concept from Ticket 7, but vertical. Each row shows: status icon + name + month + status badge. Current turn row gets highlighted background (teal tint).

---

### TICKET P3-5: Navigation Integration

**What:** Connect all the pieces — wizard, dashboard, join, detail — into a seamless flow.

**Changes:**

1. **Wizard completion → Dashboard:**
   - Step 5 saves circle to localStorage
   - "شاهد لوحة التحكم" button added
   - WhatsApp share sends real working link

2. **Landing page:**
   - If user has circles in localStorage, show a floating "لوحة التحكم" button in nav
   - This lets returning users jump to dashboard

3. **Dashboard → Wizard:**
   - "+ سِلفة جديدة" button → `location.hash = '#'` (opens wizard)
   - Dashed create card → same

4. **Dashboard → Circle Detail:**
   - Card click → `location.hash = '#circle?id=xxx'`

5. **Circle Detail → Dashboard:**
   - Back button → `location.hash = '#dashboard'`

6. **Join → Circle Detail:**
   - After joining → "شاهد السِلفة" → `location.hash = '#circle?id=xxx'`

7. **Browser back/forward:**
   - Hash router handles all navigation
   - Back button works naturally

---

## PHASE 4 — Polish

### TICKET P4-1: View Transitions

**What:** Smooth transitions between views (dashboard ↔ detail ↔ join).

- Fade-in/out on view switch (200ms, CSS transition)
- Slide-up for modal-like views (join confirmation)
- Reuse existing `cubic-bezier(0.16, 1, 0.3, 1)` easing from wizard

### TICKET P4-2: Demo Seed Data

**What:** Add a "تجربة" (demo) button that seeds localStorage with 2-3 pre-built circles at different states so the dashboard isn't empty during a presentation.

```js
function seedDemoData() {
  const demo = [
    { id: 'demo01', name: 'سِلفة العائلة', amount: 50000, members: 5, 
      contacts: [...], cycleOrder: [...], 
      payoutTracker: [{ status: 'paid' }, { status: 'current' }, ...] },
    { id: 'demo02', name: 'سِلفة الشغل', amount: 100000, members: 3,
      contacts: [...], cycleOrder: [...],
      payoutTracker: [{ status: 'upcoming' }, ...] }
  ];
  localStorage.setItem('silfa_circles', JSON.stringify(demo));
}
```

Trigger: query param `?demo=1` or a hidden triple-tap on the logo.

### TICKET P4-3: Notification Badges (Visual Only)

**What:** Add visual indicators for pending actions:
- Dashboard card: red dot if it's your turn to pay
- Circle detail: pulse animation on "تم الاستلام" button
- Join view: count of pending invites

No real notifications — just visual cues for the demo.

---

## Execution Order

```
P1-1  Circle Storage API          ← foundation, everything depends on this
P1-2  Hash Router                 ← foundation, enables all navigation
  │
  ├── P2-1  Join View HTML/CSS    ← can be built in parallel with P3-1
  ├── P3-1  Dashboard Shell       ← can be built in parallel with P2-1
  │
P2-2  Join View JS Logic          ← needs P1-1 + P2-1
P2-3  Post-Wizard Redirect        ← needs P1-1
P3-2  Dashboard JS Logic          ← needs P1-1 + P3-1
P3-3  Circle Detail HTML/CSS      ← needs P3-1 (shares layout patterns)
P3-4  Circle Detail JS Logic      ← needs P1-1 + P3-3
P3-5  Navigation Integration      ← needs everything above
  │
P4-1  View Transitions            ← polish pass
P4-2  Demo Seed Data              ← for presentations
P4-3  Notification Badges         ← visual polish
```

**Total tickets: 13**
**Estimated for AI agent: all in one session, single file edits**

---

## Constraints for the Agent

1. **Single file only** — all HTML, CSS, JS stays in `index.html`
2. **No external dependencies** — no new CDN imports, no npm, no build step
3. **localStorage only** — no backend, no API calls, no Supabase
4. **Preserve existing wizard** — do not break any of the 14 completed tickets
5. **RTL Arabic** — all new text in Arabic, use `toAr()` and `formatAr()` helpers
6. **Mobile-first** — all new views must work on 375px width
7. **Reuse design tokens** — navy, teal, gold, existing CSS variables
8. **Deploy with** `netlify deploy --dir=. --prod`
