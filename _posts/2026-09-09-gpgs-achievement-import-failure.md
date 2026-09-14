---
title: "Debugging a Silent Google Play Games Services Achievement Import Failure"
date: 2026-09-09
categories: [android]
tags: [android, google-play-games, debugging]
description: "A GPGS achievement ZIP passed validation but silently failed to save — the cause turned out to be an undocumented bulkCreate validation rule."
image: /assets/images/posts/gpgs-achievement-import-failure/achievements-list.png
---

While adding Google Play Games Services (GPGS) achievements to *Tap Tap Picture Book* (my Godot game), I prepared a bulk-import ZIP for Play Console's "Import achievements" feature — 110 achievements across 7 languages, following the [official CSV format](https://developer.android.com/games/pgs/integrate-achievements) (`AchievementsMetadata.csv`, `AchievementsLocalizations.csv`, `AchievementsIconsMappings.csv`, plus icon assets). What followed was a two-part debugging saga: first a locale-code mixup, then a much stranger silent save failure that took a full binary-search investigation to crack.

## Part 1: "Unsupported language/region"

The first upload failed instantly, with every single achievement flagged:

> Language/region not supported — use a language/region supported by the game.

My first instinct was to check Google's [general supported languages list](https://support.google.com/googleplay/android-developer/table/4419860). That table shows some languages with a bare code (`ko`, `ja`, `de`) and others with a region suffix (`es-419`, `pt-BR`, `fr-FR`) — I assumed the pattern was "single-variant languages use bare codes," rewrote my locale codes accordingly (`ko-KR` → `ko`, etc.), and re-uploaded.

Same error, unchanged.

**The actual fix**: the error message's "supported by the game" doesn't refer to Google's general language list at all — it refers to the specific set of languages *this project* has registered under **Play Console → Play Games Services → Configuration → Edit properties → Manage translations**. Whatever locale codes are configured there (checked by literally opening that screen) are the only ones the CSV importer will accept, regardless of what any general documentation says. My original codes (`ko-KR`, `ja-JP`, `de-DE`, plus `es-419`, `pt-BR`, `fr-FR` — these are just the languages my project happens to register; yours will depend on your own configuration) turned out to be exactly right once I matched them against that screen — my "fix" had actually broken things further.

**Lesson**: when an error says "supported by X," check X's actual live configuration before consulting general documentation. Generic references can be true in general and still not apply to your specific setup.

## Part 2: Import succeeds, but "Save" silently fails

With the locale issue fixed, uploading the ZIP now showed a green checkmark — "4 files imported successfully."

<figure>
  <img src="/assets/images/posts/gpgs-achievement-import-failure/import-success.png" alt="Play Console import screen showing the uploaded ZIP with a green checkmark and '4 files imported successfully'." />
  <figcaption>The Play Console UI is in Japanese here (my account's console language) — the screenshots below are too, but the text isn't essential to follow along.</figcaption>
</figure>

But clicking **Save** produced only:

> Changes could not be saved.

<figure>
  <img src="/assets/images/posts/gpgs-achievement-import-failure/save-failed.png" alt="Play Console achievement import page with a red-boxed error banner reading 'changes could not be saved' in the bottom-left corner." />
  <figcaption>No detail, no expandable error — just this banner.</figcaption>
</figure>

Nothing actionable. This is where things got interesting.

### Ruling things out one at a time

I approached this as a binary search, changing exactly one variable per test:

1. **Scale**: Full 110-achievement ZIP → fails. Reduced to a 2-achievement ZIP → same failure. Ruled out: data volume, and any single bad row among the other 108.
2. **Icon transparency**: The placeholder icon (reused from the app icon) had an alpha channel. Flattened it to an opaque RGB PNG on a white background, retried the 2-achievement ZIP → same failure. Ruled out: icon transparency.
3. **Re-verify the spec, verbatim**: Re-fetched the official CSV format documentation and diffed every field against my files — column order, no header row, `Name` uniqueness, `Points` (multiples of 5, 5–200), `Steps Needed` (max 10,000), ZIP constraints (file count, size, no subdirectories). Everything matched exactly. Ruled out: the CSV format itself.
4. **Open the browser console**: This is where real signal appeared. The failing call was `POST .../achievements:bulkCreate` returning HTTP 400.

   <figure>
     <img src="/assets/images/posts/gpgs-achievement-import-failure/devtools-console.png" alt="Chrome DevTools Console tab showing a 400 Bad Request error on a POST to the achievements:bulkCreate endpoint." />
     <figcaption>DevTools Console: the failing bulkCreate call, 400 Bad Request.</figcaption>
   </figure>

   The response body, decoded from its protobuf-over-JSON encoding, was:

   ```json
   {"1":3,"2":"Request contains an invalid argument."}
   ```

   <figure>
     <img src="/assets/images/posts/gpgs-achievement-import-failure/devtools-network-response.png" alt="Chrome DevTools Network tab Response pane showing the raw JSON body: {&quot;1&quot;:3,&quot;2&quot;:&quot;Request contains an invalid argument.&quot;}" />
     <figcaption>DevTools Network tab: the raw response body behind that error.</figcaption>
   </figure>

   Field `1: 3` is the gRPC status code for `INVALID_ARGUMENT`. Field `2` is a human string — but a completely generic one. Still no field-level detail.
5. **Strip down to the essentials**: Both `AchievementsLocalizations.csv` and `AchievementsIconsMappings.csv` are optional per spec. I built a ZIP with *only* `AchievementsMetadata.csv`, one achievement, no icon at all (the row below is an example test row):

   ```
   First Step,Clear 1 stage,True,1,Revealed,5,10
   ```

   Still failed, identically. This ruled out localization and icon files entirely — the bug had to be in `AchievementsMetadata.csv` itself, or somewhere outside the file altogether.
6. **Sanity check: can this project create achievements at all?** I tried creating a single achievement manually through the Play Console UI (no CSV involved). The save button threw a generic, unrelated-looking "An unexpected error occurred" toast — but refreshing the list showed the achievement had actually been created. This confirmed the project itself wasn't blocked; the problem was specific to the bulk-import (`bulkCreate`) code path.
7. **The actual culprit**: Comparing my one remaining test row against what a "normal" quick-created achievement probably looks like, I suspected the combination of `Incremental value = True` with `Steps Needed = 1`. I tested:
   - `Incremental=False` (non-incremental), same row otherwise → succeeded.
   - `Incremental=True`, `Steps Needed=50` (not 1) → succeeded.

   That pinned it down precisely: `bulkCreate` silently rejects any achievement where `Incremental value` is `True` and `Steps Needed` is `1`. This constraint appears nowhere in the official documentation. My best guess is that a single-step "incremental" achievement is functionally identical to a non-incremental one, so the backend rejects it as a degenerate case — but the API gives zero indication of this, and the UI's generic "invalid argument" error makes it effectively undiscoverable without decoding the raw network request.

### The fix

Of my 110 achievements, 23 were "do this once" achievements (first stage clear, first world clear, first ad watched, etc.) that I'd modeled as incremental with a target of 1 step — which is precisely the pattern that triggers this bug. Converting all 23 to non-incremental achievements (`Incremental=False`, blank `Steps Needed`) fixed the import completely, and arguably improved the design too: a "do it once" achievement doesn't need step tracking in the first place, so unlocking it directly via `unlock_achievement()` (instead of `set_achievement_steps()`) is both correct behavior and a workaround for an undocumented server-side validation quirk.

<figure>
  <img src="/assets/images/posts/gpgs-achievement-import-failure/achievements-list.png" alt="Play Console achievements list showing all imported achievements (First Step, Breaker Novice, Breaker Veteran, and more) with unpublished status, ready to review and publish." />
  <figcaption>All 110 achievements imported successfully after the fix.</figcaption>
</figure>

## Takeaways

- **Generic upload success ≠ generic save success.** Play Console's CSV importer validates *file format* on import, but a separate, much less transparent validation pass runs when you actually save — and it can reject perfectly spec-compliant data for reasons the UI won't tell you.
- **When an error gives you nothing, open DevTools.** The Console tab told me *which* API call was failing; the Network tab's response body told me *why* (in gRPC-status-code form, at least). Neither would have been discoverable from the Play Console UI alone.
- **Binary search beats guessing.** Cutting the achievement count, then the icon, then the auxiliary CSVs, then the specific field combination, each ruled out one axis of the problem — and each test took minutes, versus hours of speculating.
- **"Supported by the game" is not the same as "supported by Play."** Locale/region support in particular is scoped per-project, not globally — always check the live configuration screen, not the general reference table.
