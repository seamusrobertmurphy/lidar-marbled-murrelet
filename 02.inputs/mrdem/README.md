# MRDEM elevation

Medium Resolution Digital Elevation Model (MRDEM), the 30 m bare-earth terrain model of the CanElevation Series from Natural Resources Canada, read from the cloud-optimized national file https://canelevation-dem.s3.ca-central-1.amazonaws.com/mrdem-30/mrdem-30-dtm.tif (EPSG:3979, last modified 2026-06-24, 83.9 GB). Licence Open Government Licence – Canada, https://open.canada.ca/en/open-government-licence-canada. Retrieved 2026-11-22, with Seamus's approval that day.

The `dem` chunk reads only the blocks under the Figure 1 frame, reprojects them to EPSG:3157 at 30 m and saves `dtm_cs.tif`, which is not committed. It gives the shaded relief of Figure 1 and enters no analysis.
