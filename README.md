# Correlating toxic release inventory data to North Carolina's known impaired waterways using QGIS and Mapbox Studio

## Data Sources

* Toxic release inventory data (2021) was gathered from the Environmental Protection Agency's website at https://www.epa.gov/toxics-release-inventory-tri-program/tri-basic-data-files-calendar-years-1987-present

* Waterway data (2022) was gathered from the Environmental Protection Agency's website at https://www.epa.gov/waterdata/waters-geospatial-data-downloads#ATTAINSWaterQualityAssessmentGISDataset

* With these links, an individual can easily gather data specific to any other state in order to replicate these findings for said state.

## Tools

QGIS 3.22

Mapbox Studio

Mapbox Tiling Service API's

## Why this project?

I created this map to explore processing tools in QGIS, explore EPA datasets, and gain more familiarity with Mapbox's datasets, tilesets, and style tools.

After looking over the data I found I could use it to determine which toxic release sites in North Carolina were within a certain distance of polluted waterways. I further found I could determine which of these sites were the 'worst' offenders (ie, by volume of pollutant released). In this manner one could identify sites of toxic release which **could** give the greatest return (decrease in releases, improvement of waterway health metrics) on investment (mitigation efforts).

Some tasks accomplished in this project through the use of QGIS tools include:

   * Refactoring tables to better build queries and symbolize graduated data
   * Tooling vector files to find overlaps and correlations in the two datasets
   * Simplifying vector data to reduce processing time
   * Filtering shapefiles by building queries

Some tasks accomplished in this project through the use of CLI tools include:
   * Making curl requests to [Mapbox's API's](https://docs.mapbox.com/mapbox-tiling-service/guides/) to upload, download, and update data
   * Working with [Mapbox's tilesets CLI](https://github.com/mapbox/tilesets-cli/) to upload, download, and update data
   * Working with [ndjson](https://www.npmjs.com/package/ndjson-cli) and [geojson2ndjson](https://www.npmjs.com/package/geojson2ndjson) CLI tools to convert data.

## How I made this mapping project

### Questions Asked

Where are the known impaired waterways?
    Where are the known impaired waterways that have been assessed and are reported to contain toxins?

Where are the known sites of toxic release?
    Specifically release into bodies of water?

Where is there an overlap between impaired waterways and toxic release sites?

### Examine and Filter Data

After reviewing the waterways data, I imported *only* the line vector data (with a whopping 356,176 records). On review of the attribute table, one will see there is a column for 'impaired' waterways, as well as 'threatened' waterways. Luckily North Carolina has no 'threatened' waterways. If you decide to replicate this project with another state you may want to include your filter query to include both 'impaired' as well as 'threatened' waterways. I simply filtered WHERE 'impaired' = 'Y'. This brought the record number from 3,875 entries for all of North Carolina to 1,397.

Upon further examination of the attribute table, I built out the filter to include only those waterways which contain toxins. This further reduced the records to a more manageable 84.

I did this because toxins are **more likely** than the other causes of impairment (sediment, oxygen depletion, turbidity, etc) to be appropriately attributed to our toxic release inventory data. *This is an assumption and a point of potential weakness in my subsequent findings which should not be overlooked.*

After finishing up my filtration of the waterway data, I'm left with a layer which shows us the North Carolina waterways which are specifically impaired by toxins. I finish this step by saving the temporary filterd layer as a geojson.

Moving onto the toxic release data itself, I begin by adding the North Carolina-specific CSV file as a delimited text layer in QGIS, matching the point coordinates to the latitude and longitude columns provided in the CSV file. I then filter the records for waterborne releases only.

### Assess the Data

Now I have both the waterway data and the toxic release data on the same canvas. How do I determine correlations? By examining distance. I want to know which TRI sites are *close* to impaired waterways. What is close? I arbitrarily chose 1 kilometer. This is entirely subjective and ignores location specific realities that may make a correlation very unlikely. *This is another point of potential weakness in my subsequent findings which should not be overlooked.* 

I used the 'select within distance' vector selection processing tool in QGIS to select features from the TRI data layer by comparing to the filtered waterways layer where the TRI points are within 1000 meters of the waterways. I then saved the subsequently created layer as a geojson.

Now we have a selection of the TRI sites which are within 1 km of an impaired waterway which has been assessed positively for containing toxins. I then save this selection as its own geojson layer.

### Visualizing the data

What I wanted to show in my final visualization is a map of all of North Carolina that met the following specifications:
   * Shows all *assessed* waterways
   * Highlights impaired waterways
   * Shows all TRI sites in the 1 km buffer, symbolized in a graduated way to show the relative quantity of toxic releases at the site

I also want to get the top ten release sites to show in a table with more details (quantity of release, name of released substance, company reporting) to accompany the map. 

#### Working around Mapbox's upload limitations

In order to upload the assessed waterways layer to Mapbox, I had to simplify the data because the file was too large for the Mapbox upload API. I accomplished this using the simplify vector geometry tool in QGIS, afterwards exporting the resulting layer as a geojson.

#### Attempts to work around Mapbox's zoom extent limitations

Unfortunately upon loading this file to Mapbox as a tileset and attempting to style it, I saw that the zoom extent was not the correct zoom extent I needed for my map.

![](/images/incorrect-zoom-extent.png "")

##### Unsuccessful attempt using QGIS
So, I went back to QGIS and used the 'write vector tiles (MBtiles)' processing tool to create vector tiles to upload to Mapbox. I was able to specify my required zoom extents in the tool parameters before running it. The process seemed to complete correctly. Once I uploaded the tileset to Mapbox studio however, the tileset was not displayed on the map at the zoom necessary for my project (5-9) even though Mapbox reported that my zoom extent was within my needs.

![](/images/correct-zoom-extent.png "")

I attempted this repeatedly with the same result.

Afterwards I followed Mapbox's suggestion to use their Mapbox Tiling Service. 

##### Unsuccessful attempt using Mapbox Tiling Service (MTS) API
I was unsuccessful in using the [Mapbox Tilesets CLI tool](https://github.com/mapbox/tilesets-cli/) in tandem with the [MTS API](https://docs.mapbox.com/mapbox-tiling-service/guides/) to alter the zoom extent. Here are the steps I took so do so.

* Create mapbox token with tilesets.read, tilesets.write, and tilesets.list scopes under Mapbox account settings
* Install tileset CLI in terminal
   * For myself (using Windows - unsupported by Mapbox) this required installing Rasterio as a dependency.
* Upload-source file (geojson) via Mapbox tilesets CLI
* create tileset via Mapbox tilesets CLI
* write recipe via Mapbox tilesets CLI
* validate recipe via Mapbox tilesets CLI
* Update tileset recipe via Mapbox tilesets CLI
* Publish tileset to start job process (render data into vector tiles) via MTS API
   After several attempts, I was unable to overcome the following error from Mapbox
    No GeoJSON features processed. Make sure your GeoJSON is line-delimited.
   
   Upon further research I came across the following.
    MTS requires that you format tileset sources as line-delimited GeoJSON. If you upload your source as GeoJSON using the tilesets CLI, your source will be converted to the correct format.
   
   However, I **did** upload my source as a geojson using the tilesets CLI (and in fact re-uploaded via the same means to doublecheck myself and encountered the same error on attempting to publish), so I'm stumped as to why this error occurred. 

![Difference between geojson and newline delimted json](/images/geojson_formatting.png "Difference between geojson and newline delimted json")

Admittedly MTS is in beta. As another potential workaround I decided to attempt to upload my data as a dataset via Mapbox's Dataset API.

##### Unsuccessful attempt using Mapbox Dataset API

I'd initially attempted to upload my geojson of the river data as a dataset on Mapbox studio but was met with the message that uploads were limited to 5 MB. I turned to the MTS API + tileset CLI as suggested by Mapbox. After that unsuccessful attempt, I turned to the Dataset API as recommended by Mapbox as an alternative for larger file uploads. Here are the steps that required.

* Update Mapbox token with datasets.write scope under Mapbox account settings
* Use Mapbox Dataset API to upload geojson.
    curl -X POST "https://api.mapbox.com/datasets/v1/${MAPBOX_USERNAME}?access_token=${MAPBOX_ACCESS_TOKEN}" \
  -d @my_file.geojson \
  --header "Content-Type:application/json"

   The response I received was an error:
    {"message":"request entity too large"}

   Upon troubleshooting this issue, I came across this same error issue on a fresh [Stackoverflow thread](https://stackoverflow.com/questions/74240677/how-do-i-upload-a-large-geojson-file-to-a-mapbox-dataset), that as of this writing has still been unanswered.

##### An unsuccessful attempt to use Tippecanoe

After three failed attempts to work around Mapbox's zoom extent limitations, I began the process of trying Mapbox's recommendation to use Tippecanoe. I quickly learned that Windows is not supported by the project. However I did find a [workaround](https://github.com/GISupportICRC/ArcGIS2Mapbox/#installing-tippecanoe-on-windows) for installing Windows. Unfortunately, I have run out of time to delve into workarounds within workarounds, and this particular path seems like it could compromise my working environment, so I will need to save this workaround for the future.

#### Next steps after failing to overcome Mapbox's zoom extent limitations
I really wanted to add the waterways layer to this project and highlight the impaired waterways. Unfortunately, I'm simply not going to be able to do that with Mapbox. And without the waterways layer on Mapbox, my map is going to lack crucial visual insights. So, I turned my attention to creating my visualization solely in QGIS. I am disappointed by this, as Mapbox is a really cool way to have dynamic visualization. Unfortunately my map will be static until I can overcome the zoom extent issues.

#### Visualization with QGIS
My visualization with QGIS required the following steps
   * Style base layer (states layer)
   * Style assessed waterways layer
   * Highlight impaired waterways
   * Symbolize TRI sites within 1km buffer
    * Use color to indicate gradation (volume released)
   * Export map to print layout
   * Pull top ten entries from the attribute table
    * Create chart to accompany map in print layout
   * Export print layout




sk.eyJ1IjoicGhpbGxpcC1hc2hmb3JkIiwiYSI6ImNsYnFudjd4YTE0dzczcHFoaHIxa3hlaTUifQ.5z75eq0bzbFLnJY9Anv_4Q