# SHIPPOPOTAMUS GLASS BARS GOSPEL — TOP + BOTTOM + HAZE

Declared canonical by Sean, 2026-10-20. Status: CANONICAL

> **AMENDED 2026-10-20 (Sean, on-device review of the STR-153 build): the top bar is ALWAYS GLAZED.** The original transparent-at-rest state is repealed in full — in Sean's words, "The Gospel of Glaze says that it must remain glazed." The 1dp hairline is likewise permanent (it is the glazed bar's lower edge, not a scrolled-state cue). Everything below reflects the amendment; Stratum's contract codification (§12.8) rides STR-148. for BLOVIATE / STRATUM / ENCLAVE and all future Shippopotamus apps. Uses Haze 2.0 (Chris Banes). Android 15+ edge-to-edge enforced. Companions: PALETTE.md (the tints below are palette tokens), COMPOSE-GOSPEL.md (this gospel supersedes it for bars, sheets, and overlay glass — see Reconciliation), BUTTON-SHAPE-GOSPEL.md.

> Spelling note: Sean's pasted source named the styles/functions `Shipopotamus*` (one p). The canonical brand/code spelling is **Shippopotamus**, so everything below uses `Shippopotamus*`. The rulings are Sean's verbatim.

## Dependency

```kotlin
implementation("dev.chrisbanes.haze:haze:2.0.1") // core blur
implementation("dev.chrisbanes.haze:haze-glass:2.0.1") // refraction Glass (optional, for hero)
```

Haze provides hardware-accelerated visual effects — blur, refraction-driven Glass, and custom effects with reusable config.

## Foundation — edge-to-edge (required for glass to work)

- Activity: `enableEdgeToEdge()` — status + gesture nav are transparent, content draws behind
- Scaffold: `contentWindowInsets = WindowInsets(0,0,0,0)` — you handle insets manually
- Main body: `Modifier.hazeSource(state)` + `.windowInsetsPadding` + `.consumeWindowInsets`

## Haze gospel — reusable style

Create one `HazeState` per screen, share it:

```kotlin
val hazeState = rememberHazeState()
```

Define Shippopotamus reusable styles — don't inline magic numbers:

```kotlin
val ShippopotamusHazeTopBar = HazeStyle.Unspecified.copy(
  blurRadius = 24.dp, // 20–28dp for top bar — enough to read title over photos
  tint = Color(0xFF0D0E11).copy(alpha = 0.65f), // bg-base #0D0E11 at 65% — darkens enough for text legibility
  noiseFactor = 0.5f, // micro-texture prevents banding on OLED
)

val ShippopotamusHazeBottomBar = HazeStyle.Unspecified.copy(
  blurRadius = 20.dp,
  tint = Color(0xFF181A20).copy(alpha = 0.75f), // surface-1 #181A20 at 75% — slightly lighter than page to anchor
  noiseFactor = 0.5f
)

val ShippopotamusHazeSheet = HazeStyle.Thick // preset: heavy blur for bottom sheets
```

Usage pattern:

- Background scrollable = `hazeSource`
- Bars = `hazeChild`
- **Never put haze behind body text directly — behind containers only.**

## Top half bar — TopAppBar — the striking one

Design spec:

- Height: Small 64dp (default), Medium 112dp, Large 152dp for hero
- Container: ALWAYS GLASS (`ShippopotamusHazeTopBar`), at rest and scrolled alike (2026-10-20 amendment — the bar never goes transparent)
- Shape: none — full width, no rounded corners
- Content: Title `titleLarge` 22sp, icons 24dp, padding 16dp
- Border: none — blur provides separation
- Border: subtle 1dp bottom hairline #2A2E39 at 40%, always — the glazed bar’s permanent lower edge

Why it works: content scrolls underneath, blurred — you see movement but the title stays readable. Looks genuinely striking vs. a solid color that just pops.

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ShippopotamusTopBar(
  title: String,
  hazeState: HazeState,
  scrollBehavior: TopAppBarScrollBehavior
) {
  TopAppBar(
    title = { Text(title, style = MaterialTheme.typography.titleLarge) },
    navigationIcon = { IconButton({}) { Icon(Icons.Default.Menu, null) } },
    actions = { IconButton({}) { Icon(Icons.Default.Search, null) } },
    scrollBehavior = scrollBehavior,
    colors = TopAppBarDefaults.topAppBarColors(
      containerColor = Color.Transparent, // the glaze is the haze effect below
      scrolledContainerColor = Color.Transparent
    ),
    modifier = Modifier
      .hazeChild(
        state = hazeState,
        style = ShippopotamusHazeTopBar // always glazed (2026-10-20 amendment)
      )
      .drawWithContent {
        drawContent()
        drawLine(Color(0xFF2A2E39).copy(0.4f), Offset(0f, size.height), Offset(size.width, size.height), 1.dp.toPx())
      }
  )
}

// In screen:
val scrollBehavior = TopAppBarDefaults.pinnedScrollBehavior() // or enterAlwaysScrollBehavior()
Scaffold(
  modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
  topBar = { ShippopotamusTopBar("Bloviate", hazeState, scrollBehavior) },
  containerColor = Color(0xFF0D0E11),
  contentWindowInsets = WindowInsets(0,0,0,0)
) { padding ->
  LazyColumn(
    Modifier
      .fillMaxSize()
      .hazeSource(state = hazeState, zIndex = 0f) // zIndex 0 = source
      .windowInsetsPadding(WindowInsets.statusBars.only(WindowInsetsSides.Top))
      .padding(padding)
  ) { /* main body passes underneath top bar */ }
}
```

## Bottom half bar — bottom navigation bar

Design spec:

- Height: 80dp Material3 `NavigationBar` (68dp if you want tighter, but 80dp is canonical)
- Container: ALWAYS glass — never solid. Translucent tint + blur so content scrolls under.
- Shape: none for the bar itself; active indicator = full pill 32dp height, accent #7A7CFF at 15% fill + accent icon
- Icons: 24dp, label `labelMedium` 12sp Medium
- Inactive: #858994 (text-tertiary); Active: #EDEEF0 (text-primary) + accent indicator
- Border: 100% borderless — no stroke-line dividers; blur is the separation
- Gesture nav handling: `windowInsetsPadding(WindowInsets.navigationBars)` + `navigationBarsPadding` on content so the last item is not hidden behind the system pill

```kotlin
@Composable
fun ShippopotamusBottomBar(hazeState: HazeState) {
  var selected by remember { mutableStateOf(0) }

  NavigationBar(
    containerColor = Color.Transparent, // haze handles color
    tonalElevation = 0.dp,
    modifier = Modifier
      .hazeChild(
        state = hazeState,
        style = ShippopotamusHazeBottomBar
      )
      // ensure transparent system gesture bar doesn't clip
      .windowInsetsPadding(WindowInsets.navigationBars.only(WindowInsetsSides.Bottom))
  ) {
    listOf("Home", "Search", "Library").forEachIndexed { i, label ->
      NavigationBarItem(
        selected = i == selected,
        onClick = { selected = i },
        icon = { Icon(Icons.Default.Home, null) },
        label = { Text(label, style = MaterialTheme.typography.labelMedium) },
        colors = NavigationBarItemDefaults.colors(
          selectedIconColor = Color(0xFFEDEEF0),
          selectedTextColor = Color(0xFFEDEEF0),
          indicatorColor = Color(0xFF7A7CFF).copy(alpha = 0.15f), // accent-muted
          unselectedIconColor = Color(0xFF858994),
          unselectedTextColor = Color(0xFF858994)
        )
      )
    }
  }
}

// Add to Scaffold bottomBar param — same hazeState as top bar so both blur same source
```

## Where else to use Haze

1. **Bottom sheet:** sheet container = `hazeChild` with `ShippopotamusHazeSheet` + surface-3 #272A33 tint. Content underneath (map/feed) stays visible blurred — feels elevated without an opaque scrim.
2. **FAB over feed:** extended FAB that blurs the feed behind it — use a small blur, 12dp.
3. **Image overlays:** hero image with a glass chip (e.g. a "Live" badge) — use haze-glass with refraction for liquid glass.

**Performance:** Haze 2.0 has adaptive rendering quality — it automatically drops quality on a low-end GPU. Keep `blurRadius <= 28dp`; avoid stacking more than 2 `hazeChild`s in the same frame.

This is the pattern that makes top nav scroll-under look striking: blurred glass, always, with the main body as `hazeSource`. Bottom nav is always glass so it anchors but doesn't feel heavy. The top bar joined it in the 2026-10-20 amendment: the glaze is permanent.

## Reconciliation with the earlier gospels (Tensor, 2026-10-20)

- **Elevation/opaque rule:** the Compose Gospel's "elevation is tint, not shadow" model and Enclave's conflict-register C13 treated surfaces as opaque tonal. This gospel carves out **bars, sheets, and overlay glass** as translucent by design. Body/content surfaces stay opaque tonal, and haze never sits directly behind body text (containers only) — the reading-surface rules are unchanged.
- **Nav container:** the Compose Gospel's navigation spec (solid bg-subtle/surface-1 container) is superseded for the bottom bar: **always glass, never solid**. The 80dp height, 32dp pill indicator, icon/label specs are unchanged.
- **Bottom sheet:** the Compose Gospel sheet spec (surface-3 container + palette scrim) still governs non-Haze sheets; where a sheet is a Haze sheet, §"Where else" above governs and the opaque scrim treatment does not apply.
- **Title weight:** Sean's pasted spec line read "Title titleLarge 22sp Medium"; the Compose Gospel pins `titleLarge` as SemiBold and its implementation sketch uses the theme typography. The snippet above consumes `MaterialTheme.typography.titleLarge` (theme-owned), so the theme scale wins — flagged here rather than silently edited in Sean's text.
- **Per-repo state at adoption:** Enclave already ships spec-driven glass (ENC-33) on **haze 1.7.2** (1.x API, `hazeEffect`-style wiring; Tonal translucent, Neo opaque) — direction aligns, but adopting this gospel there means a Haze **2.0.1** bump + API migration (`hazeSource`/`hazeChild`), not a drop-in. Bloviate and Stratum have no Haze dependency yet. Stratum's STR-150 nav-bar work (solid bg-subtle container) predates this gospel and is transitional under the supersession above.
