# Lidar marbled murrelet

Manuscript, data and analysis. Rendering `01.manuscript/lidar-marbled-murrelet.qmd` runs the whole analysis, writes every table and figure, and fills every number in the text.

## Layout

1. `01.manuscript/` holds the master `.qmd`, its `.docx` and `.html` renders, and `archive/` for superseded drafts.
2. `02.inputs/` holds each dataset in its own folder with a README giving source, licence and download link.
3. `03.outputs/` holds `tables/` and `figures/`, written by the manuscript on render, and `archive/`.
4. `04.references/` holds `references.bib`, the citation styles, the Word and HTML style files, and `literature/` for source PDFs.
5. `05.tasks/` holds working notes and is not committed.

## Rendering

Open `lidar-marbled-murrelet.Rproj` in RStudio and render the manuscript, or run `quarto render 01.manuscript/lidar-marbled-murrelet.qmd` from the repository root. There are no separate scripts to run.
