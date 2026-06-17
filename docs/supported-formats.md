# Supported formats

You call `->parse()` on anything below; we detect the type and route it for you. There's
nothing to configure per format.

| Format | Extensions | What you get |
|:--|:--|:--|
| **PDF** | `.pdf` | Clean Markdown. **Scanned PDFs are OCR'd automatically**, no flag needed. |
| **Word** | `.docx`, `.doc` | Markdown with headings, lists, and tables preserved. |
| **PowerPoint** | `.pptx`, `.ppt` | One `## Slide N` section per slide, including slide tables. |
| **Spreadsheet** | `.xlsx`, `.xls`, `.csv` | Each sheet (or the CSV) rendered as a Markdown table, one `## Sheet` per tab. |
| **Email** | `.eml`, `.msg` | Headers (from/to/subject/date) as YAML frontmatter, body text, and an attachment list. |

> Legacy `.doc` / `.ppt` / `.xls` (the pre-2007 binary formats) are supported alongside the
> modern `.docx` / `.pptx` / `.xlsx`. More formats are on the way.

## Parsing features

- **Automatic OCR.** Scanned or image-only PDFs are detected and run through OCR; you don't
  pick a mode.
- **Multi-column layouts.** Two- and multi-column pages (think scientific papers) are read in
  the correct reading order, not jumbled left-to-right across columns.
- **Tables.** Tables are detected and emitted as proper Markdown tables.
- **Document structure.** Headings, lists, bold/italic, and code blocks come through as real
  Markdown, not a flat wall of text.
- **Hyperlinks preserved.** Links in the source stay as Markdown links in the output.
- **Email fields & attachments.** Email comes back with from/to/subject/date as frontmatter,
  the body, and a list of attachments (name + size).
- **Type auto-detection.** You never pass a format; we detect it and route automatically.
- **Optional frontmatter.** Add `->frontmatter()` to prepend YAML metadata (author, dates,
  page/slide/sheet counts).
- **Page ranges.** Parse just the pages you want with `->pages('1-20')`.
- **Large files.** PDFs up to **1 GB** each; other formats top out lower (see
  [Limits](handling-results.md#limits--errors)).
- **Massive bulk.** Queue **tens of thousands of files at once**; the backend scales out to
  absorb the burst and writes each result straight back to your bucket.
- **One consistent format.** Every input type (PDF, Word, Excel, email) comes out as the
  same clean Markdown.
- **Private by design.** With your own bucket, your file bytes never pass through us: your
  bucket in, your bucket out.
