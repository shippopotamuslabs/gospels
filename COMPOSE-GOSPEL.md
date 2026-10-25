# SHIPPOPOTAMUS ANDROID JETPACK COMPOSE GOSPEL — OFFICIAL UI SYSTEM

Declared canonical by Sean, 2026-10-19. Material 3 + M3 Expressive (Android 16 QPR1). Canonical for BLOVIATE / STRATUM / ENCLAVE. Companion: PALETTE.md (colors). Button shape is governed by BUTTON-SHAPE-GOSPEL.md (canonical 2026-10-20), which supersedes this document on button shape and button press morph. Bars, sheets, and overlay glass are governed by GLASS-BARS-GOSPEL.md (canonical 2026-10-20, Haze 2.0), which supersedes this document's solid nav-bar container and opaque-sheet treatment for those surfaces.

## Why
1. 8dp grid + 4dp micro: consistent spacing per Material; everything aligns, no random 5dp/7dp.
2. 48dp touch targets are law: Modifier.minimumInteractiveComponentSize() on custom interactives; hit area auto-expands to 48dp even when the visual is smaller.
3. Elevation is tint, not shadow, in dark: surface tint washes; raised = lighter than base in dark mode.
4. Shape system = hierarchy: nested radius = outer radius − padding (28dp card + 12dp padding → 16dp inner). Buttons capsule; cards fixed radii. Don't invent radii.
5. M3 Expressive is the baseline: morphable shapes, spring physics default (MotionScheme.expressive()), emphasized typography.

## Foundation

### Spacing & layout
- Base unit 4dp; major grid 8dp (margins, section gaps, card padding multiples of 8); micro 4dp (icon-to-text, chip gaps).
- Screen margin 16dp mobile / 24dp tablet. Content max-width 600dp centered on tablet.
- Vertical rhythm: 8 / 16 / 24 / 32 / 48 / 64. NEVER 5, 7, 10, 15.
- Spacing object: s4, s8, s12, s16, s24, s32, s48, s64 — only these values.
- Stacked button pairs: 8dp (s8) between stacked full-width buttons (CTA stacks, bottom-bar action pairs). The 4dp micro value is only for inside a component (icon-to-text, chip gaps), never between two controls; section rhythm (16dp) separates sections, not buttons in a pair.

### Shapes (corner radius)
- extraSmall 4dp (text field top, chips, small containers), small 8dp (default cards/menus), medium 12dp (cards, dialogs, sheets), large 16dp (large cards, FAB), extraLarge 28dp (bottom sheets, large dialogs, hero), full = pill (buttons, FABs, toggle track, active nav indicator).
- Nested rule: child radius = parent radius − padding.
- Expressive: Cookie4Sided, Clover, Squircle, Arch for loaders/FAB morph/press morph. Buttons morph pill → 12dp + scale 0.96 on press, Spring.StiffnessHigh (pressed shape per BUTTON-SHAPE-GOSPEL.md, 2026-10-20 — supersedes the earlier 16dp).

### Typography (M3 scale; font: Shipopotamus Sans, fallback Roboto Flex / Google Sans Flex variable)
- displayLarge 57/64/−0.25/400 (hero only), displayMedium 45/52, displaySmall 36/44
- headlineLarge 32/40/400, headlineMedium 28/36, headlineSmall 24/32
- titleLarge 22/28/500 SemiBold (card titles, top bar; emphasized in Expressive), titleMedium 16/24/500, titleSmall 14/20/500
- labelLarge 14/20/500 (ALL button labels), labelMedium 12/16/500, labelSmall 11/16/500
- bodyLarge 16/24/400 (main reading), bodyMedium 14/20/400 (default text-field input), bodySmall 12/16/400 (captions)
- Dark astigmatism rule: never weight 300 on dark; 400–500 body, 600 emphasis; letter-spacing +0.15 for bodySmall.

### Elevation (dark)
- No drop shadows in dark. Tonal: 0dp bg-base (no tint); 1dp surface-1 (5% primary tint); 3dp surface-2 (8%); 6dp surface-3 (11%). Use surfaceContainer / High / Highest; surfaceTint = accent.

## Component specs (defaults)
- BUTTONS: height 40dp (Large/primary CTA 56dp), full pill, labelLarge, padding 24dp text-only / 16dp with leading icon, icon 18dp, gap 8dp. Emphasis: Filled → Tonal → Elevated → Outlined → Text. Filled: accent container; Tonal: accent-muted container, text-primary on; Outlined: 1dp border-default, no fill; Text: 12dp horizontal padding. Icon button 40×40, icon 24dp. Segmented/ToggleButton 40dp, selected = tonal. Press: corner morph 50%→12dp + scale 0.96 + ripple (spring). **Button shape (BUTTON-SHAPE-GOSPEL.md, canonical 2026-10-20):** default = FULL PILL (`ShippopotamusButtonShape = CircleShape`) for consumer surfaces; allowed SQUARE variant = `RoundedCornerShape(12.dp)` medium / 16dp large (`ShippopotamusButtonShapeSquare`) for toolbars, secondary, and data-dense/enterprise contexts (inside a 12dp-radius card use an 8dp inner button, nested radius = outer − padding). Never mix randomly in the same row — one screen = one button shape language. Both variants morph on press to the same pressed shape, `ShippopotamusPressedShape = RoundedCornerShape(12.dp)`.
- FAB: small 40 / medium 56 / large 96; shape large 16dp or full (extended); accent container, icon 24dp. Extended FAB 56dp height, labelLarge, 16dp icon gap.
- TEXT FIELDS: height 56dp; padding 8dp vertical, 16dp horizontal (12dp with icons); icon-to-text gap 16dp; supporting-text top gap 4dp. Filled: surface-2 container, extraSmall top-only radius, active indicator 2dp accent / inactive 1dp. Outlined: transparent, 1dp border-default (2dp accent focused), extraSmall all corners, label padding 4dp. Input bodyLarge; label floats bodySmall→labelSmall; supporting bodySmall. Never pure-white label: text-secondary.
- CARDS: Elevated = surface-1 + 1dp tonal; Filled = surface-2 flat; Outlined = transparent + 1dp border. Shape medium 12dp (large 16dp hero), padding 16dp, click = ripple + elevate to 3dp. Nested inner radius = 12 − 8 = 4dp.
- CHIPS: all 32dp height, label labelLarge, horizontal padding 16dp, icon gap 8dp. Assist: small 8dp, 1dp outline. Filter: small, check 18dp, selected = tonal. Input: avatar 24dp + close 18dp. Suggestion: tonal.
- SWITCH: track 32dp high × 52dp wide, full pill; thumb 24dp; inactive track border-default color; active track accent; thumb white; touch auto-48dp.
- CHECKBOX/RADIO: 20dp visible, 48dp touch; checkbox small 4dp, radio full; active accent.
- SLIDER: track 4dp, active accent, inactive border-default, thumb 20dp, halo 40dp press, 48dp touch height.
- BOTTOM SHEET: extraLarge 28dp top-only, surface-3 container at 6dp/11% tint, drag handle 32×4 #6E7280, scrim palette scrim. DIALOG: extraLarge 28dp, surface-3, padding 24dp, button gap 8dp, title titleLarge. (Haze sheets follow GLASS-BARS-GOSPEL.md: hazeChild + ShippopotamusHazeSheet + surface-3 tint, no opaque scrim.)
- NAVIGATION: bar 80dp height, 3–5 items, active indicator 32dp full pill, icon 24dp, label labelMedium; container is GLASS per GLASS-BARS-GOSPEL.md (always glass, never solid — supersedes the earlier bg-subtle/surface-1 solid container). Rail 80dp wide. TopAppBar small 64dp (titleLarge), medium 112dp, large 152dp — top bar transparent at rest, glass when scrolled, per GLASS-BARS-GOSPEL.md.
- LIST ITEM: one-line 56dp / two-line 72dp / three-line 88dp; leading icon 24dp, padding 16dp, gap 16dp; title titleMedium, subtitle bodyMedium text-secondary.

## Accessibility & motion (non-negotiable)
- 48×48dp minimum for EVERY interactive (minimumInteractiveComponentSize). Radio/checkbox hit areas expand to 48dp.
- Motion: MotionScheme.expressive() spring physics default; emphasized easing for hero, standard for containers, 300–500ms. Shape morphing is a first-class interaction primitive.
- Haptics: light click on toggles/switches, medium on FAB press (HapticFeedback).

## Theme.kt shape (Sean's sketch)
- darkColorScheme(surface = #0D0E11 [bg-base], surfaceContainer = #181A20, surfaceContainerHigh = #1F222A, surfaceContainerHighest = #272A33, surfaceTint = #7A7CFF, primary = #7A7CFF, outline = #2A2E39) + palette gospel hexes.
- Typography per the scale above; Shapes(extraSmall 4, small 8, medium 12, large 16, extraLarge 28) RoundedCornerShape.
- Component defaults: buttons 40dp (56 CTA) CircleShape, contentPadding 24dp, labelLarge; text fields min 56dp, top 4dp shape; cards 12dp + 1dp tonal; switch track 32/thumb 24; minimumInteractiveComponentSize everywhere; Spacing object only.
