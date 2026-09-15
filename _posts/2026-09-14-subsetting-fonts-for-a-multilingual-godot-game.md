---
title: "Subsetting Fonts for a Multilingual Godot Game: 5 MB Down to 0.56 MB"
date: 2026-09-14
categories: [godot]
tags: [godot, i18n, fonts, optimization]
description: "How Tap Tap Picture Book cut its bundled fonts by roughly 89% by keeping only the characters the game actually uses — and the missing-glyph bug that taught us to pair the tool with a workflow rule."
image: /icon.png
---

*Tap Tap Picture Book* ships in seven languages — German, English, Spanish, French, Japanese, Korean, and Brazilian Portuguese — and the fonts that make that possible were quietly the heaviest assets in the build. Japanese and Korean fonts carry thousands to tens of thousands of glyphs, and on mobile every megabyte lands directly in the download size.

This post covers:

- how the game subsets its fonts down to only the characters it really uses,
- what that saved, and
- the one bug that shaped how the tooling is used.

For how these fonts are wired together at runtime (fallback chains, baseline fixes, emboldening), see the earlier post, [CJK Font Fallback in Godot 4: One Theme, Three Scripts]({% post_url 2026-09-09-cjk-font-fallback-in-godot-4 %}).

## The fonts and the problem

The game bundles three fonts:

- **[M PLUS Rounded 1c](https://fonts.google.com/specimen/M+PLUS+Rounded+1c) Medium** — the base UI font, covering Latin and Japanese.
- **BM JUA** — the Korean fallback, needed because M PLUS has zero Hangul glyphs.
- **[Luckiest Guy](https://fonts.google.com/specimen/Luckiest+Guy)** — a display font used only for combo counters and the countdown.

The two CJK fonts are several megabytes each, while the game's entire script — every menu, tutorial line, and button label across seven languages — touches only a tiny fraction of those glyphs. Everything else is dead weight in the package.

## The approach: keep only what's referenced

The idea is simple: collect every character that can actually appear on screen, then strip everything else out of the font files with [fontTools](https://github.com/fonttools/fonttools)' [subsetter](https://fonttools.readthedocs.io/en/latest/subset/index.html).

To keep this repeatable:

- **Originals:** kept untouched alongside the generated subsets, as the input for regeneration.
- **Subsets:** the only files the game actually loads.
- **Export:** Godot's [export exclude filter](https://docs.godotengine.org/en/stable/tutorials/export/exporting_projects.html) keeps the originals out of the shipped build, so they cost nothing at runtime.

## Three sources of characters

A small Python script gathers characters from three places:

| Source | What it catches | Filtered per font? |
|---|---|---|
| Translation files | Nearly all on-screen text | Yes, by language |
| String literals in code and scenes | Untranslated symbols like `★` or `×` | No |
| Printable ASCII | Digits and basic punctuation | No |

**1. Translation files.** This is the bulk of the on-screen text. Each font only pulls from the languages it's responsible for, via a font-to-language map. The keys and language codes below are placeholders modeled on this game — swap in your own font files and languages:

```python
FONT_LOCALE_MAP = {
    BASE_FONT: ["de", "en", "es", "fr", "ja", "ko", "pt_BR"],  # M PLUS Rounded 1c
    KOREAN_FALLBACK_FONT: ["ko"],                               # BM JUA
    # Combo counter and countdown only. Fixed, untranslated text
    # ("%d COMBO", "START!"), so the ASCII baseline is enough.
    DISPLAY_FONT: [],                                           # Luckiest Guy
}
```

BM JUA exists solely to render Korean, so it's built from Korean strings alone. Luckiest Guy only ever displays text that's identical in every language, so its list is empty.

**2. String literals in code and scenes.** Not every visible character goes through translation — symbols like `★` or `×` are sometimes written directly in scripts and scene files.

- **Literals only:** the script scans those files, but only inside quoted `"..."` string literals.
- **Why it matters:** if code comments are written in a CJK language, scanning whole files would drag hundreds of unused glyphs into the subset. Matching only literals excludes comments naturally, with no comment parser needed.
- **Escapes:** `\uXXXX` escapes are decoded back to real characters before counting.

Which files to pass in as `files` depends on your project layout:

```python
_STRING_LITERAL_RE = re.compile(r'"((?:[^"\\]|\\.)*)"')
_UNICODE_ESCAPE_RE = re.compile(r"\\u([0-9a-fA-F]{4})")


def _unescape(raw: str) -> str:
    text = _UNICODE_ESCAPE_RE.sub(lambda m: chr(int(m.group(1), 16)), raw)
    return text.replace('\\"', '"').replace("\\\\", "\\")


def chars_from_literals(files) -> set[str]:
    chars: set[str] = set()
    for path in files:
        text = path.read_text(encoding="utf-8")
        for match in _STRING_LITERAL_RE.finditer(text):
            chars.update(_unescape(match.group(1)))
    return chars
```

The regex only matches double-quoted strings. That's fine when all display text uses double quotes, but it's worth checking if you adapt the idea.

**3. Printable ASCII (`0x20`–`0x7E`).** A minimum safety set, so digits and basic punctuation never go missing no matter what the other two sources return.

Only the translation source is filtered per font. The literal characters and the ASCII baseline are added to every font:

```python
ASCII_BASELINE = set(range(0x20, 0x7F))


def codepoints_for_font(locales, literal_chars, chars_from_locale) -> set[int]:
    chars = set(literal_chars)
    for locale in locales:
        chars |= chars_from_locale(locale)
    return {ord(c) for c in chars} | ASCII_BASELINE
```

That works because of one property of the fontTools subsetter: code points a font doesn't contain are silently ignored. M PLUS just skips the Hangul, and BM JUA just skips the kana.

The subset step itself:

- calls `fontTools.subset.main()` directly with an argument list, rather than launching the `pyftsubset` CLI as a subprocess, and
- passes `--layout-features=*` to keep all OpenType layout features, so kerning and other shaping data survive.

```python
from fontTools import subset


def run_subset(input_path, output_path, codepoints) -> None:
    unicodes_arg = ",".join(f"U+{cp:04X}" for cp in sorted(codepoints))
    subset.main([
        str(input_path),
        f"--unicodes={unicodes_arg}",
        f"--output-file={output_path}",
        "--layout-features=*",
    ])
```

## Results

These numbers are for this game's text and fonts. Your savings will depend on how much text you ship and which fonts you use.

| Font | Original | Subset | Reduction |
|---|---|---|---|
| M PLUS Rounded 1c Medium | 3.43 MB | 249 KB | 92.8% |
| BM JUA | 1.52 MB | 283 KB | 81.4% |
| Luckiest Guy | 58 KB | 25 KB | 56.8% |
| **Total** | **5.01 MB** | **557 KB** | **~89%** |

- **First version:** when subsetting was introduced, the game used M PLUS's Black weight, which went from 3.53 MB to 212 KB (94.0% smaller).
- **Now:** a single Medium font, with heavier weights produced at runtime through `FontVariation.variation_embolden` — so one font file covers both the regular and bold looks.

## The bug: changed a sentence, lost the glyphs

Subsetting has one sharp edge, and we hit it.

- **What happened:** after rewording a Korean tutorial line, some of the new text didn't render on device.
- **Why:** the new wording used Hangul syllables that had never appeared in the translations before, so they weren't in the subset font.
- **Why it's easy to miss:** nothing fails at build time and the text can look fine during development. The problem only shows up when someone reads that screen on a real device.

The fix wasn't just regenerating the fonts. It was making the regeneration hard to forget:

- **A dry-run mode.** It builds the subset in a temporary directory without overwriting anything and reports the resulting sizes, so checking is cheap and safe. This is an example excerpt: `input_path`, `output_path`, and `report` stand in for your own paths and logging:

  ```python
  if args.dry_run:
      with tempfile.TemporaryDirectory() as tmp_dir:
          tmp_output = Path(tmp_dir) / font_name
          run_subset(input_path, tmp_output, codepoints)
          report(font_name, len(codepoints), before, tmp_output.stat().st_size)
  else:
      run_subset(input_path, output_path, codepoints)
      report(font_name, len(codepoints), before, output_path.stat().st_size)
  ```

- **A workflow rule.** Any change to translation text gets a dry-run check before it's committed. If the subset is out of date, the fonts are regenerated and committed *together with* the text change.

The lesson: subsetting is a system that breaks whenever someone forgets to add characters. The tool alone isn't enough. It needs a rule about when to run it.

## When to rerun

After regenerating, let the Godot editor reimport the updated font files. Rerun the subset whenever:

- translation text changes,
- an original font is added or replaced, or
- a hardcoded string with new characters is added to code or scenes.

## What's still open

- **Detection is manual.** Nothing flags a stale subset automatically yet. A CI check or pre-commit hook that runs the dry run and fails on missing characters would close that gap.
- **Free-form input breaks the model.** If the game ever adds player text input, like entering a name, the set of possible characters is no longer known ahead of time, and this approach won't work for those fields.
- **Runtime-built strings need care.** Text assembled with `%` formatting or concatenation is only safe if the values being inserted are also covered by the subset.

## Takeaways

- For a game with a fixed script, CJK fonts are mostly unused glyphs. Subsetting to referenced characters cut this game's fonts by roughly 89%.
- Keep untouched originals next to the generated subsets, and exclude the originals from export.
- Scan only string literals in code, not whole files. It's a cheap way to keep comments (in any language) out of the character set.
- The fontTools subsetter ignores missing code points, so shared character sources can be passed to every font safely.
- Pair the tool with a workflow rule. A subset that isn't regenerated when text changes fails silently, and usually only on a real device.
