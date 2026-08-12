# prompt-v1-archive

Superseded prompts and the transcriptions made with them, kept so the model
comparison in `../model-comparison.md` can be re-checked and so the effect of
each prompt revision stays visible.

Nothing here is read by any pipeline stage. Filenames deliberately use
`-gemini-…` rather than `_gemini-…`: `utils.models._TXT_RX` and
`extract_entries._detect_aligned_slug` both parse a model slug out of any
`*_gemini-….txt` name, so an underscore here would let these archived files
masquerade as live OCR output and skew model auto-detection.

## Prompt versions

| Version | File | What it was |
|---|---|---|
| v0 | `ocr_prompt.v0-generated.md`, `ner_prompt.v0-generated.md` | Straight output of `--generate-prompts`. Never used for a scored run. |
| v1 | `ocr_prompt.v1.md`, `ner_prompt.v1.md` | Hand-rewritten after reading the pages. Used for every transcription in this folder. |
| v2 | `../ocr_prompt.md`, `../ner_prompt.md` | Current. v1 plus the fixes described below. |

### What v0 got wrong

Worth keeping because the errors are instructive — they are what a
prompt generated from sample images, without anyone reading the key, looks like.

- **Wrong column count.** "The content is organized into a two-column layout."
  The pages have three columns. A wrong column count corrupts reading order,
  and reading order is what binds entries to their headings.
- **A fabricated example with the wrong mapping.** `(e.g., "Marked thus * are
  Publishers")`. The page's key says `*` is **Printers** and `†` is
  **Publishers**. An invented example carrying the opposite of the truth is
  worse than no example.
- **Symbols listed as decoration, not data.** `||` for `‖`, "lowercase
  superscript letters (a, b, c)" where the page uses `a`–`e`, and no mention
  that marks stack, that their order varies, or that a column rule sits exactly
  where a `‖` would.
- **NER side:** no symbol→role key at all, `markers` described only as
  indicating "roles like Publisher or Printer", and heading fields named
  `trade_category`/`sub_category`, which sit outside `extract_entries`'
  `_HEADING_FIELDS` and so silently opt out of the bounding-box safeguard.

### v1 → v2

- **OCR:** added a rule that `a`–`e` are the only letters that can be markers.
  Every model was reading the dagger in `e†Carpenter N.` as a lowercase `t`.
  Also added a `†` vs `‖` disambiguation note.
- **NER:** added an explicit in-scope list of the Booksellers sub-headings and
  a "do not skip ahead to `Booksellers in General`" instruction. Under v1 the
  extractor began at the first entry of `Booksellers in General` and silently
  dropped the seven specialist sub-sections above it — about 50 entries.

Note that the OCR change measurably **hurt** `gemini-3.1-flash-lite`
(17/24 → 15/24 on the gold set) while leaving the two larger models unchanged.
See "Two operational findings" in `../model-comparison.md`.

## Transcriptions

Twelve files: 3 models × 4 pages, all produced with prompt **v1**.

    {page}_{image-id}-{model}.txt

`gemini-3.1-pro-preview` files here were run with the API's default (uncapped)
thinking budget, before `utils.gemini.thinking_config_for` capped pro at 4096.
They are *not* truncated — the truncation that prompted the cap happened on a
later v2 run of page 0001, which is not kept here.

To diff a model across prompt versions:

    diff "prompt-v1-archive/0001_p16445coll4:27074-gemini-3.1-pro-preview.txt" \
         "0001_p16445coll4:27074_gemini-3.1-pro-preview.txt"
