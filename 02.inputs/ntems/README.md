# NTEMS harvest

Canada Forest Harvest 1985-2022, the year of greatest harvest disturbance on a 30 m grid, from the National Terrestrial Ecosystem Monitoring System, https://opendata.nfis.org/mapserver/nfis-change_eng.html, downloaded from https://opendata.nfis.org/downloads/forest_change/CA_Forest_Harvest_1985-2022.zip. The page asks that the data be cited as Hermosilla et al. (2016), https://doi.org/10.1080/17538947.2016.1187673. No licence is stated on the retrieval page.

The `ntems` chunk downloads the national file, crops it to the study area and saves `harvest_cs.tif`. The zip and the national raster are not committed.
