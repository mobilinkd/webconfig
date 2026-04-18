# Mobilinkd Brand Theme System

> This document defines the Mobilinkd design system for consistent cross-platform branding.
> Applies to: iOS apps, Android apps, web apps, mobilinkd.com, and the Mobilinkd store.
> Last updated: 2026-04-18

---

## 1. Brand Foundation

### 1.1 Logo Colors

The Mobilinkd logo uses a warm amber/gold on a dark background. Two variants exist:

| Variant | Background | Foreground |
|---------|-----------|------------|
| Dark (default) | `#0f1117` (near-black) | `#F7BC60` (warm amber) |
| Light | `#ffffff` | `#C48A30` (deep amber) |

The logo **must never** be recolored — use the official SVG asset per variant.

---

## 2. Color Palettes

### 2.1 Dark Theme

```
--bg:           #0B0D13   /* Page background — deep navy-black */
--surface:      #131620   /* Cards, sidebar — slightly lighter */
--surface2:     #1A1E2C   /* Inputs, inner panels */
--border:       #252A3A   /* Dividers, outlines */
--text:         #D8DCE8   /* Primary text — cool off-white */
--text-muted:   #7A8099   /* Labels, secondary text */
--accent:       #F7BC60   /* Amber — brand color, used sparingly for logo/branding only */
--accent-btn:   #3A7CCC   /* Action buttons — steel blue */
--accent-btn-h: #2D66B0   /* Button hover */
--ok:           #3DBE7A   /* Connected, success, battery good */
--warn:         #D4A83A   /* Warnings, battery medium */
--crit:         #D45050   /* Errors, battery low, disconnected */
--rx-active:    #3DBE7A   /* Receive audio active indicator */
--tx-active:    #3A7CCC   /* Transmit audio active indicator */
```

### 2.2 Light Theme

```
--bg:           #F4F5F7   /* Page background — cool gray-white */
--surface:      #FFFFFF   /* Cards, sidebar */
--surface2:     #ECEEF2   /* Inputs, inner panels */
--border:       #D0D4DE   /* Dividers, outlines */
--text:         #1A1D27   /* Primary text — deep navy */
--text-muted:   #6B7086   /* Labels, secondary text */
--accent:       #C48A30   /* Deep amber — logo on light bg */
--accent-btn:   #2A62B0   /* Action buttons — deeper blue for contrast */
--accent-btn-h: #1E4E8A   /* Button hover */
--ok:           #1E8A47   /* Connected, success, battery good */
--warn:         #A07820   /* Warnings, battery medium */
--crit:         #B83030   /* Errors, battery low, disconnected */
--rx-active:    #1E8A47   /* Receive audio active */
--tx-active:    #2A62B0   /* Transmit audio active */
```

### 2.3 Status Color Semantics

| Status | Dark Theme | Light Theme | Usage |
|--------|-----------|-------------|-------|
| Connected / OK | `#3DBE7A` | `#1E8A47` | BLE connected, battery good, saved |
| Warning | `#D4A83A` | `#A07820` | Battery medium, unsaved changes |
| Error / Critical | `#D45050` | `#B83030` | Disconnected, battery low, errors |
| Active TX | `#3A7CCC` | `#2A62B0` | PTT active, transmit mode |
| Active RX | `#3DBE7A` | `#1E8A47` | Receive audio streaming |

### 2.4 Amber Accent Usage Rule

> **Amber (`#F7BC60` / `#C48A30`) is reserved for the Mobilinkd logo and app title only.**

Interactive elements (buttons, toggles, links, active states) use `--accent-btn` (steel blue) in both themes. This prevents the amber from clashing with standard UI affordances and maintains high contrast for accessibility.

---

## 3. Typography

| Element | Font | Weight | Size | Line-height |
|---------|------|--------|------|-------------|
| App title | System UI stack | 600 | 1.1rem | 1.2 |
| Screen heading | System UI stack | 600 | 1.15rem | 1.3 |
| Section label | System UI stack | 600 | 0.78rem | 1.0 |
| Body text | System UI stack | 400 | 0.85rem | 1.5 |
| Form label | System UI stack | 400 | 0.78rem | 1.4 |
| Value / mono | `Courier New`, monospace | 400 | 0.88rem | 1.4 |
| Caption | System UI stack | 400 | 0.72rem | 1.4 |

**Font stack (all platforms):**
```css
font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
```

---

## 4. Spacing & Layout

| Token | Value | Usage |
|-------|-------|-------|
| `--space-xs` | 4px | Icon-to-text gaps |
| `--space-sm` | 8px | Inner input padding, dense lists |
| `--space-md` | 14px | Section padding, card padding |
| `--space-lg` | 20px | Between cards |
| `--space-xl` | 28px | Screen header margin |

| Element | Radius |
|---------|--------|
| Buttons | 6px |
| Cards | 10px |
| Inputs | 6px |
| Toggles | 11px (pill) |
| Badges | 10px (pill) |

---

## 5. Motion

- Transition duration: **150ms** for hover/focus states
- Transition duration: **250ms** for theme switching (color transitions)
- Easing: `ease-out` for entrances, `ease-in` for exits
- Battery bar fill: **400ms** ease transition
- No decorative or ambient animations

---

## 6. Icons & Emoji

Emoji may be used for screen/section icons in the web app (no icon font dependency):

| Screen | Emoji | Notes |
|--------|-------|-------|
| TNC Information | 🖥 | — |
| KISS Parameters | 📡 | — |
| Modem Configuration | 📶 | — |
| Receive Audio | 🎧 | — |
| Transmit Audio | 📤 | — |
| Power Settings | 🔋 | — |
| PTT buttons | — | Text labels only: Off / Mark / Space / Both |

---

## 7. Platform-Specific Notes

### iOS (SwiftUI)
```swift
// Use Color extension for brand colors
extension Color {
    static let mobilinkdBackground = Color(hex: "0B0D13")
    static let mobilinkdSurface = Color(hex: "131620")
    static let mobilinkdAccentBtn = Color(hex: "3A7CCC")
    static let mobilinkdOk = Color(hex: "3DBE7A")
    static let mobilinkdWarn = Color(hex: "D4A83A")
    static let mobilinkdCrit = Color(hex: "D45050")
}
```

### Android (XML / Compose)
```xml
<!-- colors.xml -->
<color name="mobilinkd_background">#0B0D13</color>
<color name="mobilinkd_surface">#131620</color>
<color name="mobilinkd_accent_btn">#3A7CCC</color>
<color name="mobilinkd_ok">#3DBE7A</color>
<color name="mobilinkd_warn">#D4A83A</color>
<color name="mobilinkd_crit">#D45050</color>
```

### CSS Custom Properties (Web)
```css
:root {
  --bg:           #0B0D13;
  --surface:      #131620;
  --surface2:     #1A1E2C;
  --border:       #252A3A;
  --text:         #D8DCE8;
  --text-muted:   #7A8099;
  --accent:       #F7BC60;
  --accent-btn:   #3A7CCC;
  --accent-btn-h: #2D66B0;
  --ok:           #3DBE7A;
  --warn:         #D4A83A;
  --crit:         #D45050;
  --rx-active:    #3DBE7A;
  --tx-active:    #3A7CCC;
}
```

### Light Theme (all platforms)
```css
:root[data-theme="light"] {
  --bg:           #F4F5F7;
  --surface:      #FFFFFF;
  --surface2:     #ECEEF2;
  --border:       #D0D4DE;
  --text:         #1A1D27;
  --text-muted:   #6B7086;
  --accent:       #C48A30;
  --accent-btn:   #2A62B0;
  --accent-btn-h: #1E4E8A;
  --ok:           #1E8A47;
  --warn:         #A07820;
  --crit:         #B83030;
  --rx-active:    #1E8A47;
  --tx-active:    #2A62B0;
}
```

---

## 8. Accessibility

- All interactive elements must have a minimum 4.5:1 contrast ratio (WCAG AA)
- The `--ok` / `--warn` / `--crit` status colors meet contrast requirements on both themes
- Do not rely on color alone to convey state — always pair with text or icon
- Focus indicators: 2px solid `--accent-btn` offset 2px

---

## 9. What to Avoid

❌ Do not use `--accent` (amber) for buttons, toggles, or active states  
❌ Do not use amber for large background fills  
❌ Do not introduce additional brand colors outside this system  
❌ Do not use gradients on large surfaces  
❌ Do not animate purely for decoration  
✅ Use amber only for: logo, app title, small branding marks
