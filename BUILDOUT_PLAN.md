# Google Calendar Quick Duplicate — Fork Buildout Plan

> **Purpose:** Complete rewrite of [fabiosangregorio/google-calendar-quick-duplicate](https://github.com/fabiosangregorio/google-calendar-quick-duplicate) to fix chronic selector breakage, modernize the architecture, and add key missing features.
>
> **Target repo:** `mattlampson/google-calendar-quick-duplicate` (fork)
>
> **Target devices:** MacBook Air M4, Mac Mini M1, Chromebook (all Chrome/Chromium)
>
> **This document is the execution spec for Claude Code.** Follow it phase-by-phase. Each phase has explicit deliverables and acceptance criteria. Do not skip phases or combine them — commit after each phase passes its checks.

---

## Table of Contents

1. [Context & Problem Statement](#1-context--problem-statement)
2. [Architecture Overview](#2-architecture-overview)
3. [File Structure](#3-file-structure)
4. [Phase 0 — Fork & Scaffold](#phase-0--fork--scaffold)
5. [Phase 1 — Resilient Selector Engine](#phase-1--resilient-selector-engine)
6. [Phase 2 — MutationObserver Core](#phase-2--mutationobserver-core)
7. [Phase 3 — Duplicate Button Injection](#phase-3--duplicate-button-injection)
8. [Phase 4 — Duplication Flow](#phase-4--duplication-flow)
9. [Phase 5 — View Preservation](#phase-5--view-preservation)
10. [Phase 6 — Options Page & Storage](#phase-6--options-page--storage)
11. [Phase 7 — Theming & Polish](#phase-7--theming--polish)
12. [Phase 8 — Packaging & Documentation](#phase-8--packaging--documentation)
13. [Selector Strategy Reference](#selector-strategy-reference)
14. [Known Google Calendar DOM Patterns](#known-google-calendar-dom-patterns)

---

## 1. Context & Problem Statement

The upstream extension works by injecting a "Duplicate" button into Google Calendar's event preview popup, then automating Google's own duplicate flow (open options menu → click "Duplicate" → auto-save). It also supports Alt+click as a shortcut.

### Why it breaks repeatedly

The extension hardcodes obfuscated CSS class names from Google Calendar's compiled output (e.g., `.NlL62b`, `.pPTZAe`, `.d29e1c`). Google periodically recompiles their frontend, changing these class names. Every time this happens, the extension stops working entirely. This has caused breakage at least 3 times (issues #19, #22, #29) and is the #1 source of user complaints.

### Why a rewrite is better than a patch

The entire architecture — `setInterval` polling loops, hardcoded class selectors, fragile DOM cloning hacks for view restoration — needs to change. Patching selectors is a treadmill. A rewrite targeting stable DOM attributes and modern APIs (MutationObserver, URL-based navigation) will be dramatically more maintainable.

### Chrome Extension Platform Status (as of March 2026)

The upstream extension already uses Manifest V3, which is correct. MV2 was fully deprecated — disabled by default in March 2025, with the re-enable option removed in July 2025. MV3 is the only option for Chrome extensions now. Our rewrite stays on MV3 with no service worker needed (content script only + options page). The `storage` permission is the only permission we request.

### What Google added natively (August 2025)

Google Calendar now supports Ctrl+drag (Cmd+drag on Mac) to duplicate events in Day/Week/Month views. You hold Ctrl/Cmd, then drag an existing event to a new time slot to create a copy. This does NOT work in Schedule/Agenda view or Year view. It also requires dragging — a stationary Ctrl+click does nothing natively.

The extension still provides value beyond this native shortcut:
- A **visible button** in the event popup (discoverability — no shortcut to memorize)
- **"Edit before save" mode** (the native drag creates the copy immediately)
- Works in **Schedule view** (native shortcut does not)
- **Stationary click** duplication (native requires dragging to a new slot)

**CRITICAL for our extension:** Our modifier+click handler MUST NOT interfere with native Ctrl+drag. See ADD-1 in the Addendum for the drag detection implementation.

---

## 2. Architecture Overview

```
manifest.json          — MV3 manifest, content script + options page
src/
  logger.js            — Structured logging + health check diagnostics
  selectors.js         — Selector engine: resilient discovery with fallback chains
  observer.js          — MutationObserver: watches for event panel open/close
  button.js            — Duplicate button: creation, injection, styling
  duplicator.js        — Duplication flow: orchestrates the actual duplicate action
  view-guard.js        — View preservation: captures/restores URL state
  constants.js         — Shared constants and default config
  utils.js             — Small utility functions + toast notifications
styles/
  button.css           — Self-contained button + toast styles (not dependent on GCal classes)
options/
  options.html         — Options page markup
  options.css          — Options page styles
  options-page.js      — Options page logic
assets/
  icon16.png           — Extension icons (reuse from upstream)
  icon48.png
  icon128.png
```

### Key Design Principles

1. **No hardcoded obfuscated class names.** Every selector uses a fallback chain: `[data-attribute]` → `[jsname]` → `[aria-label]` → structural query. Obfuscated classes are only used as a last-resort fallback and are isolated in `selectors.js` so they can be updated in one place.

2. **No `setInterval` polling.** All DOM waiting uses `MutationObserver` with Promise-based wrappers and timeouts. (Exception: a 1/second URL heartbeat for SPA navigation detection.)

3. **URL-based view preservation** instead of mini-calendar DOM hacking.

4. **Self-contained styles.** The duplicate button uses its own CSS, injected via a separate stylesheet, not dependent on Google's class names for appearance.

5. **All config in `chrome.storage.sync`** so settings roam across devices.

6. **Structured logging and diagnostics.** `__GCQD_HEALTH()` console command for instant selector diagnostics. `__GCQD_DEBUG = true` for verbose logging.

---

## 3. File Structure

After the rewrite, the repo root should look like this:

```
google-calendar-quick-duplicate/
├── manifest.json
├── src/
│   ├── logger.js           ← structured logging + health check
│   ├── constants.js
│   ├── utils.js             ← includes showToast()
│   ├── selectors.js
│   ├── observer.js          ← debounced, isConnected checks
│   ├── button.js
│   ├── duplicator.js        ← toast feedback, graceful degradation
│   ├── view-guard.js
│   └── content.js           ← entry point (replaces app.js)
├── styles/
│   └── button.css           ← includes toast styles
├── options/
│   ├── options.html         ← script paths from extension root
│   ├── options.css
│   └── options-page.js
├── assets/
│   ├── icon16.png
│   ├── icon48.png
│   └── icon128.png
├── README.md
├── CHANGELOG.md
├── LICENSE
└── .gitignore
```

Delete the old `app.js` and `.idea/` directory from the upstream repo.

---

## Phase 0 — Fork & Scaffold

### Tasks

1. Fork `fabiosangregorio/google-calendar-quick-duplicate` to `mattlampson/google-calendar-quick-duplicate`. **(ALREADY DONE)**
2. Clone locally.
3. Delete `app.js` and `.idea/` directory.
4. Create the directory structure shown above (`src/`, `styles/`, `options/`).
5. Create empty placeholder files for each module listed in the file structure.
6. Update `.gitignore`:
   ```
   .idea/
   .DS_Store
   *.crx
   *.pem
   *.zip
   node_modules/
   ```
7. Update `manifest.json` to the new structure (see below).

### Updated `manifest.json`

```json
{
  "name": "Google Calendar Quick Duplicate",
  "icons": {
    "16": "assets/icon16.png",
    "48": "assets/icon48.png",
    "128": "assets/icon128.png"
  },
  "version": "2.0.0",
  "description": "Quickly duplicate Google Calendar events with a single click or keyboard shortcut.",
  "manifest_version": 3,
  "permissions": ["storage"],
  "content_scripts": [
    {
      "matches": ["https://calendar.google.com/calendar/*"],
      "js": [
        "src/logger.js",
        "src/constants.js",
        "src/utils.js",
        "src/selectors.js",
        "src/observer.js",
        "src/button.js",
        "src/duplicator.js",
        "src/view-guard.js",
        "src/content.js"
      ],
      "css": ["styles/button.css"],
      "run_at": "document_idle"
    }
  ],
  "options_ui": {
    "page": "options/options.html",
    "open_in_tab": false
  }
}
```

### Acceptance Criteria

- [ ] Repo forked and cloned
- [ ] Old files removed (`app.js`, `.idea/`)
- [ ] New directory structure exists with placeholder files
- [ ] `manifest.json` updated and valid JSON
- [ ] Extension loads in Chrome via `chrome://extensions` (Load unpacked) without errors
- [ ] Commit: `"Phase 0: Fork scaffold and project restructure"`

---

## Phase 1 — Resilient Selector Engine + Logger

### File: `src/logger.js`

Load this FIRST in the manifest. See ADD-3 in the Addendum for the full implementation including `__GCQD_HEALTH()` and `__GCQD_DEBUG`.

### File: `src/selectors.js`

This is the most critical module. It replaces all hardcoded class selectors with resilient fallback chains.

### Design

Create a `Selectors` object that exposes methods for finding each key element. Each method tries multiple strategies in order of stability:

```javascript
"use strict";

/**
 * Resilient selector engine for Google Calendar DOM elements.
 *
 * Strategy priority (most stable → least stable):
 *   1. data-* attributes (semantic, rarely change)
 *   2. jsname attributes (Google's internal binding names, moderately stable)
 *   3. aria-label / role attributes (accessibility, rarely change)
 *   4. Structural queries (tag + position relationships)
 *   5. Obfuscated class names (LAST RESORT — isolated here for easy updates)
 *
 * If Google changes their DOM and selectors break, only this file needs updating.
 */
const Selectors = (() => {
  // --- LAST-RESORT CLASS NAMES (update these when Google recompiles) ---
  // Timestamped so you know when they were last verified.
  // Last verified: [DATE OF INITIAL BUILD]
  const FALLBACK_CLASSES = {
    calendarEvent: "NlL62b",        // Calendar event chip on the grid
    eventPanel: "pPTZAe",           // Event preview popup container
    optionsButton: "d29e1c",        // "..." options button in event popup
    circleButton: "VbA1ue",         // Dark circle background on image events
    miniCalDay: "IOneve",           // Mini calendar day cell
    miniCalCurrentDay: "pWJCO",     // Mini calendar highlighted (current) day
    miniCalNotThisMonth: "q2d9Ze",  // Mini calendar grayed-out day
  };

  /**
   * Try each selector in order, return first match or null.
   * @param {Element|Document} root - Element to search within
   * @param {string[]} selectors - Array of CSS selectors to try
   * @returns {Element|null}
   */
  function queryFirst(root, selectors) {
    for (const sel of selectors) {
      try {
        const el = root.querySelector(sel);
        if (el) return el;
      } catch (e) {
        Logger.debug("Invalid selector skipped:", sel);
      }
    }
    return null;
  }

  /**
   * Try each selector in order, return first non-empty NodeList as Array.
   * @param {Element|Document} root
   * @param {string[]} selectors
   * @returns {Element[]}
   */
  function queryAll(root, selectors) {
    for (const sel of selectors) {
      try {
        const els = root.querySelectorAll(sel);
        if (els.length > 0) return Array.from(els);
      } catch (e) {
        Logger.debug("Invalid selector skipped:", sel);
      }
    }
    return [];
  }

  /**
   * Find a menu item by its visible text content.
   * Used as final fallback when jsname/aria selectors fail.
   * Includes common translations for i18n support (ADD-14).
   * @param {string} text - The text to search for (case-insensitive)
   * @returns {Element|null}
   */
  function findMenuItemByText(text) {
    const variants = [
      text,
      "Duplicate", "Dupliquer", "Duplicar", "Duplizieren",
      "Duplica", "Dupliceren", "複製", "Дублировать", "複製する"
    ];
    const menuItems = document.querySelectorAll(
      '[role="menuitem"], [role="option"], li[data-action], li[jsname]'
    );
    for (const item of menuItems) {
      const itemText = item.textContent.trim().toLowerCase();
      for (const variant of variants) {
        if (itemText.includes(variant.toLowerCase())) {
          return item;
        }
      }
    }
    return null;
  }

  return {
    FALLBACK_CLASSES,
    queryFirst,
    queryAll,

    findEvent(eventId) {
      if (eventId) {
        return queryFirst(document, [
          `[data-eventid="${eventId}"]`,
          `.${FALLBACK_CLASSES.calendarEvent}[data-eventid="${eventId}"]`,
        ]);
      }
      return queryAll(document, [
        "[data-eventid]",
        `.${FALLBACK_CLASSES.calendarEvent}[data-eventid]`,
      ]);
    },

    findEventPanel() {
      // Don't detect panels when we're in the edit form (ADD-8)
      if (window.location.href.includes("/eventedit") ||
          window.location.href.includes("/duplicate")) {
        return null;
      }
      return queryFirst(document, [
        '[data-eventid][role="dialog"]',
        '[jsname="rDHQYd"]',
        `div.${FALLBACK_CLASSES.eventPanel}`,
        '[role="dialog"][data-eventid]',
      ]);
    },

    findOptionsButton(panel) {
      const root = panel || document;
      return queryFirst(root, [
        '[aria-label="Options"]',
        '[aria-label="More options"]',
        '[data-tooltip="Options"]',
        `[jsname="VkLyEc"].${FALLBACK_CLASSES.optionsButton}`,
        `.${FALLBACK_CLASSES.optionsButton}`,
      ]);
    },

    findDuplicateMenuItem() {
      return queryFirst(document, [
        '[jsname="lbYRR"]',
        '[data-action="duplicate"]',
      ]) || findMenuItemByText("Duplicate");
    },

    findSaveButton() {
      return queryFirst(document, [
        '[jsname="x8hlje"]',
        '[aria-label="Save"]',
        '[data-tooltip="Save"]',
        'button[jsname="x8hlje"]',
      ]);
    },

    hasCircleButtons(panel) {
      const root = panel || document;
      return queryFirst(root, [
        `.${FALLBACK_CLASSES.circleButton}`,
      ]) != null;
    },

    findMiniCalCurrentDay() {
      return queryFirst(document, [
        '[data-date][aria-selected="true"]',
        `.${FALLBACK_CLASSES.miniCalCurrentDay}[data-date]`,
        `td.${FALLBACK_CLASSES.miniCalCurrentDay}`,
      ]);
    },
  };
})();
```

### Acceptance Criteria

- [ ] `src/logger.js` implements Logger with debug/info/warn/error, `__GCQD_HEALTH()`, `__GCQD_DEBUG`
- [ ] `src/selectors.js` implements the full `Selectors` API as specified above
- [ ] All fallback class names are isolated in `FALLBACK_CLASSES` with a date comment
- [ ] No hardcoded obfuscated class names exist outside of `selectors.js`
- [ ] `queryFirst` and `queryAll` gracefully handle invalid selectors (try/catch)
- [ ] `findEventPanel()` excludes edit form context (ADD-8)
- [ ] `findMenuItemByText()` includes i18n variants (ADD-14)
- [ ] All modules use `Logger.*` instead of raw `console.*` (ADD-3)
- [ ] Commit: `"Phase 1: Resilient selector engine with fallback chains + structured logger"`

---

## Phase 2 — MutationObserver Core

### File: `src/observer.js`

Replace all `setInterval` polling with a centralized `MutationObserver` system.

### Design

```javascript
"use strict";

/**
 * MutationObserver-based DOM watcher for Google Calendar.
 * Replaces all setInterval polling from the original extension.
 *
 * Provides:
 *   - waitForElement(selectorFn, opts): Promise that resolves when an element appears
 *   - watchEventPanel(onOpen, onClose): watches for event panel open/close
 *   - destroy(): disconnects all observers
 */
const Observer = (() => {
  const observers = [];

  /**
   * Wait for an element to appear in the DOM.
   * Uses requestAnimationFrame for debouncing (ADD-2).
   */
  function waitForElement(selectorFn, opts = {}) {
    const { timeout = 5000, root = document.body } = opts;

    return new Promise((resolve, reject) => {
      const existing = selectorFn();
      if (existing) {
        resolve(existing);
        return;
      }

      let observer;
      let timeoutId;
      let pendingCheck = false;

      const cleanup = () => {
        if (observer) {
          observer.disconnect();
          const idx = observers.indexOf(observer);
          if (idx > -1) observers.splice(idx, 1);
        }
        if (timeoutId) clearTimeout(timeoutId);
      };

      observer = new MutationObserver(() => {
        if (!pendingCheck) {
          pendingCheck = true;
          requestAnimationFrame(() => {
            pendingCheck = false;
            const el = selectorFn();
            if (el) {
              cleanup();
              resolve(el);
            }
          });
        }
      });

      observer.observe(root, { childList: true, subtree: true });
      observers.push(observer);

      if (timeout > 0) {
        timeoutId = setTimeout(() => {
          cleanup();
          reject(new Error(`[GCQD] waitForElement timed out after ${timeout}ms`));
        }, timeout);
      }
    });
  }

  /**
   * Watch for the event preview panel to open/close.
   * Debounced to avoid excessive DOM queries (ADD-2).
   * Checks isConnected to handle SPA teardowns (ADD-12).
   */
  function watchEventPanel(onOpen, onClose) {
    let currentPanel = null;
    let debounceTimer = null;
    const DEBOUNCE_MS = 50;

    const observer = new MutationObserver(() => {
      if (debounceTimer) clearTimeout(debounceTimer);
      debounceTimer = setTimeout(() => {
        const panel = Selectors.findEventPanel();

        if (panel && panel !== currentPanel && panel.isConnected) {
          currentPanel = panel;
          onOpen(panel);
        } else if (currentPanel && !currentPanel.isConnected) {
          currentPanel = null;
          if (onClose) onClose();
        } else if (!panel && currentPanel) {
          currentPanel = null;
          if (onClose) onClose();
        }
      }, DEBOUNCE_MS);
    });

    observer.observe(document.body, { childList: true, subtree: true });
    observers.push(observer);

    const existing = Selectors.findEventPanel();
    if (existing) {
      currentPanel = existing;
      onOpen(existing);
    }

    return () => {
      observer.disconnect();
      const idx = observers.indexOf(observer);
      if (idx > -1) observers.splice(idx, 1);
    };
  }

  function destroy() {
    for (const obs of observers) {
      obs.disconnect();
    }
    observers.length = 0;
  }

  return { waitForElement, watchEventPanel, destroy };
})();
```

### Acceptance Criteria

- [ ] `waitForElement` uses `requestAnimationFrame` debouncing (ADD-2)
- [ ] `watchEventPanel` uses `setTimeout` debouncing (ADD-2)
- [ ] `watchEventPanel` checks `panel.isConnected` (ADD-12)
- [ ] `waitForElement` resolves immediately if element already exists
- [ ] `waitForElement` rejects on timeout (default 5s)
- [ ] No `setInterval` calls exist in observer.js
- [ ] Commit: `"Phase 2: MutationObserver-based DOM watching with debouncing"`

---

## Phase 3 — Duplicate Button Injection

### Files: `src/button.js`, `styles/button.css`

### Design: `src/button.js`

The button is a simple, self-styled element. It does NOT use Google's internal jslog/jscontroller/jsaction attributes.

```javascript
"use strict";

const DuplicateButton = (() => {
  const BUTTON_CLASS = "gcqd-dup-btn";
  const BUTTON_SELECTOR = `.${BUTTON_CLASS}`;

  function create(eventId) {
    const wrapper = document.createElement("div");
    wrapper.className = BUTTON_CLASS;
    wrapper.setAttribute("data-gcqd-event-id", eventId);

    const btn = document.createElement("button");
    btn.type = "button";
    btn.className = "gcqd-dup-btn__button";
    btn.setAttribute("aria-label", "Duplicate event");
    btn.setAttribute("title", "Duplicate event");
    btn.setAttribute("tabindex", "0");

    btn.innerHTML = `
      <svg class="gcqd-dup-btn__icon" height="20" width="20" viewBox="0 0 24 24" focusable="false">
        <path d="M0 0h24v24H0V0z" fill="none"/>
        <path d="M16 1H4c-1.1 0-2 .9-2 2v14h2V3h12V1zm-1 4H8c-1.1 0-1.99.9-1.99 2L6 21c0 1.1.89 2 1.99 2H19c1.1 0 2-.9 2-2V11l-6-6zM8 21V7h6v5h5v9H8z"/>
      </svg>
    `;

    wrapper.appendChild(btn);
    return wrapper;
  }

  function inject(panel, eventId) {
    if (panel.querySelector(BUTTON_SELECTOR)) return;

    const btn = create(eventId);

    // Add image-event class if needed
    if (Selectors.hasCircleButtons(panel)) {
      btn.classList.add("gcqd-dup-btn--on-image");
    }

    const actionsContainer = Selectors.queryFirst(panel, [
      '[data-eventid] [jscontroller]',
    ]);

    if (actionsContainer && actionsContainer.parentNode) {
      actionsContainer.parentNode.insertBefore(btn, actionsContainer);
    } else {
      panel.prepend(btn);
    }

    Logger.debug("Duplicate button injected for event:", eventId);
  }

  function isButtonClick(event) {
    return event.target.closest(BUTTON_SELECTOR) != null;
  }

  function getEventIdFromClick(event) {
    const btn = event.target.closest(BUTTON_SELECTOR);
    return btn ? btn.getAttribute("data-gcqd-event-id") : null;
  }

  return { BUTTON_CLASS, BUTTON_SELECTOR, create, inject, isButtonClick, getEventIdFromClick };
})();
```

### Design: `styles/button.css`

```css
/* GCQD Duplicate Button Styles — Self-contained */

.gcqd-dup-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  position: relative;
  z-index: 1;
}

.gcqd-dup-btn__button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  padding: 0;
  margin: 0;
  border: none;
  border-radius: 50%;
  background: transparent;
  cursor: pointer;
  transition: background-color 0.2s ease;
  -webkit-appearance: none;
  appearance: none;
  outline: none;
}

.gcqd-dup-btn__button:hover { background-color: rgba(0, 0, 0, 0.08); }
.gcqd-dup-btn__button:focus-visible { outline: 2px solid #1a73e8; outline-offset: 2px; }
.gcqd-dup-btn__button:active { background-color: rgba(0, 0, 0, 0.12); }
.gcqd-dup-btn__icon { fill: #5f6368; pointer-events: none; }

/* Dark theme: prefers-color-scheme */
@media (prefers-color-scheme: dark) {
  .gcqd-dup-btn__button:hover { background-color: rgba(255, 255, 255, 0.08); }
  .gcqd-dup-btn__button:active { background-color: rgba(255, 255, 255, 0.12); }
  .gcqd-dup-btn__icon { fill: #e8eaed; }
}

/* Dark theme: GCal-specific selectors */
body[data-darkmode="true"] .gcqd-dup-btn__icon,
html.dark .gcqd-dup-btn__icon,
.yDmH0d .gcqd-dup-btn__icon { fill: #e8eaed; }
body[data-darkmode="true"] .gcqd-dup-btn__button:hover,
html.dark .gcqd-dup-btn__button:hover,
.yDmH0d .gcqd-dup-btn__button:hover { background-color: rgba(255, 255, 255, 0.08); }

/* Events with background images */
.gcqd-dup-btn--on-image .gcqd-dup-btn__button { background-color: rgba(0, 0, 0, 0.54); }
.gcqd-dup-btn--on-image .gcqd-dup-btn__icon { fill: #ffffff; }
.gcqd-dup-btn--on-image .gcqd-dup-btn__button:hover { background-color: rgba(0, 0, 0, 0.7); }

/* Hide GCal options menu flash during duplication */
body.gcqd-duplicating .JPdR6b.Q3pIde.qjTEB { display: none !important; }

/* Toast notifications (ADD-4) */
.gcqd-toast {
  position: fixed;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%) translateY(20px);
  background: #323232;
  color: #fff;
  padding: 10px 24px;
  border-radius: 8px;
  font-size: 14px;
  font-family: "Google Sans", Roboto, Arial, sans-serif;
  z-index: 99999;
  opacity: 0;
  transition: opacity 0.3s ease, transform 0.3s ease;
  pointer-events: none;
}
.gcqd-toast--visible { opacity: 1; transform: translateX(-50%) translateY(0); }
.gcqd-toast--error { background: #d93025; }
```

### Acceptance Criteria

- [ ] Button uses `gcqd-*` classes exclusively, no Google-internal attributes
- [ ] Image event detection adds `gcqd-dup-btn--on-image` class
- [ ] CSS includes dark theme, image event, and toast styles
- [ ] Commit: `"Phase 3: Self-styled duplicate button with theme support"`

---

## Phase 4 — Duplication Flow

### File: `src/duplicator.js`

```javascript
"use strict";

const Duplicator = (() => {
  let isRunning = false;

  async function duplicate(opts = {}) {
    const { autoSave = true } = opts;

    if (isRunning) {
      Logger.debug("Duplication already in progress, skipping");
      return;
    }

    isRunning = true;
    document.body.classList.add("gcqd-duplicating");

    try {
      // Step 1: Find and click the options button
      Logger.debug("Step 1: Opening options menu");
      const optionsBtn = Selectors.findOptionsButton();
      if (!optionsBtn) {
        throw new Error("Could not find options button in event panel");
      }
      Utils.simulateClick(optionsBtn);

      // Step 2: Wait for "Duplicate" menu item (ADD-10: graceful failure)
      Logger.debug("Step 2: Waiting for Duplicate menu item");
      let dupItem;
      try {
        dupItem = await Observer.waitForElement(
          () => Selectors.findDuplicateMenuItem(),
          { timeout: 3000 }
        );
      } catch (timeoutErr) {
        Logger.error("Could not find 'Duplicate' in the menu. Google may have changed the menu structure.");
        Utils.showToast("'Duplicate' not found in menu — Google may have changed it", true);
        document.body.click(); // dismiss open menu
        return;
      }

      // Step 3: Capture current view state BEFORE clicking duplicate
      Logger.debug("Step 3: Capturing view state");
      ViewGuard.capture();

      // Step 4: Click "Duplicate"
      Logger.debug("Step 4: Clicking Duplicate");
      Utils.simulateClick(dupItem);

      if (autoSave) {
        Logger.debug("Step 5: Auto-saving (waiting for save button)");
        const saveBtn = await Observer.waitForElement(
          () => Selectors.findSaveButton(),
          { timeout: 5000 }
        );
        saveBtn.click();
        Logger.debug("Step 6: Event saved, restoring view");
        await Utils.delay(300);
        ViewGuard.restore();
        Utils.showToast("Event duplicated");
      } else {
        Logger.debug("Edit mode: leaving user in edit form");
        Utils.showToast("Event duplicated — edit and save when ready");
      }
    } catch (err) {
      Logger.error("Duplication failed:", err.message);
      Utils.showToast("Duplicate failed — try the ⋮ menu instead", true);
    } finally {
      isRunning = false;
      document.body.classList.remove("gcqd-duplicating");
      Logger.debug("Duplication flow complete");
    }
  }

  return { duplicate };
})();
```

### File: `src/utils.js`

```javascript
"use strict";

const Utils = (() => {
  // Dispatch in natural browser order, no element.click() (ADD-5)
  function simulateClick(element) {
    const opts = { bubbles: true, cancelable: true, view: window };
    element.dispatchEvent(new MouseEvent("mousedown", opts));
    element.dispatchEvent(new MouseEvent("mouseup", opts));
    element.dispatchEvent(new MouseEvent("click", opts));
  }

  function delay(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  // Toast notification (ADD-4)
  function showToast(message, isError = false) {
    const existing = document.querySelector(".gcqd-toast");
    if (existing) existing.remove();

    const toast = document.createElement("div");
    toast.className = `gcqd-toast ${isError ? "gcqd-toast--error" : ""}`;
    toast.textContent = message;
    document.body.appendChild(toast);

    toast.offsetHeight; // force reflow
    toast.classList.add("gcqd-toast--visible");

    setTimeout(() => {
      toast.classList.remove("gcqd-toast--visible");
      setTimeout(() => toast.remove(), 300);
    }, 3000);
  }

  return { simulateClick, delay, showToast };
})();
```

### File: `src/constants.js`

```javascript
"use strict";

const GCQD_DEFAULTS = {
  autoSave: true,
  shortcutModifier: "alt",
};
```

### Acceptance Criteria

- [ ] `simulateClick` dispatches mousedown/mouseup/click as separate MouseEvents (ADD-5)
- [ ] `showToast` provides user feedback on success and failure (ADD-4)
- [ ] Duplication gracefully handles missing "Duplicate" menu item (ADD-10)
- [ ] `isRunning` guard prevents double-execution
- [ ] Commit: `"Phase 4: Async duplication flow with auto-save toggle"`

---

## Phase 5 — View Preservation

### File: `src/view-guard.js`

```javascript
"use strict";

const ViewGuard = (() => {
  let savedUrl = null;

  function capture() {
    savedUrl = window.location.href;
    Logger.debug("ViewGuard captured:", savedUrl);
  }

  async function restore() {
    if (!savedUrl) {
      Logger.debug("ViewGuard: nothing to restore");
      return;
    }

    const targetUrl = savedUrl;
    savedUrl = null;

    const maxWait = 5000;
    const start = Date.now();

    while (Date.now() - start < maxWait) {
      const currentUrl = window.location.href;

      if (currentUrl.includes("/eventedit") || currentUrl.includes("/duplicate")) {
        await Utils.delay(200);
        continue;
      }

      if (currentUrl !== targetUrl) {
        Logger.debug("ViewGuard restoring:", currentUrl, "→", targetUrl);
        window.location.href = targetUrl;
      } else {
        Logger.debug("ViewGuard: URL unchanged, no restore needed");
      }
      return;
    }

    Logger.warn("ViewGuard: timed out waiting for URL to settle");
  }

  return { capture, restore };
})();
```

### Acceptance Criteria

- [ ] URL-based capture/restore, no DOM manipulation
- [ ] Waits for GCal navigation to settle before restoring
- [ ] Commit: `"Phase 5: URL-based view preservation"`

---

## Phase 6 — Options Page & Storage

### File: `options/options.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <link rel="stylesheet" href="options.css">
</head>
<body>
  <div class="gcqd-options">
    <h2>Quick Duplicate Settings</h2>
    <div class="gcqd-options__group">
      <label class="gcqd-options__label">
        <input type="checkbox" id="autoSave" class="gcqd-options__checkbox">
        <span>Auto-save duplicated event</span>
      </label>
      <p class="gcqd-options__help">
        When enabled, the duplicated event is saved immediately with the same date/time.
        When disabled, the edit form opens so you can change details before saving.
      </p>
    </div>
    <div class="gcqd-options__group">
      <label class="gcqd-options__label" for="shortcutModifier">Keyboard shortcut modifier</label>
      <select id="shortcutModifier" class="gcqd-options__select">
        <option value="alt">Alt / Option</option>
        <option value="ctrl">Ctrl / Control</option>
        <option value="meta">Cmd / Windows</option>
      </select>
      <p class="gcqd-options__help">Hold this key and click an event to duplicate it instantly.</p>
    </div>
    <div class="gcqd-options__status" id="status"></div>
  </div>
  <!-- Paths from extension root, NOT relative to this file (ADD-7) -->
  <script src="src/constants.js"></script>
  <script src="options/options-page.js"></script>
</body>
</html>
```

### File: `options/options-page.js`

Reads/writes `chrome.storage.sync` with `chrome.runtime.lastError` checks (ADD-6).

### File: `options/options.css`

Standard Google-style form styling.

### Content Script Settings Loading

In `content.js`, load settings with error handling (ADD-6):

```javascript
try {
  chrome.storage.sync.get(GCQD_DEFAULTS, (stored) => {
    if (chrome.runtime.lastError) {
      Logger.warn("Settings load failed, using defaults:", chrome.runtime.lastError.message);
      settings = { ...GCQD_DEFAULTS };
    } else {
      settings = { ...GCQD_DEFAULTS, ...stored };
    }
    Logger.info("Settings loaded:", settings);
    init();
  });
} catch (err) {
  Logger.error("Storage API unavailable:", err.message);
  settings = { ...GCQD_DEFAULTS };
  init();
}
```

### Acceptance Criteria

- [ ] Script paths in options.html resolve from extension root (ADD-7)
- [ ] All storage calls check `chrome.runtime.lastError` (ADD-6)
- [ ] Falls back to defaults on storage errors
- [ ] `"permissions": ["storage"]` in manifest
- [ ] Commit: `"Phase 6: Options page with synced settings"`

---

## Phase 7 — Theming & Polish

### Tasks

1. Refine dark theme detection by inspecting current GCal DOM
2. Image event detection (already wired in Phase 3 button.js)
3. Keyboard shortcut with drag detection (ADD-1 — CRITICAL):

```javascript
// Track drag detection (ADD-1)
let mouseDownPos = null;
const DRAG_THRESHOLD = 5;

document.addEventListener("mousedown", (event) => {
  mouseDownPos = { x: event.clientX, y: event.clientY };
}, true);

document.addEventListener("click", (event) => {
  const modKey = settings.shortcutModifier;
  const isModHeld =
    (modKey === "alt" && event.altKey) ||
    (modKey === "ctrl" && event.ctrlKey) ||
    (modKey === "meta" && event.metaKey);

  if (!isModHeld) return;

  // Don't intercept drags — let native Ctrl+drag work
  if (mouseDownPos) {
    const dx = Math.abs(event.clientX - mouseDownPos.x);
    const dy = Math.abs(event.clientY - mouseDownPos.y);
    if (dx > DRAG_THRESHOLD || dy > DRAG_THRESHOLD) {
      mouseDownPos = null;
      return;
    }
  }
  mouseDownPos = null;

  const eventEl = event.target.closest("[data-eventid]");
  if (!eventEl) return;

  // Don't intercept in edit form
  const isInEditForm = event.target.closest('[data-view="EDIT"]') ||
                       event.target.closest('form') ||
                       window.location.href.includes("/eventedit");
  if (isInEditForm) return;

  event.preventDefault();
  event.stopPropagation();
  Duplicator.duplicate({ autoSave: settings.autoSave });
}, true);
```

4. Test in Day, Week, Month views
5. Verify Ctrl+drag still works natively

### Acceptance Criteria

- [ ] Native Ctrl+drag NOT broken (ADD-1)
- [ ] Dark theme icon is light-colored
- [ ] Modifier+click works for Alt/Ctrl/Meta
- [ ] Commit: `"Phase 7: Theme polish, shortcut wiring, and multi-view testing"`

---

## Phase 8 — Entry Point, Packaging & Documentation

### File: `src/content.js`

Wires everything together. Includes:
- Settings loading with error handling (ADD-6)
- Live settings updates via `chrome.storage.onChanged`
- Version stamp on init (ADD-15)
- URL change heartbeat for SPA navigation (ADD-9)
- Event panel watcher with button injection
- Duplicate button click handler
- Modifier+click shortcut with drag detection (ADD-1)

### README.md

Rewrite with features, install (including Chromebook notes per ADD-11), settings, architecture, maintenance guide.

### CHANGELOG.md

Document all changes from upstream v1.1.3.

### Acceptance Criteria

- [ ] Full end-to-end test: click event → see button → click → duplicates → view preserved
- [ ] `__GCQD_HEALTH()` works in DevTools console
- [ ] `__GCQD_DEBUG = true` enables verbose logging
- [ ] Extension loads without errors
- [ ] Commit: `"Phase 8: Entry point, README, changelog — v2.0.0"`

---

## Selector Strategy Reference

| Element | Primary Selector | Secondary | Tertiary | Last Resort |
|---|---|---|---|---|
| Calendar event | `[data-eventid]` | — | — | `.NlL62b[data-eventid]` |
| Event popup panel | `[data-eventid][role="dialog"]` | `[jsname="rDHQYd"]` | — | `.pPTZAe` |
| Options "..." button | `[aria-label="Options"]` | `[aria-label="More options"]` | `[data-tooltip="Options"]` | `.d29e1c` |
| Duplicate menu item | `[jsname="lbYRR"]` | `[data-action="duplicate"]` | Text search "Duplicate" (i18n) | — |
| Save button | `[jsname="x8hlje"]` | `[aria-label="Save"]` | `[data-tooltip="Save"]` | — |
| Mini cal current day | `[data-date][aria-selected="true"]` | `.pWJCO[data-date]` | — | — |

---

## Known Google Calendar DOM Patterns

### URL Routing
- `/calendar/r/week/YYYY/M/D` — Week view
- `/calendar/r/day/YYYY/M/D` — Day view
- `/calendar/r/month/YYYY/M/1` — Month view
- `/calendar/r/customday/YYYY/M/D` — Custom range
- `/calendar/r/agenda` — Schedule/Agenda view
- `/calendar/r/eventedit/...` — Event editing form
- URLs with `/duplicate` — duplicate form is open

### Native Ctrl+Drag (August 2025)
- Ctrl+drag (Cmd+drag Mac) duplicates events in Day/Week/Month views
- Does NOT work in Schedule or Year view
- Requires dragging — stationary Ctrl+click does nothing natively
- Our extension MUST NOT break this

### Dark Mode
- `prefers-color-scheme: dark` (system-level)
- Class or data attribute on html/body (varies; verify at build time)
- Some builds use `[data-darkmode="true"]`

### Events with Images
- Keywords like "Gym", "Lunch", "Coffee" trigger auto-generated backgrounds
- Use circular icon buttons with dark backgrounds for visibility

---

## Execution Notes for Claude Code

1. **Work phase-by-phase.** Commit after each phase passes its acceptance criteria.

2. **Addendum items are already integrated into the phase specs above.** All 15 ADD-* items have been folded into the relevant phases. Follow the specs as written.

3. **Test by loading unpacked.** After each phase, verify via `chrome://extensions` → Load unpacked.

4. **No build system.** Vanilla JS, no TypeScript, no bundler. Loaded directly by the manifest.

5. **Selector verification.** During Phase 7, inspect Google Calendar DOM to verify selectors. Update `FALLBACK_CLASSES`.

6. **Script paths in extension HTML** resolve from the extension root, not relative to the HTML file.

7. **`run_at: "document_idle"`** — no DOMContentLoaded listeners needed.

8. **Git workflow:** Work on `master` branch.

9. **Always check `chrome.runtime.lastError`** after storage calls.

10. **Content script load order:** `logger.js` first, `content.js` last.

11. **Native Ctrl+drag coexistence is critical.** Use drag distance detection.

12. **Quick debugging:** `__GCQD_HEALTH()` for diagnostics, `__GCQD_DEBUG = true` for verbose logging.
