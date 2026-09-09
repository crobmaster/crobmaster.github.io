---
title: "CJK Font Fallback in Godot 4: One Theme, Three Scripts"
date: 2026-09-09
categories: [godot]
tags: [godot, i18n, fonts]
---

*Tap Tap Picture Book* ships in seven languages, three of which — English, Japanese, and Korean — need real script coverage rather than a placeholder tofu box. Godot 4 makes this workable with a single global `Theme`, but getting there took a few non-obvious steps: a base font with a glyph hole, a fallback chain, a baseline mismatch, and a font-size gotcha in the theme system itself. This is a walkthrough of how the UI font pipeline in `game_state.gd` ended up shaped the way it did.

## The gap: a base font with zero Hangul glyphs

The game's base UI font is MPLUS Rounded 1c (Medium weight). It covers Latin — including accented forms — and Japanese cleanly. What it does **not** cover is Hangul: zero Korean glyphs. Assign it directly to a `Theme` and any Korean string silently falls through to whatever CJK font the OS happens to have installed, which breaks visual consistency between platforms and between Korean and the other two scripts on the same screen.

Godot's answer to "font A doesn't have this glyph, try font B" is `FontVariation.fallbacks` — an array of fonts consulted in order whenever the primary font can't shape a character:

```gdscript
var base_font := load("res://src/fonts/MPLUSRounded1c-Medium.ttf") as FontFile
var korean_font := load("res://src/fonts/BMJUA.ttf") as FontFile

var ui_font := FontVariation.new()
ui_font.base_font = base_font
ui_font.fallbacks = [korean_font]
```

Any label that renders Latin or Japanese text uses `base_font` as normal; the moment it hits a Hangul syllable, Godot transparently falls back to `korean_font` for just that glyph run. One `FontVariation` now speaks three scripts, and the substitution happens per-character, so mixed strings render correctly without any manual splitting.

## Fixing the baseline once two scripts share a line

Swapping in a fallback font is rarely a drop-in fix on its own — different type foundries draw their `ascent`/`descent`/`lineGap` metrics differently, so a Korean glyph and a Japanese glyph on the same baseline can end up visually offset even though the font engine considers them correctly aligned. In this project, the shipped `BMJUA.ttf` isn't the distributor's original file: its `hhea`/`OS/2` vertical metrics were rewritten to match MPLUS's values, and the glyph outlines were translated vertically, specifically to fix that baseline drift where the two scripts meet. It's a one-time, font-level fix rather than something handled per-label at runtime — which matters, because runtime nudging (padding, offsets) would have to be reapplied at every font size the UI scales to.

## Weight without a second font file

The UI needs a visual weight hierarchy — titles bolder than body text — but the project ships only one Latin/Japanese master and one Korean master, not a separate bold cut of each. Instead of doubling the font count, weight comes from `FontVariation.variation_embolden`, applied at two tiers:

```gdscript
var korean_font_title := FontVariation.new()
korean_font_title.base_font = korean_font
korean_font_title.variation_embolden = 0.5

var korean_font_body := FontVariation.new()
korean_font_body.base_font = korean_font
korean_font_body.variation_embolden = 0.2

ui_font = FontVariation.new()
ui_font.base_font = base_font
ui_font.variation_embolden = 0.5
ui_font.fallbacks = [korean_font_title]

body_font = FontVariation.new()
body_font.base_font = base_font
body_font.variation_embolden = 0.2
body_font.fallbacks = [korean_font_body]
```

`ui_font` (titles, emphasis, and the theme default) and `body_font` (body and secondary text) each carry their *own* Korean fallback tuned to the same embolden value, so the two scripts land at matching visual weight instead of Korean text looking heavier or lighter than the Latin/Japanese text next to it. The embolden numbers themselves aren't derived from any formula — they're hand-picked by comparing renders on a real device and in the editor, and they'll likely keep shifting as the UI evolves.

## The theme font-size trap

One gotcha that's easy to lose an hour to: setting `Theme.default_font_size` is **not enough** to change font size globally. Godot's built-in theme overrides font size per control type, so a fresh `Theme` still needs each type covered explicitly, and the theme has to actually be assigned to the tree root before it takes effect:

```gdscript
var theme := Theme.new()
theme.default_font_size = FONT_SIZE
for type in ["Label", "Button", "RichTextLabel", "LineEdit", "CheckButton", "CheckBox", "OptionButton"]:
    theme.set_font_size("font_size", type, FONT_SIZE)

# ... assign fonts, styleboxes, etc. ...

get_tree().root.theme = theme
```

Skip the per-type loop and `default_font_size` quietly does nothing for `Label` or `Button` — the built-in theme's per-type value wins. This is called once at startup, ahead of platform-specific window setup, so the correct theme is in place before the first frame renders.

## Keeping the binary small: subsetting on a schedule, not automatically

Shipping full CJK font files would bloat the app for glyphs the game never actually uses. A build script (`tools/fonts/subset_fonts.py`) subsets the source masters down to only the characters that are actually referenced, pulled from three places: every string in the `i18n/*.json` locale files, hardcoded string literals in `.gd`/`.tscn` files (comments excluded — the regex only matches quoted literals), and the printable ASCII range as a safety baseline so digits and symbols never go missing. Font-to-locale mapping is centralized in a `FONT_LOCALE_MAP`, and requesting a character a given font doesn't have is a silent no-op rather than an error — so overlapping character requests across fonts are harmless.

The script doesn't watch for changes automatically. It's a manual step before release, run again whenever locale text, font masters, or hardcoded strings change — with a `--dry-run` pass first to confirm new characters are actually covered before regenerating the shipped fonts.

## Takeaways

- `FontVariation.fallbacks` handles missing-glyph fallback per character, not per string — mixed-script text just works.
- A fallback font swap can still need a metrics-level fix (ascent/descent/lineGap, glyph offset) if the two typefaces don't share a baseline convention.
- Weight hierarchy doesn't require more font files — `variation_embolden` on a `FontVariation` gets you there, as long as every script's fallback gets its own matching value.
- `Theme.default_font_size` alone won't touch per-type overrides; set `font_size` per control type explicitly.
- Subsetting is a build-time discipline, not a runtime concern — the tooling matters as much as the font choice itself.
