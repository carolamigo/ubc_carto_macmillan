# [MacMillan Bloedel Limited fonds](https://open.library.ubc.ca/collections/macmillan)

Linked data web application providing a georeferenced visualization of UBC Open Collection MacMillan Bloedel Limited fonds metadata, linked to [Geonames](http://www.geonames.org/ontology/documentation.html).

[Link for the application on CARTO.](https://carolamigo.carto.com/builder/881d9644-0439-4131-ad92-c16bcb1e2608/embed)

![mcmillan_v02.gif](https://github.com/carolamigo/ubc_carto_macmillan/blob/master/mcmillan_v02.gif)

### How to do it

#### Modelling

- Download collection metadata using the Open Collections Research API. A php script to batch download is provided at OC API Documentation page > Download Collection Data. This script returns a folder containing one RDF file per collection item (or XML, JSON, any format preferred). We are going to use N-triples because the file is cleaner (no headers or footers), what makes the merging easier later. Edit the script following the instructions on the documentation page and run it using the command:

`$ php collection_downloader.php --cid macmillan --fmt ntriples`

- Merge the files using the Unix cat command:

`$ cat * > merged_filename`

- Convert merged file obtained to a tabular format. Import project in Open Refine using the RDF/N3 files option. No character encoding selection is needed.

#### Cleaning

- Examine the metadata for Geographic Locations. With the tabular data open in OpenRefine, look for the column “http://purl.org/dc/terms/spatial”. In the column options, select “Facet” > “Text facet”. The facets show you all the unique values for geographic locations in this dataset. From that list, it is possible to see that:

  - The location names are following Library of Congress formatting style, with the province following the name of the city, and that they are in between double quotes with the language notation following: 

    `e.g. “Alberni (B.C.)”@en`

  - Some location names have small typos: 

     `"Namaimo River (B.C.)”@en`

  - Some resources have more than one geographic location associated with it: 

    `e.g. "Powell River (B.C.) ; Nanaimo (B.C)"@en`

- Split the cells containing more than one geographic location value. 

  - Duplicate the “http://purl.org/dc/terms/spatial” column using “Edit column” > “Add column based on this column” in order to preserve original values. Name the new column: spatial_cleaned
  - On the spatial_cleaned column, select “Edit cells” > “Split multi-valued cells”. The separator is “;”.

- Remove double quotes, provinces and “@en” from location names. Select “Edit cells” > “Transform” and write the following expression:

`value.replace("\"", " ").replace("@en"," ").replace(/\(([^\)]+)\)/," ")`

- Trim leading and trailing whitespaces by selecting “Edit cells” > “Common transforms” > “Trim leading and trailing whitespaces”.

- Cluster location names in order to combine entries with typos and small inconsistencies under just one geographic location name. On the spatial_cleaned column, select “Facet” > “Text facet”, then select “Cluster” in the facet window. In the cluster window, select Nearest neighbour” method. Select the “merge” box for “Nanaimo River”, correct the typo, and select “Merge selected and close”.

- Fill down column “subject”, “http://purl.org/dc/terms/title”, “http://purl.org/dc/terms/created” and “http://www.europeana.eu/schemas/edm/isShownAt” as we have several orphan cells resulting from the triple to tabular data format conversion. Go to each column, “Edit cells” > “Fill down”.

#### Reconciling

- Configure Geonames reconciliation service in OpenRefine following the procedure described here: https://github.com/cmh2166/geonames-reconcile. This procedure involves getting a Geonames API username, installing python packages, cloning the code in the GitHub repository above, and running the script provided in the code. 

- Perform reconciliation on the “spatial_cleaned” columns using “geonames/name_equals” option. Follow the steps described here: http://christinaharlow.com/walkthrough-of-geonames-recon-service.

- When reconciliation is finished, review the results. An easy way to do it is to facet by text (“Facet” > “Text facet”) and filter results by clicking on a location name on the facet menu. On the spatial_cleaned column, click on the link for the location to check the reconciled value. This will open the Geonames window with location information. If it is correct, no further action is required. If it is wrong (e.g. Alice Lake found is in B.C. but geonames returned a lake with the same name in Michigan), click on “Choose a new match” under any of the wrong entries on the spatial_cleaned column. Three options will show up. Select the correct one by using the double checked box, which means your decision will be applied to all other cells that match this condition. If no correct option show up in the cell, click on the double checked box of “Create new topic”, meaning that no reconciliation value will be added to cells that match this condition.    There are 66 unique values in this dataset, so it is possible to review one by one until it is done.

- Verify reconciliation results that didn’t find a match (including the ones you had to “create a new topic” for) by selecting the “none” facet in the judgment box. I have found the following ones with no matches, so I had to add coordinates manually for those by looking up manually in the Geonames database using the Geonames search box (http://www.geonames.org/). To mass edit a value, click on the edit link that appear next to the exclude link for the value in the facet window. Enter the value and click “Apply”.

- Extract reconciled data retrieving name, id and latitude/longitude, as a string separated by “|”. Select spatial_cleaned, “Edit cells” > “Transform”, and entering the following expression:

`cell.recon.match.name + " | " + cell.recon.match.id`

- Split the values obtained in the reconciliation, that should be in this format:

`Nanaimo River | 49.1304, -123.89385 | http://sws.geonames.org/6951400`

- Select spatial_cleaned, “Edit column” > “Split into several columns”. Select the “|” separator and name the columns according after split: geonames_names, geonames_coord, geonames_uri.

- The “geonames_coord” column has to be further split in latitudes and longitudes using the same command above, “Edit column” > “Split into several columns”, with separator “,”. Name the columns “lat” and “long”. Trim leading and trailing whitespaces by selecting “Edit cells” > “Common transforms” > “Trim leading and trailing whitespaces”.

#### Building interface

- Prepare the data to interface. In order to have links and images on CARTO interface, we have to add html tags in the source dataset. 

  - Remove double quotes and language. Create a new column “title” based on the column “http://purl.org/dc/terms/title”, using the following expression:
    
    `value.replace("\"", " ").replace("@en"," ")`

  - Add html tags for title links. Create a new column “title_link” based on the column “subject”, using the following expression:

    `"<a href=\""+value+"\">"+if(isBlank(cells["title"].value), " ", cells["title"].value)+"<\/a>"`

  - Remove double quotes and language. Create a new column “date” based on the column “http://purl.org/dc/terms/created”, using the following expression:
    
    `value.replace("\"", " ").replace("@en"," ")`

  - Add html tags for location links. Create a new column “geoname_link” based on the column “geonames_uri”, using the following expression:

    `"<a href=\""+value+"\">"+if(isBlank(cells["geonames_names"].value), " ", cells["geonames_names"].value)+"<\/a>"`

  - Add html tags and links for images. Create a new column “image_link” based on the column “http://www.europeana.eu/schemas/edm/isShownAt”, using the following expression:

    `"<img width=\"188\" src=\"http://iiif.library.ubc.ca/image/cdm.macmillan."+value.substring(10,19).replace(".","-")+".0000" + "/full/150,/0/default.jpg\"/>"`

- Export the dataset from OpenRefine in .csv format. Name the file “mcmillan_cleaned”.

- Sign up or Log in to CARTO Builder: https://carto.com/signup/. Create a new map and import the Open Refine exported file to your map.

- Georeference your dataset following the instructions here: https://carto.com/learn/guides/analysis/georeference. Once in your map, click on the dataset on the left menu, then click on “Analysys” > “Add analysys” > Georeference. Select the corresponding column names in your dataset for latitude and longitude (lat and long). Note that the application is plotting just one resource per location, so we will need to aggregate the results to have all the resources plotted. 

- Export the georeferenced dataset from CARTO in csv format, in order to incorporate the “the_geom_webmercator” column (with georeferenced data) in your dataset. Name the file “mcmillan_cleaned_geo”. Import the dataset back into CARTO map, and delete the previous dataset from your map. This step is necessary since CARTO does not allow georeference analysis and SQL manipulation (that we will need for aggregation) of the data concomitantly. 

- Click on the dataset on the lateral menu and select the “Data” tab. At the bottom of the lateral panel, enable SQL option. Paste the following query in the editor and click “Apply”:

`SELECT string_agg(DISTINCT CONCAT (date, ' <br>', title_link, ' <br><br>', image_link, ' '),' <br><br><br><br>') as new_column_aggregated, geoname_link, the_geom_webmercator,
Min(cartodb_id) cartodb_id
FROM mcmillan_cleaned_geo
group by geoname_link, the_geom_webmercator`

- Click on the “Pop-up” tab, and enter the following settings:

  - Style: color
  - Window size: 240
  - Header color: #b6dc9
  - Show item: check all boxes

- To build the filters:

  - Click on “mcmillan_cleaned_geo” dataset, “style” tab, and use the following settings:

    - Aggregation: none
    - Size 10, color #d1c4c4
    - Stroke 1 color #FFFFFF
    - Blending: darken

  - Exit “mcmillan_cleaned_geo” dataset and click on the “add” button to re import the same dataset. You will get copy named “mcmillan_cleaned_geo_1”. Click on this new dataset, “style” tab, and use the following settings:

    - Aggregation: none
    - Size 15, color #ee4d5a
    - Stroke 1 color #FFFFFF
    - Blending: none

  - Exit the dataset and make it the second one in the list showing on the lateral panel, dragging and dropping it. We want this new layer behind the one that has the pop-up with photos. 

  - Click on “Widget” tab to add filters. Add widgets following the instructions here: https://carto.com/learn/guides/widgets/exploring-widgets.

- Publish your map using the “Publish” button on the left lateral panel, just under the map name.
