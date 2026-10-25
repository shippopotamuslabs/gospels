# SHIPPOPOTAMUS COLOR PALETTE GOSPEL — LIGHT THEME

✅ **CANONICAL — ratified by Sean 2026-10-19**, all four open choices ruled in the draft's favor: (1) white cards on a grey canvas, (2) cool indigo-grey neutrals, (3) the safe accent step #5457E8, (4) text-faint stays at ~3.3:1, restricted to disabled/incidental use. Companion to `PALETTE.md` (the dark gospel, canonical 2026-10-19). Same token names, same structure, derived the same way: research first, hexes second. All contrast ratios below were computed with the WCAG 2.x relative-luminance formula, not eyeballed.

## Why (empirical basis)

1. **Light mode is the astigmatism-friendly polarity — halation is a dark-mode disease.** The dark gospel exists to fight halation (bright glyphs bleeding into black for the ~1 in 3 people with astigmatism). In positive polarity the failure inverts: a bright display constricts the pupil, which shrinks spherical aberration and increases depth of field — the "positive polarity advantage" replicated across legibility and proofreading studies (Piepenbrock, Mund, Mayr & Buchner, *Ergonomics*; Dobres et al. on glance legibility). Light mode's enemy is not blur; it is **glare and vibration at maximum contrast**. So this gospel never pairs the extremes, exactly as the dark one doesn't — it just refuses them from the other side.
   - https://www.psychologie.hhu.de/fileadmin/redaktion/Oeffentliche_Medien/Fakultaeten/Mathematisch-Naturwissenschaftliche_Fakultaet/Psychologie/AAP/Publikationen/in_press/Piepenbrock-in_press-Smaller_pupil_size_and_better.pdf
   - https://pmc.ncbi.nlm.nih.gov/articles/PMC6925951/
2. **Don't run 21:1. Primary text lands at ~15.5:1 — the same landing zone as the dark gospel's 15.8:1.** Pure black (#000) on pure white (#FFF) is the light-mode mirror of the dark gospel's banned pairing: maximum luminance difference drives glare and edge vibration, and no major system ships it. Material's own light scheme pairs onSurface ≈ #1D1B20 with surface ≈ #FEF7FF (~15–16:1); Notion pairs 95%-alpha black with white. Our text-primary #1B1C22 on bg-base measures **15.5:1** — near-black with the brand's indigo tint, deliberately not #000.
3. **WCAG floors, with headroom where text actually lives.** AA requires 4.5:1 (normal text) / 3:1 (large text, UI component boundaries per SC 1.4.11); AAA is 7:1. For calibration, #767676 is the lightest grey that passes AA on white (4.54:1) and #595959 is the AAA grey (7.0:1) — most "designer greys" fail outright. This ramp is built so *secondary body text* clears AAA (7.8:1) and even captions clear AA (5.2:1); only the faint step drops to non-text grade, and it is labeled as such. APCA (the WCAG 3 candidate) corroborates aiming high in light mode: Lc 75 minimum / Lc 90 preferred for body text, and its polarity model says dark-on-light needs *more* ink, not less, to resolve letterforms.
   - https://storypress.app/blog/accessibility-basics
   - https://testingbot.com/support/accessibility/web/rules/color-contrast-enhanced
   - https://git.apcacontrast.com/documentation/README.html
   - https://inkbotdesign.com/wcag-3-0-typography-standards/
4. **Accents must get darker and richer on light — the dark accent cannot cross over.** The dark gospel's accent #7A7CFF measures only **3.4:1 on white**: unusable as text and washed out as a fill. This is physics, not taste: saturated colors look brighter than their luminance (Helmholtz–Kohlrausch effect), bright surrounds suppress that boost via simultaneous contrast, and light areas appear larger than identical dark ones (irradiation illusion) — a pastel indigo that commands a dark screen simply evaporates on a bright one. Both reference systems therefore *re-derive* the accent per mode instead of inverting it: Material uses primary tone 40 in light vs tone 80 in dark; Radix reserves step 9 (the highest-chroma step) as the light-mode solid and pairs it with white text. Our accent keeps the gospel hue (~239°) and drops to tone-40 depth: #5457E8 carries white label text at 5.4:1 and works as link text at 4.9:1 on the canvas.
   - https://www.academia.edu/2786902/The_brightness_of_colour (Corney, Haynes, Rees & Lotto, *The Brightness of Colour*, PLoS ONE — the Helmholtz–Kohlrausch study)
   - https://deepwiki.com/klorman/Material3.Avalonia/2.1-color-system-and-token-hierarchy (M3 tone assignments: primary 40 light / 80 dark, surface Neutral 98, outline NeutralVariant 50)
   - https://pub.dev/packages/radix_colors (Radix 12-step anatomy: 1–2 backgrounds, 3–5 component fills, 6–8 borders, 9–10 solids, 11–12 text)
5. **Elevation inverts: shadows come back, and surfaces climb toward white instead of away from black.** In light mode the practitioner convergence is a layered system, not a white slab: an off-white canvas, content surfaces at or near white, pure white reserved for raised cards/dialogs — depth carried by shadow plus small lightness steps (Notion: paper canvas, 1px rgba(0,0,0,0.1) "whisper" borders, shadow stacks capped at 0.05 cumulative opacity; Radix steps 1–2 as backgrounds under step 3–5 components). Background luminance also matters more than raw contrast for legibility under bright ambient light — another reason the canvas is a controlled off-white rather than a blazing #FFFFFF field.
   - https://stopwhitebackgrounds.com/
   - https://opendesigner.io/design-systems/notion
   - http://dx.doi.org/10.3390%2Fapp11188679 (tablet visual comfort under varying ambient light)
6. **Blue light is not a palette input.** The AAO's position: there is no scientific evidence that blue light from screens damages eyes, and digital eye strain is behavioral — blink rate collapses from ~15/min to 5–7/min at screens, drying the eye and fatiguing focus. Blue-filtering products have not been shown to reduce that strain. So no token in this file is justified by "eye health" folklore; the neutrals keep the dark gospel's cool indigo tint for *brand continuity*, not medicine. (Blue light's one real effect is circadian — an argument about evening dark-mode use, not about light-theme hexes.)
   - https://www.phillyvoice.com/screens-eye-fatigue-prevention-tips/
   - https://factfizz.com/2026/09/17/screens-disrupt-your-blinks-but-the-biggest-dry-eye-culprit-may-be-hiding-in-your-eyelids/

## Tokens

### Background layers
Elevation works by climbing toward white and adding shadow (see "What changes vs dark"). Steps are small on purpose — on light, separation comes from shadow and borders, not big lightness jumps.
- bg-base (app canvas, deepest): #F4F4F8
- bg-subtle (navbar, tab bar): #ECECF1
- surface-1 (cards, sheets): #FFFFFF
- surface-2 (hover/pressed fills, tonal containers): #EFEFF4
- surface-3 (modal, popover): #FFFFFF (same hex as surface-1 by design — the shadow scale, not lightness, separates the top layer)

### Borders / dividers
On light, borders are *darker* than the surface (dark gospel: lighter). Two are decorative-grade; border-strong is specced to a hard standard because light borders vanish where dark ones glow.
- border-subtle (hairline dividers, decorative only): #E2E2EA
- border-default (cards, resting inputs): #C6C7D1
- border-strong (interactive boundaries — 3.35:1 vs bg-base, meets WCAG 1.4.11's 3:1 for component boundaries): #83858F

### Text layers
Same ~15:1 primary target as dark, from the other direction; chroma fades down the ramp so faint text reads as grey, not dirty blue.
- text-primary (near-black, indigo-tinted, 15.5:1 on bg-base / 17.0:1 on white): #1B1C22
- text-high (secondary headings, 12.2:1): #2C2E3E
- text-secondary (body secondary, 7.8:1 — AAA): #494B5C
- text-tertiary (captions, metadata, 5.2:1 — AA): #63667A
- text-faint (icons, disabled, placeholder — 3.3:1, NON-text grade only): #83868F

### Indigo accent family (primary action)
Same hue as the dark gospel, re-derived at light-mode depth (rationale 4). State direction inverts: hover/active go *darker*, not lighter.
- accent-base: #5457E8 (white text on it 5.4:1; as text on bg-base 4.9:1 — both AA)
- accent-hover: #4849D6
- accent-active: #3739B8
- accent-border-subtle: #7A7CFF (the dark gospel's base, demoted to a boundary tint: 3.4:1 vs white, passes the 3:1 non-text bar)
- accent-deep: #2F31A0 (text-grade indigo for use on tinted fills: 10.2:1 vs white, 7.9:1 on accent-muted-bg)
- accent-muted-bg (12% opacity): rgba(84,87,232,0.12) — composites to ≈#E1E1F6 on bg-base; pair with accent-deep text, never accent-base (4.2:1, sub-AA)
- focus-ring: rgba(84,87,232,0.60) — see the translucency warning under "What changes"

### Semantic colors (adjusted for light)
The dark gospel's pastels measure ~2:1 on white — invisible. Light semantics go deep enough to be *text-grade* (≥4.5:1 on white) while staying desaturated in spirit; each also works as a fill under white text at the same ratio.
- success-muted-green: #2E7D43 (5.1:1)
- warning-amber: #9C6B10 (4.6:1 — an ochre, because no true yellow can be text on white; every shipping system makes this same compromise)
- danger-soft-red: #C02E3A (5.7:1)
- info-sky-blue: #1F6FB4 (5.3:1)

### Mobile extras
- bg-scrim (bottom sheet scrim): rgba(13,14,17,0.45) — same near-black hue as the dark gospel's scrim; alpha drops from 0.80 because a bright UI needs far less dimming to isolate a sheet
- surface-overlay: #FFFFFFF2

## What changes vs dark (the inversion table)
- **Elevation model:** dark climbs a lightness ladder (each surface 4–8% lighter, zero shadow). Light *whitens* toward the card layer, then depth is carried by shadow (soft, layered, low-opacity — Notion's stacks never exceed ~0.05 per layer) plus a primary-tinted surfaceTint at low alpha, per M3 light. "Raised" means *brighter + shadowed*, the exact mirror of dark.
- **State direction:** hover/pressed fills get *darker* in light (dark gospel: lighter). State overlays composite black/accent over the surface instead of white.
- **Saturation direction flips:** dark *desaturates* accents (≤60% sat) so they don't vibrate against black. On white, desaturated = invisible; light accents and semantics run *deeper and richer* than their dark counterparts. The ≤60% cap is a dark-mode rule and does not carry over.
- **Translucency doesn't mirror:** the dark focus ring works at 40% alpha because indigo glows against near-black; 45% alpha of the light accent composites to only ~1.9:1 on the canvas. Hence focus-ring at 60% here (≈2.4:1 composite), and components should render critical focus indicators in solid accent-base (5.4:1) wherever the platform allows. Alpha tokens must be re-tuned per polarity, never copied.
- **Text:** 87%-white on near-black inverts to ~89%-black (#1B1C22) on off-white, at the same ~15.5:1 working contrast — neither mode ships the 21:1 extreme. The Compose Gospel's "never weight 300" rule survives the crossing: APCA's model says dark-on-light letterforms need *more* ink to resolve, so light mode is not a license for thin type.
- **Borders flip sign:** darker than the surface they divide, and the strong step is held to WCAG 1.4.11's 3:1 because a too-light border on white doesn't look subtle — it looks absent.
- **Scrim:** same hue, roughly half the alpha (0.80 → 0.45).

## Open choices for Sean (research genuinely conflicts — not silently decided)
1. **Surface model: white-card-on-grey-canvas (this draft) vs the M3-classic grey ladder.** The draft follows the web/SaaS convergence (Notion, Radix-based systems): canvas #F4F4F8, cards pure white. Google's own M3 light scheme never uses pure white for containers — its ladder runs *downward* from a #FEF7FF surface (containers ≈ #F3EDF7 → #E7E0EC). The grey ladder is more Material-native and subtler; the white-card model reads as "raised" instantly and matches the dark gospel's "move toward the extreme" rhythm. Aesthetics and platform-orthodoxy genuinely split here; the draft picks white-card, flagging the alternative before any implementation.
2. **Neutral temperature: cool indigo-grey (this draft) vs warm paper.** The draft keeps the dark gospel's cool blue-grey tint so light and dark feel like one product. The strongest counterexample is Notion, whose warm ecru neutrals (#F6F5F4 family) are widely credited for its reading comfort. Nothing in the vision research adjudicates temperature — this is a brand call wearing a lab coat either way.
3. **Accent step: #5457E8 (this draft) vs one step brighter.** Brighter candidates near the dark accent's lightness (e.g. Linear's #5E6AD2) are prettier to some eyes but fail as text: white-on-fill drops to 4.7:1 and canvas link text to 4.3:1 (sub-AA), forcing all text-links onto accent-deep. The draft takes the step that passes everywhere; ratifying the brighter step means accepting the deep-text constraint as a permanent rule.
4. **The text-faint floor.** At 3.3:1, text-faint is legal only for disabled and incidental content. If placeholders and hints should be fully AA-readable (a defensible strictness — APCA puts this zone at ~Lc 45, "spot text" grade), then text-tertiary (5.2:1) becomes the floor and text-faint is icons-only. The draft keeps the conventional floor and labels it; Sean decides how strict the house rule is.
