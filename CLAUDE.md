# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file Arabic RTL landing page + circle-creation wizard for **سِلفة** (Silfa), a rotating savings circle (جمعية/جمعيات) app targeting Iraq. The entire project lives in one file: `index.html`.

## Deploy

```bash
netlify deploy --dir=. --prod
```

Site is live at: https://silfaa.netlify.app

No build step. Edit `index.html` and deploy directly.

## Architecture

`index.html` is structured in three major sections:

### 1. `<head>` — Dependencies & Config
- Google Fonts: Noto Sans Arabic
- Tailwind CSS via CDN Play (`https://cdn.tailwindcss.com`) with custom color tokens: `navy`, `teal`, `gold`
- All custom CSS in a single `<style>` block

### 2. `<body>` — Two views in one page
- **Landing page** (`<nav>`, `<section class="hero">`, features, how-it-works, FAQ, CTA sections) — the descriptive marketing page
- **Wizard overlay** (`<div class="wizard-overlay active" id="wizardOverlay">`) — full-screen circle-creation flow, shown by default on load via `active` class in HTML

### 3. `<script>` — All JS inline at bottom
Single script block handles both the landing page (scroll effects, mobile menu, intersection observer) and the wizard.

## Wizard Architecture

The wizard is a **5-step full-screen flow**:
1. **تفاصيل** — Circle name + monthly amount (presets or custom)
2. **الأعضاء** — Member count stepper (3–20)
3. **الدعوات** — Tap-to-select contacts (X of Y counter pill)
4. **الجدول** — Cycle planning with drag-to-reorder + shuffle
5. **تم** — Success screen with pending invites + share link

Key state variables: `currentStep`, `circleData`, `selectedContacts`, `cycleOrder`, `dragSrcIndex`.

### View switching pattern
- Wizard is **always `active` in HTML** — it's the default landing view
- Close/Home button calls `goToLanding()` → removes `active` class, restores `body.style.overflowY`, scrolls to top
- CTA buttons (`.open-wizard`) re-add `active` and lock scroll
- Do **not** call `resetWizard()` before `const membersCountEl / minusBtn / plusBtn` are declared (they're declared later in the script) — this caused a ReferenceError that silently broke all event listeners

### Layout constraint (critical)
The flex column must be: `wizard-fs (100dvh, overflow:hidden)` → `wizard-logo-bar (flex-shrink:0)` → `wizard-topbar (flex-shrink:0)` → `wizard-body (flex:1, min-height:0, overflow-y:auto)` → `wizard-bottom-nav (flex-shrink:0)`. Without `min-height:0` on `.wizard-body`, the bottom nav is pushed off-screen.

## RTL / Arabic conventions
- `lang="ar" dir="rtl"` on `<html>`
- Arabic numerals via `toAr(n)` helper (converts ASCII digits to ٠١٢...)
- Currency formatted as `formatAr(n) + ' د.ع'`
- Month names in `arMonths[]` array
