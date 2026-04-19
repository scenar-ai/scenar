# Five New Shell Views for @scenar/react

**Date**: April 19, 2026

## Summary

Added five new shell view components to `@scenar/react` — MobileView, ChatView, SlideView, DashboardView, and APIClientView — each built to the same screenshot-level fidelity as the existing BrowserView (Chrome), TerminalView (iTerm2), and CodeEditorView (VS Code). This brings the total shell library from 3 to 8 components, covering the most common application chrome patterns that scenario authors need when building product demos.

## Problem Statement

Scenar's three original shells covered developer tools (terminal, code editor) and web applications (browser). However, many common demo scenarios required authors to build custom chrome from scratch: mobile app walkthroughs needed phone frames, AI assistant demos needed chat interfaces, product presentations needed slide decks, ops dashboards needed monitoring frames, and API integration demos needed REST client interfaces.

### Pain Points

- Scenario authors repeatedly built one-off device frames for mobile demos, each with slightly different proportions and fidelity
- Chat-based demos (AI agents, support bots) had no reusable bubble layout or typing indicator
- Presentation/onboarding walkthrough demos required custom slide frames
- Monitoring/analytics demos needed dashboard chrome (sidebar, breadcrumbs, time range)
- API integration demos required Postman-style request/response layouts with status badges

## Solution

Five new shell components, each referencing a specific real-world application for visual accuracy, following the exact conventions established by the existing three shells.

## Implementation Details

### MobileView — iPhone 15 Pro (iOS 17+)

Titanium-tinted bezel (`#2a2a2e`) with inner shadow, Dynamic Island pill centered at top, iOS status bar (time, Signal/Wifi/Battery icons), edge-to-edge content area, and home indicator bar. Aspect ratio locked to iPhone 15 Pro proportions (~393×852pt). Height driven by new `MOBILE_SHELL_HEIGHT_DEFAULT` token (500px).

Deliberate constraint: iPhone-only. No device toggle — Android's fundamentally different chrome would require its own dedicated `AndroidMobileView` to look convincing.

### ChatView — iMessage/WhatsApp

Header bar with avatar (gradient circle with online dot), title/subtitle, action icons (phone, video, ellipsis). Dark message scroll area. Bottom input bar with attachment button, text field, and blue send circle. Ships two composable sub-components:

- **ChatBubble** — `role: "user" | "assistant"` controls alignment (right/left) and colour (blue `#0b93f6` / dark gray `#2a2a2e`) with asymmetric rounded corners
- **TypingIndicator** — Three dots with staggered Framer Motion opacity pulse (1.2s cycle, 0.3s offset per dot)

Children-based composition (not a data array) so authors can mix bubbles with arbitrary content like code blocks, tool-call cards, or images.

### SlideView — Google Slides

Dark toolbar with traffic lights, presentation title, blue "Present" pill button, and share icon. Centered 16:9 slide canvas on dark stage surround with border and drop shadow. Slide counter ("Slide N of M"). Optional speaker-notes panel with drag handle and monospace text. Bottom toolbar with zoom percentage and layout icons. Height driven by new `SLIDE_SHELL_HEIGHT_DEFAULT` token (460px).

### DashboardView — Grafana (dark mode)

Top nav (`#181b1f`) with orange gradient logo mark, breadcrumb trail with `>` separators, time-range pill ("Last 6 hours"), refresh icon, and user avatar. Optional sidebar (`#111217`) with icon-based nav items and blue accent bar (`#3274d9`) for active state. Dark content area for dashboard panels. Thin status bar with "Ready" text and green connected indicator.

Default sidebar items (Search, Dashboards, Alerting, Connections) render automatically when no custom items are provided.

### APIClientView — Postman (dark mode)

Title bar with traffic lights and request tab. Signature Postman request bar: color-coded method badge (GET=`#49cc90`, POST=`#fca130`, PUT=`#61affe`, PATCH=`#e8a838`, DELETE=`#f93e3e`), URL input, blue Send button. Tabbed request pane (Params/Headers/Body/Auth) with optional code body. Response section with tab bar (Body/Headers/Cookies), color-coded status badge (2xx green, 3xx yellow, 4xx+ red) with auto-derived reason text from a comprehensive status code map, response time, and response size. Both code panels use line-numbered monospace style matching CodeEditorView.

### Supporting changes

| File | Change |
|------|--------|
| `tokens.ts` | Added `MOBILE_SHELL_HEIGHT_DEFAULT` (500) and `SLIDE_SHELL_HEIGHT_DEFAULT` (460) |
| `shells/index.ts` | Barrel exports for all 5 new shells + sub-components + new tokens |
| `src/index.ts` | Re-exports for the full public API surface |

### All shells follow established conventions

- `"use client"` directive
- Own exported props interface
- `contentKey: string` for Framer Motion remount animation
- `slideDirection?: "forward" | "backward"` for horizontal slide transition
- Height via `var(--scenar-shell-height, <fallback>)` with tokens
- Tailwind utilities for layout, hard-coded hex for chrome fidelity
- `lucide-react` for all icons

### Test coverage

5 new test files (49 total tests across all 8 shell test files):

| Test file | Tests | Key assertions |
|-----------|-------|----------------|
| MobileView | 5 | Default time "9:41", custom time, carrier label, zoom prop |
| ChatView | 9 | Title, subtitle, placeholder, bubble alignment by role, timestamps, typing indicator dots |
| SlideView | 7 | Title, slide counter, Present button, speaker notes conditional render |
| DashboardView | 8 | Time range, breadcrumbs, default sidebar items, status text, children |
| APIClientView | 9 | URL, method badge colours, request body, status badges (200/404), response time/size |

## Benefits

- **8 shells, not 3**: Scenario authors can now cover terminal, code editor, browser, mobile app, chat/AI, presentation, dashboard, and API client demos without building custom chrome
- **Realistic fidelity**: Each shell references a specific real product (iPhone 15 Pro, iMessage, Google Slides, Grafana, Postman) with accurate hex colours and chrome anatomy
- **Composable ChatView**: Children-based API with `ChatBubble` and `TypingIndicator` sub-components lets authors mix any content inside the chat frame
- **Consistent conventions**: All new shells follow the same `contentKey`/`slideDirection`/`--scenar-shell-height` contract, drop-in compatible with `DemoViewport` and `ScenarioPlayer`
- **Full test coverage**: 49 tests across all shells, TypeScript compiles cleanly

## Impact

- `@scenar/react` public API gains 5 components, 3 sub-components/types, 2 height tokens (all additive)
- No breaking changes to existing shells or APIs
- Enables new categories of Scenar demos that were previously impractical without custom chrome

## Related Work

- **Shell Chrome Realism & Universal CSS** (April 18): Established the fidelity standard and CSS architecture these new shells build on
- **Chrome Shell Primitives** (April 17): Original extraction of BrowserView, TerminalView, CodeEditorView into `@scenar/react`

---

**Status**: ✅ Production Ready
**Timeline**: Single session
