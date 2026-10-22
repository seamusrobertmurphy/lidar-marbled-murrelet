# LidarBC scans

Airborne laser scanning point clouds of the 2019 NDMP Vancouver Island acquisition (contract OP20BMC013), from LidarBC, the Province of British Columbia's open lidar portal, https://lidar.gov.bc.ca/. Licence Open Government Licence British Columbia, per the BC Data Catalogue record https://catalogue.data.gov.bc.ca/dataset/lidar.

The tile index is read by the `tiles` chunk from the feature service https://services6.arcgis.com/ubm4tcTYICKBpist/arcgis/rest/services/LiDAR_BC_S3_Public/FeatureServer/4 and saved as `tiles.gpkg`. Tiles are downloaded by the `laz` chunk into `laz/`, which is not committed because the files run to about 1 GB each.
