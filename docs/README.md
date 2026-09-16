# The one-page quickstart

`quickstart.html` is the source; `quickstart.pdf` is what it renders to. Two pages, for somebody
who wants the install on a sheet of paper rather than a document to read.

Rebuild the PDF after editing the HTML — it does not regenerate itself, and a stale PDF beside a
corrected page is worse than no PDF:

```sh
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu \
  --no-pdf-header-footer --print-to-pdf=docs/quickstart.pdf docs/quickstart.html
```

Any Chromium does it; `chromium --headless --print-to-pdf` on Linux, Edge on Windows. The layout
is tuned to land on exactly two A4 pages, so check the page count afterwards: adding a
troubleshooting row or a Q&A entry is usually what pushes it to three.

It is a summary. `INSTRUCTIONS.md` is the document it summarises, and when the two disagree,
INSTRUCTIONS.md is right.
