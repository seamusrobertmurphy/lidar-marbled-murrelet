# Project rules

These hold for every session in this repository, alongside the global rules in `~/.claude/CLAUDE.md`.

## Code location

Every line of code that builds, samples, processes, fits, tests or draws any part of the analysis is a chunk in `01.manuscript/lidar-marbled-murrelet.qmd`, shown with `echo: true`. A script anywhere else does not count, and an output whose code is not in the manuscript is not evidence and is not cited. A chunk that writes a file to `03.outputs/` wraps that work in `build_once()`, defined in the `setup` chunk, so a render rebuilds the output only when it is missing.

## Reported numbers

No number is typed into the prose. Each chunk writes its result to a table in `03.outputs/tables/`, and the text reads it back with inline R, `` `r tbl("T1_example.csv")$value[1]` ``. A number is written into a sentence only after the chunk has run and its output has been read.

## Data sources

Every line of code that reads a third-party dataset carries, on the comment lines directly above it, the dataset's name and the link it was retrieved from, the DOI as an https link where one exists, taken from the dataset's README in `02.inputs/`.

## Tables and figures

Tables are numbered T1, T2 and figures F1, F2 in `03.outputs/`, in order of first citation. A table caption sits above its table and a figure caption below its figure.

## Citations

Every entry in `04.references/references.bib` is taken from a CrossRef query, never from memory. A quotation is checked verbatim against the PDF in `04.references/literature/`.

## Study design

Approved by Seamus on 2026-09-28 in the words "yes build the clayoquot design first", in answer to this proposal. The model of Cosgrove et al. (2024), MaxEnt on forest vertical complexity and internal forest gaps at 100 m, is rebuilt in Clayoquot Sound from that area's own radio-tracked nests (Movebank CS_nests.csv) and LidarBC scans alone. It is tested by holding out groups of nests that sit far apart, and it is compared with the model Cosgrove et al. trained in Desolation Sound and transferred to Clayoquot Sound, which scored an AUC of 0.63. The question is whether a model built from local nests beats one borrowed from 150 km away. The small nest sample is stated as a limitation. No nest location is mapped in the paper.

The train in Desolation Sound, test in Clayoquot repeat was the first design. It was dropped on 2026-09-28 because LidarBC covers only 11 of the 127 Desolation Sound nests.
