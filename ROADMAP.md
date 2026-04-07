# سِلفة — Wizard Improvement Roadmap

> Agent instructions: All work is in `/Users/amr/development/silfa/index.html` — a single self-contained file with inline CSS (`<style>` block) and inline JS (`<script>` block at bottom of `<body>`). No build system. Deploy with `netlify deploy --dir=. --prod`. The file is ~3466 lines. Read the relevant sections before editing. RTL layout (`dir="rtl"`). Arabic numerals via `toAr(n)`. Currency formatted as `formatAr(n) + ' د.ع'`.

---

## ~~TICKET 1 — Step Transitions (Slide Animation)~~ ✅ DONE

**Status:** Completed 2026-04-07. CSS slide keyframes added (`stepSlideIn/Out` + `going-back` variants). `goToStep()` updated to animate the exiting step out first, then slide the new step in after 30ms. Direction detection uses `step < currentStep` before `currentStep = step` is updated.

**Why:** Currently steps swap instantly via `classList.add('active')`. The flow feels like a webpage, not an app.

**What to change:**

### CSS
Add these rules to the `<style>` block, near the existing `.wizard-step` rule:

```css
.wizard-step {
    display: none;
    animation: none;
}
.wizard-step.active {
    display: block;
    animation: stepSlideIn 0.32s cubic-bezier(0.16, 1, 0.3, 1) both;
}
.wizard-step.exit {
    animation: stepSlideOut 0.22s ease-in both;
}
@keyframes stepSlideIn {
    from { opacity: 0; transform: translateX(-28px); }
    to   { opacity: 1; transform: translateX(0); }
}
@keyframes stepSlideOut {
    from { opacity: 1; transform: translateX(0); }
    to   { opacity: 0; transform: translateX(28px); }
}
```

Note: RTL means "next step" enters from the left (negative X), previous step exits to the right (positive X). Going back should reverse: new step enters from right, old exits left. Handle this by toggling a `.going-back` class on `.wizard-body`.

Add to CSS:
```css
.wizard-body.going-back .wizard-step.active {
    animation: stepSlideInBack 0.32s cubic-bezier(0.16, 1, 0.3, 1) both;
}
.wizard-body.going-back .wizard-step.exit {
    animation: stepSlideOutBack 0.22s ease-in both;
}
@keyframes stepSlideInBack {
    from { opacity: 0; transform: translateX(28px); }
    to   { opacity: 1; transform: translateX(0); }
}
@keyframes stepSlideOutBack {
    from { opacity: 1; transform: translateX(0); }
    to   { opacity: 0; transform: translateX(-28px); }
}
```

### JS
Replace the `goToStep(step)` function's step-switching logic. Currently it does:
```js
stepEls.forEach(s => s.classList.remove('active'));
const target = document.querySelector(`.wizard-step[data-step="${step}"]`);
if (target) target.classList.add('active');
```

Replace with:
```js
const goingBack = step < currentStep;
const wizardBody = document.querySelector('.wizard-body');
wizardBody.classList.toggle('going-back', goingBack);

const current = document.querySelector('.wizard-step.active');
if (current) {
    current.classList.add('exit');
    current.addEventListener('animationend', () => {
        current.classList.remove('active', 'exit');
    }, { once: true });
}
const target = document.querySelector(`.wizard-step[data-step="${step}"]`);
if (target) {
    // Small delay so exit animation starts first
    setTimeout(() => target.classList.add('active'), 30);
}
```

Also update `backBtn` listener to signal direction — it already calls `goToStep(currentStep - 1)` and `currentStep` hasn't changed yet at that point, so the `step < currentStep` check in `goToStep` will correctly detect going-back direction.

---

## ~~TICKET 2 — Contextual Next Button Text~~ ✅ DONE

**Status:** Completed 2026-04-07. Replaced the ternary in `goToStep()` with a `nextLabels` map. Step 4 label is `'🚀 أطلق السِلفة'` (matches `LAST_INPUT_STEP = 4`). Success step (5) is unaffected — button is hidden before reaching the label logic.

**Why:** "التالي" is generic. Telling users exactly what happens next reduces hesitation.

**What to change:**

In `goToStep(step)`, find this line:
```js
nextBtn.textContent = step === LAST_INPUT_STEP ? '🎉 أنشئ السِلفة وأرسل الدعوات' : 'التالي';
```

Replace with a per-step map:
```js
const nextLabels = {
    1: 'اختر الأعضاء ←',
    2: 'ابدأ اختيار الأعضاء ←',
    3: 'رتّب جدول الاستلام ←',
    4: '🚀 أطلق السِلفة',
};
nextBtn.textContent = nextLabels[step] || 'التالي';
```

No other changes needed.

---

## ~~TICKET 3 — Circle Type Selector (Step 1)~~ ⏭ SKIPPED (redundant)

**Why:** Asking "what kind of circle?" instantly personalizes the experience and pre-fills the name, making step 1 take half the effort.

**What to change:**

### HTML
In the wizard body, inside `<div class="wizard-step active" data-step="1">`, insert this block **before** the `<div class="wizard-input-group">` that contains the circle name input:

```html
<div class="circle-type-group">
    <label>نوع السِلفة</label>
    <div class="circle-type-options">
        <button class="circle-type-btn" data-type="family" data-label="سِلفة العائلة">
            <span class="ct-icon">👨‍👩‍👧</span>
            <span class="ct-label">عائلي</span>
        </button>
        <button class="circle-type-btn" data-type="work" data-label="سِلفة الشغل">
            <span class="ct-icon">💼</span>
            <span class="ct-label">عمل</span>
        </button>
        <button class="circle-type-btn" data-type="friends" data-label="سِلفة الأصحاب">
            <span class="ct-icon">👥</span>
            <span class="ct-label">أصدقاء</span>
        </button>
    </div>
</div>
```

### CSS
Add to the `<style>` block:
```css
.circle-type-group { margin-bottom: 20px; }
.circle-type-group label { display: block; font-size: 13px; font-weight: 600; color: var(--text-light); margin-bottom: 10px; text-align: right; }
.circle-type-options { display: flex; gap: 10px; }
.circle-type-btn {
    flex: 1; display: flex; flex-direction: column; align-items: center; gap: 6px;
    padding: 14px 10px; border-radius: var(--radius-sm); border: 2px solid var(--border);
    background: var(--white); cursor: pointer; transition: all 0.2s; font-family: inherit;
}
.circle-type-btn:hover { border-color: var(--teal); background: rgba(13,148,136,0.04); }
.circle-type-btn.selected { border-color: var(--teal); background: rgba(13,148,136,0.08); }
.ct-icon { font-size: 24px; line-height: 1; }
.ct-label { font-size: 13px; font-weight: 700; color: var(--navy); }
.circle-type-btn.selected .ct-label { color: var(--teal); }
```

### JS
Add this listener block in the Step 1 JS section (after the `customAmount` event listener):
```js
document.querySelectorAll('.circle-type-btn').forEach(btn => {
    btn.addEventListener('click', () => {
        document.querySelectorAll('.circle-type-btn').forEach(b => b.classList.remove('selected'));
        btn.classList.add('selected');
        const nameInput = document.getElementById('circleName');
        // Only pre-fill if user hasn't typed something custom
        if (!nameInput.value.trim() || document.querySelectorAll('.circle-type-btn.selected').length) {
            nameInput.value = btn.dataset.label;
            circleData.name = btn.dataset.label;
        }
        validateStep();
    });
});
```

Also add `circleData.type = ''` to the state object at the top (`let circleData = { name: '', amount: 0, members: 5, type: '' }`).

---

## ~~TICKET 4 — Amount Presets Expansion + Auto-format~~ ✅ DONE

**Status:** Completed 2026-04-07. Added 250k and 500k preset buttons (2×3 grid, CSS already 2-col). Updated `customAmount` listener to strip non-digits before parsing so letters/symbols don't corrupt `circleData.amount`.

**Why:** 50k–200k misses the most common Iraqi جمعية amounts (250k, 500k). Auto-formatting makes large numbers readable.

### HTML
Find the `.amount-presets` div in Step 1. Replace all 4 buttons with 6:
```html
<div class="amount-presets">
    <button class="amount-btn" data-amount="50000">٥٠,٠٠٠ <span>د.ع</span></button>
    <button class="amount-btn" data-amount="100000">١٠٠,٠٠٠ <span>د.ع</span></button>
    <button class="amount-btn" data-amount="150000">١٥٠,٠٠٠ <span>د.ع</span></button>
    <button class="amount-btn" data-amount="200000">٢٠٠,٠٠٠ <span>د.ع</span></button>
    <button class="amount-btn" data-amount="250000">٢٥٠,٠٠٠ <span>د.ع</span></button>
    <button class="amount-btn" data-amount="500000">٥٠٠,٠٠٠ <span>د.ع</span></button>
</div>
```

### CSS
The `.amount-presets` grid is currently 2 columns. Change to accommodate 6 buttons in a 2×3 grid — find the existing `.amount-presets` CSS rule and ensure it has:
```css
.amount-presets { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 14px; }
```
No change needed if it's already 2-column grid — the 6th button will naturally wrap.

### JS
Replace the `customAmount` input listener with one that formats as Arabic numerals with commas:
```js
document.getElementById('customAmount').addEventListener('input', (e) => {
    // Strip non-digits
    const raw = e.target.value.replace(/[^0-9]/g, '');
    const val = parseInt(raw);
    document.querySelectorAll('.amount-btn').forEach(b => b.classList.remove('selected'));
    if (raw && val > 0) {
        circleData.amount = val;
        // Highlight matching preset if any
        document.querySelectorAll('.amount-btn').forEach(b => {
            if (parseInt(b.dataset.amount) === val) b.classList.add('selected');
        });
    } else {
        circleData.amount = 0;
    }
    validateStep();
});
```

---

## ~~TICKET 5 — Live Preview Chip (Step 1)~~ ✅ DONE

**Status:** Completed 2026-04-07. Added `.step1-preview-chip` with teal styling below the "مجاني" line. `updateStep1Preview()` function shows payout estimate using default 5 members; called from both `amount-btn` click handler and `customAmount` input handler.

**Why:** Users should see the cost/payout impact while still on Step 1, before committing to go to Step 2. One line of dynamic text creates immediate understanding.

### HTML
Add this block at the bottom of the Step 1 `<div class="wizard-step" data-step="1">`, just before its closing tag (after the "مجاني" line):
```html
<div class="step1-preview-chip" id="step1PreviewChip" style="display:none;">
    <span id="step1PreviewText"></span>
</div>
```

### CSS
```css
.step1-preview-chip {
    margin-top: 18px; padding: 12px 16px; border-radius: var(--radius-sm);
    background: rgba(13,148,136,0.07); border: 1.5px solid rgba(13,148,136,0.2);
    text-align: center; font-size: 14px; font-weight: 600; color: var(--teal);
    transition: all 0.3s;
}
```

### JS
Add a call to `updateStep1Preview()` inside both the `amount-btn` click handler and the `customAmount` input handler (after `validateStep()`). Add the function itself in the Step 1 JS section:
```js
function updateStep1Preview() {
    const chip = document.getElementById('step1PreviewChip');
    const text = document.getElementById('step1PreviewText');
    if (!chip || !text) return;
    if (circleData.amount > 0 && circleData.members > 0) {
        const payout = circleData.amount * circleData.members;
        text.textContent = `كل عضو يدفع ${formatAr(circleData.amount)} د.ع شهرياً ويستلم ${formatAr(payout)} د.ع في دوره`;
        chip.style.display = 'block';
    } else {
        chip.style.display = 'none';
    }
}
```

Note: `circleData.members` defaults to 5 at this point, which is fine — the preview is an estimate.

---

## TICKET 6 — Seat Visualizer (Step 2)

**Why:** "5 أعضاء" is abstract. Seeing 5 circles — one filled gold (you) and 4 empty outlines — makes the concept of a circle feel real. This is the highest-impact visual change in the whole wizard.

### HTML
In Step 2 (`<div class="wizard-step" data-step="2">`), add this block between the `<p class="wizard-step-desc">` and the `<div class="members-stepper">`:
```html
<div class="seats-visualizer" id="seatsVisualizer"></div>
```

### CSS
```css
.seats-visualizer {
    display: flex; flex-wrap: wrap; gap: 10px; justify-content: center;
    padding: 20px 0 10px; min-height: 60px;
}
.seat {
    width: 46px; height: 46px; border-radius: 50%;
    display: flex; align-items: center; justify-content: center;
    font-size: 13px; font-weight: 700; transition: all 0.25s ease;
}
.seat.you {
    background: var(--gold); color: var(--navy);
    box-shadow: 0 0 0 3px rgba(245,197,66,0.3);
}
.seat.you::after { content: 'أنت'; font-size: 11px; }
.seat.empty {
    background: var(--white); border: 2px dashed var(--border); color: var(--text-light);
}
.seat.empty-entering {
    animation: seatPop 0.25s cubic-bezier(0.34, 1.56, 0.64, 1) both;
}
@keyframes seatPop {
    from { opacity: 0; transform: scale(0.4); }
    to   { opacity: 1; transform: scale(1); }
}
```

### JS
Add a `renderSeats()` function and call it from `updateMembersDisplay()`:
```js
function renderSeats() {
    const viz = document.getElementById('seatsVisualizer');
    if (!viz) return;
    viz.innerHTML = '';
    for (let i = 0; i < circleData.members; i++) {
        const seat = document.createElement('div');
        seat.className = i === 0 ? 'seat you' : 'seat empty seat-entering';
        if (i !== 0) seat.textContent = toAr(i + 1);
        viz.appendChild(seat);
        // Stagger pop animation
        if (i > 0) setTimeout(() => seat.classList.add('seat-entering'), i * 30);
    }
}
```

Call `renderSeats()` at the end of `updateMembersDisplay()` and also in `goToStep` when `step === 2`.

---

## TICKET 7 — Payout Hero Number + Duration Strip (Step 2)

**Why:** The emotional hook of a جمعية is "I will receive a large lump sum." That number must be the visual hero of Step 2. The duration strip converts "5 أشهر" from abstract to concrete calendar months.

### HTML
Replace the current `.live-preview` div in Step 2 with:
```html
<div class="payout-hero">
    <div class="payout-hero-label">ستستلم في دورك</div>
    <div class="payout-hero-amount" id="previewPayout">٢٥٠,٠٠٠ د.ع</div>
</div>

<div class="preview-meta-row">
    <div class="preview-meta-item">
        <span class="preview-meta-label">حصتك الشهرية</span>
        <span class="preview-meta-value" id="previewMonthly">٥٠,٠٠٠ د.ع</span>
    </div>
    <div class="preview-meta-divider"></div>
    <div class="preview-meta-item">
        <span class="preview-meta-label">مدة السِلفة</span>
        <span class="preview-meta-value" id="previewDuration">٥ أشهر</span>
    </div>
</div>

<div class="duration-strip" id="durationStrip"></div>
```

### CSS
```css
.payout-hero { text-align: center; padding: 24px 0 8px; }
.payout-hero-label { font-size: 13px; color: var(--text-light); font-weight: 500; margin-bottom: 6px; }
.payout-hero-amount { font-size: 42px; font-weight: 900; color: var(--teal); letter-spacing: -1px; line-height: 1.1; }

.preview-meta-row { display: flex; align-items: center; justify-content: center; gap: 0; margin: 16px 0 20px; background: var(--light-bg); border-radius: var(--radius-sm); padding: 14px; }
.preview-meta-item { flex: 1; text-align: center; }
.preview-meta-label { display: block; font-size: 11px; color: var(--text-light); font-weight: 500; margin-bottom: 4px; }
.preview-meta-value { display: block; font-size: 16px; font-weight: 700; color: var(--navy); }
.preview-meta-divider { width: 1px; height: 36px; background: var(--border); }

.duration-strip { display: flex; gap: 6px; overflow-x: auto; padding: 4px 0 12px; scrollbar-width: none; }
.duration-strip::-webkit-scrollbar { display: none; }
.duration-month {
    flex-shrink: 0; padding: 8px 14px; border-radius: 50px;
    font-size: 12px; font-weight: 600; background: var(--light-bg); color: var(--text-light);
    border: 1.5px solid var(--border); white-space: nowrap;
}
.duration-month.first { background: rgba(245,197,66,0.15); border-color: var(--gold); color: var(--navy); }
```

### JS
Update `updateMembersPreview()` to also render the duration strip:
```js
function updateMembersPreview() {
    const payout = circleData.amount * circleData.members;
    document.getElementById('previewMonthly').textContent = circleData.amount > 0 ? formatAr(circleData.amount) + ' د.ع' : '— د.ع';
    document.getElementById('previewPayout').textContent  = payout > 0 ? formatAr(payout) + ' د.ع' : '— د.ع';
    document.getElementById('previewDuration').textContent = toAr(circleData.members) + ' أشهر';

    // Duration strip
    const strip = document.getElementById('durationStrip');
    if (!strip) return;
    strip.innerHTML = '';
    for (let i = 0; i < circleData.members; i++) {
        const el = document.createElement('div');
        el.className = 'duration-month' + (i === 0 ? ' first' : '');
        el.textContent = getPayoutMonth(i);
        strip.appendChild(el);
    }
}
```

---

## TICKET 8 — "أنت" Pinned Row in Step 3

**Why:** The organizer is always a member. Showing them pinned at the top of the contacts list (gold, non-removable) reinforces their identity and makes the slot math obvious: "I need 4 more people."

### HTML
In Step 3, add a pinned "you" row immediately after the `<div class="selection-counter-pill">` block and before the search box:
```html
<div class="contact-item you-contact-pin">
    <div class="contact-avatar" style="background: var(--navy);">أ</div>
    <div class="contact-info">
        <span class="contact-name">أنت <span style="font-size:11px;color:var(--teal);font-weight:600;">(المنشئ)</span></span>
        <span class="contact-phone">مضاف تلقائياً</span>
    </div>
    <div class="contact-check selected-check">
        <svg viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6L9 17l-5-5"/></svg>
    </div>
</div>
```

### CSS
```css
.you-contact-pin {
    display: flex; align-items: center; gap: 12px;
    padding: 12px 14px; border-radius: var(--radius-sm);
    background: rgba(245,197,66,0.08); border: 1.5px solid rgba(245,197,66,0.4);
    margin-bottom: 10px; cursor: default;
}
.selected-check {
    width: 22px; height: 22px; border-radius: 50%;
    background: var(--teal); border: 2px solid var(--teal);
    display: flex; align-items: center; justify-content: center; flex-shrink: 0;
}
.selected-check svg { width: 11px; height: 11px; }
```

No JS changes needed — the "أنت" row is purely decorative in the HTML; the actual `cycleOrder` already prepends `{ id: 'you', name: 'أنت', isYou: true }` in `renderCyclePlanning()`.

---

## TICKET 9 — Full-circle Celebration Pulse (Step 3)

**Why:** When the last contact slot is filled, there's no satisfaction signal. A short animation on the counter pill tells the user "you're done — hit next."

### CSS
Add a keyframe animation:
```css
@keyframes pillPulse {
    0%   { transform: scale(1); }
    40%  { transform: scale(1.06); box-shadow: 0 0 0 6px rgba(245,197,66,0.2); }
    100% { transform: scale(1); box-shadow: none; }
}
.selection-counter-pill.full {
    background: rgba(245,197,66,0.1);
    border-color: rgba(245,197,66,0.5);
}
.selection-counter-pill.full.just-filled {
    animation: pillPulse 0.5s ease forwards;
}
```

### JS
In `updateSelectionCounter()`, after the line `if (pill) pill.classList.toggle('full', n >= slots && slots > 0);`, add:
```js
if (n >= slots && slots > 0) {
    pill.classList.remove('just-filled');
    void pill.offsetWidth; // force reflow to restart animation
    pill.classList.add('just-filled');
}
```

Also update `.sc-selected` style to gold when full — already handled by the existing `.full .sc-selected { color: #D97706; }` rule.

---

## TICKET 10 — قرعة Animation (Step 4)

**Why:** The traditional جمعية uses قرعة (a draw/lottery) to assign order. This is culturally trusted and exciting. Renaming "خلط عشوائي" to "سحب القرعة" and adding a brief shake-then-snap animation makes the app feel authentically Iraqi.

### HTML
Find the shuffle button in Step 4 and replace it:
```html
<button class="shuffle-btn quraa-btn" id="shuffleBtn">
    🎲 سحب القرعة
</button>
```

### CSS
Add to existing `.shuffle-btn` rules:
```css
.quraa-btn { font-size: 15px; font-weight: 700; }

@keyframes rowShake {
    0%   { transform: translateX(0); }
    20%  { transform: translateX(-6px) rotate(-1deg); }
    40%  { transform: translateX(6px) rotate(1deg); }
    60%  { transform: translateX(-4px); }
    80%  { transform: translateX(4px); }
    100% { transform: translateX(0); }
}
.cycle-row.shaking { animation: rowShake 0.35s ease both; }
```

### JS
Replace the `shuffleBtn` click listener with:
```js
document.getElementById('shuffleBtn').addEventListener('click', () => {
    const rows = document.querySelectorAll('.cycle-row');

    // Phase 1: shake all rows
    rows.forEach((r, i) => {
        r.classList.remove('shaking');
        void r.offsetWidth;
        setTimeout(() => r.classList.add('shaking'), i * 25);
    });

    // Phase 2: after shake, shuffle data and re-render
    setTimeout(() => {
        for (let i = cycleOrder.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [cycleOrder[i], cycleOrder[j]] = [cycleOrder[j], cycleOrder[i]];
        }
        renderCycleRows();
    }, 400);
});
```

---

## TICKET 11 — Touch Drag Fix (Step 4)

**Why:** HTML5 `draggable="true"` + `dragstart/dragover/drop` events do not work on iOS Safari. Iraqi users are predominantly on mobile. This makes the reorder feature completely non-functional for most users.

**Strategy:** Replace HTML5 drag with tap-based up/down arrow buttons on each row. Simpler, universally reliable.

### HTML change in `renderCycleRows()`
Replace the current template literal inside `renderCycleRows()`. The current row template:
```js
`<div class="cycle-row ${m.isYou ? 'you-row' : ''}" draggable="true" data-index="${i}">
    <div class="cycle-drag">⠿</div>
    ...
</div>`
```

Replace with:
```js
`<div class="cycle-row ${m.isYou ? 'you-row' : ''}" data-index="${i}">
    <div class="cycle-turn">${toAr(i + 1)}</div>
    <div class="cycle-avatar" style="background:${m.color};">${m.name.charAt(0)}</div>
    <div class="cycle-info">
        <span class="cycle-name">${m.name}${m.isYou ? ' ⭐' : ''}</span>
        <span class="cycle-month">${getPayoutMonth(i)}</span>
    </div>
    <div class="cycle-amount">${formatAr(payout)} د.ع</div>
    <div class="cycle-arrows">
        <button class="cycle-arrow-btn cycle-up" data-index="${i}" ${i === 0 ? 'disabled' : ''}>▲</button>
        <button class="cycle-arrow-btn cycle-down" data-index="${i}" ${i === cycleOrder.length - 1 ? 'disabled' : ''}>▼</button>
    </div>
</div>`
```

### CSS
```css
.cycle-arrows { display: flex; flex-direction: column; gap: 2px; flex-shrink: 0; }
.cycle-arrow-btn {
    width: 26px; height: 26px; border-radius: 6px; border: 1.5px solid var(--border);
    background: var(--light-bg); cursor: pointer; font-size: 10px; color: var(--text-light);
    display: flex; align-items: center; justify-content: center; transition: all 0.15s;
    font-family: inherit;
}
.cycle-arrow-btn:disabled { opacity: 0.3; cursor: not-allowed; }
.cycle-arrow-btn:not(:disabled):hover { border-color: var(--teal); color: var(--teal); background: rgba(13,148,136,0.06); }
```

### JS
Remove all `dragstart`, `dragover`, `drop`, `dragend` event attachment from `renderCycleRows()`. Replace with:
```js
list.querySelectorAll('.cycle-up').forEach(btn => {
    btn.addEventListener('click', () => {
        const idx = parseInt(btn.dataset.index);
        if (idx > 0) {
            [cycleOrder[idx - 1], cycleOrder[idx]] = [cycleOrder[idx], cycleOrder[idx - 1]];
            renderCycleRows();
        }
    });
});
list.querySelectorAll('.cycle-down').forEach(btn => {
    btn.addEventListener('click', () => {
        const idx = parseInt(btn.dataset.index);
        if (idx < cycleOrder.length - 1) {
            [cycleOrder[idx], cycleOrder[idx + 1]] = [cycleOrder[idx + 1], cycleOrder[idx]];
            renderCycleRows();
        }
    });
});
```

Also remove `let dragSrcIndex = null;` from state variables and remove `draggable="true"` from all rows (already removed in new template above).

---

## TICKET 12 — Schedule Summary Bar (Step 4)

**Why:** "الجدول" feels abstract without knowing when it starts and ends. One line grounds the whole schedule in reality.

### HTML
Add this block immediately after `<div class="wizard-step" data-step="4">` opening tag, before the title:
```html
<div class="schedule-summary-bar" id="scheduleSummaryBar"></div>
```

### CSS
```css
.schedule-summary-bar {
    background: rgba(13,148,136,0.06); border: 1.5px solid rgba(13,148,136,0.15);
    border-radius: var(--radius-sm); padding: 10px 14px; margin-bottom: 16px;
    font-size: 13px; color: var(--teal); font-weight: 600; text-align: center;
}
```

### JS
Add a call to `renderScheduleSummary()` inside `renderCyclePlanning()`. Define the function:
```js
function renderScheduleSummary() {
    const bar = document.getElementById('scheduleSummaryBar');
    if (!bar || cycleOrder.length === 0) return;
    const start = getPayoutMonth(0);
    const end   = getPayoutMonth(cycleOrder.length - 1);
    bar.textContent = `السِلفة تبدأ ${start} وتنتهي ${end}`;
}
```

Call `renderScheduleSummary()` at the end of `renderCycleRows()` as well (since re-ordering changes nothing here, but shuffle might change start month display if months are recalculated — they aren't, so this is low-cost insurance).

---

## TICKET 13 — Digital Summary Card + WhatsApp Share (Step 5)

**Why:** (1) The success screen shows no summary of what was created — if you screenshot it, it's meaningless. (2) The WhatsApp button currently does nothing. A pre-written Arabic message template is the single highest-value feature for growth — one tap and the invite is ready to send.

### HTML
Replace the entire `<div class="wizard-success">` content in Step 5 with:

```html
<div class="wizard-success">
    <div class="success-icon">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6L9 17l-5-5"/></svg>
    </div>
    <h2>سِلفتك جاهزة!</h2>
    <p style="color:var(--text-light);font-size:14px;margin-bottom:20px;">أرسل الدعوات لأعضائك للانضمام</p>

    <!-- Summary card -->
    <div class="success-summary-card" id="successSummaryCard">
        <div class="ssc-name" id="sscName">—</div>
        <div class="ssc-grid">
            <div class="ssc-item">
                <span class="ssc-label">الحصة الشهرية</span>
                <span class="ssc-value" id="sscMonthly">—</span>
            </div>
            <div class="ssc-item">
                <span class="ssc-label">إجمالي الاستلام</span>
                <span class="ssc-value highlight" id="sscPayout">—</span>
            </div>
            <div class="ssc-item">
                <span class="ssc-label">عدد الأعضاء</span>
                <span class="ssc-value" id="sscMembers">—</span>
            </div>
            <div class="ssc-item">
                <span class="ssc-label">مدة السِلفة</span>
                <span class="ssc-value" id="sscDuration">—</span>
            </div>
        </div>
        <div class="ssc-dates" id="sscDates">—</div>
    </div>

    <!-- Pending members -->
    <div class="pending-members-section">
        <div class="pending-members-label">الأعضاء المدعوون</div>
        <div class="pending-members-list" id="pendingMembersList"></div>
    </div>

    <p class="success-notify-nudge">🔔 سنُعلمك فور موافقة كل عضو</p>

    <!-- WhatsApp primary share -->
    <button class="whatsapp-share-btn" id="whatsappShareBtn">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor"><path d="M17.472 14.382c-.297-.149-1.758-.867-2.03-.967-.273-.099-.471-.148-.67.15-.197.297-.767.966-.94 1.164-.173.199-.347.223-.644.075-.297-.15-1.255-.463-2.39-1.475-.883-.788-1.48-1.761-1.653-2.059-.173-.297-.018-.458.13-.606.134-.133.298-.347.446-.52.149-.174.198-.298.298-.497.099-.198.05-.371-.025-.52-.075-.149-.669-1.612-.916-2.207-.242-.579-.487-.5-.669-.51-.173-.008-.371-.01-.57-.01-.198 0-.52.074-.792.372-.272.297-1.04 1.016-1.04 2.479 0 1.462 1.065 2.875 1.213 3.074.149.198 2.096 3.2 5.077 4.487.709.306 1.262.489 1.694.625.712.227 1.36.195 1.871.118.571-.085 1.758-.719 2.006-1.413.248-.694.248-1.289.173-1.413-.074-.124-.272-.198-.57-.347m-5.421 7.403h-.004a9.87 9.87 0 01-5.031-1.378l-.361-.214-3.741.982.998-3.648-.235-.374a9.86 9.86 0 01-1.51-5.26c.001-5.45 4.436-9.884 9.888-9.884 2.64 0 5.122 1.03 6.988 2.898a9.825 9.825 0 012.893 6.994c-.003 5.45-4.437 9.884-9.885 9.884m8.413-18.297A11.815 11.815 0 0012.05 0C5.495 0 .16 5.335.157 11.892c0 2.096.547 4.142 1.588 5.945L.057 24l6.305-1.654a11.882 11.882 0 005.683 1.448h.005c6.554 0 11.89-5.335 11.893-11.893a11.821 11.821 0 00-3.48-8.413z"/></svg>
        دعوة الجميع عبر واتساب
    </button>

    <!-- Copy link fallback -->
    <div class="invite-alt-divider">أو شارك الرابط مباشرة</div>
    <div class="invite-link-box">
        <input type="text" id="inviteLink" value="silfa.app/join/xK9mQ2" readonly>
        <button class="copy-btn" id="copyBtn">نسخ الرابط</button>
    </div>
</div>
```

### CSS
```css
.success-summary-card {
    background: var(--light-bg); border: 1.5px solid var(--border);
    border-radius: var(--radius); padding: 18px 16px; margin-bottom: 20px; text-align: right;
}
.ssc-name { font-size: 18px; font-weight: 800; color: var(--navy); margin-bottom: 14px; text-align: center; }
.ssc-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-bottom: 12px; }
.ssc-item { display: flex; flex-direction: column; gap: 3px; }
.ssc-label { font-size: 11px; color: var(--text-light); font-weight: 500; }
.ssc-value { font-size: 15px; font-weight: 700; color: var(--navy); }
.ssc-value.highlight { color: var(--teal); }
.ssc-dates { font-size: 12px; color: var(--text-light); text-align: center; padding-top: 10px; border-top: 1px solid var(--border); }

.whatsapp-share-btn {
    display: flex; align-items: center; justify-content: center; gap: 10px;
    width: 100%; padding: 16px; border-radius: var(--radius-sm); border: none;
    background: #25D366; color: white; font-size: 16px; font-weight: 700;
    font-family: inherit; cursor: pointer; margin-bottom: 16px;
    transition: background 0.2s; box-shadow: 0 4px 16px rgba(37,211,102,0.3);
}
.whatsapp-share-btn:hover { background: #1fbe5a; }
```

### JS
Update `populatePendingList()` to also fill the summary card:
```js
function populatePendingList() {
    // Fill summary card
    const name    = circleData.name || 'سِلفتي';
    const monthly = circleData.amount;
    const payout  = monthly * circleData.members;
    const total   = circleData.members;

    const el = id => document.getElementById(id);
    if (el('sscName'))     el('sscName').textContent     = name;
    if (el('sscMonthly'))  el('sscMonthly').textContent  = formatAr(monthly) + ' د.ع';
    if (el('sscPayout'))   el('sscPayout').textContent   = formatAr(payout) + ' د.ع';
    if (el('sscMembers'))  el('sscMembers').textContent  = toAr(total) + ' أعضاء';
    if (el('sscDuration')) el('sscDuration').textContent = toAr(total) + ' أشهر';
    if (el('sscDates'))    el('sscDates').textContent    = `من ${getPayoutMonth(0)} إلى ${getPayoutMonth(total - 1)}`;

    // Fill pending members list
    const list = document.getElementById('pendingMembersList');
    if (!list) return;
    if (selectedContacts.length === 0) {
        list.innerHTML = '<div style="font-size:12px;color:var(--text-light);text-align:center;padding:10px 0;">لم يُرسَل أي دعوات</div>';
        return;
    }
    list.innerHTML = selectedContacts.map((c, i) => `
        <div class="pending-member-row" style="animation: wizardFadeIn 0.35s ease ${i * 0.07}s both;">
            <div class="pending-member-avatar" style="background:${c.color};">${c.name.charAt(0)}</div>
            <span class="pending-member-name">${c.name}</span>
            <span class="pending-status-badge">⏳ في الانتظار</span>
        </div>
    `).join('');
}
```

Add WhatsApp share button listener in the JS (near the `copyBtn` listener):
```js
document.getElementById('whatsappShareBtn').addEventListener('click', () => {
    const name    = circleData.name || 'سِلفتي';
    const monthly = formatAr(circleData.amount);
    const payout  = formatAr(circleData.amount * circleData.members);
    const link    = document.getElementById('inviteLink').value;

    const msg = `السلام عليكم! 👋\n\nدعوتك للانضمام إلى سِلفة "${name}" 🏦\n\n📅 الحصة الشهرية: ${monthly} د.ع\n💰 ستستلم في دورك: ${payout} د.ع\n\nاضغط للانضمام:\n${link}\n\n— عبر تطبيق سِلفة`;

    const encoded = encodeURIComponent(msg);
    window.open(`https://wa.me/?text=${encoded}`, '_blank');
});
```

---

## TICKET 14 — Iraqi Phone Validation (Step 3 Manual Add)

**Why:** Iraqi numbers follow the format `07XX-XXX-XXXX` (10 digits starting with 07). Validating this specifically prevents garbage data and signals the app knows its audience.

### JS
In `addManualContact()`, after `const phone = phoneInput.value.trim();`, add:
```js
// Validate Iraqi phone format if provided
if (phone && !/^07[0-9]{9}$/.test(phone.replace(/[\s\-]/g, ''))) {
    phoneInput.style.borderColor = '#DC2626';
    phoneInput.placeholder = 'مثال: 07701234567';
    setTimeout(() => {
        phoneInput.style.borderColor = '';
        phoneInput.placeholder = 'رقم الهاتف';
    }, 2000);
    return;
}
```

No CSS changes needed — uses existing `.wizard-input` border.

---

## Deployment

After each ticket, run:
```bash
netlify deploy --dir=. --prod
```

Site: https://silfaa.netlify.app

Tickets are independent and can be implemented in any order. Tickets 1, 6, 10, and 13 have the highest user-visible impact.
