# Lidar marbled murrelet

Manuscript, data and analysis for "Mapping Marbled Murrelet habitat in the Clayoquot Sound using ALS LiDAR data", prepared for Forest Ecology and Management. Rendering the manuscript runs the whole analysis, writes every table and figure, and fills every number in the text.

## Current draft

The current draft is dated 15 November 2026. Read it on GitHub as [lidar-marbled-murrelet-20261115.md](01.manuscript/lidar-marbled-murrelet-20261115.md), which prints every code chunk and gives each table as text. Its source is `01.manuscript/lidar-marbled-murrelet-20261115.qmd`, and its Word and HTML renders sit beside it.

Each working day's edits go into copies dated with that day. The undated `lidar-marbled-murrelet.qmd` holds the draft of 26 October 2026.

## Layout

1. `01.manuscript/` holds the dated manuscripts with their `.docx`, `.html` and `.md` renders, `submission/` for the cover letter and highlights, and `archive/` for superseded drafts.
2. `02.inputs/` holds each dataset in its own folder with a README giving source, licence and download link.
3. `03.outputs/` holds `tables/` and `figures/`, written by the manuscript on render, and `archive/`.
4. `04.references/` holds `references.bib`, the citation styles, the Word and HTML style files, and `literature/` for source PDFs.
5. `05.tasks/` holds working notes and is not committed.

## Rendering

Open `lidar-marbled-murrelet.Rproj` in RStudio and render the dated manuscript, or run `quarto render lidar-marbled-murrelet-20261115.qmd --to docx` from `01.manuscript/`, then again with `html` and `gfm`. There are no separate scripts to run.
