# SHIPPOPOTAMUS BUTTON SHAPE GOSPEL

Declared canonical by Sean, 2026-10-20. Canonical for BLOVIATE / STRATUM / ENCLAVE and all future Shippopotamus apps. Companion to COMPOSE-GOSPEL.md (this gospel governs button shape specifically and supersedes the Compose Gospel on button shape and button press morph) and PALETTE.md / PALETTE-LIGHT.md (colors).

> Spelling note: Sean's pasted source named the tokens `ShipopotamusButtonShape` (one p). The canonical brand/code spelling is **Shippopotamus** (cf. Enclave's `ShippopotamusShapes`), so the tokens below use `Shippopotamus*`. The ruling itself is Sean's verbatim.

## Default for Bloviate / consumer: FULL PILL

- Shape: `CircleShape` / 9999px / full token
- Why: empirically friendliest, highest tap recognition, matches Google Phone, Gboard, etc. More rounded corners + pill buttons are the M3 signature.

## Allowed variant for toolbars / secondary / dense enterprise: SQUARE

- Shape: `RoundedCornerShape(12.dp)` for medium buttons, `RoundedCornerShape(16.dp)` for large buttons
- When to use:
  - Inside a card that already has 12dp radius — use an 8dp inner button (nested radius = outer − padding)
  - Data-dense toolbar, filter bar, table actions
  - When you want the Linear/Raycast serious vibe vs. bubbly consumer

## Rules (non-negotiable)

- **Never mix randomly in the same row. One screen = one button shape language.**
- If you use the square variant, it still morphs on press to the **same pressed shape** as the pill variant (12dp, below).

## Compose implementation

```kotlin
val ShippopotamusButtonShape = CircleShape // DEFAULT PILL
val ShippopotamusButtonShapeSquare = RoundedCornerShape(12.dp) // ALLOWED VARIANT (medium; 16.dp for large)
val ShippopotamusPressedShape = RoundedCornerShape(12.dp) // Both morph to this on press

Button(
  shape = ShippopotamusButtonShape,
  modifier = Modifier.minimumInteractiveComponentSize() // 48dp enforced
) { Text("Primary", style = MaterialTheme.typography.labelLarge) }

// Expressive press morph
val corner by animateDpAsState(if (pressed) 12.dp else 100.dp, spring(StiffnessHigh))
val scale by animateFloatAsState(if (pressed) 0.96f else 1f)
```

## Reconciliation with the earlier Compose Gospel

- The Compose Gospel (2026-10-19) and the repo contracts derived from it said buttons morph pill → **16dp** on press. This gospel sets the pressed shape to **12dp** for both the pill and square variants; **12dp wins** wherever the two disagree on button press.
- Button heights, label style (`labelLarge`), padding, emphasis order, and the 48dp touch-target law are unchanged — they remain governed by the Compose Gospel's component specs.
- Sanctioned carve-outs stand: Stratum's Neo Brutalist theme keeps its own button geometry (4dp) under its permanent carve-out; this gospel governs the Material/Tonal family there, as with the other gospels.
