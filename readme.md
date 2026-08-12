# Process notes

A series of OCR and NER extraction tests on four pages from the *Post Office London Directory, 1841* (Part 1: Trades' Directory), printed pages 646–649, covering the Booksellers section. Canvases 674–677 of an 890-page volume held by the University of Leicester in CONTENTdm.

**Source:** [Post Office London Directory, 1841 on CONTENTdm](https://leicester.contentdm.oclc.org/digital/collection/p16445coll4/id/8844) · [full IIIF manifest](https://leicester.contentdm.oclc.org/iiif/info/p16445coll4/8844/manifest.json) (890 canvases)

The four pages individually: [p. 646](https://leicester.contentdm.oclc.org/digital/collection/p16445coll4/id/27074) · [p. 647](https://leicester.contentdm.oclc.org/digital/collection/p16445coll4/id/27075) · [p. 648](https://leicester.contentdm.oclc.org/digital/collection/p16445coll4/id/27076) · [p. 649](https://leicester.contentdm.oclc.org/digital/collection/p16445coll4/id/27077)

## Extraction notes

### OCR
- I put all the files and outputs from the whole process in this repository -- a full machine-generated description of everything is in the [File Structure](#file-structure) section down below
- I used a few different gemini models to try prompted OCR extraction, since there was only 4 pages to work with. The full evaluation and writeup from Claude is at [`model-comparison.md`](model-comparison.md)
- Biggest takeaway: Gemini, especially Gemini 3 Pro, seems to handle the character extraction with no issues given the right prompting. I didn't do a full-scale evaluation of the OCR compared to ground truth, so perhaps I missed some character error rate or word error rate in there that wasn't immediately obvious, but at a glance it seems to handle the marker extraction fine. You can see the `ocr_prompt.md` and `ner_prompt.md` to see what I used to do the OCR and data extraction with Gemini.
- My usual strategy of sending sample images to Gemini and asking it to create a prompt for itself only partially worked in this case. The prompts needed substantial fine-tuning from there to be fully successful. So definitely still a manual element to the OCR side of things. The first pass prompts are in [`prompt-v1-archive`](./prompt-v1-archive/), the improved prompts are the `ocr_prompt.md` and `ner_prompt.md` files.
- You're right that Surya out of the box can't really handle that extremely tight 3-column layout. However, using the manual alignment correction tool in my pipeline, I was able to get 100% line matches. It was a little tedious, but if you're only doing a handful here and there instead of dozens of pages, it's doable. I only did this for the first and the third page p. 646 (scan 674) and p. 648 (scan 676) -- I purposely left the second and the fourth page untouched so the data explorer would show what happens in both the manual correction and the unmediated output. 
You can see in the data explorer that many of the entries for pages 2 and 4 either have image snippets crossing column boundaries, or default to showing the entire page (which means that the pipeline couldn't match the gemini lines to any of the surya lines, which happens when surya doesn't parse finely enough and therefore there are more gemini lines than surya lines)
- I suspect in 6-12 months, there will be open weight models that can match Gemini quality on tasks like this

### NER / data extraction
- There was a substantial issue on this front, which is that while the NER prompt seemed to do very well for the entries under the "bookseller" headings, it captured subsequent entries from later sections, and even worse it mis-categorized them as part of the bookseller section (e.g., entries clearly under bootmaker were listed as booksellers, particularly bad because the same markers are used but mean different things). That's pretty bad, and if I was running this for real I'd probably try to clip the data from the OCR pages to _just_ the sections I cared about, in order to head off that kind of heading drift despite the detailed exhortations of the NER prompt.

## File structure

### Inputs

| File | What it is |
|---|---|
| `manifest.json` | Synthetic IIIF manifest holding only the four canvases we want, sliced out of the full 890-canvas volume manifest. Canvas ids and image service URLs are preserved from the source, so `canvas_fragment` values still resolve against Leicester's viewer. |
| `NNNN_p16445coll4:NNNNN.jpg` | The four page scans, 4550 × 6275 px. CONTENTdm ignores the pipeline's width request and serves full size. |
| `ocr_prompt.md` | OCR system prompt. Carries the three-column layout, the reference-mark repertoire, and the warning that the column rule is not a `‖`. |
| `ner_prompt.md` | NER system prompt. Carries the section-scoped symbol→role keys and the Booksellers-only scoping rules. |
| `selection.txt`, `select_pages.html` | Sample-page selection from the calibration step, used to generate the first-draft prompts. |

### OCR output

| File | What it is |
|---|---|
| `*_gemini-3.1-flash-lite.txt` | Transcription per model, one printed line per output line. Three models were run |
| `*_gemini-3-flash-preview.txt` | over the identical prompt so they could be compared directly — see |
| `*_gemini-3.1-pro-preview.txt` | `model-comparison.md`. |
| `*_surya.json`, `*_surya.txt` | Surya line detection: bounding boxes and their text, the spatial half of the alignment step. |
| `*_gemini-3.1-pro-preview_aligned.json` | Gemini text matched to Surya boxes, giving each line an `#xywh` region. Pages 674 and 676 were hand-corrected in the review UI (`"confidence": "manual"`, full coverage); 675 and 677 are left as the raw automatic output for comparison. |

### Extracted data

| File | What it is |
|---|---|
| `entries_gemini-3.1-flash-lite.csv` | **The main output** — 635 entries. Columns: `section`, `subsection`, `page`, `markers`, `marker_roles`, `name`, `qualifier`, `address`, `cross_reference`, `canvas_fragment`, `image`. `markers` holds the reference marks verbatim; `marker_roles` resolves them against whichever key governs that sub-section. |
| `*_entries.json` | Per-page NER cache, one per page. Lets extraction resume without re-billing pages that already succeeded. |
| `extraction_context_*.json` | Heading context carried across page boundaries, so entries at the top of a page inherit the right section. |
| `explorer.html` | Self-contained browser UI over the CSV — facet filters, per-page density, and IIIF thumbnails of each entry's region. |

### Documentation

| File | What it is |
|---|---|
| `model-comparison.md` | Full writeup of the three OCR models scored on reference-mark fidelity, plus the two operational findings (pro's thinking budget, and prompt elaboration hurting the smallest model). |
| `prompt-v1-archive/` | Superseded prompts and the transcriptions made with them, so the effect of each prompt revision stays visible. Its own `README.md` explains what the auto-generated v0 prompts got wrong. |



## Pipeline commands summary 

```python
# 1. Slice the 890-canvas volume down to the four Booksellers pages
python tools/slice_manifest.py \
    "https://leicester.contentdm.oclc.org/iiif/info/p16445coll4/8844/manifest.json" \
    --from-id 27074 --to-id 27077 --slug london-1841-booksellers
#    → output/london-1841-booksellers/manifest.json

# 2. Download the four page images
python main.py output/london-1841-booksellers/manifest.json \
    --slug london-1841-booksellers --download

# 3. Calibrate: pick sample pages, generate first-draft prompts
python main.py output/london-1841-booksellers/manifest.json \
    --slug london-1841-booksellers --select-pages --generate-prompts
#    → then hand-rewrite ocr_prompt.md and ner_prompt.md

# 4. OCR — one pass per model compared
for M in gemini-3.1-flash-lite gemini-3-flash-preview gemini-3.1-pro-preview; do
  python main.py output/london-1841-booksellers/ --slug london-1841-booksellers \
      --gemini-ocr --ocr-model "$M" --no-flex
done

# 5. Surya line detection + alignment against the pro transcription
pipeline ocr output/london-1841-booksellers --ocr-model gemini-3.1-pro-preview

# 6. Interactive alignment review (browser UI at localhost:5000)
pipeline review output/london-1841-booksellers --model gemini-3.1-pro-preview

# 7. NER extraction
python -m pipeline.extract_entries output/london-1841-booksellers \
    --model gemini-3.1-flash-lite --aligned-model gemini-3.1-pro-preview \
    --mode multimodal --force

# 8. Build the explorer
python -m pipeline.explore_entries \
    output/london-1841-booksellers/entries_gemini-3.1-flash-lite.csv \
    --out output/london-1841-booksellers/explorer.html
```

