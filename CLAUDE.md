# Project rules

These hold for every session in this repository, alongside the global rules in `~/.claude/CLAUDE.md`.

## Code location

Every line of code that builds, samples, processes, fits, tests or draws any part of the analysis is a chunk in `01.manuscript/lidar-marbled-murrelet.qmd`, shown with `echo: true`. A script anywhere else does not count, and an output whose code is not in the manuscript is not evidence and is not cited. A chunk that writes a file to `03.outputs/` wraps that work in `build_once()`, defined in the `setup` chunk, so a render rebuilds the output only when it is missing.

On 2026-10-22 Seamus asked, in the words "hide the code cells in the docx render", for the Word render to leave the code out. A knitr option hook in the `setup` chunk sets `echo` to false when the output is docx, and the html render still prints every chunk. The same day Seamus set the title in the words "change the title to the following: Mapping Marbled Murrelet habitat in the Clayoquot Sound using ALS LiDAR data".

## Target journal

Forest Ecology and Management, chosen by Seamus on 2026-10-22 over Avian Conservation and Ecology, whose 1300 USD fee breaks the no-author-fee rule. Its subscription route charges authors nothing. It asks for an abstract of at most 250 words, 1 to 7 keywords, double spacing with continuous line numbers, highlights as a separate file of 3 to 5 bullets of at most 85 characters, and single anonymized review, so the author's name stays on the manuscript.

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

On 2026-11-15 Seamus kept the ground and noise classes that the contractor, Quantum Spatial, delivered in the LidarBC files, in the words "definitely keep the contractors classification and make sure that the classifications are reported in the method section, that they were inherited". He chose this over his lidar-forestry ebook's cloth simulation ground filter and statistical outlier removal. On three hectares of tall forest, outlier removal had dropped 1.6 to 1.8 percent of points at mean heights of 10 to 16 m, and the cloth filter had moved forest height by up to 4.6 m. He also asked for the Methods to say that further noise removal weakened the predictions. That claim waits on a test that refits the model with outlier removal on the same cells and folds, and it has not been run.

The train in Desolation Sound, test in Clayoquot repeat was the first design. It was dropped on 2026-09-28 because LidarBC covers only 11 of the 127 Desolation Sound nests.

## Current task

On 2026-10-26 Seamus asked for a heavy redraft of the whole manuscript, above all the abstract, Introduction, Methods and Discussion, in the words "Really condense it down to the bare minimum, but improve the flow of the prose and the wording throughout using the skills /writing-flow and /writing-scientific-prose and /writing-scientific-articles", with at most 9,000 words excluding references. The redraft was committed and pushed the same day as 6c80a1a. The main text fell from 3,054 to 2,719 words, the abstract from 245 to 218 and the total excluding references from 3,436 to 3,073, and the mean sentence fell from 36 words to 28. No number, citation key or quotation was changed. RMSE, MAE, the Brier score, relative RMSE, R² with its F test, the Durbin-Watson statistic, overall accuracy and the omission and commission rates were left to Table 4 and no longer appear in the Results text. The Introduction lost the provincial model's age and height thresholds and the nest-tree AUC of Hagar et al. (2014). Running `git diff --word-diff 25baa72 6c80a1a -- 01.manuscript/lidar-marbled-murrelet.qmd` shows every change word by word. Word counts were taken from the Word render with `pandoc -t plain` after a Lua filter removed every table, figure and image, and were counted by section heading.

On 2026-10-27 six steps remained before submission.

1. Seamus's read-through of the redraft, with a ruling on the one present-tense verb left, "is a seabird that nests inland" in the first sentence of the Introduction, which the global past-tense rule forbids and the writing-flow skill allows.
2. Seamus's affiliation on the title page.
3. Confirmation of the funding statement.
4. A full reading of the 7 retained records not yet reached, listed in `02.inputs/derived/review-records.csv`, Zharikov et al. (2007) first because it tested a model transferred from Desolation Sound to Clayoquot Sound.
5. A check of the limits under Target journal against the journal's live guide for authors.
6. Submission of the manuscript with `01.manuscript/submission/cover-letter.docx` and `01.manuscript/submission/highlights.docx`.

On 2026-11-11 Seamus approved North et al. (2017), "Cover of tall trees best predicts California spotted owl habitat", Forest Ecology and Management 405, 166 to 178, https://doi.org/10.1016/j.foreco.2017.09.019, as the exemplar paper, in the words "The North paper is great". He asked for the manuscript to be built out from its 2,719 words, and for the literature review subsection to leave the Methods, in the words "Yes, definitely remove this literature subsection from the sections. That just needs to, the results of that literature review needs to build out the introduction around the current debates in the literature, but also it will, the introduction will finish then on the hypothesis of the study or the main objectives, so that that can then lead into the meta section." ("meta section" is speech-to-text for Methods section.) The day's edits go into `01.manuscript/lidar-marbled-murrelet-20261111.qmd` and dated copies of the cover letter and highlights. The undated files stay as they were.

Later on 2026-11-11 Seamus approved the nine-paragraph Introduction of 1,375 words for the dated manuscript, but ruled out the claim that no laser-scan model had been fitted to the Clayoquot Sound nests, in the words "I don't think we should make that claim". He asked for the review to widen, in the words "definitely try and find studies that have done this or have done similar kinds of studies", "What kind of old growth studies are out there? With habitat in mind", and "start looking at all the other murrelet and habitat literature that's out there. See what kind of important questions there are and concerns, what kind of weaknesses". The Introduction is to keep growing from that wider reading. The same evening he approved, in the word "yes", a 13-paragraph Introduction of 2,078 words built from that reading, with 15 new references, and the removal of the "only", "first" and "no earlier" claims from the abstract and cover letter lines 21 and 27. All three went into the 2026-11-11 dated files.

Since 2026-11-11, each day's edits have gone into copies dated with that day, so the latest dated copy, not the undated file, holds the current text and code. On 2026-11-13 Seamus approved new Table 1 rows in the words "Yes, add them in that way", and they went into `02.inputs/derived/review-attributes-20261113.csv`, which only the dated manuscripts read. The same day the Introduction's count of Wilk et al. (2016) nest sites was corrected from 20 to 18, and the van Rooyen surname was braced in `references.bib` so the particle prints.

On 2026-11-15 the Laser scans paragraph was reworded to say the files came as compressed LAS with ground and noise classes inherited from the contractor. Seamus approved it in the words "Excellent, write this into the manuscript and don't forget the working code. I would like to build this markdown to demonstrate on GitHub". The manuscript gained a `gfm` format with `wrap: none`, whose tables print as pipe tables. At his "Yes", the 15 November set, `references.bib` and the Table 1 CSV were committed and pushed as e1303c7.

On 2026-11-16 the open tasks were written into `05.tasks/2026-11-16-task-brief.md`, which git ignores, and the README was updated to point at the 15 November manuscript. The working manuscript is `01.manuscript/lidar-marbled-murrelet-20261115.qmd`.

Nothing is pushed without Seamus's word, and every commit is authored by Seamus alone, with no co-author or attribution line.
