# OCR model comparison — 1841 London Trades' Directory, Booksellers

Three Gemini models, one identical OCR prompt, four pages, scored on how faithfully each recovers the reference marks printed in front of entries.

Those marks are what makes this page hard. In this directory they are the only record of the secondary trades a firm carried on — `†` Publishers, `*` Printers, `§` Librarians, `‖` General Stationers, `‡` Fancy Stationers, `¶` Printsellers, and superscript `a`–`e` for Newsvenders / Account-book makers / Bookbinders / Foreign / Juvenile. They are small, they stack (`‖†‡¶Ackermann & Co. 96 Strand`), and a dropped one is indistinguishable from an entry that never had one. The body text is comparatively easy; every model gets it substantially right.

## Setup

| | |
|---|---|
| Source | *Post Office London Directory, 1841*, Part 1 — Trades' Directory, printed pages 646–649 |
| Manifest | Leicester CONTENTdm `p16445coll4:8844`, canvases 674–677 of 890 |
| Images | 4550 × 6275 px native JPEG (CONTENTdm ignores `--width` and serves full size) |
| Layout | 3 columns, ~85 lines per column, ~257 lines per page |
| Prompt | `ocr_prompt.md` in this directory — **identical for all three runs** |
| Command | `main.py output/london-1841-booksellers/ --gemini-ocr --ocr-model <MODEL> --no-flex` |
| Date | 2026-08-11 |

Two model ids are not served under the names you might expect: `gemini-3.1-pro` is **`gemini-3.1-pro-preview`** and `gemini-3-flash` is **`gemini-3-flash-preview`**. Check `client.models.list()` before pinning either.

No model truncated. Line counts ran 254–260 per page against ~257 printed.

## Headline result

Scored against a 24-entry gold set transcribed by hand from full-resolution IIIF crops of the source pages:

| | flash-lite | 3-flash-preview | 3.1-pro-preview |
|---|---|---|---|
| Exact marker string | 15 / 24 | **23 / 24** | **23 / 24** |
| Symbol precision | 86% | **97%** | **97%** |
| Symbol recall | 75% | **97%** | **97%** |
| Entry lines lost entirely | 1 | **0** | **0** |
| Deviation from 2-of-3 majority (945 lines) | 7.9% | **1.7%** | 4.3% |

`3-flash-preview` and `3.1-pro-preview` are indistinguishable on the gold set — 23 of 24 each, failing on *different* entries. `flash-lite` is materially worse and fails in the direction that is hardest to detect downstream.

## Recall, not precision, is the axis

All three are reasonably careful about what they claim. The gap is in what they miss, and it is concentrated in one glyph.

| Symbol | flash-lite | 3-flash-preview | 3.1-pro-preview |
|---|---|---|---|
| `†` | 287 | 234 | 232 |
| `‖` | **141** | 211 | **222** |
| `§` | 75 | 69 | 73 |
| `¶` | 20 | 24 | **39** |
| `a` | 37 | 52 | **65** |
| `*` | 50 | 46 | 46 |
| `‡` | 9 | 5 | 10 |
| `b` / `c` / `d` / `e` | 18 / 15 / 1 / 1 | 21 / 15 / 3 / 4 | 22 / 19 / 2 / 2 |
| **Total marks** | **654** | 684 | **732** |

`flash-lite` finds 141 `‖` against pro's 222 — a 36% shortfall on **General Stationers**, one of the most common attributes in the section. It simultaneously reports the *most* `†` (287 vs 232), which is the signature of substitution rather than clean omission: it is reading a `‖` as a `†`.

## Which direction each model errs

Across the 945 lines where all three align, counting only cases where the other two agree with each other:

| | fails to report a mark the other two both see | reports a mark neither other sees |
|---|---|---|
| flash-lite | **14** | 4 |
| 3-flash-preview | 1 | **0** |
| 3.1-pro-preview | **0** | 18 |

Pro's 18 solo detections look like over-reporting until you check them. All 18 are `‖`, and adjudicating a cluster against the page image settles it:

```
‖Collins Miss Julia, 29 Orchard street     lite: —  3-flash: —  pro: ‖   truth: ‖
‖Collins Samuel, 84 St. John street rd     lite: —  3-flash: —  pro: ‖   truth: ‖
‖Collyer E. & A. 5 Prospect pl. Chelsea    lite: —  3-flash: —  pro: ‖   truth: ‖
 Corbett Thomas, 218 Tottenham ct. rd      lite: —  3-flash: —  pro: —   truth: none
```

`Corbett Thomas` directly beneath them carries no mark, so this is not the column rule being misread. Pro is right and the other two are both missing real marks. Its extra 48 marks over `3-flash-preview` are recall, not noise — so "deviation from majority" should be read as a divergence measure, not an error rate: on this material the majority is sometimes wrong in the same direction.

Six further cases were adjudicated where `flash-lite` reported nothing and the other two reported `‖` (`Egley`, `Elder`, `Elkins Wm. Henry`, `Elt`, `Cowie`, plus the dropped `†` in `Ackermann`). All six went against `flash-lite`.

Pairwise agreement on marker strings across the 945 aligned lines:

| | agreement |
|---|---|
| 3-flash-preview vs 3.1-pro-preview | 90.3% |
| flash-lite vs 3-flash-preview | 86.7% |
| flash-lite vs 3.1-pro-preview | 84.0% |

Unanimous on 82.3%.

## Two operational findings

**Pro needs a thinking cap or it truncates.** Left on the API's default dynamic thinking budget, `gemini-3.1-pro-preview` spent **62,915 thinking tokens** against 2,617 output tokens on page 1 and returned `MAX_TOKENS` at 210 of 257 lines — a silently truncated page. Capping at `thinking_budget=4096` used 3,965 thinking tokens and completed; 8,192 produced byte-identical output, so 4,096 is not binding. The cap also takes pro from roughly **$0.79/page to $0.085/page**, since thinking bills as output. This is now the pipeline default for pro-tier models (`utils.gemini.thinking_config_for`), and truncation is no longer silent — `run_gemini_ocr` warns on `MAX_TOKENS`.

**Prompt elaboration helped nothing and hurt the small model.** An earlier version of `ocr_prompt.md` lacked the explicit `†`-is-not-a-`t` rule and the `†`/`‖` disambiguation guidance. Adding them left `3-flash-preview` and `pro` unchanged at 23/24, but moved `flash-lite` from 17/24 **down** to 15/24 — and changed the character of its errors. Under the shorter prompt it mostly omitted marks; under the longer one it also began substituting them (`‖‡b`→`‖†b`, `‖¶§`→`‖*§`, `‖e`→`e‡`). More instruction about which glyphs are confusable appears to have given the smaller model more ways to guess wrong. Don't assume prompt detail scales down.

## Recommendation

**`gemini-3-flash-preview`** is the default choice: it ties pro on the gold set, sits closest to consensus, and costs a sixth as much.

**`gemini-3.1-pro-preview`** is worth it where the rarer glyphs matter — it is the only model with no missed marks against consensus, and it leads clearly on `¶` (39 vs 24) and superscript `a` (65 vs 52). At $0.085/page with the thinking cap this is no longer an expensive choice.

**Do not use `gemini-3.1-flash-lite`** — the pipeline default — on any source where prefix marks carry data. At 75% symbol recall roughly one attribute in four is lost, mostly `‖`, with nothing downstream to flag it.

For maximum recall, run two models and diff the marker strings: the ~100 disagreements localize nearly all the real errors into a set small enough to adjudicate by hand against the page images.

## Reproducing

```bash
for M in gemini-3.1-flash-lite gemini-3-flash-preview gemini-3.1-pro-preview; do
  python main.py output/london-1841-booksellers/ --slug london-1841-booksellers \
      --gemini-ocr --ocr-model "$M" --no-flex
done
```

Each run writes `{page}_{image-id}_{model}.txt` alongside the images, so all three transcriptions coexist and can be diffed directly. Note that `run_gemini_ocr` skips pages whose `.txt` already exists and has no `--force` flag — delete the files to re-run a model.

## Caveats

- The gold set is 24 entries from two of the four pages, chosen because they were legible in crops pulled for other checks — not a random sample. It separates `flash-lite` from the other two comfortably; it does not separate `3-flash-preview` from `3.1-pro-preview`.
- One run per model, so run-to-run variance is unmeasured. An earlier pro run scored 24/24 on this same set; treat single-entry differences at the top as noise.
- Marker strings were compared after aligning lines by normalised body text. Lines a model garbled badly enough to break alignment (~3%) are excluded from the 945.
- No timing was recorded. Costs are derived from the published per-token rates in `docs/costs.md` plus measured token counts, not from billing.
- The `*_gemini-arbitrated.txt` files in this directory are from an earlier experiment merging two superseded runs. They are stale and safe to delete.
