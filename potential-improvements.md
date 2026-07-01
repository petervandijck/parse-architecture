# Potential Improvements

Ideas for improving the Parse for Artisans parsing service, from a review against [LiteParse](https://github.com/run-llama/liteparse) (run-llama, Apache-2.0).

## Comparison: Parse for Artisans vs. LiteParse

They target different jobs. Parse for Artisans is a hosted SaaS API; LiteParse is an open-source local library/CLI. LiteParse is closer to a component you would embed than a competitor a customer would pick instead.

| | Parse for Artisans | LiteParse |
|---|---|---|
| What it is | Hosted SaaS API, send a doc get Markdown back async | Open-source local library/CLI (Apache-2.0) |
| Deployment | Serverless Modal backend, managed | Self-installed (pip/npm/cargo/WASM) |
| Input formats | PDF, DOCX, PPTX, XLSX/CSV, legacy Office (.doc/.ppt/.xls), email (.eml/.msg) | PDF native; Office via LibreOffice (wider set incl .odt/.rtf/.pages/.key/.numbers); raw images via ImageMagick |
| Output | Markdown only (optional YAML frontmatter) | JSON (text + bounding boxes), plain text, Markdown, PNG page screenshots |
| Engine | PyMuPDF + Tesseract, mammoth, python-pptx, openpyxl, LibreOffice | PDFium + Tesseract (or pluggable EasyOCR/PaddleOCR/custom HTTP OCR) |
| OCR | Auto-detect + forced, ~120 langs, Tesseract only | Selective/conditional, complexity pre-detection, pluggable engines |
| Structured data | None | Bounding boxes, grid projection, spatial layout |
| Page images | None | PNG per page at configurable DPI |
| Async/webhooks/batch | Yes (HMAC webhooks, batch, idempotency, tiered limits, billing) | N/A (library) |
| Licensing | Ships PyMuPDF (AGPL, flagged risk) | Apache-2.0 |

LiteParse's real edges over us: it parses raw images, emits structured/spatial output and page screenshots, and has a cleaner license than our PyMuPDF dependency.

## Improvement ideas

### High-value, aligned with roadmap

1. **Raw image input** (.jpg/.png/.tiff/.webp/.heic). We already ship `tesseract-ocr-all` and OCR scanned PDFs, so an `image` route is mostly a thin wrapper. Most-requested missing format vs. LiteParse and the `competition/` corpus. Bills as 1 credit = 1 image.

2. **Page-image / screenshot generation option.** A `render` / `images: true` option returning PNG-per-page at configurable DPI to the same BYO/managed storage. Killer feature for "Built for AI": feeds multimodal LLM pipelines (page image + Markdown to a vision model). LiteParse markets this as "screenshots for LLM agents." Storage/delivery plumbing already exists.

3. **Optional structured/JSON output mode** (bounding boxes + reading order). Turns us from "Markdown converter" into "document intelligence API" (where Textract/Azure/Reducto sell). PyMuPDF already exposes block coordinates; this is a serialization/option layer.

### Medium-value

4. **LLM/vision-assisted parsing tier.** Premium route where a vision model (latest Claude, e.g. Opus 4.8 or Sonnet 5, via the `Laravel\Ai` package already referenced in docs) cleans up dense tables, multi-column layouts, and handwriting Tesseract mangles. Exactly the gap LiteParse punts on ("use LlamaParse for complex docs"). Natural premium upsell, fits the founder story (undercut LlamaParse on cost).

5. **Resolve the PyMuPDF AGPL risk.** Evaluate LiteParse (PDFium, Apache-2.0), Docling, or Marker as the PDF engine before charging at scale. Adopting LiteParse as the PDF route would also deliver images + structured output + screenshots in one move (partly covers 1-3). Worth a serious spike.

6. **More Office/OpenDocument formats** (.odt/.ods/.odp/.rtf). We already run LibreOffice headless for legacy Office; extending the `legacy-office` route is low incremental cost.

7. **Pluggable/better OCR** (EasyOCR or PaddleOCR) for non-Latin scripts where Tesseract is weak.

### Lower-value / later

8. **HTML and plain-text input.** Cheap to add, rounds out "any document."
9. **Chunking option for RAG** (semantic or fixed-size with overlap). Deliberately out of v1; natural paid feature once the core is stable.
10. **Image extraction** (pull embedded images to files instead of reducing to alt text). Pairs with #2.

## Recommended sequencing

1. Image input (#1) and page screenshots (#2) first, both small given existing infrastructure.
2. Run the LiteParse-as-PDF-engine spike (#5), the cheapest path to also getting structured output (#3).
3. Vision tier (#4) is the biggest revenue idea but the most build.
