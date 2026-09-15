---
title: "CJK Font Fallback in Godot 4: One Theme, Three Scripts"
date: 2026-09-09
categories: [godot]
tags: [godot, i18n, fonts]
description: "How Tap Tap Picture Book renders Latin, Japanese, and Korean from one Godot Theme, using FontVariation fallbacks and a font-level baseline fix."
image: /icon.png
---

*Tap Tap Picture Book* ships in seven languages, three of which — English, Japanese, and Korean — need real script coverage rather than a placeholder tofu box. Godot 4 makes this workable with a single global `Theme`, but getting there took a few non-obvious steps:

- a base font with a glyph hole,
- a fallback chain,
- a baseline mismatch, and
- a font-size gotcha in the theme system itself.

This is a walkthrough of how the game's UI font setup ended up shaped the way it did.

## The gap: a base font with zero Hangul glyphs

The game's base UI font is MPLUS Rounded 1c (Medium weight).

- **Covers:** Latin (including accented forms) and Japanese.
- **Doesn't cover:** Hangul — zero Korean glyphs.
- **What goes wrong:** assign it directly to a `Theme` and Korean strings silently fall through to whatever CJK font the OS has installed. That breaks visual consistency across platforms, and between Korean and the other two scripts on the same screen.

Godot's answer to "font A doesn't have this glyph, try font B" is [`FontVariation.fallbacks`](https://docs.godotengine.org/en/stable/classes/class_font.html#class-font-property-fallbacks) — an array of fonts consulted in order whenever the primary font can't shape a character. `BASE_FONT_PATH` and `KOREAN_FONT_PATH` stand in for wherever your own font files live:

```gdscript
var base_font := load(BASE_FONT_PATH) as FontFile
var korean_font := load(KOREAN_FONT_PATH) as FontFile

var ui_font := FontVariation.new()
ui_font.base_font = base_font
ui_font.fallbacks = [korean_font]
```

- Latin and Japanese text renders with `base_font` as normal.
- The moment a Hangul syllable appears, Godot falls back to `korean_font` for just that glyph run.
- Substitution is per character, so mixed-script strings render correctly without any manual splitting.

## Fixing the baseline once two scripts share a line

Swapping in a fallback font is rarely a drop-in fix on its own.

- **The problem:** type foundries set their `ascent`/`descent`/`lineGap` metrics differently. A Korean glyph and a Japanese glyph on the same line can look offset even though the font engine considers them aligned.
- **The fix here:** the shipped BM JUA font isn't the distributor's original file. Its [`hhea`](https://learn.microsoft.com/en-us/typography/opentype/spec/hhea)/[`OS/2`](https://learn.microsoft.com/en-us/typography/opentype/spec/os2) vertical metrics were rewritten to match MPLUS's values, and its glyph outlines were shifted vertically.
- **Why at the font level:** it's a one-time fix. Runtime nudging (padding, offsets) would have to be reapplied at every font size the UI scales to.
- **Note:** the exact offsets are specific to this font pairing. A different pair of fonts needs its own measurements.

## Weight without a second font file

The UI needs a visual weight hierarchy — titles bolder than body text — but the project ships only one Latin/Japanese font and one Korean font, not a separate bold cut of each. Instead of doubling the font count, weight comes from [`FontVariation.variation_embolden`](https://docs.godotengine.org/en/stable/classes/class_fontvariation.html), applied at two tiers:

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

| Font | Used for | Embolden | Korean fallback |
|---|---|---|---|
| `ui_font` | Titles, emphasis, theme default | `0.5` | `korean_font_title` (`0.5`) |
| `body_font` | Body and secondary text | `0.2` | `korean_font_body` (`0.2`) |

- **Matching fallbacks:** each tier carries its *own* Korean fallback with the same embolden value, so Korean text doesn't look heavier or lighter than the Latin/Japanese text next to it.
- **Hand-tuned values:** the numbers aren't derived from a formula. They were picked by comparing renders on a real device and in the editor, and will likely keep shifting as the UI evolves.
- **Treat `0.5` and `0.2` as examples**, not recommended values.

## The theme font-size trap

One gotcha that's easy to lose an hour to: setting [`Theme.default_font_size`](https://docs.godotengine.org/en/stable/classes/class_theme.html) is **not enough** to change font size globally.

- Godot's built-in theme sets font size per control type, and those per-type values win over `default_font_size`.
- So a fresh `Theme` needs each control type covered explicitly.
- The theme also has to be assigned to the tree root before it takes effect.

`FONT_SIZE` and the list of control types below are examples; cover whichever controls your UI uses:

```gdscript
var theme := Theme.new()
theme.default_font_size = FONT_SIZE
for type in ["Label", "Button", "RichTextLabel", "LineEdit", "CheckButton", "CheckBox", "OptionButton"]:
    theme.set_font_size("font_size", type, FONT_SIZE)

# ... assign fonts, styleboxes, etc. ...

get_tree().root.theme = theme
```

- Skip the per-type loop and `default_font_size` quietly does nothing for `Label` or `Button`.
- This runs once at startup, before platform-specific window setup, so the right theme is in place before the first frame renders.

## Keeping the binary small: subsetting on a schedule, not automatically

Shipping full CJK font files would bloat the app with glyphs the game never uses, so a build script subsets the original fonts down to the characters that are actually referenced.

- **Character sources:** translation files, string literals in scripts and scenes (comments excluded), and printable ASCII as a safety baseline.
- **Per-font mapping:** each font is built only from the languages it serves. Requesting a character a font doesn't have is silently ignored, so overlapping requests are harmless.
- **Manual, not automatic:** the script is rerun whenever translation text, fonts, or hardcoded strings change, with a dry run first to confirm new characters are covered.

The details are in [Subsetting Fonts for a Multilingual Godot Game]({% post_url 2026-09-14-subsetting-fonts-for-a-multilingual-godot-game %}).

## Takeaways

- `FontVariation.fallbacks` handles missing-glyph fallback per character, not per string — mixed-script text just works.
- A fallback font swap can still need a metrics-level fix (ascent/descent/lineGap, glyph offset) if the two typefaces don't share a baseline convention.
- Weight hierarchy doesn't require more font files — `variation_embolden` on a `FontVariation` gets you there, as long as every script's fallback gets its own matching value.
- `Theme.default_font_size` alone won't touch per-type overrides; set `font_size` per control type explicitly.
- Subsetting is a build-time discipline, not a runtime concern — the tooling matters as much as the font choice itself.
