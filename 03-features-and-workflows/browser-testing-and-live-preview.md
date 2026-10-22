# Headful Browser Testing & Live Staging Preview

## Overview

When coding agents modify frontend code, they must verify that their changes work in a real browser environment without breaking user layouts, throwing JavaScript exceptions, or misaligning UI components.

On the Micro-VM tier, the agent has access to a dedicated headful Chromium browser running inside the XFCE desktop environment, coupled with instantaneous live staging URLs.

---

## 1. How the Agent Performs Browser Verification

1. **Headful Browser Control:** Inside the XFCE desktop, the agent launches Chromium with a dedicated user data profile and remote debugging enabled:
   ```bash
   chromium --no-first-run --remote-debugging-port=9222 http://localhost:3000
   ```
2. **Action-Observation Loop:**
   - The agent takes in-memory screenshots to inspect rendered pages and CSS styling.
   - It identifies coordinates and interactive elements.
   - It issues clicks, typing events, and scrolls through `/dev/uinput`.
   - It checks the browser runtime console (`window.errors`) to guarantee zero unhandled exceptions.

---

## 2. Instant Live Staging URLs

Instead of manual local checkouts, every agent run exposes an automated live preview URL:
* `https://<workspace-id>-3000.preview.yourdomain.dev`
* Enables instantaneous verification by human reviewers, designers, and teammates.

---

## 3. Pull Request Summary Format

When submitting a Pull Request, the agent provides a clean, executive summary with direct links to the live preview and test results:

```markdown
## 🤖 Agent Summary: Fixed Checkout Modal Responsiveness

### Changes Made:
- Refactored `CheckoutModal.tsx` to handle mobile viewport bounds (<480px).
- Fixed Tailwind class collision causing button clipping on mobile screens.
- Added unit tests for cart state persistence.

### Verification:
> **Status:** ✅ Unit Tests Passed (14/14) | ✅ Zero Browser Console Errors  
> **🚀 Live Preview:** [Open Live Staging](https://box-9912-3000.preview.yourdomain.dev)
```
