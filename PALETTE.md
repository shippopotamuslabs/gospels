# SHIPPOPOTAMUS COLOR PALETTE GOSPEL — OFFICIAL DARK THEME

Declared canonical by Sean, 2026-10-19. Status: CANONICAL FOR ALL APPS (Bloviate, Stratum, Enclave, and all future Shippopotamus products). Where any mockup/image disagrees with this text, THIS TEXT WINS.

## Why (empirical basis, Sean's decree)
1. No pure black/white halation: #000 on #FFF is 21:1 and glows for ~50% of users with astigmatism. Base #0D0E11 with text #EDEEF0 = 15.8:1, the Material/Google sweet spot.
2. Elevation via lightness, not shadows (shadows vanish on dark). Each surface 4–8% lighter than the one below. The Linear/Raycast/Vercel convergence.
3. Accents desaturated 20–30% (saturation 40–60% max) so they don't vibrate on dark. Indigo: most universally preferred, least halation-prone.
4. Warm off-white (#EDEEF0, 87% white) over blue-white for eye fatigue. Secondary text stays WCAG AA 4.5:1+.

## Tokens

### Background layers
- bg-base (app canvas, deepest): #0D0E11
- bg-subtle (navbar, tab bar): #111216
- surface-1 (cards, sheets): #181A20
- surface-2 (elevated/pressed): #1F222A
- surface-3 (hover, modal, popover): #272A33

### Borders / dividers
- border-default: #2A2E39
- border-strong: #343A48
- border-subtle: #20222E

### Text layers
- text-primary (87% white, 15.8:1): #EDEEF0
- text-high (secondary headings): #C5C7DD
- text-secondary (body secondary): #A1A5B0
- text-tertiary (captions, metadata): #858994
- text-faint (icons, disabled, placeholder): #6E7280

### Indigo accent family (primary action)
- accent-base: #7A7CFF
- accent-hover: #8D8FFF
- accent-active: #5F62B9
- accent-border-subtle: #4B4EBF
- accent-deep: #3B3BAA
- accent-muted-bg: rgba(122,124,255,0.15)
- focus-ring: rgba(122,124,255,0.40)

### Semantic colors (all desaturated ≤60%)
- success-muted-green: #7DBE8C
- warning-amber: #D9B07A
- danger-soft-red: #E07A7A
- info-sky-blue: #7BB5D6

### Mobile extras
- bg-scrim: rgba(13,14,17,0.80)
- surface-overlay: #1F222AF2
