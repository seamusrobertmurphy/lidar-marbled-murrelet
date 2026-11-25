# Mapping Marbled Murrelet habitat in the Clayoquot Sound using ALS LiDAR data
Seamus Murphy
2026-11-22

``` r
# Rendering this document runs the whole analysis. Every table and figure is written
# by a chunk below into 03.outputs, and every number in the prose is read back from
# those files by inline R, so the text cannot drift from the results. The code is
# printed in the html render and left out of the Word render.
library(knitr)
knitr::opts_hooks$set(echo = \(options) { if (knitr::pandoc_to("docx")) options$echo <- FALSE; options })
knitr::opts_chunk$set(echo = TRUE, warning = FALSE, message = FALSE)
# GitHub shows a flextable as a picture, so the markdown render prints each
# table's cells as a pipe table instead.
if (knitr::pandoc_to(c("gfm", "commonmark"))) {
  loadNamespace("flextable")
  registerS3method("knit_print", "flextable", \(x, ...) knitr::knit_print(knitr::kable(x$body$dataset, format = "pipe")), envir = asNamespace("knitr"))
}

repo <- normalizePath("..")
INP  <- file.path(repo, "02.inputs")
TBL  <- file.path(repo, "03.outputs", "tables")
FIG  <- file.path(repo, "03.outputs", "figures")
dir.create(TBL, recursive = TRUE, showWarnings = FALSE)
dir.create(FIG, recursive = TRUE, showWarnings = FALSE)

tbl <- function(f) read.csv(file.path(TBL, f))
fmt <- \(x, d = 2) formatC(x, format = "f", digits = d, big.mark = ",")
pv <- \(p) if (p < 0.001) "*P* < 0.001" else paste0("*P* = ", fmt(p))
ev <- \(what, col = "value", m = "local") { t <- tbl("T5_evaluation.csv"); t[[col]][t$model == m & t$metric == what] }
sep <- \(what, col) { t <- tbl("T4_separability.csv"); t[[col]][t$measure == what] }
smp <- \(what) { t <- tbl("T2_sample.csv"); t$n[t$step == what] }
nz <- \(what, col = "value", m = "outlier removal") { t <- tbl("T6_noise.csv"); t[[col]][t$model == m & t$metric == what] }
cs <- \(what, size, col = "value") { t <- tbl("T7_cell_size.csv"); t[[col]][t$metric == what & t$size_ha == size] }
rf <- \(what, a, col = "value") { t <- tbl("T8_refits.csv"); t[[col]][t$metric == what & t$analysis == a] }
acc <- \(cls) if (cls == "nest") 100 * ev("Nest cells above threshold") / ev("Nest cells") else 100 * (1 - ev("Background cells above threshold") / ev("Background cells"))

# Run `code` only when `path` is missing, so a render rebuilds an output that was
# deleted and leaves a finished one alone. Delete the file to force a rebuild.
build_once <- function(path, code) {
  if (!file.exists(path)) force(code)
  invisible(path)
}
```

# Abstract

A nesting-habitat model for the threatened Marbled Murrelet, built from airborne laser scans and nests in Desolation Sound, British Columbia, lost accuracy when transferred to Clayoquot Sound. Models fitted within one region had predicted nesting habitat better than pooled or transferred models, so a local model was expected to do better. This study rebuilt that model from 2019 scans and nests of birds radio-tracked from 2000 to 2002 (16 nest and 3,827 background cells in tall forest unharvested since 1998). Maximum entropy modeling of forest vertical complexity and internal forest gaps in 1 ha cells was scored by five-fold spatial cross-validation in 10 km blocks. Neither measure separated nest cells from background cells (forest vertical complexity, *P* = 0.54; internal forest gaps, *P* = 0.10). Nest cells had more internal forest gaps than background cells, though not significantly. The model classed 62.5 percent of held-out nest cells and 40.8 percent of background cells correctly, an average accuracy of 51.7 percent, which could not be distinguished from chance. The result held in 4 and 9 ha cells and across further checks. The local model’s ranking of nest cells could not be shown to differ from that of the model transferred from Desolation Sound. With so few located nests, a local model gave no reason to replace the transferred model in mapping nesting habitat in Clayoquot Sound.

# Keywords

Airborne laser scanning; *Brachyramphus marmoratus*; habitat suitability; lidar; MaxEnt; model transferability; old-growth forest

# Introduction

The Marbled Murrelet, *Brachyramphus marmoratus* (Gmelin, 1789), was a seabird that nested inland in old-growth coniferous forest. It was listed as threatened in Canada and the United States, and loss of that forest was the primary reason given for its decline (Burger, 2001). In a review of science for federal forests in Washington, Oregon and northern California, Raphael et al. (2018, p. 333) wrote that the studies they summarized pointed “toward nesting habitat as the primary driver, but all these studies concede that relationships are correlational”. The murrelet’s nests were cryptic and difficult to locate (Burger, 2001), so the nesting habitat retained for it in British Columbia was mapped from forest inventory, air photographs and aerial surveys (Burger et al., 2018; Mather et al., 2010). Those maps were judged against the few nests found by radio telemetry (Burger et al., 2018). Burger et al. (2009, p. 12) did “not anticipate having a much larger sample of actual nest sites to test, due to the high costs of telemetry studies needed to find large numbers of nests”.

Nesting habitat was defined by nest platforms, the large limbs and thick epiphyte mats on which murrelets nested (Burger et al., 2010). Ralph et al. (1995, p. 7) wrote that “it is the presence of adequate nesting platforms that defines suitable nesting habitat”, and later work confirmed platforms as “the most important characteristic of murrelet nesting habitat” (Raphael et al., 2018, p. 303). Across six regions of British Columbia, tree diameter was the most important predictor of platforms (Burger et al., 2010). Platforms began “to appear regularly in trees \>30 m tall” and were “most abundant in the large emergent trees” (Burger et al., 2010, p. 736). In Washington and Oregon, nests sat on platforms with more horizontal cover, a larger diameter and more moss than other platforms in the same stands (Hamer et al., 2021). Silvergieter and Lank (2011a, p. 15) noted that the number of platforms, a key term in their Clayoquot Sound models, “cannot be quantitatively measured by remote methods”.

The provincial habitat model classed forest-cover polygons as suitable by stand age, height, elevation and distance to the sea (Mather et al., 2010). Its authors judged it “suited to broad-scale estimation of Marbled Murrelet nesting habitat area, but not its precise location” (Mather et al., 2010, p. 98). Against 171 radio-tracked nests, it placed 52.6 percent in suitable habitat, 69.4 percent in Clayoquot Sound and 52.1 percent in Desolation Sound (Burger et al., 2018). In Clayoquot Sound, the model applied a regional algorithm built from inventory maps drawn for that area (Bahn and Newsom, 2002; Mather et al., 2010), and its better result there “might be attributed” to that algorithm (Burger et al., 2018, p. 13).

Air photo interpretation and low-level aerial surveys ranked stands by canopy features seen from above (Burger et al., 2009; Waterhouse et al., 2008). Fine-scale aerial surveys placed 85.2 percent of 135 nests in suitable habitat and were the only method that estimated nest platforms (Burger et al., 2018). Their ranks separated nests from random points in Desolation Sound and Toba Inlet but not in Clayoquot Sound (Waterhouse et al., 2009). Re-examining the first comparison of radio-tracked nests with random points in the two sounds, Burger and Page (2007, p. 1276) contended that “coarse-scale satellite imagery and forest cover mapping lack the spatial resolution to adequately test effects of patch size and forest edges on marbled murrelets”. They found “little or no support” for that study’s main conclusions and named “tree size, moss development, canopy micro-structure, aspect, topographic complexity within stands” as likely to matter more (Burger and Page, 2007, p. 1279). All of these methods inferred forest structure from maps or from observers’ ranks rather than measuring it.

Airborne laser scanning measured canopy height and the vertical arrangement of vegetation directly, and it had been linked to the habitat of many forest animals. Across 173 studies, Acebes et al. (2021, p. 7) found that “canopy vertical distribution traits were less important for birds than canopy height and cover”. Across 50 bird studies, measures of canopy vertical variability showed a relationship in only 44 percent of their uses (Bakx et al., 2019). In the Sierra Nevada, cover in trees taller than 48 m was “the canopy structure most highly selected for” by California spotted owls (North et al., 2017, p. 166). Cover from 2 to 16 m was avoided. In that study, measures of gap area were “among the least distinct” (North et al., 2017, p. 171). Northern spotted owls in Oregon selected activity centers with more canopy cover and a more uneven canopy surface within 100 m (Sovern et al., 2019). Sovern et al. (2019, p. 722) linked weak selection for a contiguous canopy to maturing stands, where trees “senesce, break, and fall over creating small openings”. In western Washington, conifer stands older than about 75 years showed a rougher canopy surface and a lower canopy density as they aged (Kane et al., 2010).

For the murrelet, Burger et al. (2009, p. 12) wrote that laser scanning “might provide detailed three-dimensional measures of canopy structure that could be extremely valuable for large-scale assessments of Marbled Murrelet nesting habitats”. In southwestern Oregon, a laser-scan model of stand occupancy scored 0.74 on the area under the receiver operating characteristic curve (AUC), against 0.64 for a model built from vegetation maps (Hagar et al., 2014). An AUC of 0.5 marked a model that ranked occupied stands no better than chance. On northern Vancouver Island, measures mostly from laser scans flown in 2012 predicted helicopter survey ranks with an overall accuracy of 71.05 percent, and elevation outranked every canopy measure (Clyde, 2017). In northern California redwood timberland scanned in 2017 and 2018, measures of tall individual trees predicted habitat suitability, while area-based canopy and height-percentile measures “displayed little predictive value” beside them (Lamphear et al., 2025, p. 12). Lamphear et al. (2025, p. 23) wrote that the tree measures were “more direct measures of small-scale marbled murrelet habitat requirements than indirect area-based point cloud metrics”. All three models were fitted to survey records of occupancy or habitat rank rather than to nests (Clyde, 2017; Hagar et al., 2014; Lamphear et al., 2025). Hagar et al. (2014) also compared 11 climbed nest trees with random points. Two measures related to tree height and cover were larger at the nests, with an average AUC of 0.91.

In Desolation Sound, Cosgrove et al. (2024) trained a laser-scan model on 58 radio-tracked nests, using maximum entropy modeling (MaxEnt) in 1 ha cells. Predicted suitability rose with forest vertical complexity, the spread of point heights, and fell with internal forest gaps, the empty space below the top of the canopy. The fall with gaps reversed the authors’ expectation that gaps would favor nesting by “providing canopy openings for nest access and flight maneuvers” (Cosgrove et al., 2024, p. 4). Burger et al. (2009, p. 5) reported that “small gaps in the canopy are typically found next to nest trees”. On Haida Gwaii, small gaps were prevalent on air photos at all 7 nest patches found in 2000 (Waterhouse et al., 2007). On the south coast, a model was fitted to 118 nests tracked from 1998 to 2002 in three areas (Waterhouse et al., 2008). Judged “most applicable to Desolation Sound”, it predicted nests where “small gaps are sporadic but where large gaps do occur” (Waterhouse et al., 2008, p. 27).

The effect of edges and openings also changed with their kind and their scale. In four regions of southwestern British Columbia, disturbance of simulated nests was 2.5 times higher at hard edges, beside clearcuts 5 to 11 years old, than in forest interiors (Malt and Lank, 2009). Malt and Lank (2009, p. 1274) found “no edge effect at natural edges”, which lay beside large rivers or avalanche chutes. Natural-edged patches also held “the greatest levels of epiphyte cover and platform tree density” (<span class="nocase">van Rooyen et al.</span>, 2011, p. 549). Across Washington, Oregon and northern California, occupancy fell with edge within 2 km of survey stations and rose with edge within 100 m (Valente et al., 2023). Those authors “did not distinguish natural and anthropogenic canopy gaps” (Valente et al., 2023, p. 8). In western Washington and southwest Vancouver Island, 18 radio-tracked nest sites had less canopy cover of dominant conifers than nearby control plots, 39.3 against 48.6 percent (Wilk et al., 2016). Wilk et al. (2016, p. 170) read this as “a more open structure around nest sites than in other areas of the forest”.

For vertical complexity, Silvergieter and Lank (2011b, p. 10) wrote that “preference for heterogeneous forest patches with increased vertical canopy complexity has been well established by other studies”. On southwest Vancouver Island from 1991 to 1999, the spread of tree heights in ground plots at 27 survey stations correlated with occupied murrelet activity \[Spearman *r*<sub>s</sub> = 0.495, *P* \< 0.01\] (Burger and Bahn, 2004). In Washington and Oregon from 1995 to 1999, however, climbers searching random plots found the effect of “tree diameter standard deviation (a measure of canopy complexity)” on nesting unclear (Hamer et al., 2021, p. 8). In Clayoquot Sound, ground plots at 27 nests found from 2000 to 2002 showed canopy shorter and less varied in height than at random sites (Silvergieter and Lank, 2011a). Mean canopy height was 33.4 against 37.6 m and its standard deviation 4.9 against 6.0 m, whereas at 37 Desolation Sound nests both ran the other way (Silvergieter and Lank, 2011a). From laser scans at 1 ha, by contrast, nests in both sounds lay in taller and more vertically complex forest than random cells (Cosgrove et al., 2024). Waterhouse et al. (2009, p. 91) found “only weak evidence” that vertical complexity rated from a helicopter helped to predict nests in Desolation Sound, Toba Inlet and Clayoquot Sound.

Models had been moved from Desolation Sound to Clayoquot Sound before laser scans were used. Zharikov et al. (2007) fitted landscape models from forest inventory maps to 90 Desolation Sound nests and tested them on 36 Clayoquot Sound nests. There, the models scored AUCs of 0.558 to 0.663 (Zharikov et al., 2007). The authors traced the weak transfer to old growth, which covered 18 percent of the training extent against 55 percent of the test extent (Zharikov et al., 2007). Transferred to Clayoquot Sound, the Desolation Sound laser-scan model scored an AUC of 0.63, against 0.77 where it was trained (Cosgrove et al., 2024). Its Boyce index, a measure of how closely predicted suitability tracked nest frequency, fell from 0.99 to 0.67 (Cosgrove et al., 2024). The AUC fell below the 0.7 those authors set for a good model, yet they judged that the transferred model “maintained reasonable accuracy” (Cosgrove et al., 2024, p. 9). In Oregon, a laser-scan occupancy model transferred from the Coos Bay District to the Siuslaw National Forest scored an AUC of 0.73, against 0.78 for a locally selected model (Hagar et al., 2018). Hagar et al. (2018, p. 1) found that “neither of these transferred models were considered competitive with a model trained on local data”. Ground-plot models fitted to each sound placed 59 percent of Clayoquot Sound nests above a score of 0.7, against 15 percent for a pooled model without regional terms (Silvergieter and Lank, 2011a). Silvergieter and Lank (2011a, p. 13) concluded that one resource selection function “does not describe habitat selection nearly as well as region-specific models”. They judged that “the limited size of our dataset and strong regional interactions precluded” testing Desolation Sound models against Clayoquot Sound nests (Silvergieter and Lank, 2011a, p. 11). Of the provincial habitat classifications, Burger et al. (2018, p. 15) wrote that “the regional differences that have been found suggest caution in applying results from one region to another”.

Desolation Sound had lost about 80 percent of its original nesting habitat to logging (Burger et al., 2018). Burger et al. (2018, p. 11) wrote that the distribution of nests in what remained “might not be typical for the species in less disturbed landscapes, such as Clayoquot Sound”. Burger and Page (2007, p. 1278) had asked whether the results from Clayoquot Sound, “with relatively little forest loss”, might “represent a truer reflection on the habitat preferences of nesting murrelets”. Air photo interpretation detected the fewest signs of nest-site selection in Clayoquot Sound, in part because “random sites would more often fall in poorer habitats on the Sunshine Coast compared to Clayoquot Sound with its extensive areas of available habitat” (Waterhouse et al., 2008, p. 31).

The way nests were compared with other forest also limited what these studies could detect. Murrelet use of random sites was unknown, so Waterhouse et al. (2008, p. 21) compared nests with available sites “rather than the more powerful contrast of used versus unused sites”. In Clayoquot Sound, the nesting dates of radio-tracked birds “were substantially biased towards later nests” (Tranquilla et al., 2005, p. 357). In Oregon, Valente et al. (2024, p. 10) were “only able to find 17 active nests despite radiotagging 239 individuals” over four breeding seasons. Raphael et al. (2018, p. 333) proposed to “build models exclusively from training sites that represent actual murrelet nests”.

The Clayoquot Sound nests had served to test models transferred from Desolation Sound, built from forest inventory (Zharikov et al., 2007) and from laser scans (Cosgrove et al., 2024). The provincial scans flown there in 2019 and the public release of the nest locations (Cooke et al., 2022) made a local model possible. The first objective was to rebuild the two-measure model of Cosgrove et al. (2024) from local nests and laser scans and to score it on spatially separate groups of nests. The second was to compare that score with that of the Desolation Sound model transferred to Clayoquot Sound. Forest vertical complexity was expected to be higher at nest cells of 1 ha than in other available forest, as in Desolation Sound (Cosgrove et al., 2024) and earlier ground studies (Burger and Bahn, 2004; Silvergieter and Lank, 2011b). Ground plots at Clayoquot Sound nests had shown the reverse (Silvergieter and Lank, 2011a), so this direction was contested there. Internal forest gaps were expected to be fewer at nest cells, as in Desolation Sound (Cosgrove et al., 2024). Small gaps had nonetheless been reported at nest sites (Burger et al., 2009; Waterhouse et al., 2007), and nest sites had been more open than nearby forest (Wilk et al., 2016). Models fitted within one region had described nest sites or occupied stands there better than pooled or transferred models (Hagar et al., 2018; Silvergieter and Lank, 2011a). The local model was therefore expected to score an AUC above the 0.63 of the transferred model. The expected direction of each measure and its source were listed in <a href="#tbl-attributes" class="quarto-xref">Table 1</a>.

``` r
readr::read_csv("../02.inputs/derived/review-attributes-20261122.csv", show_col_types = FALSE) |>
  dplyr::transmute(Measure = measure, `Expected direction` = expected_direction, Reasoning = reasoning, Source = paste0("[", gsub("; ", "; @", paste0("@", citation)), "]")) |>
  knitr::kable(format = "pipe")
```

<div id="tbl-attributes">

Table 1: Laser-scan measures entered in the model, with the direction of their relation to nest sites expected from prior work.

<div class="cell-output-display">

| Measure | Expected direction | Reasoning | Source |
|:---|:---|:---|:---|
| Forest vertical complexity | Positive | Taller forest with more variable structure was expected to offer more nest platforms and overhead cover. In Desolation Sound, predicted suitability rose with forest vertical complexity, and earlier ground studies had linked murrelets to vertically complex forest. Ground plots at Clayoquot Sound nests found shorter and less varied canopy than at random sites, so the direction was contested there. | (Burger and Bahn, 2004; Cosgrove et al., 2024; Silvergieter and Lank, 2011a, 2011b) |
| Internal forest gaps | Negative | In Desolation Sound, predicted suitability fell as internal forest gaps increased, although gaps had been expected to give flight access to nests. On the south coast, an air photo model placed nests where small gaps were sporadic but large gaps occurred. Small gaps had nonetheless been reported beside nest trees and at all 7 nest patches studied on Haida Gwaii. In Washington and on southwest Vancouver Island, nest sites were more open than nearby forest. | (Burger et al., 2009; Cosgrove et al., 2024; Waterhouse et al., 2008; Waterhouse et al., 2007; Wilk et al., 2016) |

</div>

</div>

# Methods

## Study area

Clayoquot Sound lay on the west coast of Vancouver Island, British Columbia, at 49°12′ N, 126°06′ W (<a href="#fig-study-area" class="quarto-xref">Figure 1</a>). Its terrain was mountainous, with steep cliffs, glacial valleys, avalanche chutes and streams that broke the forest naturally, and it rose from sea level to 2,200 m (Zharikov et al., 2006). From April to August, mean temperature was 11.9 °C and rainfall 720 mm, cooler and wetter than the 13.4 °C and 300 mm of Desolation Sound (Zharikov et al., 2006). Old growth on the lower slopes was of western redcedar (*Thuja plicata* Donn ex D. Don) and western hemlock (*Tsuga heterophylla* (Raf.) Sarg.) (Zharikov et al., 2006). It also held Douglas-fir (*Pseudotsuga menziesii* (Mirb.) Franco) and Sitka spruce (*Picea sitchensis* (Bong.) Carrière) (Zharikov et al., 2006). Commercial logging there began after 1954 (Zharikov et al., 2006). The sound supported one of the highest concentrations of breeding Marbled Murrelets south of Alaska, including more than 10 percent of the estimated provincial population (Burger, 2001). The study area was the smallest convex polygon around the nests, widened by 5 km.

## Data and processing

From 2000 to 2002, Marbled Murrelets were captured at sea by spotlighting from late April to early June, fitted with radio transmitters and followed by helicopter to their nests (Zharikov et al., 2006). Each nest was photographed from the air and recorded by GPS, and it was confirmed by tree climbing where it could be reached from the ground (Zharikov et al., 2006). The published data set listed 36 nests, 8 in 2000, 10 in 2001 and 18 in 2002, with a positional error of about 50 m (Cooke et al., 2022). Zharikov et al. (2007, p. 751) reported that 8 of the 36 nests located in Clayoquot Sound “could not be accessed from the ground”.

``` r
# Marbled Murrelets at Clayoquot Sound, British Columbia, Canada (2000-2002), CS_nests.csv
# https://doi.org/10.5441/001/1.kg210889
build_once("../02.inputs/movebank/CS_nests.csv", curl::curl_download(
  "https://datarepository.movebank.org/server/api/core/bitstreams/1504af80-ae53-4e69-a6f6-3b2ce5d53958/content",
  "../02.inputs/movebank/CS_nests.csv"))
nests <- readr::read_csv("../02.inputs/movebank/CS_nests.csv", show_col_types = FALSE) |>
  sf::st_as_sf(coords = c("Long", "Lat"), crs = 4326) |>
  sf::st_transform(3157)
```

``` r
# LidarBC Open LiDAR Data Index, point cloud layer
# https://services6.arcgis.com/ubm4tcTYICKBpist/arcgis/rest/services/LiDAR_BC_S3_Public/FeatureServer/4
build_once("../02.inputs/lidarbc/tiles.gpkg", {
  area <- nests |> sf::st_union() |> sf::st_convex_hull() |> sf::st_buffer(5000)
  query <- paste0(
    "https://services6.arcgis.com/ubm4tcTYICKBpist/arcgis/rest/services/LiDAR_BC_S3_Public/",
    "FeatureServer/4/query?where=year%3D2019&geometryType=esriGeometryEnvelope&inSR=4326",
    "&outFields=filename,s3Url&f=geojson&geometry=",
    area |> sf::st_transform(4326) |> sf::st_bbox() |> paste(collapse = ","))
  set.seed(2019)
  purrr::map(0:5, \(i) sf::read_sf(paste0(query, "&resultOffset=", i * 1000))) |>
    dplyr::bind_rows() |>
    sf::st_transform(3157) |>
    dplyr::filter(lengths(sf::st_intersects(sf::st_centroid(geometry), area)) > 0) |>
    dplyr::mutate(
      nest = lengths(sf::st_intersects(geometry, sf::st_buffer(nests, 150))) > 0,
      background = dplyr::row_number() %in% sample(which(!nest), 40)) |>
    sf::write_sf("../02.inputs/lidarbc/tiles.gpkg")
})
tiles <- sf::read_sf("../02.inputs/lidarbc/tiles.gpkg")
```

``` r
# LidarBC 2019 point clouds, https://lidar.gov.bc.ca/
dir.create("../02.inputs/lidarbc/laz", showWarnings = FALSE)
tiles |>
  sf::st_drop_geometry() |>
  dplyr::filter(nest | background) |>
  purrr::pwalk(\(filename, s3Url, ...) {
    laz <- file.path("../02.inputs/lidarbc/laz", filename)
    build_once(laz, curl::curl_download(s3Url, laz))
  })
```

``` r
dir.create("../02.inputs/derived/thin", recursive = TRUE, showWarnings = FALSE)
list.files("../02.inputs/lidarbc/laz", "laz$") |>
  purrr::walk(\(f) build_once(file.path("../02.inputs/derived/thin", f), {
    h <- lidR::readLASheader(file.path("../02.inputs/lidarbc/laz", f))
    keep <- min(1, 15 * (h$`Max X` - h$`Min X`) * (h$`Max Y` - h$`Min Y`) /
      h$`Number of point records`)
    las <- lidR::readLAS(file.path("../02.inputs/lidarbc/laz", f), select = "xyzc",
        filter = paste("-drop_class 7 -drop_withheld -keep_random_fraction", keep)) |>
      lidR::decimate_points(lidR::homogenize(12, 10))
    if (!lidR::is.empty(las)) {
      lidR::writeLAS(las, file.path("../02.inputs/derived/thin", f))
      rlas::writelax(file.path("../02.inputs/derived/thin", f))
    }
  }))
```

Point clouds came from LidarBC, the open lidar program of the Province of British Columbia, and only nests inside its 2019 Vancouver Island acquisition were used. The acquisition used a Riegl VQ-1560i sensor flown 1,500 m above the ground, with 30 percent swath overlap and a target of 8 pulses per square meter (GeoBC, 2023). Its vertical accuracy on open ground was 0.067 m at the 95 percent level, from 203 check points (GeoBC, 2023). The point clouds were downloaded as compressed LAS files, with ground and noise classes inherited from the contractor. Of the 322 tiles centered in the study area, the 29 within 150 m of a nest were used as nest tiles. A further 40, drawn at random from the rest, were used as background tiles. Points classed as noise, and points the contractor had flagged as withheld from processing, were dropped. Each tile was thinned at random to at most 12 points per square meter within 10 m squares, so that density did not differ between nest and background tiles.

Forest harvested from 1998 to 2022 was screened out with the Canada Forest Harvest 1985–2022 product of the National Terrestrial Ecosystem Monitoring System (Hermosilla et al., 2016). The product recorded the year of stand-replacing harvest on a 30 m grid.

``` r
# Canada Forest Harvest 1985-2022, National Terrestrial Ecosystem Monitoring System
# https://opendata.nfis.org/downloads/forest_change/CA_Forest_Harvest_1985-2022.zip
build_once("../02.inputs/ntems/harvest_cs.tif", {
  curl::curl_download(
    "https://opendata.nfis.org/downloads/forest_change/CA_Forest_Harvest_1985-2022.zip",
    "../02.inputs/ntems/CA_Forest_Harvest_1985-2022.zip")
  unzip("../02.inputs/ntems/CA_Forest_Harvest_1985-2022.zip", exdir = "../02.inputs/ntems")
  harvest <- list.files("../02.inputs/ntems", "\\.tif$", recursive = TRUE, full.names = TRUE)[1] |>
    terra::rast()
  terra::crop(harvest, tiles |> sf::st_transform(terra::crs(harvest)) |> terra::vect(),
    filename = "../02.inputs/ntems/harvest_cs.tif")
})
```

``` r
# Medium Resolution Digital Elevation Model (MRDEM), 30 m, Natural Resources Canada
# https://canelevation-dem.s3.ca-central-1.amazonaws.com/mrdem-30/mrdem-30-dtm.tif
dir.create("../02.inputs/mrdem", showWarnings = FALSE)
build_once("../02.inputs/mrdem/dtm_cs.tif", {
  # The map frame with 4 km to spare, so the relief has no edge effects
  window <- sf::st_as_sfc(sf::st_bbox(c(xmin = 262000, ymin = 5450000, xmax = 328000, ymax = 5513000), crs = 3157))
  dtm <- terra::rast("/vsicurl/https://canelevation-dem.s3.ca-central-1.amazonaws.com/mrdem-30/mrdem-30-dtm.tif")
  terra::crop(dtm, terra::ext(sf::st_bbox(sf::st_transform(window, terra::crs(dtm))))) |>
    terra::project("EPSG:3157", res = 30, method = "bilinear") |>
    terra::crop(terra::vect(window), filename = "../02.inputs/mrdem/dtm_cs.tif", gdal = "COMPRESS=DEFLATE")
})
```

``` r
# Freshwater Atlas Coastlines, Province of British Columbia
# https://catalogue.data.gov.bc.ca/dataset/freshwater-atlas-coastlines
dir.create("../02.inputs/fwa", showWarnings = FALSE)
build_once("../02.inputs/fwa/coastline.gpkg", sf::read_sf("https://openmaps.gov.bc.ca/geo/pub/wfs?service=WFS&version=2.0.0&request=GetFeature&typeNames=pub:WHSE_BASEMAPPING.FWA_COASTLINES_SP&outputFormat=application/json&srsName=EPSG:3157&bbox=980000,460000,1045000,525000,urn:ogc:def:crs:EPSG::3005") |> sf::write_sf("../02.inputs/fwa/coastline.gpkg"))
build_once("../03.outputs/figures/F1_study_area.png", {
  frame <- sf::st_bbox(c(xmin = 266000, ymin = 5454000, xmax = 324000, ymax = 5509000), crs = 3157)
  coast <- sf::read_sf("../02.inputs/fwa/coastline.gpkg")
  # Natural Earth 1:10 million admin 0 countries, read through the rnaturalearthhires package
  # https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-admin-0-countries/
  land <- rnaturalearth::ne_countries(scale = 10, country = c("Canada", "United States of America"), returnclass = "sf") |> sf::st_transform(3157)
  harvest <- terra::rast("../02.inputs/ntems/harvest_cs.tif") |> terra::project("EPSG:3157", method = "near", res = 30) |> terra::aggregate(3, fun = "max") |> (\(r) r >= 1998)() |> terra::as.data.frame(xy = TRUE) |> dplyr::filter(dplyr::if_any(3))
  area <- nests |> sf::st_union() |> sf::st_convex_hull() |> sf::st_buffer(5000)
  # Shaded relief lit from the northwest at 40 degrees, drawn as a grey image under every layer
  dtm <- terra::rast("../02.inputs/mrdem/dtm_cs.tif")
  relief <- terra::shade(terra::terrain(dtm, "slope", unit = "radians"), terra::terrain(dtm, "aspect", unit = "radians"), angle = 40, direction = 315)
  relief <- terra::ifel(dtm > 1, relief, NA)
  shade <- terra::as.matrix(relief, wide = TRUE)
  tone <- 0.62 + 0.38 * pmax(0, pmin(1, shade))
  shade <- matrix(ifelse(is.na(tone), "#FFFFFF00", grDevices::grey(ifelse(is.na(tone), 1, tone))), nrow(shade))
  grat <- sf::st_graticule(sf::st_as_sfc(frame), crs = sf::st_crs(3157), datum = sf::st_crs(4326), lon = seq(-126.4, -125.2, 0.2), lat = seq(49.1, 49.8, 0.1))
  main <- ggplot2::ggplot() +
    ggplot2::annotation_raster(grDevices::as.raster(shade), xmin = terra::xmin(relief), xmax = terra::xmax(relief), ymin = terra::ymin(relief), ymax = terra::ymax(relief)) +
    ggplot2::geom_sf(data = grat, colour = "grey25", linewidth = 0.15) +
    ggplot2::geom_tile(data = harvest, ggplot2::aes(x, y, fill = "Harvested 1998 to 2022"), width = 90, height = 90) +
    ggplot2::geom_sf(data = coast, colour = "grey35", linewidth = 0.2) +
    ggplot2::geom_sf(data = sf::st_union(tiles), ggplot2::aes(colour = "LidarBC 2019 scan tiles"), fill = NA, linewidth = 0.4) +
    ggplot2::geom_sf(data = area, ggplot2::aes(linetype = "Study area"), fill = NA, colour = "black", linewidth = 0.4) +
    ggplot2::scale_fill_manual(NULL, values = "#D55E00") + ggplot2::scale_colour_manual(NULL, values = "#0072B2") + ggplot2::scale_linetype_manual(NULL, values = "dashed") +
    ggplot2::annotate("text", x = 267200, y = 5456300, label = "Pacific Ocean", hjust = 0, fontface = "italic", size = 2.6, colour = "grey30") +
    ggplot2::annotate("rect", xmin = 316400, xmax = 323600, ymin = 5454400, ymax = 5461600, fill = "white", colour = "grey40", linewidth = 0.3) +
    ggplot2::annotate("text", x = 320000, y = 5455500, label = "1:400,000", size = 2.4) +
    ggspatial::annotation_north_arrow(location = "br", which_north = "true", height = grid::unit(0.95, "cm"), width = grid::unit(0.6, "cm"),
      pad_x = grid::unit(0.7, "cm"), pad_y = grid::unit(0.6, "cm"), style = ggspatial::north_arrow_fancy_orienteering(text_size = 6, line_width = 0.5)) +
    ggplot2::scale_x_continuous(breaks = seq(-126.4, -125.2, 0.2)) + ggplot2::scale_y_continuous(breaks = seq(49.1, 49.8, 0.1)) +
    ggplot2::coord_sf(crs = 3157, datum = 4326, xlim = frame[c(1, 3)], ylim = frame[c(2, 4)], expand = FALSE, label_axes = "ENEN") +
    ggplot2::theme_void(base_size = 8) +
    ggplot2::theme(panel.border = ggplot2::element_rect(fill = NA, colour = "black", linewidth = 0.5), panel.background = ggplot2::element_rect(fill = "white", colour = NA),
      axis.text = ggplot2::element_text(size = 6.5, colour = "black"), axis.ticks = ggplot2::element_line(colour = "black", linewidth = 0.3), axis.ticks.length = grid::unit(2, "pt"),
      legend.position = "none", plot.margin = ggplot2::margin(2, 2, 2, 2))
  key <- cowplot::get_plot_component(main + ggplot2::theme(legend.position = "right", legend.spacing.y = grid::unit(0, "pt"), legend.margin = ggplot2::margin(0, 0, 0, 0), legend.key.size = grid::unit(9, "pt")), "guide-box-right")
  locator <- ggplot2::ggplot() +
    ggplot2::geom_sf(data = land, fill = "grey80", colour = NA) +
    ggplot2::geom_sf(data = sf::st_as_sfc(frame), fill = NA, colour = "black", linewidth = 0.4) +
    ggplot2::coord_sf(crs = 3157, xlim = c(100000, 800000), ylim = c(5250000, 5900000), expand = FALSE, datum = NA) +
    ggplot2::theme_void() + ggplot2::theme(panel.background = ggplot2::element_rect(fill = "white", colour = NA))
  pw <- 58000 / 4e3; ph <- 55000 / 4e3
  g <- ggplot2::ggplotGrob(main)
  panel <- g$layout[g$layout$name == "panel", ]
  g$widths[panel$l] <- grid::unit(pw, "cm"); g$heights[panel$t] <- grid::unit(ph, "cm")
  grDevices::pdf(NULL)
  cm <- \(u) grid::convertUnit(u, "cm", valueOnly = TRUE)
  fw <- sum(cm(g$widths)); fh <- sum(cm(g$heights))
  left <- sum(cm(g$widths[seq_len(panel$l - 1)])); bottom <- sum(cm(g$heights[-seq_len(panel$b)]))
  grDevices::dev.off()
  at <- \(x, y) c((left + x * pw) / fw, (bottom + y * ph) / fh)
  fig <- cowplot::ggdraw(g) +
    cowplot::draw_grob(grid::rectGrob(gp = grid::gpar(fill = "white", col = "grey40", lwd = 0.8)), x = at(0.012, 0)[1], y = at(0, 0.665)[2], width = 0.245 * pw / fw, height = 0.323 * ph / fh) +
    cowplot::draw_plot(locator, x = at(0.022, 0)[1], y = at(0, 0.79)[2], width = 0.225 * pw / fw, height = 0.19 * ph / fh) +
    cowplot::draw_grob(key, x = at(0.022, 0)[1], y = at(0, 0.672)[2], width = 0.225 * pw / fw, height = 0.11 * ph / fh, hjust = 0, vjust = 0) +
    ggplot2::theme(plot.background = ggplot2::element_rect(fill = "white", colour = NA))
  ggplot2::ggsave("../03.outputs/figures/F1_study_area.png", fig, width = fw, height = fh, units = "cm", dpi = 300)
})
knitr::include_graphics("../03.outputs/figures/F1_study_area.png")
```

<div id="fig-study-area">

![](../03.outputs/figures/F1_study_area.png)

Figure 1: Clayoquot Sound on the west coast of Vancouver Island, British Columbia. The dashed line bounded the study area, drawn 5 km around the radio-tracked nests. The blue line bounded the 322 LidarBC scan tiles flown in 2019, each about 2.5 km², and orange marked forest harvested from 1998 to 2022. Shading showed relief from the 30 m Medium Resolution Digital Elevation Model of Natural Resources Canada. Nest locations were not shown. The inset placed the study area on Vancouver Island. The main map was drawn at a scale of 1:400,000. Map lines delineate study areas and do not necessarily depict accepted national boundaries.

</div>

Heights were normalized to the ground by a triangulated surface on the thinned ground points, and points below 0 m or at or above 100 m were discarded. Following the grid and definitions of Cosgrove et al. (2024), forest height, forest vertical complexity and internal forest gaps were computed in 1 ha cells with lidR (<span class="nocase">Roussel et al.</span>, 2020). Forest height was the 95th percentile of all point heights, ground points included, and forest vertical complexity was their standard deviation. Internal forest gaps were the number of empty 1 m cubes, or voxels, below the highest filled voxel of each 1 m column, summed over the cell. An empty voxel could be open space or space that no laser pulse reached, and the two could not be told apart. Internal forest gaps depended on point density, so they were not comparable with those of Cosgrove et al. (2024, p. 3), whose Clayoquot Sound scans averaged 29.9 points per square meter. Two scan checks, the share of 1 m columns with returns and the point density, were used only to screen cells.

``` r
structure_metrics <- function(X, Y, Z) {
  voxels <- unique(data.table::data.table(x = floor(X), y = floor(Y), z = floor(Z)))
  columns <- voxels[, list(n = .N, top = max(z)), by = c("x", "y")]
  list(
    fh = quantile(Z, 0.95),
    fvc = sd(Z),
    empty_gaps = sum(columns$top + 1 - columns$n),
    cover = nrow(columns) / 1e4,
    density = length(Z) / 1e4)
}
build_once("../02.inputs/derived/metrics.tif", {
  ctg <- lidR::readLAScatalog("../02.inputs/derived/thin")
  lidR::opt_chunk_size(ctg) <- 1000
  lidR::opt_chunk_buffer(ctg) <- 20
  lidR::opt_chunk_alignment(ctg) <- c(0, 0)
  lidR::opt_stop_early(ctg) <- FALSE
  lidR::opt_progress(ctg) <- FALSE
  lidR::catalog_map(ctg, \(las) las |>
      lidR::normalize_height(lidR::tin()) |>
      lidR::filter_poi(Z >= 0, Z < 100) |>
      lidR::pixel_metrics(~structure_metrics(X, Y, Z), res = 100, start = c(0, 0))) |>
    terra::writeRaster("../02.inputs/derived/metrics.tif")
})
```

Available forest was unharvested since 1998 and at least 20 m tall, as in Cosgrove et al. (2024), though here the 20 m applied to each cell’s forest height. A cell counted as harvested when any 30 m pixel in it had been harvested from 1998 to 2022. The screen ran to the product’s last year, so cells cut after the scans were also left out. The product recorded only stand-replacing harvest, so partial cutting and natural disturbance were not screened out. A cell cut before 1998 could still count as available forest if its forest height reached 20 m by 2019. The 20 m minimum was lower than the height at which nest platforms began to appear regularly, in trees over 30 m tall (Burger et al., 2010). Cells where returns reached fewer than 90 percent of 1 m columns, or fewer than 10 points per square meter after thinning, were also left out. Neighboring tiles were processed together, so cells lost columns only at the outer edge of the downloaded tiles. These cells were incompletely scanned, so their structure was not comparable with that of other cells. Internal forest gaps were summed only over columns with returns, so missing columns lowered a cell’s count, while sparse points raised it. Nest cells were available-forest cells with a nest, and background cells were all available-forest cells in the background tiles (<a href="#tbl-sample" class="quarto-xref">Table 2</a>). The nest sample was small, and the nests were found 17 to 19 years before the scans were flown. Each step from the nest file and scans to the modeled sample was listed with its data, dates, operation and output in <a href="#tbl-steps" class="quarto-xref">Table 3</a>.

``` r
metrics <- terra::rast("../02.inputs/derived/metrics.tif")
cells <- terra::rast("../02.inputs/ntems/harvest_cs.tif") |>
  (\(r) terra::project(r >= 1998, metrics, method = "max"))() |>
  stats::setNames("cut") |>
  c(metrics) |>
  terra::as.data.frame(xy = TRUE, cells = TRUE) |>
  dplyr::mutate(
    forest = dplyr::coalesce(fh >= 20 & cover >= 0.9 & density >= 10 & dplyr::coalesce(cut, 0) == 0, FALSE),
    nest = cell %in% terra::cellFromXY(metrics, sf::st_coordinates(nests)),
    background = !nest & lengths(sf::st_intersects(
      sf::st_as_sf(dplyr::pick(x, y), coords = c("x", "y"), crs = 3157),
      dplyr::filter(tiles, background))) > 0)
scanned <- dplyr::filter(cells, nest, !is.na(fh))
build_once("../03.outputs/tables/T2_sample.csv", tibble::tibble(
  step = c("nests in file", "nests found in 2000", "nests found in 2001", "nests found in 2002", "nest cells scanned", "nest cells under 20 m tall",
    "nest cells with returns in under 90 percent of 1 m columns", "nest cells under 10 points per square meter",
    "nest cells harvested from 1998 to 2022", "nest cells in available forest",
    "background cells scanned", "background cells in available forest", "lowest forest height of nest cells removed, m"),
  n = c(nrow(nests), sum(nests$Date == 2000), sum(nests$Date == 2001), sum(nests$Date == 2002), nrow(scanned), sum(scanned$fh < 20), sum(scanned$cover < 0.9), sum(scanned$density < 10),
    sum(dplyr::coalesce(scanned$cut, 0) > 0), sum(scanned$forest),
    sum(cells$background & !is.na(cells$fh)), sum(cells$background & cells$forest), min(scanned$fh[!scanned$forest]))) |>
  readr::write_csv("../03.outputs/tables/T2_sample.csv"))
sample <- cells |>
  dplyr::filter(forest, nest | background) |>
  dplyr::mutate(p = as.integer(nest))
```

``` r
tbl("T2_sample.csv") |>
  dplyr::filter(!grepl(", m$|^nests found in", step)) |>
  dplyr::transmute(Step = stringr::str_to_sentence(step) |> stringr::str_replace(" percent", "%") |> stringr::str_replace("points per square meter", "points per m²"), Count = formatC(n, format = "d", big.mark = ",")) |>
  flextable::flextable() |> flextable::align(j = 2, align = "right", part = "all") |> flextable::set_table_properties(layout = "autofit", width = 1)
```

<div id="tbl-sample">

Table 2: Nests and 1 ha cells at each step from the nest file to the modeled sample. Available forest was at least 20 m tall and unharvested from 1998 to 2022. It also passed the two scan checks, returns in at least 90% of 1 m columns and at least 10 points per m². A cell could fail more than one screen.

<div class="cell-output-display">

| Step                                                | Count |
|:----------------------------------------------------|:------|
| Nests in file                                       | 36    |
| Nest cells scanned                                  | 20    |
| Nest cells under 20 m tall                          | 0     |
| Nest cells with returns in under 90% of 1 m columns | 4     |
| Nest cells under 10 points per m²                   | 4     |
| Nest cells harvested from 1998 to 2022              | 0     |
| Nest cells in available forest                      | 16    |
| Background cells scanned                            | 7,869 |
| Background cells in available forest                | 3,827 |

</div>

</div>

<div id="tbl-steps">

Table 3: Steps from the nest file and scans to the modeled sample, with the data, dates, operation and output of each.

| Step | Data | Dates | Operation | Output |
|----|----|----|----|----|
| Nests | Movebank nest file (Cooke et al., 2022) | 2000 to 2002 | Nests located by radio telemetry, position error about 50 m | 36 nests |
| Scan tiles | LidarBC tile index | 2019 | Tiles within 150 m of a nest, and 40 drawn at random | 69 tiles |
| Point clouds | LidarBC compressed LAS files | 2019 | Contractor’s noise and withheld points dropped, thinned to at most 12 points per m² | Thinned tiles |
| Heights | Thinned points | 2019 | Normalized to a triangulated ground surface, 0 to 100 m kept | Heights above ground |
| Measures | Heights above ground | 2019 | Five measures in 1 ha cells | Grid of 1 ha cells |
| Harvest | Canada Forest Harvest (Hermosilla et al., 2016) | 1998 to 2022 | Stand-replacing harvest on a 30 m grid | Harvested cells |
| Available forest | Measures and harvest | 2019 | Height, harvest and scan checks | 16 nest and 3,827 background cells |
| Separability | Available forest | 2019 | *M* and Wilcoxon rank-sum test for each measure | <a href="#tbl-separability" class="quarto-xref">Table 4</a> |
| Model | Available forest | 2019 | MaxEnt on two measures, five folds of 10 km hexagonal blocks, repeated over 20 other assignments | Held-out predictions |
| Noise test | Thinned points | 2019 | Statistical outlier removal, refit on the same cells and folds | Paired AUC comparison |
| Larger cells | Thinned points | 2019 | Measures, screens, separability tests and model repeated in 4 and 9 ha cells | Separability and held-out predictions at 4 and 9 ha |
| Refits | Measures and harvest | 2019 | Sample redrawn without the scan checks, with nest-tile background, or with a 30 m minimum height, and the model refitted | Held-out predictions for each refit |

</div>

## Nest-site separability

On each measure, the separability of nest and background cells was given by the index *M* of Kaufman and Remer (1994). *M* was the absolute difference between the two means divided by the sum of the two standard deviations. The difference was tested with a Wilcoxon rank-sum test, as in Cosgrove et al. (2024). For each measure, the Spearman correlation with forest height, the skewness and a Shapiro-Wilk test of normality were computed over all cells.

``` r
build_once("../03.outputs/tables/T4_separability.csv", sample |>
  dplyr::mutate(height = fh) |>
  tidyr::pivot_longer(c(fvc, empty_gaps, fh, cover, density), names_to = "measure") |>
  dplyr::summarise(.by = measure,
    nest_mean = mean(value[p == 1]), nest_sd = sd(value[p == 1]),
    nest_low = t.test(value[p == 1])$conf.int[1], nest_high = t.test(value[p == 1])$conf.int[2],
    background_mean = mean(value[p == 0]), background_sd = sd(value[p == 0]),
    background_low = t.test(value[p == 0])$conf.int[1], background_high = t.test(value[p == 0])$conf.int[2],
    m = abs(nest_mean - background_mean) / (nest_sd + background_sd),
    w = wilcox.test(value[p == 1], value[p == 0])$statistic,
    p_value = wilcox.test(value[p == 1], value[p == 0])$p.value,
    rho = cor(value, height, method = "spearman"),
    skewness = mean(((value - mean(value)) / sd(value))^3),
    sw = shapiro.test(value)$statistic,
    sw_p = shapiro.test(value)$p.value) |>
  readr::write_csv("../03.outputs/tables/T4_separability.csv"))
```

## Model comparison

Following the top model of Cosgrove et al. (2024), MaxEnt was fitted to forest vertical complexity and internal forest gaps in the maxnet package (Phillips et al., 2017). The model used linear and product features and a regularization multiplier of 0.2. Predictions were taken on the complementary log-log scale, a 0 to 1 index of relative suitability. For spatial block cross-validation, cells were grouped in hexagonal blocks 10 km across, and the blocks were assigned at random to five folds (Valavi et al., 2019). Of 100 random assignments, the one that spread nest and background cells most evenly among the folds was kept. Each fold was predicted by a model trained on the other four. The assignment was repeated with 20 other random seeds, to show how far held-out AUC depended on one assignment. Each model’s classification threshold maximized the sum of sensitivity and specificity on its training cells (Liu et al., 2005). Sensitivity was the share of nest cells classed as nesting habitat, and specificity was the share of background cells not classed as nesting habitat. Average accuracy was the mean of sensitivity and specificity.

``` r
folds <- sample |>
  sf::st_as_sf(coords = c("x", "y"), crs = 3157) |>
  blockCV::cv_spatial(column = "p", k = 5, size = 10000, selection = "random",
    seed = 2019, plot = FALSE, progress = FALSE, report = FALSE)
fit_maxent <- function(d) {
  x <- as.data.frame(d[c("fvc", "empty_gaps")])
  maxnet::maxnet(d$p, x, maxnet::maxnet.formula(d$p, x, classes = "lp"), regmult = 0.2)
}
score <- \(fit, d) predict(fit, as.data.frame(d[c("fvc", "empty_gaps")]), type = "cloglog")[, 1]
runs <- purrr::map(1:5, \(k) {
  train <- sample[folds$folds_ids != k, ]
  fit <- fit_maxent(train)
  threshold <- pROC::coords(pROC::roc(train$p, score(fit, train), levels = c(0, 1), direction = "<", quiet = TRUE), "best", ret = "threshold")$threshold[1]
  test <- sample[folds$folds_ids == k, ] |> dplyr::mutate(fold = k)
  list(test = dplyr::mutate(test, pred = score(fit, test), class = as.integer(pred >= threshold)), train_rmse = sqrt(mean((score(fit, train) - train$p)^2)),
    train_auc = as.numeric(pROC::auc(pROC::roc(train$p, score(fit, train), levels = c(0, 1), direction = "<", quiet = TRUE))))
})
held_out <- purrr::map(runs, "test") |> purrr::list_rbind()
```

Held-out predictions were pooled across folds. Discrimination was measured by AUC with a DeLong 95 percent confidence interval (CI) (Robin et al., 2011) and the continuous Boyce index (Hirzel et al., 2006). The Boyce index took a 95 percent interval from 1,000 bootstrap resamples of the held-out nest and background cells.

Classification at each fold’s threshold was summarized by sensitivity, specificity, overall accuracy with an exact binomial interval, Cohen’s kappa and the commission and omission error of each class. Agreement between prediction and the observed 0 or 1 outcome was measured by root mean square error (RMSE), relative RMSE, mean absolute error and the Brier score. Relative RMSE was RMSE over the mean outcome. The coefficient of determination (*R*²) of outcome on prediction was tested with its *F* test. Theil’s *U* bias gave the share of squared error due to the gap between mean prediction and mean outcome.

Overfitting was measured by the RMSE ratio, held-out over training RMSE, with a model rejected above 1.1 (Murphy et al., 2026). Training AUC, from each fold’s model scored on its own training cells, was set beside held-out AUC as a second test. Residual autocorrelation was measured by Moran’s *I* of prediction minus outcome, with inverse-distance weights. The Durbin-Watson test was applied to residuals of the regression of outcome on prediction, ordered by easting. The model was compared with that of Cosgrove et al. (2024), in Desolation Sound and transferred to Clayoquot Sound, on the AUC and Boyce index those authors reported. The Desolation Sound scores came from bootstrapped random splits with 70 percent of the data for training, not from spatial blocks (Cosgrove et al., 2024). The transferred model’s scores came from 21 nests and a random background sample (Cosgrove et al., 2024), so the local and transferred models were not scored on the same cells.

``` r
roc <- pROC::roc(held_out$p, held_out$pred, levels = c(0, 1), direction = "<", quiet = TRUE)
auc <- pROC::ci.auc(roc)
cm <- table(predicted = factor(held_out$class, 0:1), observed = factor(held_out$p, 0:1))
n <- sum(cm); oa <- sum(diag(cm)) / n; pe <- sum(rowSums(cm) * colSums(cm)) / n^2
kappa <- (oa - pe) / (1 - pe); kappa_se <- sqrt(oa * (1 - oa) / (n * (1 - pe)^2))
e <- held_out$pred - held_out$p
r2 <- summary(lm(p ~ pred, held_out))
w <- 1 / as.matrix(dist(held_out[c("x", "y")])); diag(w) <- 0
moran <- ape::Moran.I(e, w)
dw <- lmtest::dwtest(p ~ pred, data = dplyr::arrange(held_out, x, y))
# Desolation Sound scores from Cosgrove et al. (2024), Avian Conservation and Ecology 19(1):5
# https://doi.org/10.5751/ACE-02585-190105
boyce <- \(d) ecospat::ecospat.boyce(d$pred[d$p == 0], d$pred[d$p == 1], PEplot = FALSE)$cor
build_once("../03.outputs/tables/T5_evaluation.csv", {
  # Held-out AUC with the 10 km blocks allocated to folds by 20 other seeds
  draws <- purrr::map_dbl(1:20, \(s) {
    f <- blockCV::cv_spatial(sf::st_as_sf(sample, coords = c("x", "y"), crs = 3157), column = "p", k = 5, size = 10000,
      selection = "random", seed = s, plot = FALSE, progress = FALSE, report = FALSE)$folds_ids
    h <- purrr::map(1:5, \(k) dplyr::mutate(sample[f == k, ], pred = score(fit_maxent(sample[f != k, ]), sample[f == k, ]))) |> purrr::list_rbind()
    as.numeric(pROC::auc(pROC::roc(h$p, h$pred, levels = c(0, 1), direction = "<", quiet = TRUE)))
  })
  # Boyce index on 1,000 bootstrap resamples of the held-out nest and background cells
  boot <- withr::with_seed(2019, replicate(1000, boyce(dplyr::slice_sample(held_out, prop = 1, replace = TRUE, by = p))))
  train_auc <- purrr::map_dbl(runs, "train_auc")
  per_fold <- tabulate(held_out$fold[held_out$p == 1], 5)
  tibble::tribble(
  ~model, ~metric, ~value, ~low, ~high, ~p_value,
  "local", "Nest cells", sum(held_out$p), NA, NA, NA,
  "local", "Background cells", sum(held_out$p == 0), NA, NA, NA,
  "local", "Nest cells above threshold", cm["1", "1"], NA, NA, NA,
  "local", "Background cells above threshold", cm["1", "0"], NA, NA, NA,
  "local", "Mean prediction", mean(held_out$pred), NA, NA, NA,
  "local", "Nest share of cells", mean(held_out$p), NA, NA, NA,
  "local", "F statistic", r2$fstatistic[1], r2$fstatistic[2], r2$fstatistic[3], NA,
  "local", "AUC", auc[2], auc[1], auc[3], NA,
  "local", "Boyce index", boyce(held_out), unname(quantile(boot, 0.025, na.rm = TRUE)), unname(quantile(boot, 0.975, na.rm = TRUE)), NA,
  "local", "Training AUC", mean(train_auc), t.test(train_auc)$conf.int[1], t.test(train_auc)$conf.int[2], NA,
  "local", "Training AUC SD", sd(train_auc), NA, NA, NA,
  "local", "AUC over 20 fold draws", mean(draws), min(draws), max(draws), NA,
  "local", "AUC over 20 fold draws SD", sd(draws), NA, NA, NA,
  "local", "Fewest nest cells in a fold", min(per_fold), NA, NA, NA,
  "local", "Most nest cells in a fold", max(per_fold), NA, NA, NA,
  "local", "Overall accuracy", oa, binom.test(sum(diag(cm)), n)$conf.int[1], binom.test(sum(diag(cm)), n)$conf.int[2], NA,
  "local", "Kappa", kappa, kappa - 1.96 * kappa_se, kappa + 1.96 * kappa_se, NA,
  "local", "Commission error, nest", cm["1", "0"] / sum(cm["1", ]), NA, NA, NA,
  "local", "Omission error, nest", cm["0", "1"] / sum(cm[, "1"]), NA, NA, NA,
  "local", "Commission error, background", cm["0", "1"] / sum(cm["0", ]), NA, NA, NA,
  "local", "Omission error, background", cm["1", "0"] / sum(cm[, "0"]), NA, NA, NA,
  "local", "R2", r2$r.squared, NA, NA, pf(r2$fstatistic[1], r2$fstatistic[2], r2$fstatistic[3], lower.tail = FALSE),
  "local", "RMSE", sqrt(mean(e^2)), NA, NA, NA,
  "local", "RMSE ratio", sqrt(mean(e^2)) / mean(purrr::map_dbl(runs, "train_rmse")), NA, NA, NA,
  "local", "Relative RMSE", sqrt(mean(e^2)) / mean(held_out$p), NA, NA, NA,
  "local", "MAE", mean(abs(e)), NA, NA, NA,
  "local", "Theil's U bias", (mean(held_out$pred) - mean(held_out$p))^2 / mean(e^2), NA, NA, NA,
  "local", "Brier score", mean(e^2), NA, NA, NA,
  "local", "Moran's I of residuals", moran$observed, NA, NA, moran$p.value,
  "local", "Durbin-Watson", unname(dw$statistic), NA, NA, dw$p.value,
  "desolation", "AUC", 0.77, NA, NA, NA,
  "desolation", "Boyce index", 0.99, NA, NA, NA,
  "transferred", "AUC", 0.63, NA, NA, NA,
  "transferred", "Boyce index", 0.67, NA, NA, NA) |>
  readr::write_csv("../03.outputs/tables/T5_evaluation.csv")
})
```

To test whether further noise removal changed the result, statistical outlier removal was applied to all thinned points with lidR, ground points included (<span class="nocase">Roussel et al.</span>, 2020). A point was removed when its mean distance to its 10 nearest neighbors exceeded the average for its 1 km processing block by more than 3 standard deviations. Any ground point removed was also left out of the triangulated ground surface. Forest vertical complexity and internal forest gaps were then recomputed, and the model was refitted on the same cells and folds. The two held-out AUCs were compared with a paired DeLong test (Robin et al., 2011).

Each nest position carried an error of about 50 m, so a 1 ha cell could hold only part of the forest around a nest. For California spotted owls, North et al. (2017, p. 168) defined the nest site as “a four-hectare area immediately surrounding each nest tree or snag”. The separability tests and the model were therefore repeated in 4 and 9 ha cells, on 200 and 300 m grids aligned with the 1 ha grid. Each measure was recomputed from the points of each larger cell, and a larger cell holding more than one nest counted as one nest cell. The 1 ha formulas for internal forest gaps and the two scan checks were scaled to each cell’s area, so the 1 ha screens applied unchanged. The same height and harvest screens and model settings were used at each size. Hexagonal blocks 10 km across were drawn over the cells of each size and reassigned to five folds.

Three refits in 1 ha cells tested choices behind the sample. The first dropped the two scan checks, so that all 20 scanned nest cells entered. The second added the available-forest cells of the nest tiles to the background. The third raised the minimum forest height to 30 m, above which nest platforms appeared regularly (Burger et al., 2010). The block size, regularization multiplier, feature classes, thinning density and draw of background tiles were not varied. These alternative analyses were exploratory, so their *P* values were not adjusted for multiple tests.

Analyses were run in R 4.6.2 (R Core Team, 2024) with lidR 4.3.3, maxnet 0.1.4, blockCV 4.0.1, pROC 1.19.1 and ecospat 4.1.4 (Di Cola et al., 2017).

``` r
build_once("../02.inputs/derived/metrics_sor.tif", {
  ctg <- lidR::readLAScatalog("../02.inputs/derived/thin")
  lidR::opt_chunk_size(ctg) <- 1000
  lidR::opt_chunk_buffer(ctg) <- 20
  lidR::opt_chunk_alignment(ctg) <- c(0, 0)
  lidR::opt_stop_early(ctg) <- FALSE
  lidR::opt_progress(ctg) <- FALSE
  lidR::opt_select(ctg) <- "xyzc"
  future::plan(future::multisession, workers = 2)
  lidR::catalog_map(ctg, \(las) las |>
      lidR::classify_noise(lidR::sor(k = 10, m = 3)) |>
      lidR::normalize_height(lidR::tin()) |>
      lidR::filter_poi(Z >= 0, Z < 100) |>
      lidR::pixel_metrics(~c(structure_metrics(X[Classification != lidR::LASNOISE], Y[Classification != lidR::LASNOISE], Z[Classification != lidR::LASNOISE]), list(removed = mean(Classification == lidR::LASNOISE))), res = 100, start = c(0, 0))) |>
    terra::writeRaster("../02.inputs/derived/metrics_sor.tif")
  future::plan(future::sequential)
})
build_once("../03.outputs/tables/T6_noise.csv", {
  cleaned <- sample |>
    dplyr::select(-fvc, -empty_gaps) |>
    dplyr::mutate(terra::extract(terra::rast("../02.inputs/derived/metrics_sor.tif")[[c("fvc", "empty_gaps", "removed")]], as.matrix(dplyr::pick(x, y))))
  paired <- purrr::map(1:5, \(k) {
    fit <- fit_maxent(cleaned[folds$folds_ids != k, ])
    test <- cleaned[folds$folds_ids == k, ]
    dplyr::transmute(test, cell, pred_sor = score(fit, test))
  }) |> purrr::list_rbind() |> dplyr::inner_join(held_out, by = "cell")
  roc_sor <- pROC::roc(paired$p, paired$pred_sor, levels = c(0, 1), direction = "<", quiet = TRUE)
  roc_vendor <- pROC::roc(paired$p, paired$pred, levels = c(0, 1), direction = "<", quiet = TRUE)
  delong <- pROC::roc.test(roc_vendor, roc_sor, method = "delong", paired = TRUE)
  removed <- t.test(100 * cleaned$removed)
  tibble::tribble(
    ~model, ~metric, ~value, ~low, ~high, ~p_value,
    "contractor classes", "AUC", pROC::ci.auc(roc_vendor)[2], pROC::ci.auc(roc_vendor)[1], pROC::ci.auc(roc_vendor)[3], NA,
    "outlier removal", "AUC", pROC::ci.auc(roc_sor)[2], pROC::ci.auc(roc_sor)[1], pROC::ci.auc(roc_sor)[3], NA,
    "outlier removal", "DeLong Z", unname(delong$statistic), NA, NA, delong$p.value,
    "outlier removal", "Points removed, percent", unname(removed$estimate), removed$conf.int[1], removed$conf.int[2], NA,
    "outlier removal", "Points removed SD, percent", sd(100 * cleaned$removed), NA, NA, NA,
    "outlier removal", "Cells compared", nrow(paired), NA, NA, NA) |>
    readr::write_csv("../03.outputs/tables/T6_noise.csv")
})
```

``` r
purrr::walk(c(200, 300), \(res) build_once(sprintf("../02.inputs/derived/metrics_%d.tif", res), {
  # LidarBC 2019 point clouds, https://lidar.gov.bc.ca/
  ctg <- lidR::readLAScatalog("../02.inputs/derived/thin")
  # Chunks two cells wide, read one at a time, kept the pass within an 8 GB machine
  lidR::opt_chunk_size(ctg) <- 2 * res
  lidR::opt_chunk_buffer(ctg) <- 20
  lidR::opt_chunk_alignment(ctg) <- c(0, 0)
  lidR::opt_stop_early(ctg) <- FALSE
  lidR::opt_progress(ctg) <- FALSE
  lidR::opt_select(ctg) <- "xyzc"
  future::plan(future::sequential)
  lidR::catalog_map(ctg, \(las) las |>
      lidR::normalize_height(lidR::tin()) |>
      lidR::filter_poi(Z >= 0, Z < 100) |>
      lidR::pixel_metrics(~structure_metrics(X, Y, Z), res = res, start = c(0, 0))) |>
    terra::writeRaster(sprintf("../02.inputs/derived/metrics_%d.tif", res))
}))
```

``` r
# Repeats the sample, separability tests and model for one cell size and set of screens
rerun <- function(res = 100, screens = TRUE, nest_tiles = FALSE, min_height = 20) {
  grid <- terra::rast(if (res == 100) "../02.inputs/derived/metrics.tif" else sprintf("../02.inputs/derived/metrics_%d.tif", res))
  # Canada Forest Harvest 1985-2022, National Terrestrial Ecosystem Monitoring System
  # https://opendata.nfis.org/downloads/forest_change/CA_Forest_Harvest_1985-2022.zip
  all <- terra::rast("../02.inputs/ntems/harvest_cs.tif") |>
    (\(r) terra::project(r >= 1998, grid, method = "max"))() |>
    stats::setNames("cut") |>
    c(grid) |>
    terra::as.data.frame(xy = TRUE, cells = TRUE) |>
    dplyr::mutate(dplyr::across(c(empty_gaps, cover, density), \(v) v / (res / 100)^2),
      forest = dplyr::coalesce(fh >= min_height & (!screens | (cover >= 0.9 & density >= 10)) & dplyr::coalesce(cut, 0) == 0, FALSE),
      nest = cell %in% terra::cellFromXY(grid, sf::st_coordinates(nests)),
      background = !nest & lengths(sf::st_intersects(
        sf::st_as_sf(dplyr::pick(x, y), coords = c("x", "y"), crs = 3157),
        if (nest_tiles) tiles else dplyr::filter(tiles, background))) > 0)
  d <- dplyr::filter(all, forest, nest | background) |> dplyr::mutate(p = as.integer(nest))
  f <- blockCV::cv_spatial(sf::st_as_sf(d, coords = c("x", "y"), crs = 3157), column = "p", k = 5, size = 10000,
    selection = "random", seed = 2019, plot = FALSE, progress = FALSE, report = FALSE)$folds_ids
  r <- purrr::map(1:5, \(k) {
    train <- d[f != k, ]
    fit <- fit_maxent(train)
    threshold <- pROC::coords(pROC::roc(train$p, score(fit, train), levels = c(0, 1), direction = "<", quiet = TRUE), "best", ret = "threshold")$threshold[1]
    list(test = dplyr::mutate(d[f == k, ], pred = score(fit, d[f == k, ]), class = as.integer(pred >= threshold)),
      train_rmse = sqrt(mean((score(fit, train) - train$p)^2)),
      train_auc = as.numeric(pROC::auc(pROC::roc(train$p, score(fit, train), levels = c(0, 1), direction = "<", quiet = TRUE))))
  })
  h <- purrr::map(r, "test") |> purrr::list_rbind()
  boot <- withr::with_seed(2019, replicate(1000, boyce(dplyr::slice_sample(h, prop = 1, replace = TRUE, by = p))))
  ta <- purrr::map_dbl(r, "train_auc")
  auc <- pROC::ci.auc(pROC::roc(h$p, h$pred, levels = c(0, 1), direction = "<", quiet = TRUE))
  cm <- table(predicted = factor(h$class, 0:1), observed = factor(h$p, 0:1))
  n <- sum(cm); oa <- sum(diag(cm)) / n; pe <- sum(rowSums(cm) * colSums(cm)) / n^2
  kappa <- (oa - pe) / (1 - pe); kappa_se <- sqrt(oa * (1 - oa) / (n * (1 - pe)^2))
  sens <- cm["1", "1"] / sum(cm[, "1"]); spec <- cm["0", "0"] / sum(cm[, "0"])
  e <- h$pred - h$p
  w <- 1 / as.matrix(dist(h[c("x", "y")])); diag(w) <- 0
  moran <- ape::Moran.I(e, w)
  separation <- purrr::map(c("fvc", "empty_gaps", "fh"), \(k) {
    a <- d[[k]][d$p == 1]; b <- d[[k]][d$p == 0]; test <- wilcox.test(a, b)
    tibble::tibble(metric = paste(k, c("nest", "background", "M", "W")),
      value = c(mean(a), mean(b), abs(mean(a) - mean(b)) / (sd(a) + sd(b)), unname(test$statistic)),
      sd = c(sd(a), sd(b), NA, NA),
      low = c(t.test(a)$conf.int[1], t.test(b)$conf.int[1], NA, NA),
      high = c(t.test(a)$conf.int[2], t.test(b)$conf.int[2], NA, NA),
      p_value = c(NA, NA, NA, test$p.value))
  }) |> purrr::list_rbind()
  # Without the scan checks, the same predictions scored against background cells that pass them,
  # and the background cells that fail them, with those cut by the edge of the downloaded tiles
  extra <- if (screens) NULL else {
    pass <- h$cover >= 0.9 & h$density >= 10
    rp <- pROC::ci.auc(pROC::roc(h$p[h$p == 1 | pass], h$pred[h$p == 1 | pass], levels = c(0, 1), direction = "<", quiet = TRUE))
    fail <- dplyr::filter(d, p == 0, !(cover >= 0.9 & density >= 10))
    inside <- sf::st_as_sf(fail, coords = c("x", "y"), crs = 3157) |>
      sf::st_buffer(res / 2, endCapStyle = "SQUARE") |>
      sf::st_intersection(sf::st_union(dplyr::filter(tiles, nest | background))) |>
      sf::st_area() |> as.numeric() / res^2
    tibble::tribble(
      ~metric, ~value, ~sd, ~low, ~high, ~p_value,
      "AUC against passing background", rp[2], NA, rp[1], rp[3], NA,
      "Background cells failing the checks", nrow(fail), NA, NA, NA, NA,
      "Failing background cells cut by the tile edge", sum(inside < 0.999), NA, NA, NA, NA,
      "empty_gaps failing background", mean(fail$empty_gaps), sd(fail$empty_gaps), t.test(fail$empty_gaps)$conf.int[1], t.test(fail$empty_gaps)$conf.int[2], NA)
  }
  tibble::tribble(
    ~metric, ~value, ~sd, ~low, ~high, ~p_value,
    "Nest cells scanned", sum(all$nest & !is.na(all$fh)), NA, NA, NA, NA,
    "Nest cells", sum(d$p), NA, NA, NA, NA,
    "Background cells", sum(d$p == 0), NA, NA, NA, NA,
    "AUC", auc[2], NA, auc[1], auc[3], NA,
    "Training AUC", mean(ta), sd(ta), t.test(ta)$conf.int[1], t.test(ta)$conf.int[2], NA,
    "Boyce index", boyce(h), NA, unname(quantile(boot, 0.025, na.rm = TRUE)), unname(quantile(boot, 0.975, na.rm = TRUE)), NA,
    "Sensitivity", sens, NA, NA, NA, NA,
    "Specificity", spec, NA, NA, NA, NA,
    "Average accuracy", (sens + spec) / 2, NA, NA, NA, NA,
    "Overall accuracy", oa, NA, binom.test(sum(diag(cm)), n)$conf.int[1], binom.test(sum(diag(cm)), n)$conf.int[2], NA,
    "Kappa", kappa, NA, kappa - 1.96 * kappa_se, kappa + 1.96 * kappa_se, NA,
    "RMSE ratio", sqrt(mean(e^2)) / mean(purrr::map_dbl(r, "train_rmse")), NA, NA, NA, NA,
    "Theil's U bias", (mean(h$pred) - mean(h$p))^2 / mean(e^2), NA, NA, NA, NA,
    "Moran's I of residuals", moran$observed, NA, NA, NA, moran$p.value) |>
    dplyr::bind_rows(separation, extra) |>
    dplyr::mutate(size_ha = (res / 100)^2, .before = 1)
}
```

``` r
build_once("../03.outputs/tables/T7_cell_size.csv", purrr::map(c(100, 200, 300), rerun) |>
  purrr::list_rbind() |>
  readr::write_csv("../03.outputs/tables/T7_cell_size.csv"))
```

``` r
build_once("../03.outputs/tables/T8_refits.csv", dplyr::bind_rows(
  "Without scan checks" = rerun(screens = FALSE),
  "Nest-tile background" = rerun(nest_tiles = TRUE),
  "30 m minimum height" = rerun(min_height = 30), .id = "analysis") |>
  readr::write_csv("../03.outputs/tables/T8_refits.csv"))
```

``` r
build_once("../03.outputs/figures/F2_structure.png", {
  fig <- sample |>
    dplyr::mutate(empty_gaps = empty_gaps / 1000, group = factor(ifelse(p == 1, "Nest", "Background"), c("Background", "Nest"))) |>
    tidyr::pivot_longer(c(fvc, empty_gaps), names_to = "measure") |>
    dplyr::mutate(measure = factor(measure, c("fvc", "empty_gaps"), c("Forest vertical complexity (m)", "Internal forest gaps (thousand empty 1 m³ voxels per ha)"))) |>
    ggplot2::ggplot(ggplot2::aes(group, value)) +
    ggplot2::geom_boxplot(ggplot2::aes(fill = group), outlier.shape = NA, width = 0.5, linewidth = 0.3) +
    ggplot2::geom_jitter(data = \(d) dplyr::filter(d, group == "Nest"), width = 0.12, height = 0, size = 1.2, colour = "black") +
    ggplot2::facet_wrap(~measure, scales = "free_y") +
    ggplot2::scale_fill_manual(values = c(Background = "grey85", Nest = "#E69F00"), guide = "none") +
    ggplot2::labs(x = NULL, y = NULL) +
    ggplot2::theme_bw(base_size = 8) + ggplot2::theme(strip.background = ggplot2::element_blank(), panel.grid.major.x = ggplot2::element_blank(), panel.grid.minor = ggplot2::element_blank())
  ggplot2::ggsave("../03.outputs/figures/F2_structure.png", fig, width = 16, height = 7, units = "cm", dpi = 300)
})
```

# Results

## Sample

The 2019 scans covered 20 of the cells containing the 36 nests, and 16 of these were available forest. In the background tiles, 3,827 of the 7,869 scanned cells were available forest (<a href="#tbl-sample" class="quarto-xref">Table 2</a>).

## Nest-site separability

Neither forest vertical complexity nor internal forest gaps separated nest cells from background cells (<a href="#tbl-separability" class="quarto-xref">Table 4</a>, <a href="#fig-structure" class="quarto-xref">Figure 2</a>). Forest vertical complexity did not differ \[*M* = 0.08, Wilcoxon rank-sum test, *W* = 33,355, *P* = 0.54\]. It averaged 10.67 ± 1.47 m (95 percent CI 9.89 to 11.46) at nest cells and 10.37 ± 2.43 m (10.29 to 10.45) at background cells. Nest cells had more internal forest gaps, though not significantly \[*M* = 0.21, *W* = 37,889, *P* = 0.10\]. Internal forest gaps averaged 177.5 ± 40.9 thousand empty voxels per hectare (155.7 to 199.3) at nest cells and 157.5 ± 56.8 thousand (155.7 to 159.3) at background cells. Forest height did not differ either \[*M* = 0.09, *W* = 33,762, *P* = 0.48\]. It was 34.74 ± 5.76 m (31.66 to 37.81) at nest cells and 33.54 ± 7.81 m (33.29 to 33.78) at background cells.

Across all cells, forest vertical complexity and internal forest gaps rose with forest height, with Spearman correlations of 0.96 and 0.86. Point density did not differ between nest and background cells \[*M* = 0.03, *W* = 30,220, *P* = 0.93\]. It was 11.80 ± 0.41 (11.58 to 12.02) points per square meter at nest cells and 11.78 ± 0.41 (11.76 to 11.79) at background cells. No measure was normally distributed \[Shapiro-Wilk test, *P* \< 0.001 for each\].

``` r
star <- \(p) dplyr::case_when(is.na(p) ~ "", p <= 0.0001 ~ "****", p <= 0.001 ~ "***", p <= 0.01 ~ "**", p <= 0.05 ~ "*", TRUE ~ "")
tbl("T4_separability.csv") |>
  dplyr::mutate(k = dplyr::case_when(measure == "empty_gaps" ~ 1000, measure == "cover" ~ 0.01, TRUE ~ 1), dplyr::across(nest_mean:background_high, \(x) x / k)) |>
  dplyr::mutate(dplyr::across(c(nest_high, background_high), \(x) dplyr::if_else(measure == "cover", pmin(x, 100), x))) |>
  dplyr::arrange(match(measure, c("fvc", "empty_gaps", "fh", "cover", "density"))) |>
  dplyr::transmute(Measure = c(fvc = "Forest vertical complexity (m)", empty_gaps = "Internal forest gaps (thousand voxels per ha)", fh = "Forest height (m)", cover = "Columns with returns (%)", density = "Point density (points per m²)")[measure],
    Nest = paste0(fmt(nest_mean), " ± ", fmt(nest_sd), " (", fmt(nest_low), "–", fmt(nest_high), ")"),
    Background = paste0(fmt(background_mean), " ± ", fmt(background_sd), " (", fmt(background_low), "–", fmt(background_high), ")"),
    M = fmt(m), W = paste0(fmt(w, 0), star(p_value)), `ρ` = fmt(rho), Skewness = fmt(skewness), `Shapiro-Wilk` = paste0(fmt(sw, 3), star(sw_p))) |>
  flextable::flextable() |> flextable::align(j = 2:8, align = "right", part = "body") |> flextable::align(align = "center", part = "header") |> flextable::set_table_properties(layout = "autofit", width = 1)
```

<div id="tbl-separability">

Table 4: Laser-scan measures at nest and background cells in available forest, as mean ± standard deviation (95% confidence interval). M was the separability index and W the Wilcoxon rank-sum statistic of the two groups. The Spearman correlation ρ with forest height, the skewness and the Shapiro-Wilk statistic were computed over all cells. The first two measures entered the model and the last three defined available forest. Intervals for columns with returns were truncated at 100%. \* P ≤ 0.05, \*\* P ≤ 0.01, \*\*\* P ≤ 0.001, \*\*\*\* P ≤ 0.0001.

<div class="cell-output-display">

| Measure | Nest | Background | M | W | ρ | Skewness | Shapiro-Wilk |
|:---|:---|:---|:---|:---|:---|:---|:---|
| Forest vertical complexity (m) | 10.67 ± 1.47 (9.89–11.46) | 10.37 ± 2.43 (10.29–10.45) | 0.08 | 33,355 | 0.96 | 0.14 | 0.994\*\*\*\* |
| Internal forest gaps (thousand voxels per ha) | 177.52 ± 40.90 (155.73–199.31) | 157.46 ± 56.84 (155.66–159.26) | 0.21 | 37,889 | 0.86 | 0.35 | 0.984\*\*\*\* |
| Forest height (m) | 34.74 ± 5.76 (31.66–37.81) | 33.54 ± 7.81 (33.29–33.78) | 0.09 | 33,762 | 1.00 | 0.16 | 0.975\*\*\*\* |
| Columns with returns (%) | 99.23 ± 1.60 (98.37–100.00) | 99.46 ± 1.31 (99.41–99.50) | 0.08 | 24,994 | -0.36 | -3.84 | 0.491\*\*\*\* |
| Point density (points per m²) | 11.80 ± 0.41 (11.58–12.02) | 11.78 ± 0.41 (11.76–11.79) | 0.03 | 30,220 | -0.03 | -2.03 | 0.683\*\*\*\* |

</div>

</div>

``` r
knitr::include_graphics("../03.outputs/figures/F2_structure.png")
```

<div id="fig-structure">

![](../03.outputs/figures/F2_structure.png)

Figure 2: Forest vertical complexity and internal forest gaps at 16 nest cells (points) and 3,827 background cells in available forest, Clayoquot Sound. Boxes spanned the middle half of the values, the bar marked the median, and whiskers reached the furthest value within 1.5 box lengths. Values of background cells beyond the whiskers were not drawn.

</div>

## Model comparison

The local model’s ranking of held-out nest cells above background cells could not be distinguished from chance (<a href="#tbl-evaluation" class="quarto-xref">Table 5</a>). Its AUC was 0.53 (95 percent CI 0.39 to 0.67). The published values were 0.77 for the Desolation Sound model in its own area and 0.63 for that model transferred to Clayoquot Sound (Cosgrove et al., 2024). The local model’s Boyce index was 0.10 (95 percent bootstrap interval -0.59 to 0.70), against 0.99 and 0.67. At the training thresholds, 10 of 16 held-out nest cells and 2,264 of 3,827 background cells were classed as nesting habitat. Sensitivity was 62.5 percent and specificity 40.8 percent, for an average accuracy of 51.7 percent. Cohen’s kappa, the agreement beyond chance, was 0.00 (95 percent CI -0.03 to 0.03).

Nest cells per fold ranged from 2 to 6. The RMSE ratio of 1.01 lay below the rejection threshold of 1.1. Nest cells added almost nothing to the squared error, however, so the ratio could not have detected overfitting. Training AUC averaged 0.64 ± 0.04 across the five folds (95 percent CI 0.60 to 0.69), against 0.53 on held-out cells. Over 20 other assignments of blocks to folds, held-out AUC averaged 0.51 ± 0.04 and ranged from 0.44 to 0.56. Nearly all of the model’s error was systematic \[Theil’s *U* bias = 0.96\], because predictions averaged 0.60 while nest cells made up 0.004 of all cells. Residuals were spatially autocorrelated \[Moran’s *I* = 0.12, *P* \< 0.001\].

Statistical outlier removal took out 1.67 ± 1.27 percent of the points in each cell (95 percent CI 1.63 to 1.71). Refitted on the same cells and folds, the model did not differ in held-out AUC from the model with the contractor’s classes alone \[paired DeLong test, *Z* = 0.61, *P* = 0.54\]. The two scored 0.528 (95 percent CI 0.385 to 0.672) and 0.530, respectively.

``` r
show <- c("AUC" = "AUC", "Training AUC" = "AUC on training cells, mean of five folds", "AUC over 20 fold draws" = "AUC over 20 other fold draws, mean (range)", "Boyce index" = "Boyce index", "Overall accuracy" = "Overall accuracy", "Kappa" = "Kappa", "Commission error, nest" = "Commission error, nest", "Omission error, nest" = "Omission error, nest", "Commission error, background" = "Commission error, background", "Omission error, background" = "Omission error, background", "R2" = "R²", "RMSE" = "RMSE", "RMSE ratio" = "RMSE ratio", "Relative RMSE" = "Relative RMSE", "MAE" = "MAE", "Brier score" = "Brier score", "Theil's U bias" = "Theil's U bias", "Moran's I of residuals" = "Moran's I of residuals", "Durbin-Watson" = "Durbin-Watson")
tbl("T5_evaluation.csv") |>
  dplyr::filter(metric %in% names(show)) |>
  dplyr::mutate(cell = paste0(dplyr::case_when(metric == "R2" ~ fmt(value, 5), grepl("^Commission", metric) ~ fmt(value, 3), TRUE ~ fmt(value)), dplyr::if_else(is.na(low), "", paste0(" (", fmt(low), " to ", fmt(high), ")")), star(p_value))) |>
  tidyr::pivot_wider(id_cols = metric, names_from = model, values_from = cell, values_fill = "") |>
  dplyr::arrange(match(metric, names(show))) |>
  dplyr::transmute(Measure = show[metric], `Clayoquot Sound, local` = local, `Desolation Sound, own area` = desolation, `Desolation Sound, transferred` = transferred) |>
  flextable::flextable() |> flextable::align(j = 2:4, align = "right", part = "body") |> flextable::align(align = "center", part = "header") |> flextable::set_table_properties(layout = "autofit", width = 1)
```

<div id="tbl-evaluation">

Table 5: Performance of the Clayoquot Sound model on held-out cells pooled over five spatial folds. The Desolation Sound model of Cosgrove et al. (2024) was shown beside it, in its own area and transferred to Clayoquot Sound. Intervals were 95% confidence intervals, from 1,000 bootstrap resamples for the Boyce index, and the row of fold draws gave the range. AUC was the area under the receiver operating characteristic curve, RMSE root mean square error and MAE mean absolute error. \* P ≤ 0.05, \*\* P ≤ 0.01, \*\*\* P ≤ 0.001, \*\*\*\* P ≤ 0.0001.

<div class="cell-output-display">

| Measure | Clayoquot Sound, local | Desolation Sound, own area | Desolation Sound, transferred |
|:---|:---|:---|:---|
| AUC | 0.53 (0.39 to 0.67) | 0.77 | 0.63 |
| AUC on training cells, mean of five folds | 0.64 (0.60 to 0.69) |  |  |
| AUC over 20 other fold draws, mean (range) | 0.51 (0.44 to 0.56) |  |  |
| Boyce index | 0.10 (-0.59 to 0.70) | 0.99 | 0.67 |
| Overall accuracy | 0.41 (0.39 to 0.43) |  |  |
| Kappa | 0.00 (-0.03 to 0.03) |  |  |
| Commission error, nest | 0.996 |  |  |
| Omission error, nest | 0.38 |  |  |
| Commission error, background | 0.004 |  |  |
| Omission error, background | 0.59 |  |  |
| R² | 0.00003 |  |  |
| RMSE | 0.61 |  |  |
| RMSE ratio | 1.01 |  |  |
| Relative RMSE | 145.98 |  |  |
| MAE | 0.60 |  |  |
| Brier score | 0.37 |  |  |
| Theil’s U bias | 0.96 |  |  |
| Moran’s I of residuals | 0.12\*\*\*\* |  |  |
| Durbin-Watson | 2.01 |  |  |

</div>

</div>

## Alternative analyses

Repeating the separability tests and the model in 4 and 9 ha cells left the result unchanged (<a href="#tbl-cell-size" class="quarto-xref">Table 6</a>). Held-out AUC was 0.57 with 4 ha cells (95 percent CI 0.43 to 0.72) and 0.60 with 9 ha cells (0.48 to 0.73). Both intervals included chance, and neither forest vertical complexity nor internal forest gaps separated nest cells from background cells at either size.

Neither the nest-tile refit nor the 30 m refit ranked nest cells above chance (<a href="#tbl-cell-size" class="quarto-xref">Table 6</a>). With background cells added from the nest tiles, held-out AUC was 0.49 (95 percent CI 0.34 to 0.64). With a minimum forest height of 30 m, it was 0.37 (0.24 to 0.50), from 12 of the 16 nest cells. Without the scan checks, all 20 scanned nest cells entered, and held-out AUC rose to 0.60 (0.52 to 0.69). Internal forest gaps then differed between nest and background cells \[*M* = 0.26, Wilcoxon rank-sum test, *W* = 62,071, *P* = 0.03\]. Of the background cells, 1,008 failed the checks, 370 of them cut by the edge of the downloaded tiles. These averaged 91.2 ± 55.4 thousand internal forest gaps per hectare (87.7 to 94.6). Background cells that passed averaged 157.5 ± 56.8 thousand (155.7 to 159.3). Scored against the passing background cells alone, the same predictions gave an AUC of 0.53 (0.44 to 0.63).

``` r
alt <- dplyr::bind_rows(
  tbl("T7_cell_size.csv") |> dplyr::mutate(analysis = c(`1` = "1 ha cells, primary", `4` = "4 ha cells", `9` = "9 ha cells")[as.character(size_ha)]),
  tbl("T8_refits.csv"))
pick <- \(a, what, col = "value") alt[[col]][alt$analysis == a & alt$metric == what]
rows <- c("1 ha cells, primary", "4 ha cells", "9 ha cells", "Without scan checks", "Nest-tile background", "30 m minimum height")
each <- \(f) purrr::map_chr(rows, f)
mstar <- \(a, k) paste0(fmt(pick(a, paste(k, "M"))), star(pick(a, paste(k, "W"), "p_value")))
iv <- \(a, what) paste0(fmt(pick(a, what)), " (", fmt(pick(a, what, "low")), " to ", fmt(pick(a, what, "high")), ")")
tibble::tibble(Analysis = rows,
  `Nest cells` = each(\(a) fmt(pick(a, "Nest cells"), 0)),
  `Background cells` = each(\(a) fmt(pick(a, "Background cells"), 0)),
  `M, forest vertical complexity` = each(\(a) mstar(a, "fvc")),
  `M, internal forest gaps` = each(\(a) mstar(a, "empty_gaps")),
  `M, forest height` = each(\(a) mstar(a, "fh")),
  AUC = each(\(a) iv(a, "AUC")),
  `Boyce index` = each(\(a) iv(a, "Boyce index"))) |>
  flextable::flextable() |> flextable::align(j = 2:8, align = "right", part = "body") |> flextable::align(align = "center", part = "header") |> flextable::set_table_properties(layout = "autofit", width = 1)
```

<div id="tbl-cell-size">

Table 6: Separability of nest and background cells, and held-out performance of the Clayoquot Sound model, in the primary analysis and five alternatives. The alternatives used 4 or 9 ha cells, dropped the two scan checks, added background cells from the nest tiles, or raised the minimum forest height to 30 m. M was the separability index, with asterisks from the Wilcoxon rank-sum test. Intervals were 95% confidence intervals for AUC and came from 1,000 bootstrap resamples for the Boyce index. \* P ≤ 0.05, \*\* P ≤ 0.01, \*\*\* P ≤ 0.001, \*\*\*\* P ≤ 0.0001.

<div class="cell-output-display">

| Analysis | Nest cells | Background cells | M, forest vertical complexity | M, internal forest gaps | M, forest height | AUC | Boyce index |
|:---|:---|:---|:---|:---|:---|:---|:---|
| 1 ha cells, primary | 16 | 3,827 | 0.08 | 0.21 | 0.09 | 0.49 (0.36 to 0.62) | -0.20 (-0.61 to 0.62) |
| 4 ha cells | 15 | 897 | 0.01 | 0.19 | 0.05 | 0.57 (0.43 to 0.72) | 0.31 (-0.32 to 0.66) |
| 9 ha cells | 16 | 343 | 0.10 | 0.23 | 0.13 | 0.60 (0.48 to 0.73) | 0.40 (-0.32 to 0.62) |
| Without scan checks | 20 | 4,835 | 0.13 | 0.26\* | 0.15 | 0.60 (0.52 to 0.69) | 0.16 (-0.30 to 0.67) |
| Nest-tile background | 16 | 7,539 | 0.06 | 0.15 | 0.07 | 0.49 (0.34 to 0.64) | -0.43 (-0.67 to 0.46) |
| 30 m minimum height | 12 | 2,435 | 0.19 | 0.03 | 0.14 | 0.37 (0.24 to 0.50) | -0.56 (-0.77 to 0.29) |

</div>

</div>

# Discussion

In Clayoquot Sound, a MaxEnt model was rebuilt from 16 local nest cells and public 2019 laser scans. Its ranking of held-out nest cells above background cells could not be distinguished from chance. The 95 percent interval on the local model’s AUC included both chance and the 0.63 reached by the same model transferred from Desolation Sound (Cosgrove et al., 2024). That score came from that study’s own nests and random cells, not from the cells used here. The local model therefore did not outperform the transferred one, although these data could not show that it did worse. In Oregon, a laser-scan model selected on local stands had ranked above one transferred from a neighboring district (Hagar et al., 2018). Neither measure showed the clear separation expected from Desolation Sound. Forest vertical complexity was only slightly higher at nest cells, and internal forest gaps were more numerous there, though not significantly. In the Desolation Sound model, by contrast, the fitted probability of a nest fell as internal forest gaps increased (Cosgrove et al., 2024). Three explanations were considered, a difference in nest-site selection between the two sounds, a small sample drawn from uniform forest, and the limits of 1 ha summaries of point heights.

## Two sounds compared

In Clayoquot Sound, forest height did not differ between nest and background cells, and both modeled measures rose closely with forest height (<a href="#tbl-separability" class="quarto-xref">Table 4</a>). Each measure therefore largely tracked the height of a 1 ha cell, so neither separated nest cells from the tall forest around them. For ground plots in the Ursus Valley of Clayoquot Sound, Bahn (1998, p. 92) noted that the standard deviation of tree height “is dependent on the average height”. In a coastal forest of British Columbia, median height and the coefficient of variation of heights loaded most highly on separate principal components of 30 scan metrics (White et al., 2017). A spread divided by mean height, such as the coefficient of variation, might therefore have measured vertical complexity apart from height. Cosgrove et al. (2024) found forest height and vertical complexity higher at nests than at random cells of available forest in both sounds. In Clayoquot Sound, that study compared 21 nests with an equal number of random cells, in scans of 2020 that averaged 29.9 points per square meter (Cosgrove et al., 2024). Cosgrove et al. (2024, p. 5) defined available forest so that “only one tree per pixel needed to meet the height criterion”. In this study, points were thinned to 12 per square meter, and the 95th percentile of point heights in each cell had to reach 20 m. Cells also had to pass the two scan checks, and the 16 nest cells were compared with all 3,827 background cells rather than an equal number.

Ground studies had found that the contrast in canopy between nest sites and other forest differed between the two sounds in direction (Silvergieter and Lank, 2011a) and in size. Nest trees in Clayoquot Sound exceeded other available trees in height by an average of 15 percent of total height, against 30 percent in Desolation Sound (Silvergieter and Lank, 2011b). A 1 ha summary of point heights would be less able to detect the smaller of these contrasts. Internal forest gaps were more numerous at nest cells, though not significantly, and rose closely with forest height \[Spearman ρ = 0.86\]. The measure therefore could not show that nest cells held more of the small gaps reported next to nest trees (Burger et al., 2009; Waterhouse et al., 2007). Cosgrove et al. (2024, p. 8) had come to suspect that the measure “was inappropriate for describing the fine-scale structure that determines nest access”. Canada’s recovery strategy for the species ranked canopy closure of 36 to 75 percent as more likely nesting habitat than both more open and more closed canopy (Environment Canada, 2014). Moderately uniform canopy, with a 21 to 30 percent height difference between leading trees and the average canopy, ranked above both very uniform and very non-uniform canopy (Environment Canada, 2014). In the Ursus Valley, Bahn (1998, pp. 100–101) rated canopy closure most suitable from 30 to 70 percent and wrote that it had a “non-monotonous relationship to habitat quality”. If nest-site use peaked at intermediate structure, the same preference could give relations of opposite sign in two sounds with different available forest. The rank-sum test was not designed to detect such a peak, and the model’s linear and product features could not fit one.

Relations between forest structure and use had also reversed between places in other studies. In the central Sierra Nevada in 2017 and 2018, California spotted owls at low elevations selected stands with shorter trees and sites closer to hard edges (Kramer et al., 2021). Owls at high elevations selected the opposite (Kramer et al., 2021). Across California and southern Oregon from 1991 to 1997, murrelets most often occupied redwood stands close to streams but Douglas-fir stands away from them (Meyer et al., 2004). If nest-site selection differed between the two sounds in this way, a model built on Desolation Sound relations would be expected to lose accuracy on transfer, as it did. These data could not distinguish such a regional difference from the effects of a small sample and of error in nest positions.

## Sample and forest

The small sample limited the comparison, because with 16 nest cells over five folds the 95 percent interval on AUC spanned 0.28. The result did not rest on one assignment of blocks to folds, since 20 other assignments also left held-out AUC near chance. Residuals were spatially autocorrelated, so the cells were not independent and the true uncertainty was probably wider. The two scan checks removed 4 of the 20 scanned nest cells, each in forest taller than 30 m. Without the scan checks, the rise in AUC came from the background cells that the checks removed (<a href="#tbl-cell-size" class="quarto-xref">Table 6</a>). Scored against the passing background cells alone, the same predictions stayed near chance. Gaps were summed only over columns with returns, so poorly scanned cells and cells cut by a tile edge had low counts, against which nest cells looked distinct. Of the scanned background cells, 49 percent met the definition of available forest, so nest cells were compared with forest of much the same kind. Background cells were available forest of unknown use, and some probably held nests of untagged birds, which would lower any measured contrast (Waterhouse et al., 2008). Adding background cells from the nest tiles also left held-out AUC near chance (<a href="#tbl-cell-size" class="quarto-xref">Table 6</a>). A minimum forest height of 30 m, which would exclude stands cut before 1998 and regrown past 20 m, also left the model no better than chance. In Clayoquot Sound from 2000 to 2002, murrelets had nested in old-growth fragments of each size in proportion to their availability (Zharikov et al., 2006). In Desolation Sound, where about 80 percent of the original forest had been cut against 25 percent in Clayoquot Sound, they had nested in disproportionately small fragments (Zharikov et al., 2006).

Nest positions had an error of about 50 m within 100 m cells (Cooke et al., 2022). In delineating critical habitat, the recovery strategy drew a 200 m radius around each nest record, because many had been located by telemetry or before advanced GPS (Environment Canada, 2014). In the Ursus Valley, Bahn (1998, p. 94) wrote that old-growth vegetation “often changes rapidly over small distances”. An error of about 50 m could therefore assign a nest to a neighboring cell of different structure. The 4 and 9 ha cells were more tolerant of that error, yet they too did not separate nest cells from background cells. Position error alone therefore did not explain the weak result. The nests were found 17 to 19 years before the scans were flown. At Monks Wood, England, mean canopy height related to great tit nestling mass in 2001 whether the scan was flown in 2000, 2005 or 2012 (Hill and Hinsley, 2015). Hill and Hinsley (2015, p. 3462) expected a lag to have “a much greater impact in a more dynamic system, such as early successional or coppiced woodland”. Unharvested old growth was closer to the stable case, although natural change over the interval was not measured.

The model’s error was almost entirely systematic, because MaxEnt output on the complementary log-log scale ranked relative suitability and was not scaled to the small share of nest cells. Training AUC exceeded held-out AUC, so the weak separation found within training folds did not carry over to new cells. The RMSE and Brier score therefore described calibration more than discrimination. Removing statistical outliers on top of the contractor’s noise class left held-out AUC unchanged. With AUC near chance before and after removal, the test showed only that the removed outliers had not caused the weak result.

## Measurable structure

Cosgrove et al. (2024, p. 7) judged it likely that their 1 ha predictors “did not effectively capture key microhabitat structure, such as tree-level cover and the presence of suitable nest platforms”. The recovery strategy used stand and landscape attributes, vertical canopy complexity among them, as a filter to identify areas that “require confirmation at the site level against the microhabitat attributes” (Environment Canada, 2014, p. 26). Nest platforms, which defined nesting habitat (Ralph et al., 1995), could not be counted by remote methods (Silvergieter and Lank, 2011a). Most laser-scan studies of wildlife habitat reviewed by Vogeler and Cohen (2016) summarized canopy height, density and height percentiles in grid cells, the kind of measure used here. In northern California redwood timberland, measures of single tall trees had outperformed such summaries (Lamphear et al., 2025). Finding single trees in a point cloud could be “problematic in an operational context”, given “tree occlusion, diverse tree structures and canopy architectures, as well as low point densities” (White et al., 2017, p. 1).

Scan data provided “information on vegetation height, variation in vegetation height, and vegetation density” (White et al., 2017, p. 4). Other forest attributes could be modeled with confidence only where they related statistically to these (White et al., 2017). In the Ursus Valley from 1995 to 1997, Bahn (1998) related ground plots at 51 survey stations to occupied activity, behavior taken to show nesting nearby. Epiphyte cover, which scans did not measure directly, explained 29 to 37 percent of the variation in that activity (Bahn, 1998). The spread of tree heights explained 9 to 12 percent \[least-squares regression, each *P* ≤ 0.035\] (Bahn, 1998).

Woodpecker nest models built at three burned forests in Washington, Oregon and Idaho scored AUCs of 0.660 to 0.951 where they were built and 0.430 to 0.873 elsewhere (Latif et al., 2016). Models with field measures transferred better than those built only from remotely sensed layers (Latif et al., 2016). Latif et al. (2016, p. 780) wrote that “improved resolution of remotely sensed data (e.g., LiDAR) over large landscapes could provide the fine resolution data needed to relax this limitation”.

# Conclusion

A two-measure model built from public laser scans flown in 2019 and 16 local nest cells did not separate nests from available forest in Clayoquot Sound. That result held in 4 and 9 ha cells, over 20 other assignments of blocks to folds, and in three further refits. The local model could not be shown to improve on the model transferred from Desolation Sound. For management in Clayoquot Sound, a model fitted to the few local nests gave no reason to replace the transferred model.

Two approaches might make better use of the scans than two measures summarized in 1 ha cells, and both could be tested with data already public. One would be a model fitted to nests pooled across regions with terms for region, as Silvergieter and Lank (2011a) fitted to ground plots. The other would use measures of individual tall trees taken from the scans, the predictors of Lamphear et al. (2025).

# Data availability

Every line of code that built, fitted and tested the model was included in the manuscript source at https://github.com/seamusrobertmurphy/lidar-marbled-murrelet. Nest locations were obtained under a CC0 license from the Movebank Data Repository (Cooke et al., 2022). Laser scans were obtained from LidarBC (https://lidar.gov.bc.ca/) under the Open Government Licence British Columbia, and harvest years from the National Terrestrial Ecosystem Monitoring System (Hermosilla et al., 2016).

# Author contributions

Seamus Murphy carried out the conceptualization, methodology, software, formal analysis, data curation and visualization, and the writing of the original draft and its review and editing.

# Funding

This research did not receive any specific grant from funding agencies in the public, commercial, or not-for-profit sectors.

# Competing interests

The author declares no known competing financial interests or personal relationships that could have appeared to influence the work reported in this paper.

# References

<div id="refs" class="references csl-bib-body hanging-indent" data-entry-spacing="0">

<div id="ref-acebes2021disentangling" class="csl-entry">

Acebes, P., Lillo, P., Jaime-González, C., 2021. Disentangling LiDAR contribution in modelling species–habitat structure relationships in terrestrial ecosystems worldwide. A systematic review and future directions. Remote Sensing 13, 3447. <https://doi.org/10.3390/rs13173447>

</div>

<div id="ref-bahn1998habitat" class="csl-entry">

Bahn, V., 1998. [Habitat requirements and habitat suitability index for the Marbled Murrelet (*Brachyramphus marmoratus*) as a management target species in the Ursus Valley, British Columbia](https://corescholar.libraries.wright.edu/biology/27) (Diplomarbeit). Philipps-Universität Marburg, Marburg, Germany.

</div>

<div id="ref-bahn2002habitat" class="csl-entry">

Bahn, V., Newsom, D., 2002. Habitat suitability mapping for Marbled Murrelets in Clayoquot Sound, in: Burger, A.E., Chatwin, T.A. (Eds.), Multi-Scale Studies of Populations, Distribution and Habitat Associations of Marbled Murrelets in Clayoquot Sound, British Columbia. Ministry of Water, Land and Air Protection, Victoria, British Columbia, pp. 101–119.

</div>

<div id="ref-bakx2019categorization" class="csl-entry">

Bakx, T.R.M., Koma, Z., Seijmonsbergen, A.C., Kissling, W.D., 2019. Use and categorization of <span class="nocase">Light Detection and Ranging</span> vegetation metrics in avian diversity and species distribution research. Diversity and Distributions 25, 1045–1059. <https://doi.org/10.1111/ddi.12915>

</div>

<div id="ref-burger2001using" class="csl-entry">

Burger, A.E., 2001. Using radar to estimate populations and assess habitat associations of marbled murrelets. The Journal of Wildlife Management 65, 696–715. <https://doi.org/10.2307/3803021>

</div>

<div id="ref-burger2004inland" class="csl-entry">

Burger, A.E., Bahn, V., 2004. Inland habitat associations of Marbled Murrelets on southwest Vancouver Island, British Columbia. Journal of Field Ornithology 75, 53–66. <https://doi.org/10.1648/0273-8570-75.1.53>

</div>

<div id="ref-burger2007need" class="csl-entry">

Burger, A.E., Page, R.E., 2007. The need for biological realism in habitat modeling: A reinterpretation of Zharikov et al. (2006). Landscape Ecology 22, 1273–1281. <https://doi.org/10.1007/s10980-007-9114-6>

</div>

<div id="ref-burger2010factors" class="csl-entry">

Burger, A.E., Ronconi, R.A., Silvergieter, M.P., Conroy, C., Bahn, V., Manley, I.A., Cober, A., Lank, D.B., 2010. Factors affecting the availability of thick epiphyte mats and other potential nest platforms for Marbled Murrelets in British Columbia. Canadian Journal of Forest Research 40, 727–746. <https://doi.org/10.1139/x10-034>

</div>

<div id="ref-burger2018reliability" class="csl-entry">

Burger, A.E., Waterhouse, F.L., Deal, J.A., Lank, D.B., Donald, D.S., 2018. The reliability and application of methods used to predict suitable nesting habitat for Marbled Murrelets. Journal of Ecosystems and Management 18, 1–18. <https://doi.org/10.22230/jem.2018v18n1a593>

</div>

<div id="ref-burger2009new" class="csl-entry">

Burger, A.E., Waterhouse, F.L., Donaldson, A., Whittaker, C., Lank, D.B., 2009. New methods for assessing Marbled Murrelet nesting habitat: Air photo interpretation and low-level aerial surveys. BC Journal of Ecosystems and Management 10, 4–14. <https://doi.org/10.22230/jem.2009v10n1a407>

</div>

<div id="ref-clyde2017fine" class="csl-entry">

Clyde, G.E., 2017. [A fine-scale lidar-based habitat suitability mapping methodology for the marbled murrelet (*Brachyramphus marmoratus*) on Vancouver Island, British Columbia](https://dspace.library.uvic.ca/handle/1828/7923) (Master’s thesis). University of Victoria, Victoria, British Columbia.

</div>

<div id="ref-cooke2022marbled" class="csl-entry">

Cooke, F., Lank, D.B., Bradley, R.W., Parker, N.R., Lougheed, C., Lougheed, L.W., McFarlane Tranquila, L.A., Pastran, S.A., Greene, R., Vincent, P., Bertram, D.F., 2022. Data from: Study "Marbled Murrelets at Clayoquot Sound, British Columbia, Canada (2000-2002)". <https://doi.org/10.5441/001/1.KG210889>

</div>

<div id="ref-cosgrove2024modeling" class="csl-entry">

Cosgrove, C.F., Coops, N.C., Waterhouse, F.L., Goodbody, T.R.H., 2024. Modeling Marbled Murrelet nesting habitat: A quantitative approach using airborne laser scanning data in British Columbia, Canada. Avian Conservation and Ecology 19, 5. <https://doi.org/10.5751/ace-02585-190105>

</div>

<div id="ref-dicola2017ecospat" class="csl-entry">

Di Cola, V., Broennimann, O., Petitpierre, B., Breiner, F.T., D’Amen, M., Randin, C., Engler, R., Pottier, J., Pio, D., Dubuis, A., Pellissier, L., Mateo, R.G., Hordijk, W., Salamin, N., Guisan, A., 2017. <span class="nocase">ecospat</span>: An R package to support spatial analyses and modeling of species niches and distributions. Ecography 40, 774–787. <https://doi.org/10.1111/ecog.02671>

</div>

<div id="ref-environmentcanada2014recovery" class="csl-entry">

Environment Canada, 2014. Recovery strategy for the Marbled Murrelet (*Brachyramphus marmoratus*) in Canada (Species at Risk Act Recovery Strategy Series). Environment Canada, Ottawa.

</div>

<div id="ref-geobc2023lidar" class="csl-entry">

GeoBC, 2023. LiDAR metadata summary, NDMP 2018 Vancouver Island \[WWW Document\]. URL <https://nrs.objectstore.gov.bc.ca/gdwuts/metadata/accuracy_rpts/2019/l_002_emn_19_summary.pdf> (accessed 11.20.2026).

</div>

<div id="ref-hagar2018modeling" class="csl-entry">

Hagar, J.C., Aragon, R., Haggerty, P., Hollenbeck, J.P., 2018. Modeling habitat for Marbled Murrelets on the Siuslaw National Forest, Oregon, using lidar data (Open-File Report No. 2018-1035). U.S. Geological Survey, Reston, VA. <https://doi.org/10.3133/ofr20181035>

</div>

<div id="ref-hagar2014modeling" class="csl-entry">

Hagar, J.C., Eskelson, B.N.I., Haggerty, P.K., Nelson, S.K., Vesely, D.G., 2014. Modeling marbled murrelet (*Brachyramphus marmoratus*) habitat using LiDAR-derived canopy data. Wildlife Society Bulletin 38, 237–249. <https://doi.org/10.1002/wsb.407>

</div>

<div id="ref-hamer2021marbled" class="csl-entry">

Hamer, T.E., Nelson, K., Jones, J., Verschuyl, J., 2021. Marbled Murrelet nest site selection at three fine spatial scales. Avian Conservation and Ecology 16, 4. <https://doi.org/10.5751/ace-01883-160204>

</div>

<div id="ref-hermosilla2016mass" class="csl-entry">

Hermosilla, T., Wulder, M.A., White, J.C., Coops, N.C., Hobart, G.W., Campbell, L.B., 2016. Mass data processing of time series Landsat imagery: Pixels to data products for forest monitoring. International Journal of Digital Earth 9, 1035–1054. <https://doi.org/10.1080/17538947.2016.1187673>

</div>

<div id="ref-hill2015airborne" class="csl-entry">

Hill, R., Hinsley, S., 2015. Airborne lidar for woodland habitat quality monitoring: Exploring the significance of lidar data characteristics when modelling organism-habitat relationships. Remote Sensing 7, 3446–3466. <https://doi.org/10.3390/rs70403446>

</div>

<div id="ref-hirzel2006evaluating" class="csl-entry">

Hirzel, A.H., Le Lay, G., Helfer, V., Randin, C., Guisan, A., 2006. Evaluating the ability of habitat suitability models to predict species presences. Ecological Modelling 199, 142–152. <https://doi.org/10.1016/j.ecolmodel.2006.05.017>

</div>

<div id="ref-kane2010examining" class="csl-entry">

Kane, V.R., Bakker, J.D., McGaughey, R.J., Lutz, J.A., Gersonde, R.F., Franklin, J.F., 2010. Examining conifer canopy structural complexity across forest ages and elevations with LiDAR data. Canadian Journal of Forest Research 40, 774–787. <https://doi.org/10.1139/x10-064>

</div>

<div id="ref-kaufman1994detection" class="csl-entry">

Kaufman, Y.J., Remer, L.A., 1994. Detection of forests using mid-IR reflectance: An application for aerosol studies. IEEE Transactions on Geoscience and Remote Sensing 32, 672–683. <https://doi.org/10.1109/36.297984>

</div>

<div id="ref-kramer2021elevational" class="csl-entry">

Kramer, H.A., Jones, G.M., Kane, V.R., Bartl-Geller, B., Kane, J.T., Whitmore, S.A., Berigan, W.J., Dotters, B.P., Roberts, K.N., Sawyer, S.C., Keane, J.J., North, M.P., Gutiérrez, R.J., Peery, M.Z., 2021. Elevational gradients strongly mediate habitat selection patterns in a nocturnal predator. Ecosphere 12, e03500. <https://doi.org/10.1002/ecs2.3500>

</div>

<div id="ref-lamphear2025marbled" class="csl-entry">

Lamphear, D.W., Hamm, K.A., Early, D.A., McDonald, T.L., 2025. Marbled murrelet habitat suitability in redwood timberlands of northern coastal California utilizing LiDAR-derived individual tree metrics. PLOS ONE 20, e0307726. <https://doi.org/10.1371/journal.pone.0307726>

</div>

<div id="ref-latif2016transferability" class="csl-entry">

Latif, Q.S., Saab, V.A., Hollenbeck, J.P., Dudley, J.G., 2016. Transferability of habitat suitability models for nesting woodpeckers associated with wildfire. The Condor 118, 766–790. <https://doi.org/10.1650/condor-16-86.1>

</div>

<div id="ref-liu2005selecting" class="csl-entry">

Liu, C., Berry, P.M., Dawson, T.P., Pearson, R.G., 2005. Selecting thresholds of occurrence in the prediction of species distributions. Ecography 28, 385–393. <https://doi.org/10.1111/j.0906-7590.2005.03957.x>

</div>

<div id="ref-malt2009marbled" class="csl-entry">

Malt, J.M., Lank, D.B., 2009. Marbled Murrelet nest predation risk in managed forest landscapes: Dynamic fragmentation effects at multiple scales. Ecological Applications 19, 1274–1287. <https://doi.org/10.1890/08-0598.1>

</div>

<div id="ref-mather2010marbled" class="csl-entry">

Mather, M., Chatwin, T., Cragg, J., Sinclair, L., Bertram, D.F., 2010. [Marbled Murrelet nesting habitat suitability model for the British Columbia coast](http://jem.forrex.org/index.php/jem/article/view/11/27). BC Journal of Ecosystems and Management 11, 91–102.

</div>

<div id="ref-meyer2004stand" class="csl-entry">

Meyer, C.B., Miller, S.L., Ralph, C.J., 2004. Stand-scale habitat associations across a large geographic region of an old-growth specialist, the Marbled Murrelet. The Wilson Bulletin 116, 197–210. <https://doi.org/10.1676/03-122>

</div>

<div id="ref-murphy2026spatial" class="csl-entry">

Murphy, S., Leslie, A., Wilson, J., Banks, L.K., 2026. Spatial patterns of conifer regeneration following mountain pine beetle outbreak (*Dendroctonus ponderosae*) and mixed-severity wildfire: A point process model of disturbance interactions in the southern Selkirk Mountains, British Columbia. Forest Ecology and Management 618, 123985. <https://doi.org/10.1016/j.foreco.2026.123985>

</div>

<div id="ref-north2017cover" class="csl-entry">

North, M.P., Kane, J.T., Kane, V.R., Asner, G.P., Berigan, W., Churchill, D.J., Conway, S., Gutiérrez, R.J., Jeronimo, S., Keane, J., Koltunov, A., Mark, T., Moskal, M., Munton, T., Peery, Z., Ramirez, C., Sollmann, R., White, A., Whitmore, S., 2017. Cover of tall trees best predicts California spotted owl habitat. Forest Ecology and Management 405, 166–178. <https://doi.org/10.1016/j.foreco.2017.09.019>

</div>

<div id="ref-phillips2017opening" class="csl-entry">

Phillips, S.J., Anderson, R.P., Dudík, M., Schapire, R.E., Blair, M.E., 2017. Opening the black box: An open-source release of Maxent. Ecography 40, 887–893. <https://doi.org/10.1111/ecog.03049>

</div>

<div id="ref-rcore2024r" class="csl-entry">

R Core Team, 2024. [R: A language and environment for statistical computing. Version 4.4.1](https://www.R-project.org/). R Foundation for Statistical Computing, Vienna, Austria.

</div>

<div id="ref-ralph1995ecology" class="csl-entry">

Ralph, C.J., Hunt, G.L., Raphael, M.G., Piatt, J.F., 1995. Ecology and conservation of the Marbled Murrelet in North America: An overview, in: Ralph, C.J., Hunt, G.L., Raphael, M.G., Piatt, J.F. (Eds.), Ecology and Conservation of the Marbled Murrelet, General Technical Report PSW-GTR-152. U.S. Department of Agriculture, Forest Service, Pacific Southwest Research Station, Albany, CA, pp. 3–22. <https://doi.org/10.2737/psw-gtr-152>

</div>

<div id="ref-raphael2018marbled" class="csl-entry">

Raphael, M.G., Falxa, G.A., Burger, A.E., 2018. Chapter 5: Marbled Murrelet, in: Spies, T.A., Stine, P.A., Gravenmier, R.A., Long, J.W., Reilly, M.J. (Eds.), Synthesis of Science to Inform Land Management Within the Northwest Forest Plan Area, General Technical Report PNW-GTR-966. U.S. Department of Agriculture, Forest Service, Pacific Northwest Research Station, Portland, OR, pp. 301–370. <https://doi.org/10.2737/pnw-gtr-966>

</div>

<div id="ref-robin2011proc" class="csl-entry">

Robin, X., Turck, N., Hainard, A., Tiberti, N., Lisacek, F., Sanchez, J.-C., Müller, M., 2011. <span class="nocase">pROC</span>: An open-source package for R and S+ to analyze and compare ROC curves. BMC Bioinformatics 12, 77. <https://doi.org/10.1186/1471-2105-12-77>

</div>

<div id="ref-roussel2020lidr" class="csl-entry">

<span class="nocase">Roussel, J.-R., Auty, D., Coops, N.C., Tompalski, P., Goodbody, T.R.H., Sánchez Meador, A., Bourdon, J.-F., de Boissieu, F., Achim, A.</span>, 2020. <span class="nocase">lidR</span>: An R package for analysis of airborne laser scanning (ALS) data. Remote Sensing of Environment 251, 112061. <https://doi.org/10.1016/j.rse.2020.112061>

</div>

<div id="ref-silvergieter2011patch" class="csl-entry">

Silvergieter, M.P., Lank, D.B., 2011a. Patch scale nest-site selection by Marbled Murrelets (*Brachyramphus marmoratus*). Avian Conservation and Ecology 6, 6. <https://doi.org/10.5751/ace-00483-060206>

</div>

<div id="ref-silvergieter2011marbled" class="csl-entry">

Silvergieter, M.P., Lank, D.B., 2011b. Marbled Murrelets select distinctive nest trees within old-growth forest patches. Avian Conservation and Ecology 6, 3. <https://doi.org/10.5751/ace-00462-060203>

</div>

<div id="ref-sovern2019activity" class="csl-entry">

Sovern, S.G., Lesmeister, D.B., Dugger, K.M., Pruett, M.S., Davis, R.J., Jenkins, J.M., 2019. Activity center selection by northern spotted owls. The Journal of Wildlife Management 83, 714–727. <https://doi.org/10.1002/jwmg.21632>

</div>

<div id="ref-mcfarlanetranquilla2005breeding" class="csl-entry">

Tranquilla, L.M., Parker, N.R., Bradley, R.W., Lank, D.B., Krebs, E.A., Lougheed, L., Lougheed, C., 2005. Breeding chronology of Marbled Murrelets varies between coastal and inshore sites in southern British Columbia. Journal of Field Ornithology 76, 357–367. <https://doi.org/10.1648/0273-8570-76.4.357>

</div>

<div id="ref-valavi2019blockcv" class="csl-entry">

Valavi, R., Elith, J., Lahoz-Monfort, J.J., Guillera-Arroita, G., 2019. <span class="nocase">blockCV</span>: An R package for generating spatially or environmentally separated folds for *k*-fold cross-validation of species distribution models. Methods in Ecology and Evolution 10, 225–232. <https://doi.org/10.1111/2041-210x.13107>

</div>

<div id="ref-valente2024presence" class="csl-entry">

Valente, J.J., Adrean, L.J., Nelson, S.K., Betts, M.G., Roby, D.D., Rivers, J.W., 2024. Presence-absence surveys yield spatially imprecise information about nesting sites of an endangered, forest-nesting seabird. PLOS ONE 19, e0315531. <https://doi.org/10.1371/journal.pone.0315531>

</div>

<div id="ref-valente2023fragmentation" class="csl-entry">

Valente, J.J., Rivers, J.W., Yang, Z., Nelson, S.K., Northrup, J.M., Roby, D.D., Meyer, C.B., Betts, M.G., 2023. Fragmentation effects on an endangered species across a gradient from the interior to edge of its range. Conservation Biology 37, e14091. <https://doi.org/10.1111/cobi.14091>

</div>

<div id="ref-vanrooyen2011relating" class="csl-entry">

<span class="nocase">van Rooyen, J.C., Malt, J.M., Lank, D.B.</span>, 2011. Relating microclimate to epiphyte availability: Edge effects on nesting habitat availability for the Marbled Murrelet. Northwest Science 85, 549–561. <https://doi.org/10.3955/046.085.0405>

</div>

<div id="ref-vogeler2016review" class="csl-entry">

Vogeler, J.C., Cohen, W.B., 2016. A review of the role of active remote sensing and data fusion for characterizing forest in wildlife habitat models. Revista de Teledetección 45, 1–14. <https://doi.org/10.4995/raet.2016.3981>

</div>

<div id="ref-waterhouse2007assessing" class="csl-entry">

Waterhouse, F.L., Burger, A.E., Cober, A., Donaldson, A., Ott, P.K., 2007. [Assessing habitat quality of Marbled Murrelet nest sites on the Queen Charlotte Islands/Haida Gwaii, by algorithm, airphoto interpretation, and aerial survey methods](https://www.for.gov.bc.ca/rco/research/wildlifereports/tr035.pdf) (Technical Report No. TR-035). Research Section, Coast Forest Region, BC Ministry of Forests and Range, Nanaimo, BC.

</div>

<div id="ref-waterhouse2009using" class="csl-entry">

Waterhouse, F.L., Burger, A.E., Lank, D.B., Ott, P., Krebs, E., Parker, N., 2009. Using the low-level aerial survey method to identify Marbled Murrelet nesting habitat. BC Journal of Ecosystems and Management 10, 80–96. <https://doi.org/10.22230/jem.2009v10n1a413>

</div>

<div id="ref-waterhouse2008using" class="csl-entry">

Waterhouse, F.L., Donaldson, A., Lank, D.B., Ott, P., Krebs, E., 2008. Using air photos to interpret quality of Marbled Murrelet nesting habitat in south coastal British Columbia. BC Journal of Ecosystems and Management 9, 17–37. <https://doi.org/10.22230/jem.2008v9n1a381>

</div>

<div id="ref-white2017model" class="csl-entry">

White, J.C., Tompalski, P., Vastaranta, M., Wulder, M.A., Saarinen, N., Stepper, C., Coops, N.C., 2017. A model development and application guide for generating an enhanced forest inventory using airborne laser scanning data and an area-based approach (Information Report No. FI-X-018). Natural Resources Canada, Canadian Forest Service, Canadian Wood Fibre Centre, Victoria, British Columbia.

</div>

<div id="ref-wilk2016nesting" class="csl-entry">

Wilk, R.J., Raphael, M.G., Bloxton, T.D., 2016. Nesting habitat characteristics of Marbled Murrelets occurring in near-shore waters of the Olympic Peninsula, Washington. Journal of Field Ornithology 87, 162–175. <https://doi.org/10.1111/jofo.12150>

</div>

<div id="ref-zharikov2007influence" class="csl-entry">

Zharikov, Y., Lank, D.B., Cooke, F., 2007. Influence of landscape pattern on breeding distribution and success in a threatened alcid, the marbled murrelet: Model transferability and management implications. Journal of Applied Ecology 44, 748–759. <https://doi.org/10.1111/j.1365-2664.2007.01319.x>

</div>

<div id="ref-zharikov2006habitat" class="csl-entry">

Zharikov, Y., Lank, D.B., Huettmann, F., Bradley, R.W., Parker, N., Yen, P.P.-W., Mcfarlane-Tranquilla, L.A., Cooke, F., 2006. Habitat selection and breeding success in a forest-nesting alcid, the marbled murrelet, in two landscapes with different degrees of forest fragmentation. Landscape Ecology 21, 107–120. <https://doi.org/10.1007/s10980-005-1438-5>

</div>

</div>
