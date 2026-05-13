---
name: visual-exploration
description: Layer 1 skill for browser automation and visual UX documentation. Initial reconnaissance, navigation mapping, interaction documentation, end-to-end flow capture, edge case exploration, screenshot methodology. Loaded by the analyzer agent during Layer 1.
---

# Visual Exploration Methodology

Document behavioral intelligence through browser automation and visual observation. Every screenshot is an empirical observation. Every interaction sequence is a behavioral flow. This mode captures what the user SEES and what the system DOES in response to user actions.

## When to Use This Mode

Visual exploration activates when:
- The target is a web application with a UI
- The discovery inventory identifies a running web UI accessible via HTTP
- A running instance of the target is accessible via HTTP

This mode requires a running instance of the target application and browser automation capabilities (Playwright, Puppeteer, or equivalent). All output is **RAW** (screenshots and flow documentation capture the target's UI in detail).

Targets without a web UI get no useful signal from this mode — skip it.

## Why Visual Exploration Matters

Source code tells you what the system CAN do. Tests tell you what the system MUST do. Visual exploration tells you what the system LOOKS LIKE while doing it. This is behavioral intelligence that no other mode captures:

- **Affordances** -- what actions does the UI present as available?
- **Feedback** -- how does the system communicate success, failure, and state changes?
- **Navigation structure** -- how are features organized from the user's perspective?
- **State representation** -- how does the UI change as the system's state changes?
- **Accessibility** -- is the interface usable with keyboard, screen reader, or at different sizes?

## Phase 1: Initial Reconnaissance

**Goal:** Navigate to the root URL, capture the landing page, and identify the application type, auth requirements, and available affordances.

### 1.1 Landing Page Capture

```bash
# Navigate to root URL and screenshot
# (Using Playwright as reference; adapt to available browser automation)
```

```javascript
const page = await browser.newPage();
await page.goto(ROOT_URL, { waitUntil: 'networkidle' });
await page.screenshot({ path: 'workspace/raw/runtime/visual/screenshots/001-landing.png', fullPage: true });
```

### 1.2 Initial Assessment

Document:
- **Application type** -- SPA, MPA, dashboard, wizard, documentation site, etc.
- **Authentication** -- does the landing page require login? Is there a signup flow?
- **Primary navigation** -- top nav, sidebar, hamburger menu, tabs, breadcrumbs?
- **Visible affordances** -- buttons, links, forms, search bars, dropdowns visible on the landing page
- **Branding and layout** -- header, footer, sidebar, content area dimensions

### 1.3 Auth Handling

If authentication is required:
1. Screenshot the login page
2. Document the auth method (username/password, OAuth, SSO, API key)
3. If test credentials are available, authenticate and screenshot the post-login state
4. If no credentials are available, document the gate and explore only the unauthenticated surface

## Phase 2: Navigation Mapping

**Goal:** Click every navigation element, screenshot each page, and document the complete navigation tree.

### 2.1 Identify Navigation Elements

```javascript
// Extract all navigation links
const navLinks = await page.$$eval('nav a, [role="navigation"] a, .sidebar a, .menu a, header a', 
  links => links.map(a => ({ text: a.textContent.trim(), href: a.href }))
);
```

### 2.2 Systematic Traversal

For each navigation element:
1. Click the element (or navigate to the href)
2. Wait for the page to settle (network idle or DOM stable)
3. Take a full-page screenshot
4. Record the URL, page title, and visible heading
5. Identify sub-navigation elements on the new page

```javascript
for (const [index, link] of navLinks.entries()) {
  await page.goto(link.href, { waitUntil: 'networkidle' });
  await page.screenshot({ 
    path: `workspace/raw/runtime/visual/screenshots/nav-${String(index).padStart(3, '0')}-${slugify(link.text)}.png`,
    fullPage: true 
  });
}
```

### 2.3 Navigation Tree Output

Write to `workspace/raw/runtime/visual/navigation-map.md`:

```markdown
## Navigation Tree

- **Home** (`/`) -- screenshot: 001-landing.png
  - **Dashboard** (`/dashboard`) -- screenshot: nav-001-dashboard.png
    - **Analytics** (`/dashboard/analytics`) -- screenshot: nav-002-analytics.png
    - **Reports** (`/dashboard/reports`) -- screenshot: nav-003-reports.png
  - **Settings** (`/settings`) -- screenshot: nav-004-settings.png
    - **Profile** (`/settings/profile`) -- screenshot: nav-005-profile.png
    - **Billing** (`/settings/billing`) -- screenshot: nav-006-billing.png
```

## Phase 3: Interaction Documentation

**Goal:** For each page, identify interactive elements, interact with them, and capture before/after screenshots documenting the system's feedback.

### 3.1 Element Inventory

For each page, identify:
- **Forms** -- text inputs, selects, checkboxes, radio buttons, textareas, file uploads
- **Buttons** -- submit buttons, action buttons, toggle buttons
- **Dropdowns** -- select menus, custom dropdown components
- **Modals/dialogs** -- elements that trigger overlay content
- **Expandable sections** -- accordions, collapsible panels, "show more" links
- **Interactive widgets** -- date pickers, sliders, color pickers, drag-and-drop zones

```javascript
const interactiveElements = await page.$$eval(
  'button, input, select, textarea, [role="button"], [role="tab"], [onclick], a[href="#"]',
  els => els.map(el => ({
    tag: el.tagName,
    type: el.type || '',
    text: el.textContent?.trim().substring(0, 50),
    id: el.id,
    name: el.name,
    ariaLabel: el.getAttribute('aria-label'),
  }))
);
```

### 3.2 Interaction Capture

For each interactive element:

1. **Before screenshot** -- capture the page state before interaction
2. **Interact** -- click, type, select, toggle
3. **After screenshot** -- capture the page state after interaction
4. **Document feedback** -- what changed? Error message? Success toast? New content?

```javascript
// Example: form submission
await page.screenshot({ path: `screenshots/form-${name}-before.png` });
await page.fill('input[name="email"]', 'test@example.com');
await page.click('button[type="submit"]');
await page.waitForTimeout(1000); // Allow time for feedback
await page.screenshot({ path: `screenshots/form-${name}-after.png` });
```

### 3.3 Page Inventory Output

Write to `workspace/raw/runtime/visual/page-inventory.md`:

```markdown
## Page: Settings > Profile

**URL:** `/settings/profile`
**Screenshot:** nav-005-profile.png

### Interactive Elements

| Element | Type | Action Taken | Feedback | Screenshots |
|---------|------|-------------|----------|-------------|
| Display Name | text input | Typed "Test User" | No immediate feedback | form-profile-before.png, form-profile-after.png |
| Save | submit button | Clicked | Success toast: "Profile updated" | form-profile-save.png |
| Avatar | file upload | Uploaded test.png | Preview updated | form-profile-avatar.png |
```

## Phase 4: Flow Documentation

**Goal:** Document end-to-end user flows with screenshots at every state transition.

### 4.1 Core Flows to Document

Document these flows whenever they exist:

| Flow | Description | Priority |
|------|-------------|----------|
| **Onboarding** | First-time user experience from signup to first use | High |
| **Core workflow** | The primary task the application enables | High |
| **Error recovery** | What happens when something goes wrong | High |
| **Settings/configuration** | How users customize the application | Medium |
| **Search/filter** | How users find content within the application | Medium |
| **CRUD operations** | Create, read, update, delete for primary entities | Medium |
| **Authentication flow** | Login, logout, password reset, session expiry | Medium |
| **Export/import** | How data moves in and out of the application | Lower |

### 4.2 Flow Documentation Format

For each flow, create a directory under `workspace/raw/runtime/visual/flows/`:

```
workspace/raw/runtime/visual/flows/
    onboarding/
        step-01-landing.png
        step-02-signup-form.png
        step-03-email-verification.png
        step-04-profile-setup.png
        step-05-first-dashboard.png
        flow.md
    core-workflow/
        step-01-start.png
        step-02-input.png
        step-03-processing.png
        step-04-result.png
        flow.md
```

Each `flow.md` follows this format:

```markdown
# Flow: Onboarding

## Steps

### Step 1: Landing Page
**Screenshot:** step-01-landing.png
**URL:** `/`
**Action:** Click "Sign Up" button
**State:** Unauthenticated, no account

### Step 2: Signup Form
**Screenshot:** step-02-signup-form.png
**URL:** `/signup`
**Action:** Fill email, password, click "Create Account"
**State:** Unauthenticated, form displayed
**Feedback:** Form validation on blur for email format

### Step 3: Email Verification
**Screenshot:** step-03-email-verification.png
**URL:** `/verify`
**Action:** (requires email access -- documented but not tested)
**State:** Account created, not verified
**Feedback:** "Check your email" message displayed

## Behavioral Claims

- New accounts require email verification before first use
  <!-- cite: source=visual-observation, ref=flows/onboarding/step-03-email-verification.png, confidence=confirmed, agent=visual-explorer -->
```

## Phase 5: Edge Cases

**Goal:** Explore edge cases that reveal the system's behavior at its boundaries.

### 5.1 Empty States

```javascript
// Navigate to pages likely to have empty states (lists, dashboards, search results)
// Screenshot each empty state
await page.goto('/projects'); // Assuming no projects exist
await page.screenshot({ path: 'screenshots/empty-projects.png', fullPage: true });
```

Document: What does the system show when there is no data? Does it provide guidance? Is the UI broken?

### 5.2 Long Content

```javascript
// Submit forms with very long text
await page.fill('input[name="title"]', 'A'.repeat(1000));
await page.screenshot({ path: 'screenshots/long-input.png' });
```

Document: Does the system truncate? Overflow? Reject? Scroll?

### 5.3 Responsive Layout

```javascript
// Test at different viewport sizes
const viewports = [
  { width: 375, height: 812, name: 'mobile' },     // iPhone
  { width: 768, height: 1024, name: 'tablet' },     // iPad
  { width: 1280, height: 800, name: 'laptop' },     // Laptop
  { width: 1920, height: 1080, name: 'desktop' },   // Desktop
];

for (const vp of viewports) {
  await page.setViewportSize({ width: vp.width, height: vp.height });
  await page.screenshot({ path: `screenshots/responsive-${vp.name}.png`, fullPage: true });
}
```

Document: Does the layout adapt? Is navigation accessible at mobile sizes? Are interactive elements reachable?

### 5.4 Accessibility

```javascript
// Check for keyboard navigation
await page.keyboard.press('Tab');
await page.screenshot({ path: 'screenshots/keyboard-focus-1.png' });
await page.keyboard.press('Tab');
await page.screenshot({ path: 'screenshots/keyboard-focus-2.png' });

// Check for skip links
await page.keyboard.press('Tab'); // First tab should focus skip-to-content if present
```

Document: Is keyboard navigation possible? Are focus indicators visible? Is there a skip-to-content link?

## Screenshot Naming Convention

All screenshots follow this naming scheme:

```
{phase}-{context}-{description}.png
```

| Phase Prefix | Usage |
|-------------|-------|
| `001-` through `099-` | Phase 1: Initial reconnaissance |
| `nav-NNN-` | Phase 2: Navigation mapping |
| `form-{page}-` | Phase 3: Interaction documentation |
| `step-NN-` | Phase 4: Flow documentation (within flow subdirectory) |
| `edge-{type}-` | Phase 5: Edge case exploration |

### Directory Structure

```
workspace/raw/runtime/visual/
    screenshots/                    # All screenshots (flat, except flows)
        001-landing.png
        nav-001-dashboard.png
        nav-002-analytics.png
        form-profile-before.png
        form-profile-after.png
        edge-empty-projects.png
        edge-responsive-mobile.png
    flows/                          # End-to-end flow documentation
        onboarding/
            step-01-landing.png
            step-02-signup.png
            flow.md
        core-workflow/
            step-01-start.png
            step-02-result.png
            flow.md
    navigation-map.md               # Complete navigation tree with screenshot refs
    page-inventory.md               # Per-page interactive element inventory
    visual-behaviors.md             # Behavioral claims synthesized from all observations
```

## Provenance Rules

### Source Type

All claims from visual exploration use `source=visual-observation`:

```markdown
- The dashboard displays a "No projects" message with a "Create Project" call-to-action when the project list is empty
  <!-- cite: source=visual-observation, ref=screenshots/edge-empty-projects.png, confidence=confirmed, agent=visual-explorer -->
```

### Confidence Levels

- **confirmed** -- the behavior is directly visible in a screenshot or demonstrated by a before/after screenshot pair
- **inferred** -- the behavior is implied by UI elements but not directly observed (e.g., a "Delete" button exists but was not clicked)
- **assumed** -- the behavior is guessed from visual cues (e.g., a grayed-out button is assumed to be disabled)

### Cite As You Go

Every behavioral claim gets an inline citation immediately after the claim. The `ref` field should be the screenshot path.

## Rules

1. **Screenshots are RAW** -- visual captures of proprietary UI go to `workspace/raw/`. They never reach `workspace/output/` or `workspace/public/`.
2. **Screenshot everything** -- when in doubt, take a screenshot. Storage is cheap. Missing a state transition screenshot cannot be recovered without re-running the exploration.
3. **Before AND after** -- every interaction must have a before screenshot and an after screenshot. The behavioral claim is the DIFFERENCE between the two.
4. **Wait for stability** -- after every action, wait for the page to settle before screenshotting. Use `networkidle` or explicit waits for dynamic content.
5. **Do not invent interactions** -- only interact with elements that are visible and actionable. If a button is disabled, document that it is disabled. Do not force-enable it.
6. **Auth gates are data** -- if a page requires authentication you do not have, that gate is a behavioral claim worth documenting.
7. **Cite as you go** -- every behavioral claim gets an inline `<!-- cite: -->` comment immediately after the claim. Never defer citation to a later step.
8. **Full page screenshots** -- use `fullPage: true` to capture below-the-fold content. Viewport-only screenshots miss content.
