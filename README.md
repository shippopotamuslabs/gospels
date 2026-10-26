# Shippopotamus Gospels

Gospel as in gospel truth: the authoritative word on how
Shippopotamus software looks, sounds, and gets built.

This repo holds the canon the projects share. Colors, buttons, bars,
Compose rules, voice, architecture, code style, testing. One copy of
each, in one place, with the projects pointing at it.

## The documents

- `PALETTE.md`: the dark theme. Tokens, hex values, contrast ratios.
- `PALETTE-LIGHT.md`: the light theme. Same token names, same structure.
- `COMPOSE-GOSPEL.md`: the Compose UI system: spacing, shapes, typography, components, previews.
- `BUTTON-SHAPE-GOSPEL.md`: button shape and the press morph. Full pill by default, 12dp square variant.
- `GLASS-BARS-GOSPEL.md`: top and bottom bars in glass, on Haze. The top bar stays glazed at rest.
- `WUMBO-GOSPEL.md`: the voice. Every word a user reads, in every app and on the site.
- `ARCHITECTURE-GOSPEL.md`: Clean Architecture: dependency direction, layer responsibilities, domain model policy, the presentation contract.
- `CODE-STYLE-GOSPEL.md`: Kotlin and Compose style: naming, formatting, stability, coroutines.
- `TESTING-GOSPEL.md`: testing standards: pyramid levels, classical school, fakes over mocks; Android grounding in JUnit 5, Turbine, Robolectric.

## How a project uses this repo

Add it as a git submodule at `contracts/gospel/` and pin a release
tag:

```bash
git submodule add https://github.com/shippopotamuslabs/gospels.git contracts/gospel
git -C contracts/gospel checkout v1
```

The pin is the point. A gospel change reaches a project when that
project bumps its pin, in a commit that says it did. Builds stay
repeatable, and a canon edit lands in each app on that app's schedule.

## How a change lands

1. Open a pull request here.
2. Merge it and tag a new release.
3. Each project bumps its submodule pin to the new tag.

## Contracts and gospels

Contracts live in a project's own repo and bind that project: its
module map, its package names, its screens, its gates. Gospels sit
above the contracts and state the rules every project shares. A
contract points up at a gospel for the shared rules and keeps the
local material at home.

Carve-outs live in the consuming repo. Stratum's Neo Brutalist theme
is the standing example: Stratum's contract records the carve-out,
the Palette Gospel keeps the standard tokens, and both documents say
which one governs Stratum's screens.

## License

0BSD license. Do whatever you want with the code. Just don't expect
it to work or sue me.
