# Frontend Guidelines — Bookmark AI MVP (Locked)

These guidelines codify **exact, enforceable** standards for a consistent, accessible, and fast UI. Items marked **[LOCKED]** are non‑negotiable and mirror the Source of Truth.

---

## 1) Scope & Architecture — [LOCKED]
- **App scope (MVP):** Landing, auth/billing gating, paid‑only dashboard to paste URLs → auto‑fetch metadata → **neutral English summary ≤300 chars** → search (keyword + optional semantic). **Grid cards** with a **side‑drawer** for edits; **click tracking** via server redirect. No transcripts in MVP.
- **Stack:** React + **Vite**, **React Router v7 with SSR**, TypeScript, Tailwind, **shadcn/ui**, **Lucide** (Tabler optional). Backend: Convex; Auth: Clerk; Billing: Polar; Summaries: OpenAI (GPT‑5) via Convex; Vector search: Qdrant. Deploy on Vercel.
- **Do not re‑plan providers** or versions. Follow the template pins.

---

## 2) Design Tokens — [LOCKED]
Use these tokens globally. If Tailwind variables already exist, map them 1:1.

- **Typography:** Nunito **400 / 600 / 700**
- **Colors:**
  - `primary #3B82F6` (buttons, focus, links)
  - `accent #F43F5E` (emphasis/AI search toggle)
  - `bg #F9FAFB`, `surface #FFFFFF`
  - `muted #E5E7EB`
  - `text #111827`, `subtext #4B5563`
  - `success #22C55E`, `warning #F59E0B`, `danger #EF4444`
- **Radius:** `12px` default (cards/drawers/buttons). Larger radii allowed for marketing sections only.
- **Spacing:** 4px base grid.
- **Focus ring:** 2px, primary color, `focus-visible` only.
- **Elevation:** soft shadows at 0 / 2 / 6 tiers.

**Implementation notes:**
- Load Nunito (display=swap) in the document head and set Tailwind `font-sans` to Nunito. Avoid FOIT/FOUT.

```tsx
// app/root.tsx (head <Links />)
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossOrigin="anonymous" />
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700&display=swap" />
```

```css
/* app/app.css */
:root { --radius: 12px; }
@layer base {
  :root { --font-sans: "Nunito", ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Inter, "Helvetica Neue", Arial, "Apple Color Emoji", "Segoe UI Emoji"; }
  * { @apply focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary; }
}
```

---

## 3) Layout & Components

### 3.1 Grid & Cards — [LOCKED]
- **Aspect:** strict **16:9** (≈240×135 desktop / ≈160×90 mobile).
- **Contents:** thumbnail/icon, **title**, **clickable URL** (opens new tab), **date added**.
- **Hover:** subtle elevation; no color shifts that reduce contrast.
- **Click:** opens **side‑drawer** (not a full page).

**Card skeletons (shimmer 1200ms):**
```tsx
// components/dashboard/CardSkeleton.tsx
export function CardSkeleton() {
  return (
    <div className="rounded-[12px] bg-surface shadow-sm overflow-hidden">
      <div className="relative w-full" style={{ aspectRatio: "16/9" }}>
        <div className="h-full w-full animate-pulse [animation-duration:1200ms] bg-muted" />
      </div>
      <div className="p-3 space-y-2">
        <div className="h-4 w-3/4 animate-pulse [animation-duration:1200ms] bg-muted rounded" />
        <div className="h-3 w-1/2 animate-pulse [animation-duration:1200ms] bg-muted rounded" />
      </div>
    </div>
  );
}
```

### 3.2 Drawer — [LOCKED]
- **Width:** **440–480px** on desktop; 100% on mobile.
- **Fields:** **Title** and **Summary** are editable. **Cmd/Ctrl+S saves**. **Esc closes**. Success/failure toasts.
- **No copy URL button**; link is a normal anchor (`target="_blank" rel="noopener"`).

**Save handler with Cmd/Ctrl+S:**
```tsx
// components/drawer/EditBookmarkDrawer.tsx
import { useEffect } from "react";
import { Button } from "~/components/ui/button";

export function EditBookmarkDrawer({ isOpen, onClose, onSave, formRef }: {
  isOpen: boolean; onClose: () => void; onSave: () => Promise<void>;
  formRef: React.RefObject<HTMLFormElement>;
}) {
  useEffect(() => {
    function onKey(e: KeyboardEvent) {
      if (!isOpen) return;
      const isMetaS = (e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "s";
      if (isMetaS) { e.preventDefault(); void onSave(); }
      if (e.key === "Escape") { e.preventDefault(); onClose(); }
    }
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [isOpen, onClose, onSave]);

  return (
    <aside role="dialog" aria-modal className="w-full sm:w-[460px] p-4">
      <form ref={formRef} className="space-y-3">
        {/* Title & Summary inputs here (labeled) */}
        <div className="flex gap-2 justify-end">
          <Button type="button" onClick={() => void onSave()}>Save</Button>
          <Button variant="outline" type="button" onClick={onClose}>Close (Esc)</Button>
        </div>
      </form>
    </aside>
  );
}
```

### 3.3 Inputs, Buttons, Icons
- Use **shadcn/ui** primitives from `~/components/ui/*`.
- Icons: **Lucide** for all UI (Tabler optional in marketing pages).

---

## 4) States — [LOCKED]
- **Empty (dashboard):** Centered input with friendly placeholder (e.g., *“Paste a link to save and summarize.”*) and **Enter submits**.
- **Loading:** Optimistic card shows **“Fetching…” → “Summarizing…”**.
- **Error:** Card shows **Error chip** (tooltip from `errorReason`) and a **Retry** button that re‑runs the correct stage.
- **Saved:** Toast on successful edit/save.

---

## 5) Hotkeys & Accessibility — [LOCKED]
- **Hotkeys:**
  - **Enter** → submit Add URL form
  - **Cmd/Ctrl+K** → focus search
  - **Cmd/Ctrl+S** → save in drawer
  - **Esc** → close drawer
- **A11y rules:**
  - Use **focus-visible** only (never hide focus for keyboard users).
  - Provide **semantic landmarks** (`<header>`, `<nav>`, `<main>`, `<footer>`).
  - Every input requires a **<label>** or `aria-label`.
  - **aria-live="polite"** for async status text (Fetching → Summarizing).
  - Maintain contrast **≥ WCAG AA**.

**Hotkeys util (global, input‑safe):**
```tsx
// hooks/useHotkeys.ts
import { useEffect } from "react";

export function useHotkeys({ onEnter, onSearch, onSave, onClose }: {
  onEnter?: () => void; onSearch?: () => void; onSave?: () => void; onClose?: () => void;
}) {
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      const tag = (e.target as HTMLElement)?.tagName;
      const inTextField = ["INPUT","TEXTAREA"].includes(tag) || (e.target as HTMLElement)?.isContentEditable;
      // Enter submits only for the Add URL field
      if (e.key === "Enter" && onEnter && inTextField) onEnter();
      if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "k") { e.preventDefault(); onSearch?.(); }
      if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "s") { e.preventDefault(); onSave?.(); }
      if (e.key === "Escape") { onClose?.(); }
    };
    window.addEventListener("keydown", handler);
    return () => window.removeEventListener("keydown", handler);
  }, [onEnter, onSearch, onSave, onClose]);
}
```

**Async status (screen‑reader friendly):**
```tsx
// components/AsyncStatus.tsx
export function AsyncStatus({ status }: { status: "idle"|"fetching"|"summarizing"|"error" }) {
  const text = status === "fetching" ? "Fetching…" : status === "summarizing" ? "Summarizing…" : status === "error" ? "Error" : "";
  return <p aria-live="polite" className="sr-only">{text}</p>;
}
```

---

## 6) Content Style
- Microcopy is **neutral, concise, and English‑only**. No emojis, no hype. Summaries capture the **main idea** (not timestamps/commands).
- Prefer verbs: *Add, Save, Retry, Search*.
- Tone on errors: factual and short — e.g., “Metadata fetch failed. Retry.”

> **Missing (provide to finalize):** Product name & brand voice nuances; currency and monthly price for pricing examples.

---

## 7) Performance Playbook
- **SSR (React Router v7)** + granular **code‑splitting** via route boundaries.
- **Prefetch critical routes** for authenticated users (dashboard, drawer data). Use React Router’s `prefetch="intent"` on links.
- **Skeleton‑first rendering** for cards; avoid layout shift.
- **Defer non‑critical JS** on landing; hydrate dashboard first.
- **Cache images** with proper sizes; serve thumbnails ~480w for grid.

```tsx
// Prefetch example
import { Link } from "react-router";
<Link to="/dashboard" prefetch="intent" className="btn">Open Dashboard</Link>
```

---

## 8) Client Config (safe subset)
Only consume **client‑safe** constants (no secrets). Expose a thin export that reads from `lib/config.ts` server module and re‑exports safe values.

```ts
// lib/config.ts (server‑authoritative)
export const SEARCH = { VECTOR_DIM: 1536, HNSW_M: 32, HNSW_EF_CONSTRUCT: 256, SEARCH_EF: 96, MMR_LAMBDA: 0.7, CANDIDATE_POOL: 64, FINAL_CUTOFF: 0.22, RELATIVE_TOP_RATIO: 0.70 };
export const INGEST = { SUMMARY_MAX_CHARS: 300, BACKOFF_STEPS_MS: [500,1000,2000,4000] };
```

```ts
// lib/config.client.ts (no secrets)
export { SEARCH, INGEST } from "./config";
```

```tsx
// usage in client
import { SEARCH } from "~/lib/config.client";
console.log(SEARCH.MMR_LAMBDA); // 0.7 — used to label UI controls, not to run queries client‑side
```

---

## 9) Search & Ingest UI Hooks — What the frontend must reflect
- **Add URL** input submits via **Enter**; immediately show an **optimistic card**.
- Replace placeholder text with **Fetching → Summarizing** states; on success morph to final title/thumbnail.
- **AI Search** is an explicit action (e.g., button/toggle) that triggers server‑side vector query and merges with keyword results; **do not hard cap** results in UI — render whatever the threshold returns.

---

## 10) QA Checklist (UI)
- [ ] Keyboard: Enter/Cmd+K/Cmd+S/Esc behave exactly as specified.
- [ ] Focus rings visible on all interactive elements; ESC closes drawer; landmarks present.
- [ ] Card aspect 16:9 at all breakpoints; no CLS.
- [ ] Error chip shows tooltip based on `errorReason`; Retry behaves per pipeline stage.
- [ ] Toasts on save success/failure; Saved state persists.
- [ ] Prefetch works for dashboard routes; skeletons use ~1200ms pulse.

---

## 11) Copy Library (starter examples)
- **Empty state placeholder:** “Paste a link to save and summarize.”
- **Loading labels:** “Fetching…”, “Summarizing…”
- **Error:** “Metadata fetch failed. Retry.” / “Summarization failed. Retry.”
- **Saved toast:** “Changes saved.”

---

## 12) Guardrails — [LOCKED]
- Do **not** introduce alternate providers or packages.
- All async status text must be **aria‑live**.
- UI must respect **server gating** (paid dashboard) and use the template’s Clerk/Polar/Convex wiring.
- Links to saved URLs open via the **tracked redirect** route; never direct‑count client‑side.

---

## 13) Open Items (Provide to finalize examples)
- **Product name + brand voice** (microcopy nuances and CTA tone).
- **Currency + monthly price** for pricing CTA examples (copy only; billing handled by Polar template).

