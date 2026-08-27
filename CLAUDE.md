# Pharmanomicon — project notes for Claude Code

Single-file HTML app that generates fake prescription drug names in the style of US direct-to-consumer TV ads, plus a made-up indication, side effects, and safety copy. Built as a joke with real methodology underneath.

## Files

- `index.html` — the entire app (formerly `pharmanomicon.html`; renamed so Cloudflare Pages serves it at the site root). No build step, no dependencies except Google Fonts (Sora, IBM Plex Sans, IBM Plex Mono). Deploys as-is to Cloudflare Pages, Netlify, or GitHub Pages.
- `CLAUDE.md` — this file.

Keep it single-file unless the owner says otherwise. If you split it, keep a build-free option (plain `<script src>` includes, no bundler).

## How the generator works

All logic is in the one `<script>` block, in this order:

1. **Corpus** — `DEFAULT_CORPUS`: ~216 real brand names, weighted toward current heavy TV advertisers (Skyrizi, Rinvoq, Vraylar, Wegovy, Ozempic, Zepbound, Mounjaro, Dupixent, Jardiance, Rexulti, Cosentyx, Tremfya, Nurtec, Qulipta, Biktarvy, etc.) plus historic heavy advertisers (Humira, Lyrica, Chantix, Viagra, Celebrex). Editable in-app; `parseCorpus()` dedupes and normalizes to Capitalized.
2. **Training** — `train(names)` builds:
   - `trans`: order-2 Markov transitions over lowercase letters, with `^^` start padding and `$` end token.
   - `freq`, `inits`, `bigrams`, `lengths`, `set` (for exact-match rejection).
3. **Name generation** — `genWord(minL, maxL, temp, boost)`:
   - Walks the chain. `temp` flattens/sharpens the distribution (`weight^(1/temp)`). `boost` multiplies weights for X Z Q V Y K J.
   - `$` is suppressed until `minL` is reached.
   - Accepts only if: length in range, the final two letters can end a real name (`trans[last2]['$']` exists), `pronounceable()` passes (has a vowel, no triple letters, no 4-consonant or 3-vowel runs, vowel ratio 0.25–0.65), and it is not an exact corpus match. Retries up to 60 times.
4. **Generic name** — `genGeneric()`: short Markov stem that avoids X/Z/H/J/K/W initials (a real USAN rule) + an INN-style suffix from `GENERIC_SUFFIX` (-mab, -tinib, -gliflozin, ...).
5. **Copy** — `makeDrug()` assembles brand, generic, delivery form, condition, qualifier, tagline, side effects (mild + serious), and "tell your doctor" items from the content pools at the top of the script.
6. **Scorecard** — `renderVerdict()`: 
   - Unusualness = mean English letter frequency per letter (Norvig Google Books frequencies in `ENGLISH`), the metric from Carico et al. 2022. Lower = more pharma.
   - Rare-letter count, length, nearest real drug by Levenshtein distance (`lev()`).
7. **Stats panel** — `renderStats()`: letter-frequency bars vs English, over/under-represented letters, top initials, top bigrams, length and syllable summaries.

## Research basis (so nobody re-derives it)

- Carico et al., *Exploratory Research in Clinical and Social Pharmacy*, 2022, DOI 10.1016/j.rcsop.2022.100146. All US brand names approved 1985–2020. Findings: A (11.96%), V (3.08%), X (2.31%), Z (1.91%) overrepresented; E, H, T, S underrepresented; C and N declining over time; V, Y, Z rising.
- Older industry stat: Q ~3×, X ~16×, Z ~18× English frequency; W nearly absent.
- Generic (USAN/INN) names: no H, J, K, W initials; moratorium on X and Z initials for sound-alike reasons; suffix encodes drug class.
- English baseline: Norvig letter frequencies from Google Books.

## Conventions

- Vanilla JS, no framework. Keep functions small and named; the owner iterates fast and reads the source.
- CSS variables at `:root` for palette and type. Palette: navy `#0D2B4E`, teal `#11A5A0`, saffron `#F5B335`, clinical `#F4F8FA`. Do not drift toward generic cream/terracotta or black/acid-green.
- Layout is deliberately single-column, max-width 520px, phone-first. Every grid uses `minmax(0,1fr)`; containers have `overflow:hidden`. This was the fix for horizontal overflow on mobile; do not reintroduce `1fr` grids, `padding-left:100%` marquee tricks, or `100vw` widths.
- Respect `prefers-reduced-motion` (ticker stops).
- Copy voice: dry, deadpan, parody of DTC ad language. Absurd items sit next to plausible ones; that contrast is the joke. Do not make everything wacky.
- Spacebar triggers Generate. Keep that.

## Known issues / open questions

- The in-app Claude artifact preview on mobile may render at a fixed desktop width and scale down; verify layout in a real mobile browser or on Netlify, not the preview.
- Google Fonts requires network; fallbacks are system fonts. If offline use matters, self-host or drop to system stacks.
- `pronounceable()` is heuristic. Some outputs still land on awkward clusters (e.g. "vzar", "sphn"). A syllable-onset/coda whitelist derived from the corpus would be stricter.
- Nearest-drug check is edit distance only; no phonetic (Soundex/Metaphone) similarity, which is what the FDA actually screens for.
- Condition/side-effect pools are flat arrays. No weighting, no exclusion of contradictory pairs.
- History is in-memory only; refresh clears it. `localStorage` is fine once hosted (it was avoided only because Claude artifacts block it).

## Backlog ideas (owner has not prioritized these)

- Phonetic similarity score (Metaphone) alongside edit distance.
- "Name a drug for X": user types a condition, generator produces name + full ad copy.
- Per-letter over/under toggles so the user can force, e.g., a Q.
- Export the ad card as PNG (html2canvas or an SVG rebuild).
- Seeded RNG so a given seed reproduces the same drug (shareable URLs: `?seed=...`).
- Fake commercial script generator: 30-second spot with meadow, dog, and side-effect voiceover.
- Corpus tagging by therapeutic class, then class-conditioned generation (GLP-1s sound different from oncology drugs).
- Persist edited corpus and history with `localStorage` once hosted.
