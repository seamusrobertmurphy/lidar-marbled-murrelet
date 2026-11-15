# Mapping Marbled Murrelet habitat in the Clayoquot Sound using ALS LiDAR data
Murphy, S
2026-11-15

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
ev <- \(what, col = "value", m = "local") { t <- tbl("T4_evaluation.csv"); t[[col]][t$model == m & t$metric == what] }
sep <- \(what, col) { t <- tbl("T3_separability.csv"); t[[col]][t$measure == what] }
smp <- \(what) { t <- tbl("T2_sample.csv"); t$n[t$step == what] }
acc <- \(cls) if (cls == "nest") 100 * ev("Nest cells above threshold") / ev("Nest cells") else 100 * (1 - ev("Background cells above threshold") / ev("Background cells"))

# Run `code` only when `path` is missing, so a render rebuilds an output that was
# deleted and leaves a finished one alone. Delete the file to force a rebuild.
build_once <- function(path, code) {
  if (!file.exists(path)) force(code)
  invisible(path)
}
```

# Abstract

Provincial airborne laser scans flown in 2019 measured old-growth forest structure across Clayoquot Sound, British Columbia. A Marbled Murrelet nesting-habitat model built from such scans and nests located in Desolation Sound lost accuracy when transferred there. Whether a model built from Clayoquot Sound’s own nests would do better was unknown. This study rebuilt that model from the 2019 scans and the nests of birds radio-tracked from 2000 to 2002 (16 nest cells and 3,827 background cells of 1 ha in tall, unharvested forest). Maximum entropy modeling of forest vertical complexity and internal forest gaps was scored by five-fold spatial cross-validation in 10 km blocks. Neither measure separated nest cells from background cells (forest vertical complexity, *P* = 0.54; internal forest gaps, *P* = 0.10), and nest cells had more internal gaps than background cells, the reverse of the Desolation Sound relation. The model classed 62.5 percent of held-out nest cells and 40.8 percent of background cells correctly, an average accuracy of 51.7 percent and no better than chance. Its ranking of nest cells could not be shown to differ from that of the model transferred from Desolation Sound. With so few located nests, the transferred model, or one fitted to nests pooled across regions, remained the better-supported basis for mapping nesting habitat in Clayoquot Sound.

# Keywords

Airborne laser scanning; *Brachyramphus marmoratus*; habitat suitability; lidar; MaxEnt; model transferability; old-growth forest

# Introduction

The Marbled Murrelet, *Brachyramphus marmoratus* (Gmelin, 1789), was a seabird that nested inland in old-growth coniferous forest. It was listed as threatened in Canada and the United States, and loss of that forest was the primary reason given for its decline (Burger, 2001). In a review of science for federal forests in Washington, Oregon and northern California, Raphael et al. (2018, p. 333) wrote that the studies they summarized pointed “toward nesting habitat as the primary driver, but all these studies concede that relationships are correlational”. The murrelet’s nests were cryptic and difficult to locate (Burger, 2001), so the nesting habitat retained for it in British Columbia was mapped from forest inventory, air photographs and aerial surveys (Burger et al., 2018; Mather et al., 2010). Those maps were judged against the few nests found by radio telemetry (Burger et al., 2018). Burger et al. (2009, p. 12) did “not anticipate having a much larger sample of actual nest sites to test, due to the high costs of telemetry studies needed to find large numbers of nests”.

Nesting habitat was defined by nest platforms, the large limbs and thick epiphyte mats on which murrelets nested (Burger et al., 2010). Ralph et al. (1995, p. 7) wrote that “it is the presence of adequate nesting platforms that defines suitable nesting habitat”, and later work confirmed platforms as “the most important characteristic of murrelet nesting habitat” (Raphael et al., 2018, p. 303). Across six regions of British Columbia, tree diameter was the most important predictor of platforms (Burger et al., 2010). Platforms began “to appear regularly in trees \>30 m tall” and were “most abundant in the large emergent trees” (Burger et al., 2010, p. 736). In Washington and Oregon, nests sat on platforms with more horizontal cover, a larger diameter and more moss than other platforms in the same stands (Hamer et al., 2021). Silvergieter and Lank (2011a, p. 15) noted that the number of platforms, a key term in their Clayoquot Sound models, “cannot be quantitatively measured by remote methods”.

The provincial habitat model classed forest-cover polygons as suitable by stand age, height, elevation and distance to the sea (Mather et al., 2010). Its authors judged it “suited to broad-scale estimation of Marbled Murrelet nesting habitat area, but not its precise location” (Mather et al., 2010, p. 98). Against 171 radio-tracked nests, it placed 52.6 percent in suitable habitat, 69.4 percent in Clayoquot Sound and 52.1 percent in Desolation Sound (Burger et al., 2018). In Clayoquot Sound, the model applied a regional algorithm built from inventory maps drawn for that area (Bahn and Newsom, 2002; Mather et al., 2010), and its better result there “might be attributed” to that algorithm (Burger et al., 2018, p. 13).

Air photo interpretation and low-level aerial surveys ranked stands by canopy features seen from above (Burger et al., 2009; Waterhouse et al., 2008). Fine-scale aerial surveys placed 85.2 percent of 135 nests in suitable habitat and were the only method that estimated nest platforms (Burger et al., 2018). Their ranks separated nests from random points in Desolation Sound and Toba Inlet but not in Clayoquot Sound (Waterhouse et al., 2009). Re-examining the first comparison of radio-tracked nests with random points in the two sounds, Burger and Page (2007, p. 1276) contended that “coarse-scale satellite imagery and forest cover mapping lack the spatial resolution to adequately test effects of patch size and forest edges on marbled murrelets”. They found “little or no support” for that study’s main conclusions and named “tree size, moss development, canopy micro-structure, aspect, topographic complexity within stands” as likely to matter more (Burger and Page, 2007, p. 1279). All of these methods inferred forest structure from maps or from observers’ ranks rather than measuring it.

Airborne laser scanning measured canopy height and the vertical arrangement of vegetation directly, and it had been linked to the habitat of many forest animals. Across 173 studies, Acebes et al. (2021, p. 7) found that “canopy vertical distribution traits were less important for birds than canopy height and cover”. Across 50 bird studies, measures of canopy vertical variability showed a relationship in only 44 percent of their uses (Bakx et al., 2019). In the Sierra Nevada, cover in trees taller than 48 m was “the canopy structure most highly selected for” by California spotted owls, while cover from 2 to 16 m was avoided (North et al., 2017, p. 166). In that study, measures of gap area were “among the least distinct” (North et al., 2017, p. 171). In Oregon, northern spotted owls selected activity centers with more canopy cover and a more uneven canopy surface within 100 m (Sovern et al., 2019). Sovern et al. (2019, p. 722) linked the uneven canopy to old stands, where trees “senesce, break, and fall over creating small openings in the canopy”. In western Washington, conifer stands older than about 75 years showed a rougher canopy surface and a lower canopy density as they aged (Kane et al., 2010).

For the murrelet, Burger et al. (2009, p. 12) wrote that laser scanning “might provide detailed three-dimensional measures of canopy structure that could be extremely valuable for large-scale assessments of Marbled Murrelet nesting habitats”. In southwestern Oregon, a lidar model of stand occupancy scored 0.74 on the area under the receiver operating characteristic curve (AUC), against 0.64 for a model built from vegetation maps (Hagar et al., 2014). An AUC of 0.5 marked a model that ranked occupied stands no better than chance. On northern Vancouver Island, measures mostly from laser scans flown in 2012 predicted helicopter survey ranks with an overall accuracy of 71.05 percent, and elevation outranked every canopy measure (Clyde, 2017). In northern California redwood timberland scanned in 2017 and 2018, measures of tall individual trees predicted habitat suitability, while area-based canopy and height-percentile measures “displayed little predictive value” beside them (Lamphear et al., 2025, p. 12). Lamphear et al. (2025, p. 23) wrote that the tree measures were “more direct measures of small-scale marbled murrelet habitat requirements than indirect area-based point cloud metrics”. All three models were fitted to survey records of occupancy or habitat rank rather than to nests (Clyde, 2017; Hagar et al., 2014; Lamphear et al., 2025). Hagar et al. (2014) also compared 11 climbed nest trees with random points. Two measures related to tree height and cover were larger at the nests, with an average AUC of 0.91.

In Desolation Sound, Cosgrove et al. (2024) trained a laser-scan model on 58 radio-tracked nests, using maximum entropy modeling (MaxEnt) in 1 ha cells. Predicted suitability rose with forest vertical complexity, the spread of point heights, and fell with internal forest gaps, the empty space below the top of the canopy. The fall with gaps reversed the authors’ expectation that gaps would favor nesting by “providing canopy openings for nest access and flight maneuvers” (Cosgrove et al., 2024, p. 4). Burger et al. (2009, p. 5) reported that “small gaps in the canopy are typically found next to nest trees”. On Haida Gwaii, small gaps were prevalent on air photos at all 7 nest patches found in 2000 (Waterhouse et al., 2007). On the south coast, a model fitted to 118 nests tracked from 1998 to 2002 in three areas, and judged “most applicable to Desolation Sound”, predicted nests where “small gaps are sporadic but where large gaps do occur” (Waterhouse et al., 2008, p. 27).

The effect of edges and openings also changed with their kind and their scale. In four regions of southwestern British Columbia, disturbance of simulated nests was 2.5 times higher at hard edges, beside clearcuts 5 to 11 years old, than in forest interiors (Malt and Lank, 2009). Malt and Lank (2009, p. 1274) found “no edge effect at natural edges”, which lay beside large rivers or avalanche chutes. Natural-edged patches also held “the greatest levels of epiphyte cover and platform tree density” (van Rooyen et al., 2011, p. 549). Across Washington, Oregon and northern California, occupancy fell with edge within 2 km of survey stations and rose with edge within 100 m (Valente et al., 2023). Those authors “did not distinguish natural and anthropogenic canopy gaps” (Valente et al., 2023, p. 8). In western Washington and southwest Vancouver Island, 18 radio-tracked nest sites had less canopy cover of dominant conifers than nearby control plots, 39.3 against 48.6 percent (Wilk et al., 2016). Wilk et al. (2016, p. 170) read this as “a more open structure around nest sites than in other areas of the forest”.

For vertical complexity, Silvergieter and Lank (2011b, p. 10) wrote that “preference for heterogeneous forest patches with increased vertical canopy complexity has been well established by other studies”. On southwest Vancouver Island from 1991 to 1999, the spread of tree heights in ground plots at 27 survey stations correlated with occupied murrelet activity \[Spearman *r*<sub>s</sub> = 0.495, *P* \< 0.01; Burger and Bahn (2004)\]. In Washington and Oregon from 1995 to 1999, however, climbers searching random plots found the effect of “tree diameter standard deviation (a measure of canopy complexity)” on nesting unclear (Hamer et al., 2021, p. 8). In Clayoquot Sound, ground plots at 27 nests found from 2000 to 2002 showed canopy shorter and less varied in height than at random sites (Silvergieter and Lank, 2011a). Mean canopy height was 33.4 against 37.6 m and its standard deviation 4.9 against 6.0 m, whereas at 37 Desolation Sound nests both ran the other way (Silvergieter and Lank, 2011a). From laser scans at 1 ha, by contrast, nests in both sounds lay in taller and more vertically complex forest than random cells (Cosgrove et al., 2024). Waterhouse et al. (2009, p. 91) found “only weak evidence” that vertical complexity rated from a helicopter helped to predict nests in Desolation Sound, Toba Inlet and Clayoquot Sound.

Models had been moved from Desolation Sound to Clayoquot Sound before laser scans were used. Zharikov et al. (2007) fitted landscape models from forest inventory maps to 90 Desolation Sound nests and tested them on 36 Clayoquot Sound nests. There, the models scored AUCs of 0.558 to 0.663 (ZHARIKOV et al., 2007). The authors traced the weak transfer to old growth, which covered 18 percent of the training extent against 55 percent of the test extent (ZHARIKOV et al., 2007). Transferred to Clayoquot Sound, the Desolation Sound laser-scan model scored an AUC of 0.63, against 0.77 where it was trained (Cosgrove et al., 2024). Its Boyce index, a measure of how closely predicted suitability tracked nest frequency, fell from 0.99 to 0.67 (Cosgrove et al., 2024). The AUC fell below the 0.7 those authors set for a good model, yet they judged that the transferred model “maintained reasonable accuracy” (Cosgrove et al., 2024, p. 9). In Oregon, a lidar occupancy model transferred from the Coos Bay District to the Siuslaw National Forest scored an AUC of 0.73, against 0.78 for a locally selected model (Hagar et al., 2018). Hagar et al. (2018, p. 1) found that “neither of these transferred models were considered competitive with a model trained on local data”. In the two sounds, ground-plot models fitted to each region placed 59 percent of Clayoquot Sound nests above a score of 0.7, against 15 percent for a pooled model (Silvergieter and Lank, 2011a). Silvergieter and Lank (2011a, p. 13) concluded that one resource selection function “does not describe habitat selection nearly as well as region-specific models”. They judged that “the limited size of our dataset and strong regional interactions precluded” testing Desolation Sound models against Clayoquot Sound nests (Silvergieter and Lank, 2011a, p. 11). Of the provincial habitat classifications, Burger et al. (2018, p. 15) wrote that “the regional differences that have been found suggest caution in applying results from one region to another”.

Desolation Sound had lost about 80 percent of its original nesting habitat to logging (Burger et al., 2018). Burger et al. (2018, p. 11) wrote that the distribution of nests in what remained “might not be typical for the species in less disturbed landscapes, such as Clayoquot Sound”. Burger and Page (2007, p. 1278) had asked whether the results from Clayoquot Sound, “with relatively little forest loss”, did not “represent a truer reflection on the habitat preferences of nesting murrelets”. Air photo interpretation detected the fewest signs of nest-site selection in Clayoquot Sound, in part because “random sites would more often fall in poorer habitats on the Sunshine Coast compared to Clayoquot Sound with its extensive areas of available habitat” (Waterhouse et al., 2008, p. 31).

The way nests were compared with other forest also limited what these studies could detect. Because murrelet use of random sites was unknown, Waterhouse et al. (2008, p. 21) compared nests with available sites “rather than the more powerful contrast of used versus unused sites”. At Clayoquot Sound, the nesting dates of radio-tracked birds “were substantially biased towards later nests” (Tranquilla et al., 2005, p. 357). In Oregon, Valente et al. (2024, p. 10) were “only able to find 17 active nests despite radiotagging 239 individuals” over four breeding seasons. Raphael et al. (2018, p. 333) proposed to “build models exclusively from training sites that represent actual murrelet nests”.

The Clayoquot Sound nests had served to test models transferred from Desolation Sound, built from forest inventory (ZHARIKOV et al., 2007) and from laser scans (Cosgrove et al., 2024). The provincial scans flown there in 2019 and the public release of the nest locations (Cooke et al., 2022) made a local model possible. The first objective was to rebuild the two-measure model of Cosgrove et al. (2024) from local nests and laser scans and to score it on spatially separate groups of nests. The second was to compare that score with that of the Desolation Sound model transferred to Clayoquot Sound. Forest vertical complexity was expected to be higher at nest cells than in other available forest, as it was in Desolation Sound (Cosgrove et al., 2024) and in earlier ground studies (Burger and Bahn, 2004; Silvergieter and Lank, 2011b). Ground plots at Clayoquot Sound nests had shown the reverse (Silvergieter and Lank, 2011a), so this direction was contested there. Internal forest gaps were expected to be smaller at nest cells, as in Desolation Sound (Cosgrove et al., 2024). Small gaps had nonetheless been reported at nest sites (Burger et al., 2009; Waterhouse et al., 2007), and nest sites had been more open than nearby forest (Wilk et al., 2016). Models fitted within one region had described nest sites or occupied stands there better than pooled or transferred models (Hagar et al., 2018; Silvergieter and Lank, 2011a). The local model was therefore expected to score an AUC above the 0.63 of the transferred model. The expected direction of each measure and its source were listed in <a href="#tbl-attributes" class="quarto-xref">Table 1</a>.

``` r
readr::read_csv("../02.inputs/derived/review-attributes-20261113.csv", show_col_types = FALSE) |>
  dplyr::transmute(Measure = measure, `Expected direction` = expected_direction, Reasoning = reasoning, Source = paste0("[", gsub("; ", "; @", paste0("@", citation)), "]")) |>
  knitr::kable(format = "pipe")
```

<div id="tbl-attributes">

Table 1: Laser-scan measures entered in the model, with the direction of their relation to nest sites expected from prior work.

<div class="cell-output-display">

| Measure | Expected direction | Reasoning | Source |
|:---|:---|:---|:---|
| Forest vertical complexity | Positive | Taller forest with more variable structure was expected to offer more nest platforms and overhead cover. In Desolation Sound, predicted suitability rose with vertical complexity, and earlier ground studies had linked murrelets to vertically complex forest. Ground plots at Clayoquot Sound nests found shorter and less varied canopy than at random sites, so the direction was contested there. | (Burger and Bahn, 2004; Cosgrove et al., 2024; Silvergieter and Lank, 2011a, 2011b) |
| Internal forest gaps | Negative | Gaps within and under the canopy were expected to give flight access to nests. In Desolation Sound, however, predicted suitability fell as internal gaps increased. On the south coast, an air photo model placed nests where small gaps were sporadic but large gaps occurred. Small gaps had nonetheless been reported beside nest trees and at all 7 nest patches studied on Haida Gwaii. In Washington and on southwest Vancouver Island, nest sites were more open than nearby forest. | (Burger et al., 2009; Cosgrove et al., 2024; Waterhouse et al., 2008; Waterhouse et al., 2007; Wilk et al., 2016) |

</div>

</div>

# Methods

## Study area

Clayoquot Sound, on the west coast of Vancouver Island, British Columbia (<a href="#fig-study-area" class="quarto-xref">Figure 1</a>), supported one of the highest concentrations of breeding Marbled Murrelets south of Alaska, including more than 10 percent of the estimated provincial population (Burger, 2001). The study area was the smallest convex polygon around the nests, widened by 5 km.

## Nest data

From 2000 to 2002, Marbled Murrelets captured at sea were radio-tracked to their nests in Clayoquot Sound (Cosgrove et al., 2024). The published data set listed 36 nests confirmed on the ground, with a positional error of about 50 m (Cooke et al., 2022). No nest location was mapped in this paper.

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

## Laser scans

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

Point clouds came from LidarBC, the open lidar program of the Province of British Columbia, and only nests inside its 2019 Vancouver Island acquisition were used. They were downloaded as compressed LAS files, with ground and noise classes inherited from the contractor. Of the 322 tiles centered in the study area, the 29 within 150 m of a nest and 40 drawn at random from the rest were used. Points classed as noise or withheld were dropped, and each tile was thinned at random and evened to 12 points per square meter in 10 m cells, so that point density did not differ between nest and background tiles.

## Harvest screen

Forest harvested from 1998 to 2022 was screened out with the Canada Forest Harvest 1985–2022 product of the National Terrestrial Ecosystem Monitoring System, which records the year of stand-replacing harvest on a 30 m grid (Hermosilla et al., 2016).

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
  main <- ggplot2::ggplot() +
    ggplot2::geom_tile(data = harvest, ggplot2::aes(x, y, fill = "Harvested 1998 to 2022"), width = 90, height = 90) +
    ggplot2::geom_sf(data = coast, colour = "grey35", linewidth = 0.2) +
    ggplot2::geom_sf(data = sf::st_union(tiles), ggplot2::aes(colour = "LidarBC 2019 scan tiles"), fill = NA, linewidth = 0.4) +
    ggplot2::geom_sf(data = area, ggplot2::aes(linetype = "Study area"), fill = NA, colour = "black", linewidth = 0.4) +
    ggplot2::scale_fill_manual(NULL, values = "#D55E00") + ggplot2::scale_colour_manual(NULL, values = "#0072B2") + ggplot2::scale_linetype_manual(NULL, values = "dashed") +
    ggplot2::annotate("text", x = 267200, y = 5456300, label = "Pacific Ocean", hjust = 0, fontface = "italic", size = 2.6, colour = "grey30") +
    ggplot2::annotate("rect", xmin = 306500 + c(0, 2500, 5000, 7500), xmax = 306500 + c(2500, 5000, 7500, 10000), ymin = 5488500, ymax = 5489000, fill = c("black", "white", "black", "white"), colour = "black", linewidth = 0.2) +
    ggplot2::annotate("text", x = 306500 + c(0, 5000, 10000), y = 5489800, label = c("0", "5", "10 km"), size = 2.4) +
    ggplot2::annotate("text", x = 311500, y = 5487300, label = "Scale 1:400,000", size = 2.4) +
    ggplot2::coord_sf(crs = 3157, xlim = frame[c(1, 3)], ylim = frame[c(2, 4)], expand = FALSE, datum = NA) +
    ggplot2::theme_void(base_size = 8) +
    ggplot2::theme(panel.border = ggplot2::element_rect(fill = NA, colour = "black", linewidth = 0.5), panel.background = ggplot2::element_rect(fill = "white", colour = NA), legend.position = "none", plot.margin = ggplot2::margin(0, 0, 0, 0))
  key <- cowplot::get_plot_component(main + ggplot2::theme(legend.position = "right", legend.spacing.y = grid::unit(0, "pt"), legend.margin = ggplot2::margin(0, 0, 0, 0), legend.key.size = grid::unit(9, "pt")), "guide-box-right")
  locator <- ggplot2::ggplot() +
    ggplot2::geom_sf(data = land, fill = "grey80", colour = NA) +
    ggplot2::geom_sf(data = sf::st_as_sfc(frame), fill = NA, colour = "black", linewidth = 0.4) +
    ggplot2::coord_sf(crs = 3157, xlim = c(100000, 800000), ylim = c(5250000, 5900000), expand = FALSE, datum = NA) +
    ggplot2::theme_void() + ggplot2::theme(panel.background = ggplot2::element_rect(fill = "white", colour = NA))
  fig <- cowplot::ggdraw(main) +
    cowplot::draw_grob(grid::rectGrob(gp = grid::gpar(fill = "white", col = "grey40", lwd = 0.8)), x = 0.012, y = 0.665, width = 0.245, height = 0.323) +
    cowplot::draw_plot(locator, x = 0.022, y = 0.79, width = 0.225, height = 0.19) +
    cowplot::draw_grob(key, x = 0.022, y = 0.672, width = 0.225, height = 0.11, hjust = 0, vjust = 0) +
    ggplot2::theme(plot.background = ggplot2::element_rect(fill = "white", colour = NA))
  ggplot2::ggsave("../03.outputs/figures/F1_study_area.png", fig, width = 58000 / 4e3, height = 55000 / 4e3, units = "cm", dpi = 300)
})
knitr::include_graphics("../03.outputs/figures/F1_study_area.png")
```

<div id="fig-study-area">

![](../03.outputs/figures/F1_study_area.png)

Figure 1: Clayoquot Sound on the west coast of Vancouver Island, British Columbia. The dashed line bounds the study area, drawn 5 km around the radio-tracked nests, the blue line bounds the 322 LidarBC scan tiles flown in 2019, each about 2.5 km², and orange marks forest harvested from 1998 to 2022. Nest locations are not shown. The inset places the study area on Vancouver Island. Scale 1:400,000.

</div>

## Forest structure

Heights were normalized to the ground by a triangulated surface on ground points, and points below 0 m or at or above 100 m were discarded. Following the grid and definitions of Cosgrove et al. (2024), five measures were computed in 1 ha cells with lidR (Roussel et al., 2020). Forest vertical complexity was the standard deviation of point heights, and internal forest gaps were the number of empty 1 m³ voxels below the highest filled voxel of each 1 m column, summed over the cell. Forest height, the 95th percentile of point heights, canopy cover, the share of 1 m columns with a point, and point density were used only to define available forest.

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

## Available forest

A cell was available forest if

1.  forest height was at least 20 m,
2.  canopy cover was at least 90 percent,
3.  point density was at least 10 points per square meter, and
4.  no 30 m pixel inside it was harvested from 1998 to 2022.

Nest cells were available-forest cells with a nest, and background cells were all available-forest cells in the background tiles (<a href="#tbl-sample" class="quarto-xref">Table 2</a>). The nest sample was small, and the nests were found 17 to 19 years before the scans were flown.

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
build_once("../03.outputs/tables/T2_sample.csv", tibble::tibble(
  step = c("nests in file", "nest cells scanned", "nest cells in available forest",
    "background cells scanned", "background cells in available forest"),
  n = c(nrow(nests), sum(cells$nest & !is.na(cells$fh)), sum(cells$nest & cells$forest),
    sum(cells$background & !is.na(cells$fh)), sum(cells$background & cells$forest))) |>
  readr::write_csv("../03.outputs/tables/T2_sample.csv"))
```

``` r
tbl("T2_sample.csv") |>
  dplyr::transmute(Step = stringr::str_to_sentence(step), Count = formatC(n, format = "d", big.mark = ",")) |>
  flextable::flextable() |> flextable::align(j = 2, align = "right", part = "all") |> flextable::set_table_properties(layout = "autofit", width = 1)
```

<div id="tbl-sample">

Table 2: Nests and 1 ha cells at each step from the nest file to the modeled sample. Available forest was at least 20 m tall with at least 90% canopy cover and no harvest from 1998 to 2022.

<div class="cell-output-display">

| Step                                 | Count |
|:-------------------------------------|:------|
| Nests in file                        | 36    |
| Nest cells scanned                   | 20    |
| Nest cells in available forest       | 16    |
| Background cells scanned             | 7,869 |
| Background cells in available forest | 3,827 |

</div>

</div>

## Model fitting

Following the top model of Cosgrove et al. (2024), MaxEnt was fitted to forest vertical complexity and internal forest gaps in the maxnet package (Phillips et al., 2017), with linear and product features and a regularization multiplier of 0.2. Predictions were taken on the complementary log-log scale. For spatial block cross-validation, cells were assigned to five folds by 10 km square blocks allocated at random (Valavi et al., 2018), and each fold was predicted by a model trained on the other four. Each model’s classification threshold maximized the sum of sensitivity, the share of nest cells classed as nesting habitat, and specificity, the share of background cells classed as not, on its training cells (Liu et al., 2005).

``` r
sample <- cells |>
  dplyr::filter(forest, nest | background) |>
  dplyr::mutate(p = as.integer(nest))
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
  threshold <- pROC::coords(pROC::roc(train$p, score(fit, train), quiet = TRUE), "best", ret = "threshold")$threshold[1]
  test <- sample[folds$folds_ids == k, ] |> dplyr::mutate(fold = k)
  list(test = dplyr::mutate(test, pred = score(fit, test), class = as.integer(pred >= threshold)), train_rmse = sqrt(mean((score(fit, train) - train$p)^2)))
})
held_out <- purrr::map(runs, "test") |> purrr::list_rbind()
```

## Separability

On each measure, the separability of nest and background cells was given by the index *M* of Kaufman and Remer (1994), the absolute difference between the two means divided by the sum of the two standard deviations. The difference was tested with a Wilcoxon rank-sum test, as in Cosgrove et al. (2024).

``` r
build_once("../03.outputs/tables/T3_separability.csv", sample |>
  tidyr::pivot_longer(c(fvc, empty_gaps), names_to = "measure") |>
  dplyr::summarise(.by = measure,
    nest_mean = mean(value[p == 1]), nest_sd = sd(value[p == 1]),
    nest_low = t.test(value[p == 1])$conf.int[1], nest_high = t.test(value[p == 1])$conf.int[2],
    background_mean = mean(value[p == 0]), background_sd = sd(value[p == 0]),
    background_low = t.test(value[p == 0])$conf.int[1], background_high = t.test(value[p == 0])$conf.int[2],
    m = abs(nest_mean - background_mean) / (nest_sd + background_sd),
    w = wilcox.test(value[p == 1], value[p == 0])$statistic,
    p_value = wilcox.test(value[p == 1], value[p == 0])$p.value) |>
  readr::write_csv("../03.outputs/tables/T3_separability.csv"))
```

## Evaluation

Held-out predictions were pooled across folds and scored with the measures of Murphy et al. (2026). Discrimination was measured by AUC with a DeLong 95 percent confidence interval (CI) (Robin et al., 2011) and the continuous Boyce index (Hirzel et al., 2006). Classification at each fold’s threshold was summarized by sensitivity, specificity, overall accuracy with an exact binomial interval, Cohen’s kappa and the commission and omission error of each class. Agreement between prediction and the observed 0 or 1 outcome was measured by root mean square error (RMSE), relative RMSE (RMSE over the mean outcome), mean absolute error, Brier score, the coefficient of determination (*R*²) of outcome on prediction with its *F* test. Theil’s *U* bias gave the share of squared error due to the gap between mean prediction and mean outcome. Overfitting was measured by the RMSE ratio, held-out over training RMSE, with a model rejected above 1.1 (Murphy et al., 2026), and residual autocorrelation by Moran’s *I* with inverse-distance weights and the Durbin-Watson test on residuals ordered by easting. The model was compared with that of Cosgrove et al. (2024), in Desolation Sound and transferred to Clayoquot Sound, on the AUC and Boyce index those authors reported. Analyses were run in R 4.4.1 (R Core Team, 2024) with lidR 4.2.3, maxnet 0.1.4, blockCV 3.2.0, pROC 1.19.0.1 and ecospat 4.1.3.

``` r
roc <- pROC::roc(held_out$p, held_out$pred, quiet = TRUE)
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
build_once("../03.outputs/tables/T4_evaluation.csv", tibble::tribble(
  ~model, ~metric, ~value, ~low, ~high, ~p_value,
  "local", "Nest cells", sum(held_out$p), NA, NA, NA,
  "local", "Background cells", sum(held_out$p == 0), NA, NA, NA,
  "local", "Nest cells above threshold", cm["1", "1"], NA, NA, NA,
  "local", "Background cells above threshold", cm["1", "0"], NA, NA, NA,
  "local", "Mean prediction", mean(held_out$pred), NA, NA, NA,
  "local", "Nest share of cells", mean(held_out$p), NA, NA, NA,
  "local", "F statistic", r2$fstatistic[1], r2$fstatistic[2], r2$fstatistic[3], NA,
  "local", "AUC", auc[2], auc[1], auc[3], NA,
  "local", "Boyce index", ecospat::ecospat.boyce(held_out$pred[held_out$p == 0], held_out$pred[held_out$p == 1], PEplot = FALSE)$cor, NA, NA, NA,
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
  readr::write_csv("../03.outputs/tables/T4_evaluation.csv"))
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

The 2019 scans covered 20 cells with a nest, of which 16 were available forest, against 3,827 available-forest cells among the 7,869 scanned cells of the background tiles (<a href="#tbl-sample" class="quarto-xref">Table 2</a>).

## Separability

Neither measure separated nest cells from background cells (<a href="#tbl-separability" class="quarto-xref">Table 3</a>, <a href="#fig-structure" class="quarto-xref">Figure 2</a>). Forest vertical complexity averaged 10.67 ± 1.47 m (95 percent CI 9.89 to 11.46) at nest cells and 10.37 ± 2.43 m (10.29 to 10.45) at background cells \[*M* = 0.08, Wilcoxon rank-sum test, *W* = 33,355, *P* = 0.54\]. Internal forest gaps were larger at nest cells, 177.5 ± 40.9 thousand empty voxels per hectare (155.7 to 199.3), than at background cells, 157.5 ± 56.8 thousand (155.7 to 159.3), the reverse of the negative relation found in Desolation Sound, although the difference was not significant \[*M* = 0.21, *W* = 37,889, *P* = 0.10\].

``` r
star <- \(p) dplyr::case_when(is.na(p) ~ "", p <= 0.0001 ~ "****", p <= 0.001 ~ "***", p <= 0.01 ~ "**", p <= 0.05 ~ "*", TRUE ~ "")
tbl("T3_separability.csv") |>
  dplyr::mutate(k = ifelse(measure == "empty_gaps", 1000, 1), dplyr::across(nest_mean:background_high, \(x) x / k)) |>
  dplyr::transmute(Measure = ifelse(measure == "fvc", "Forest vertical complexity (m)", "Internal forest gaps"),
    Nest = paste0(fmt(nest_mean), " ± ", fmt(nest_sd), " (", fmt(nest_low), "–", fmt(nest_high), ")"),
    Background = paste0(fmt(background_mean), " ± ", fmt(background_sd), " (", fmt(background_low), "–", fmt(background_high), ")"),
    M = fmt(m), W = paste0(fmt(w, 0), star(p_value))) |>
  flextable::flextable() |> flextable::align(j = 2:5, align = "right", part = "body") |> flextable::align(align = "center", part = "header") |> flextable::set_table_properties(layout = "autofit", width = 1)
```

<div id="tbl-separability">

Table 3: Laser-scan measures at nest and background cells in available forest, as mean ± standard deviation (95% confidence interval), with the separability index M and the Wilcoxon rank-sum statistic W. Internal forest gaps are in thousands of empty 1 m³ voxels per ha. \* P ≤ 0.05, \*\* P ≤ 0.01, \*\*\* P ≤ 0.001, \*\*\*\* P ≤ 0.0001.

<div class="cell-output-display">

| Measure | Nest | Background | M | W |
|:---|:---|:---|:---|:---|
| Forest vertical complexity (m) | 10.67 ± 1.47 (9.89–11.46) | 10.37 ± 2.43 (10.29–10.45) | 0.08 | 33,355 |
| Internal forest gaps | 177.52 ± 40.90 (155.73–199.31) | 157.46 ± 56.84 (155.66–159.26) | 0.21 | 37,889 |

</div>

</div>

``` r
knitr::include_graphics("../03.outputs/figures/F2_structure.png")
```

<div id="fig-structure">

![](../03.outputs/figures/F2_structure.png)

Figure 2: Forest vertical complexity and internal forest gaps at 16 nest cells (points) and 3,827 background cells in available forest, Clayoquot Sound. Boxes span the middle half of the values, the bar marks the median, and whiskers reach the furthest value within 1.5 box lengths.

</div>

## Model evaluation

The Clayoquot Sound model discriminated nest cells from background cells no better than chance (<a href="#tbl-evaluation" class="quarto-xref">Table 4</a>). Its AUC was 0.53 (95 percent CI 0.39 to 0.67), against 0.77 for the Desolation Sound model in its own area and 0.63 for that model transferred to Clayoquot Sound, and its Boyce index was 0.10 against 0.99 and 0.67. At the training thresholds, 10 of 16 held-out nest cells and 2,264 of 3,827 background cells were classed as nesting habitat, a sensitivity of 62.5 percent and a specificity of 40.8 percent, for an average accuracy of 51.7 percent. Cohen’s kappa, the agreement beyond chance, was 0.00 (95 percent CI -0.03 to 0.03).

The model did not overfit, its RMSE ratio of 1.01 lying below the rejection threshold of 1.1, but nearly all of its error was systematic \[Theil’s *U* bias = 0.96\], because predictions averaged 0.60 while nest cells made up 0.004 of all cells. Residuals were spatially autocorrelated \[Moran’s *I* = 0.12, *P* \< 0.001\].

``` r
show <- c("AUC" = "AUC", "Boyce index" = "Boyce index", "Overall accuracy" = "Overall accuracy", "Kappa" = "Kappa", "Commission error, nest" = "Commission error, nest", "Omission error, nest" = "Omission error, nest", "Commission error, background" = "Commission error, background", "Omission error, background" = "Omission error, background", "R2" = "R²", "RMSE" = "RMSE", "RMSE ratio" = "RMSE ratio", "Relative RMSE" = "Relative RMSE", "MAE" = "MAE", "Brier score" = "Brier score", "Theil's U bias" = "Theil's U bias", "Moran's I of residuals" = "Moran's I of residuals", "Durbin-Watson" = "Durbin-Watson")
tbl("T4_evaluation.csv") |>
  dplyr::filter(metric %in% names(show)) |>
  dplyr::mutate(cell = paste0(dplyr::if_else(metric == "R2", fmt(value, 5), fmt(value)), dplyr::if_else(is.na(low), "", paste0(" (", fmt(low), " to ", fmt(high), ")")), star(p_value))) |>
  tidyr::pivot_wider(id_cols = metric, names_from = model, values_from = cell, values_fill = "") |>
  dplyr::arrange(match(metric, names(show))) |>
  dplyr::transmute(Measure = show[metric], `Clayoquot Sound, local` = local, `Desolation Sound, own area` = desolation, `Desolation Sound, transferred` = transferred) |>
  flextable::flextable() |> flextable::align(j = 2:4, align = "right", part = "body") |> flextable::align(align = "center", part = "header") |> flextable::set_table_properties(layout = "autofit", width = 1)
```

<div id="tbl-evaluation">

Table 4: Performance of the Clayoquot Sound model on held-out cells pooled over five spatial folds, beside the Desolation Sound model of Cosgrove et al. (2024) in its own area and transferred to Clayoquot Sound. Intervals are 95% confidence intervals. AUC is the area under the receiver operating characteristic curve, RMSE root mean square error and MAE mean absolute error. \* P ≤ 0.05, \*\* P ≤ 0.01, \*\*\* P ≤ 0.001, \*\*\*\* P ≤ 0.0001.

<div class="cell-output-display">

| Measure | Clayoquot Sound, local | Desolation Sound, own area | Desolation Sound, transferred |
|:---|:---|:---|:---|
| AUC | 0.53 (0.39 to 0.67) | 0.77 | 0.63 |
| Boyce index | 0.10 | 0.99 | 0.67 |
| Overall accuracy | 0.41 (0.39 to 0.43) |  |  |
| Kappa | 0.00 (-0.03 to 0.03) |  |  |
| Commission error, nest | 1.00 |  |  |
| Omission error, nest | 0.38 |  |  |
| Commission error, background | 0.00 |  |  |
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

# Discussion

In Clayoquot Sound, a MaxEnt model rebuilt from 16 local nest cells and open 2019 laser scans ranked held-out nest cells above background cells no more often than chance. The 95 percent interval on its AUC included both chance and the 0.63 reached by the same model transferred from Desolation Sound (Cosgrove et al., 2024). The local model therefore did not outperform the transferred one, although these data could not show that it did worse. In Oregon, a locally selected lidar model also improved little on a transferred one, 0.78 against 0.73, but there the transferred model was already adequate on 772 surveyed stands (Hagar et al., 2018).

The two measures did not relate to nest sites in Clayoquot Sound as they did in Desolation Sound. Internal forest gaps were larger at nest cells than at background cells, the reverse of the Desolation Sound relation (Cosgrove et al., 2024), and forest vertical complexity hardly differed. Ground plots at 27 Clayoquot Sound and 37 Desolation Sound nests also showed opposite relations, with shorter and less varied canopy than at random sites in Clayoquot Sound but taller and more varied canopy in Desolation Sound (Silvergieter and Lank, 2011a). Nest trees in Clayoquot Sound exceeded other available trees in height by an average of 15 percent of total height, against 30 percent in Desolation Sound (Silvergieter and Lank, 2011b), a smaller contrast that a 1 ha summary of point heights would be less able to detect. If nest-site selection differed between the two sounds in this way, a model built on Desolation Sound relations would be expected to transfer poorly, as it did, and a model of the same form fitted in Clayoquot Sound would have little to separate.

The background cells may explain much of the weak result, because 49 percent of the scanned background cells met the definition of available forest, so nest cells were compared with forest of much the same kind. Waterhouse et al. (2009) gave the same reason for the failure of aerial survey ranks in Clayoquot Sound, which “has more uniform coverage of suitable forest (Zharikov et al. 2006); consequently a higher proportion of random points might fall within suitable nesting habitat, thereby minimizing the differences between nests and random points” (Waterhouse et al., 2009, p. 90). The features that decided which stand a bird used may also have gone unmeasured by a 1 ha summary of point heights. Silvergieter and Lank (2011a), who modeled nest-site selection with resource selection functions (RSF), noted that “relative numbers of potential nesting platforms, a key requirement of nesting habitat and an important component of RSF models at Clayoquot Sound, cannot be quantitatively measured by remote methods” (Silvergieter and Lank, 2011a, p. 15).

The small sample limited the comparison, because with 16 nest cells over five folds the 95 percent interval on AUC spanned 0.28, and spatially autocorrelated residuals meant that the cells were not independent, so the true uncertainty was probably wider. Nest positions had an error of about 50 m within 100 m cells (Cooke et al., 2022), and forest change other than harvest in the 17 to 19 years between the nest searches and the scans could not be measured. The error was almost entirely systematic because MaxEnt output on the complementary log-log scale ranked relative suitability and was not scaled to the small share of nest cells, so the RMSE, Brier score and *R*² described calibration more than discrimination.

For management in Clayoquot Sound, these results did not support replacing the transferred Desolation Sound model, or the provincial habitat model, which placed 69.4 percent of Clayoquot Sound nests in suitable habitat (Burger et al., 2018), with a model fitted to the few local nests. Two approaches might make better use of open scans than two stand-level measures, and both could be tested with data already public. One would be a model fitted to nests pooled across regions with terms for region, as Silvergieter and Lank (2011a) fitted to ground plots, and the other would use measures of individual tall trees taken from the scans, the predictors of Lamphear et al. (2025).

# Conclusion

Open laser scans flown in 2019 and 16 local nest cells were not enough to build a Marbled Murrelet nesting-habitat model for Clayoquot Sound that separated nests from available forest, and the local model could not be shown to improve on the model transferred from Desolation Sound. Until more nests could be located in Clayoquot Sound, the transferred model, or a model fitted to nests pooled across regions, offered the better-supported basis for mapping nesting habitat there.

# Data availability

Every line of code that built, fitted and tested the model is in the manuscript source at https://github.com/seamusrobertmurphy/lidar-marbled-murrelet. Nest locations are public under a CC0 license at the Movebank Data Repository (Cooke et al., 2022), point clouds at LidarBC (https://lidar.gov.bc.ca/) under the Open Government Licence British Columbia, and harvest years at the National Terrestrial Ecosystem Monitoring System (Hermosilla et al., 2016).

# Author contributions

Seamus Murphy carried out the conceptualization, methodology, software, formal analysis, data curation and visualization, and wrote, reviewed and edited the manuscript.

# Funding

This research did not receive any specific grant from funding agencies in the public, commercial, or not-for-profit sectors.

# Competing interests

The author declares no known competing financial interests or personal relationships that could have appeared to influence the work reported in this paper.

# References

<div id="refs" class="references csl-bib-body hanging-indent" entry-spacing="0">

<div id="ref-acebes2021disentangling" class="csl-entry">

Acebes, P., Lillo, P., Jaime-González, C., 2021. Disentangling LiDAR contribution in modelling species–habitat structure relationships in terrestrial ecosystems worldwide. A systematic review and future directions. Remote Sensing 13, 3447. <https://doi.org/10.3390/rs13173447>

</div>

<div id="ref-bahn2002habitat" class="csl-entry">

Bahn, V., Newsom, D., 2002. Habitat suitability mapping for marbled murrelets in Clayoquot Sound, in: Burger, A.E., Chatwin, T.A. (Eds.), Multi-Scale Studies of Populations, Distribution and Habitat Associations of Marbled Murrelets in Clayoquot Sound, British Columbia. Ministry of Water, Land; Air Protection, Victoria, British Columbia, pp. 101–119.

</div>

<div id="ref-bakx2019categorization" class="csl-entry">

Bakx, T.R.M., Koma, Z., Seijmonsbergen, A.C., Kissling, W.D., 2019. Use and categorization of light detection and ranging vegetation metrics in avian diversity and species distribution research. Diversity and Distributions 25, 1045–1059. <https://doi.org/10.1111/ddi.12915>

</div>

<div id="ref-burger2001using" class="csl-entry">

Burger, A.E., 2001. Using radar to estimate populations and assess habitat associations of marbled murrelets. The Journal of Wildlife Management 65, 696. <https://doi.org/10.2307/3803021>

</div>

<div id="ref-burger2004inland" class="csl-entry">

Burger, A.E., Bahn, V., 2004. Inland habitat associations of marbled murrelets on southwest Vancouver Island, British Columbia. Journal of Field Ornithology 75, 53–66. <https://doi.org/10.1648/0273-8570-75.1.53>

</div>

<div id="ref-burger2007need" class="csl-entry">

Burger, A.E., Page, R.E., 2007. The need for biological realism in habitat modeling: A reinterpretation of Zharikov et al. (2006). Landscape Ecology 22, 1273–1281. <https://doi.org/10.1007/s10980-007-9114-6>

</div>

<div id="ref-burger2010factors" class="csl-entry">

Burger, A.E., Ronconi, R.A., Silvergieter, M.P., Conroy, C., Bahn, V., Manley, I.A., Cober, A., Lank, D.B., 2010. Factors affecting the availability of thick epiphyte mats and other potential nest platforms for marbled murrelets in british columbia. Canadian Journal of Forest Research 40, 727–746. <https://doi.org/10.1139/x10-034>

</div>

<div id="ref-burger2009new" class="csl-entry">

Burger, A.E., Waterhouse, F.L., Donaldson, A., Whittaker, C., Lank, D.B., 2009. New methods for assessing marbled murrelet nesting habitat: Air photo interpretation and low-level aerial surveys. Journal of Ecosystems and Management. <https://doi.org/10.22230/jem.2009v10n1a407>

</div>

<div id="ref-burger2018reliability" class="csl-entry">

Burger, A., Waterhouse, F.L., Deal, J., Lank, D., Donald, D.S., 2018. The reliability and application of methods used to predict suitable nesting habitat for marbled murrelets. Journal of Ecosystems and Management 18. <https://doi.org/10.22230/jem.2018v18n1a593>

</div>

<div id="ref-clyde2017fine" class="csl-entry">

Clyde, G.E., 2017. [A fine-scale lidar-based habitat suitability mapping methodology for the marbled murrelet (*Brachyramphus marmoratus*) on Vancouver Island, British Columbia](https://dspace.library.uvic.ca/handle/1828/7923) (Master’s thesis). University of Victoria, Victoria, British Columbia.

</div>

<div id="ref-cooke2022marbled" class="csl-entry">

Cooke, F., Lank, D.B., Bradley, R.W., Parker, N.R., Lougheed, C., Lougheed, L.W., McFarlane Tranquila, L.A., Pastran, S.A., Greene, R., Vincent, P., Bertram, D.F., 2022. Data from: Study "marbled murrelets at Clayoquot Sound, British Columbia, Canada (2000-2002)". <https://doi.org/10.5441/001/1.KG210889>

</div>

<div id="ref-cosgrove2024modeling" class="csl-entry">

Cosgrove, C., Coops, N., Waterhouse, F., Goodbody, T., 2024. Modeling marbled murrelet nesting habitat: A quantitative approach using airborne laser scanning data in British Columbia, Canada. Avian Conservation and Ecology 19. <https://doi.org/10.5751/ace-02585-190105>

</div>

<div id="ref-hagar2018modeling" class="csl-entry">

Hagar, J.C., Aragon, R., Haggerty, P., Hollenbeck, J.P., 2018. Modeling habitat for marbled murrelets on the Siuslaw National Forest, Oregon, using lidar data, Open-File Report. US Geological Survey. <https://doi.org/10.3133/ofr20181035>

</div>

<div id="ref-hagar2014modeling" class="csl-entry">

Hagar, J.C., Eskelson, B.N.I., Haggerty, P.K., Nelson, S.K., Vesely, D.G., 2014. Modeling marbled murrelet (*Brachyramphus marmoratus*) habitat using LiDAR-derived canopy data. Wildlife Society Bulletin 38, 237–249. <https://doi.org/10.1002/wsb.407>

</div>

<div id="ref-hamer2021marbled" class="csl-entry">

Hamer, T.E., Nelson, K., Jones, J., Verschuyl, J., 2021. Marbled Murrelet nest site selection at three fine spatial scales. Avian Conservation and Ecology 16, art4. <https://doi.org/10.5751/ace-01883-160204>

</div>

<div id="ref-hermosilla2016mass" class="csl-entry">

Hermosilla, T., Wulder, M.A., White, J.C., Coops, N.C., Hobart, G.W., Campbell, L.B., 2016. Mass data processing of time series Landsat imagery: Pixels to data products for forest monitoring. International Journal of Digital Earth 9, 1035–1054. <https://doi.org/10.1080/17538947.2016.1187673>

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

<div id="ref-lamphear2025marbled" class="csl-entry">

Lamphear, D.W., Hamm, K.A., Early, D.A., McDonald, T.L., 2025. Marbled murrelet habitat suitability in redwood timberlands of northern coastal California utilizing LiDAR-derived individual tree metrics. PLOS ONE 20, e0307726. <https://doi.org/10.1371/journal.pone.0307726>

</div>

<div id="ref-liu2005selecting" class="csl-entry">

Liu, C., Berry, P.M., Dawson, T.P., Pearson, R.G., 2005. Selecting thresholds of occurrence in the prediction of species distributions. Ecography 28, 385–393. <https://doi.org/10.1111/j.0906-7590.2005.03957.x>

</div>

<div id="ref-malt2009marbled" class="csl-entry">

Malt, J.M., Lank, D.B., 2009. Marbled murrelet nest predation risk in managed forest landscapes: Dynamic fragmentation effects at multiple scales. Ecological Applications 19, 1274–1287. <https://doi.org/10.1890/08-0598.1>

</div>

<div id="ref-mather2010marbled" class="csl-entry">

Mather, M., Chatwin, T., Cragg, J., Sinclair, L., Bertram, D.F., 2010. [Marbled murrelet nesting habitat suitability model for the British Columbia coast](http://jem.forrex.org/index.php/jem/article/view/11/27). BC Journal of Ecosystems and Management 11, 91–102.

</div>

<div id="ref-murphy2026spatial" class="csl-entry">

Murphy, S., Leslie, A., Wilson, J., Banks, L.K., 2026. Spatial patterns of conifer regeneration following mountain pine beetle outbreak (*Dendroctonus ponderosae*) and mixed-severity wildfire: A point process model of disturbance interactions in the southern Selkirk Mountains, British Columbia. Forest Ecology and Management 618, 123985. <https://doi.org/10.1016/j.foreco.2026.123985>

</div>

<div id="ref-north2017cover" class="csl-entry">

North, M.P., Kane, J.T., Kane, V.R., Asner, G.P., Berigan, W., Churchill, D.J., Conway, S., Gutiérrez, R.J., Jeronimo, S., Keane, J., Koltunov, A., Mark, T., Moskal, M., Munton, T., Peery, Z., Ramirez, C., Sollmann, R., White, A., Whitmore, S., 2017. Cover of tall trees best predicts california spotted owl habitat. Forest Ecology and Management 405, 166–178. <https://doi.org/10.1016/j.foreco.2017.09.019>

</div>

<div id="ref-phillips2017opening" class="csl-entry">

Phillips, S.J., Anderson, R.P., Dudík, M., Schapire, R.E., Blair, M.E., 2017. Opening the black box: An open-source release of Maxent. Ecography 40, 887–893. <https://doi.org/10.1111/ecog.03049>

</div>

<div id="ref-rcore2024r" class="csl-entry">

R Core Team, 2024. [R: A language and environment for statistical computing. Version 4.4.1](https://www.R-project.org/). R Foundation for Statistical Computing, Vienna, Austria.

</div>

<div id="ref-ralph1995ecology" class="csl-entry">

Ralph, C.J., Hunt, G.L., Raphael, M.G., Piatt, J.F., 1995. Ecology and conservation of the Marbled Murrelet in North America: An overview, in: Ralph, C.J., Hunt, G.L., Raphael, M.G., Piatt, J.F. (Eds.), Ecology and Conservation of the Marbled Murrelet, General Technical Report. U.S. Department of Agriculture, Forest Service, Pacific Southwest Research Station, Albany, CA, pp. 3–22. <https://doi.org/10.2737/psw-gtr-152>

</div>

<div id="ref-raphael2018marbled" class="csl-entry">

Raphael, M.G., Falxa, G.A., Burger, A.E., 2018. Chapter 5: Marbled Murrelet, in: Spies, T.A., Stine, P.A., Gravenmier, R.A., Long, J.W., Reilly, M.J. (Eds.), Synthesis of Science to Inform Land Management Within the Northwest Forest Plan Area, General Technical Report. U.S. Department of Agriculture, Forest Service, Pacific Northwest Research Station, Portland, OR, pp. 301–370. <https://doi.org/10.2737/pnw-gtr-966>

</div>

<div id="ref-robin2011proc" class="csl-entry">

Robin, X., Turck, N., Hainard, A., Tiberti, N., Lisacek, F., Sanchez, J.-C., Müller, M., 2011. <span class="nocase">pROC</span>: An open-source package for R and S+ to analyze and compare ROC curves. BMC Bioinformatics 12. <https://doi.org/10.1186/1471-2105-12-77>

</div>

<div id="ref-roussel2020lidr" class="csl-entry">

Roussel, J.-R., Auty, D., Coops, N.C., Tompalski, P., Goodbody, T.R.H., Meador, A.S., Bourdon, J.-F., Boissieu, F. de, Achim, A., 2020. <span class="nocase">lidR</span>: An R package for analysis of airborne laser scanning (ALS) data. Remote Sensing of Environment 251, 112061. <https://doi.org/10.1016/j.rse.2020.112061>

</div>

<div id="ref-silvergieter2011patch" class="csl-entry">

Silvergieter, M.P., Lank, D.B., 2011a. Patch scale nest-site selection by marbled murrelets (*Brachyramphus marmoratus*). Avian Conservation and Ecology 6. <https://doi.org/10.5751/ace-00483-060206>

</div>

<div id="ref-silvergieter2011marbled" class="csl-entry">

Silvergieter, M.P., Lank, D.B., 2011b. Marbled murrelets select distinctive nest trees within old-growth forest patches. Avian Conservation and Ecology 6. <https://doi.org/10.5751/ace-00462-060203>

</div>

<div id="ref-sovern2019activity" class="csl-entry">

Sovern, S.G., Lesmeister, D.B., Dugger, K.M., Pruett, M.S., Davis, R.J., Jenkins, J.M., 2019. Activity center selection by northern spotted owls. The Journal of Wildlife Management 83, 714–727. <https://doi.org/10.1002/jwmg.21632>

</div>

<div id="ref-mcfarlanetranquilla2005breeding" class="csl-entry">

Tranquilla, L.M., Parker, N.R., Bradley, R.W., Lank, D.B., Krebs, E.A., Lougheed, L., Lougheed, C., 2005. Breeding chronology of marbled murrelets varies between coastal and inshore sites in southern british columbia. Journal of Field Ornithology 76, 357–367. <https://doi.org/10.1648/0273-8570-76.4.357>

</div>

<div id="ref-valavi2019blockcv" class="csl-entry">

Valavi, R., Elith, J., Lahoz-Monfort, J.J., Guillera-Arroita, G., 2018. <span class="nocase">blockCV</span>: An R package for generating spatially or environmentally separated folds for *k*-fold cross-validation of species distribution models. Methods in Ecology and Evolution 10, 225–232. <https://doi.org/10.1111/2041-210x.13107>

</div>

<div id="ref-valente2024presence" class="csl-entry">

Valente, J.J., Adrean, L.J., Nelson, S.K., Betts, M.G., Roby, D.D., Rivers, J.W., 2024. Presence-absence surveys yield spatially imprecise information about nesting sites of an endangered, forest-nesting seabird. PLOS ONE 19, e0315531. <https://doi.org/10.1371/journal.pone.0315531>

</div>

<div id="ref-valente2023fragmentation" class="csl-entry">

Valente, J.J., Rivers, J.W., Yang, Z., Nelson, S.K., Northrup, J.M., Roby, D.D., Meyer, C.B., Betts, M.G., 2023. Fragmentation effects on an endangered species across a gradient from the interior to edge of its range. Conservation Biology 37. <https://doi.org/10.1111/cobi.14091>

</div>

<div id="ref-vanrooyen2011relating" class="csl-entry">

van Rooyen, J.C., Malt, J.M., Lank, D.B., 2011. Relating microclimate to epiphyte availability: Edge effects on nesting habitat availability for the marbled murrelet. Northwest Science 85, 549–561. <https://doi.org/10.3955/046.085.0405>

</div>

<div id="ref-waterhouse2007assessing" class="csl-entry">

Waterhouse, F.L., Burger, A.E., Cober, A., Donaldson, A., Ott, P.K., 2007. [Assessing habitat quality of Marbled Murrelet nest sites on the Queen Charlotte Islands/Haida Gwaii, by algorithm, airphoto interpretation, and aerial survey methods](https://www.for.gov.bc.ca/rco/research/wildlifereports/tr035.pdf) (Technical Report No. TR-035). Research Section, Coast Forest Region, BC Ministry of Forests; Range, Nanaimo, BC.

</div>

<div id="ref-waterhouse2009using" class="csl-entry">

Waterhouse, F.L., Burger, A.E., Lank, D.B., Ott, P., Krebs, E., Parker, N., 2009. Using the low-level aerial survey method to identify marbled murrelet nesting habitat. Journal of Ecosystems and Management. <https://doi.org/10.22230/jem.2009v10n1a413>

</div>

<div id="ref-waterhouse2008using" class="csl-entry">

Waterhouse, F.L., Donaldson, A., Lank, D.B., Ott, P., Krebs, E., 2008. Using air photos to interpret quality of marbled murrelet nesting habitat in south coastal British Columbia. Journal of Ecosystems and Management. <https://doi.org/10.22230/jem.2008v9n1a381>

</div>

<div id="ref-wilk2016nesting" class="csl-entry">

Wilk, R.J., Raphael, M.G., Bloxton, T.D., 2016. Nesting habitat characteristics of marbled murrelets occurring in near-shore waters of the olympic peninsula, washington. Journal of Field Ornithology 87, 162–175. <https://doi.org/10.1111/jofo.12150>

</div>

<div id="ref-zharikov2007influence" class="csl-entry">

ZHARIKOV, Y., LANK, D.B., COOKE, F., 2007. Influence of landscape pattern on breeding distribution and success in a threatened alcid, the marbled murrelet: Model transferability and management implications. Journal of Applied Ecology 44, 748–759. <https://doi.org/10.1111/j.1365-2664.2007.01319.x>

</div>

</div>
