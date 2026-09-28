# Review protocol

Written 2026-09-28, before any query ran, for `lidar-marbled-murrelet`. Follows
`library-science/protocols/literature-review.md`. Frozen by the commit that adds it.
Nothing above "Deviations" is edited after that commit.

## Objectives

Approved by Seamus on 2026-09-28.

1. Rebuild the two-measure model of Cosgrove et al. (2024) from Clayoquot nests and LidarBC scans, and score it on held-out groups of nests.
2. Compare that score with the Desolation Sound model transferred to Clayoquot.

## Questions

1. Which laser-scan measures of forest structure have been linked to murrelet nest sites, and in which direction? Population marbled murrelet, concept airborne laser scanning, context nesting habitat.
2. How well do nesting-habitat models for this bird hold up when moved to a new region? Population marbled murrelet, concept habitat suitability models and their transfer, context nesting habitat.
3. Novelty. Has anyone already built a local model in Clayoquot from open scans and compared it with a borrowed one? Population marbled murrelet, concept a nesting-habitat model built from local nests, context Clayoquot Sound.

## Adjudication

1. If a study reports that forest vertical complexity or internal forest gaps relates to nest sites in the opposite direction to Cosgrove et al. (2024), the expected direction in the attribute table is stated as contested and both sources are cited.
2. If a study reports a transferred murrelet nesting-habitat model that scores as well in its new region as in its training region, the claim that local models are needed is withdrawn and the comparison is reported without a preferred direction.
3. If a study has already fitted a laser-scan model to Clayoquot Sound nests and compared it with a transferred model, the manuscript is framed as a replication of that study.

## Criteria

1. Document types admitted are peer-reviewed articles, reviews, chapters and conference papers, and agency reports or standards where they are the primary source of a method, threshold or data product.
2. Published 1990 to 2026, opening at 1990 because the Committee on the Status of Endangered Wildlife in Canada first designated the species Threatened in April 1990.
3. English only.
4. Bears on at least one question, meaning laser-scan measures at murrelet nest sites, murrelet nesting-habitat models and their transfer, or murrelet nesting habitat in Clayoquot Sound.

Exclusions use the codes in the protocol.

## Sources

1. `library-science`, searched with the `review-store` chunk.
2. Semantic Scholar bulk search, with the `review-semantic-scholar` chunk.
3. Grey literature sought by hand from the British Columbia Ministry of Water, Land and Resource Stewardship, the Resources Information Standards Committee and Environment and Climate Change Canada, for murrelet habitat standards and recovery documents.

Search terms are selected in `04.references/review/search-terms.csv`, from candidates the
`review-terms` chunk drew from the manuscript's title, subtitle and keywords, and committed
with this file. The queries are built from that selection by the `review-queries` chunk in
`01.manuscript/lidar-marbled-murrelet.qmd`.

## Deviations
