#### ICUC_Workshop
This repository includes all the necessary materials for our Workshop on 11th of July 2025 within the ICUC conference. 
Througout this workshop we will reconstruct Rotterdam using different level of detail around the area of the conference and far away. 

___

#### Python environment
First we will create a python environment to avoid any duplication issues with your own python libraries. 
This implies the following steps: 

- mkdir -p reconstructions/Rotterdam
- cd reconstructions
- mkdir python-env
- cd python-env 
- cd reconstructions/Rotterdam
- source ../python-env/bin/activate
- pip install numpy, osmnx, geopy

___

#### Input data

1. Semantic surfaces and building footprints 
2. LiDAR point clouds. Ideally point cloud density of 5 points per building allows reconstruction for LoD 1.2, while higher level of detail requires higher point cloud density. 

___

#### STEPS TO RECONSTRUCT URBAN BUILT ENVIRONMENT

- **Fetch the semantic surfaces and building footprints**

To simplify the urban reconstruction, we rely on Open-Street Map (OSM) $^2$ data that contains a large portion of the input data that is routinely updated $\left[ \sim \mathcal{O}(weeks) \right]^3$. As a result, we assume the building footprint data and the semantic surfaces can be reliably procured from this data source when no local data-set is available.

In order to fetch the buildings polygons and semantic surfaces automatically, use the `City4CFD/tools/fetch_polygons/fetch_osm.py` python script. This script is also included in the current repository. 

This python script requires the following input data

1. A text file named `cities.txt` in the same directory where `fetch_osm.py` is placed.
```{.python .numberLines}
#<city-name>, <country-code>, <center-lat>, <center-lon>, <output-EPSG-code>
Rotterdam, NL, 51.9225, 4.47917, EPSG:28992
```
Each line in the `cities.txt` file uses the following either the two letter code or the three letter coding defined in **ISO 3166-1 alpha-2**$^4$ and **ISO 3166-1 alpha-3**$^5$, respectively. The `<center-lat>` and `<center-lon>` correspond to the center of the region of interest (assuming a circular region of interest) in `EPSG:4326`. The column for input per city is the EPSG in which the final output is exported. **It is vital to have the `<output-EPSG-code>` in the same Coordinate Reference System (CRS) as the point cloud.**

To figure out the latitude, longitude and coordinate system (`https://epsg.io/`)

2. Within the `fetch_osm.py` script, the user input parameters that will be applied per city listed in the `cities.txt` file.
                 
```{.python .numberLines}
Hmax = 230                            
rbuildings = 1200                    
rpolygons = 4000                      
outdir = "data" 
```

In the listing above, the first line is used to define the region of interest based on best practice guidelines $^6$.  Line number 2 fetches the building footprint polygons within a radius specified by the value. Line number 3 does the same for semantic surfaces defined within the script as line 2. In line 3, the semantic surfaces are defined as detailed below. For most cases the below defined OSM-tags should correctly capture the various surfaces, however, in certain cases, it might be important to either remove or append additional categories to correctly include the necessary surface features. **Line number 3 is activated only when $H_{max} < 0$.** Line 4 redirects the output polygons within the output directory.


```{.python .numberLines}
tags = {
    "buildings": {
         "building": True,
         "height": True,
         "building:levels": True
     },
    "vegetation": {
        "landuse": ["forest", "grass", "meadow", "orchard"],
        "leisure": ["park", "nature_reserve", "garden", "dog_park"],
        "natural": ["grassland", "wood"]
    },
    "water": {
        "natural": ["water", "sea"],
        "waterway": True,
    },
    "ocean": {
        "natural": ["coastline"]
    }
}
```

3. Handling large water polygons: For certain cases it is important to split large water and vegetation polygons into multiple smaller polygons using QGIS so that they are imprinted within the region of interest. Typically City4CFD handles this quite well, but if the problem persists then it is advised to split the polygon into smaller polygons such that they fit within the region of interest.

Now we need to run the fetch.py script:

`python fetch.py > log.fetch`

Saving a log will allow us to know where the center of our domain is in X,Y coordinates. 

- **Fetch the Point cloud Data**

Downloading the point cloud data does not have an automated method similar to the polygon data detailed in the previous section. This is mainly a consequence of LiDAR generation and maintenance methods which are heterogenous depending on the local municipality/agency hosting the data. 

For Rotterdam we can use [geotiles.nl](https://geotiles.citg.tudelft.nl/), there we can download tiles: 

XXXX image with tiles

*Classified Point Clouds*

Step 4.a: For a classified point cloud, City4CFD requires the following point classes: *Vegetation, Water, Ground, and Buildings.* Depending on the point cloud classification ID's the class ID corresponding to these might change but for most cases these are the classifications that can be obtained by using a combination of the following commands using `lastools`$^8$.

```{.python .numberLines}
2 9 - Terrain
6 - Buildings
```

Extract classes 2 6 and 9 into a single point cloud.
```
lasmerge64 -i *.LAZ -keep_class 2 6 9 -o out.LAZ
```

Extract terrain point cloud after thinning it by keeping every 10th point in the point cloud
```
las2las64 -i out.LAZ -keep_every_nth 10 -keep_class 2 9 -o terrain.LAZ
```

Extract building point cloud after thinning it by keeping every 2nd point in the point cloud
```
las2las64 -i out.LAZ -keep_every_nth 2 -keep_class 6 -o buildings.LAZ
```
*NOTE on Unclassified Point Clouds*

While most point cloud datasets offered by local authorities are typically classified (e.g., The Netherlands, Germany, Japan [Tokyo]), this may not always be the case thus making the above detailed steps relatively challenging to split the data into terrain and buildings. However, there is a relatively simple solution for situations like this using the Cloth Simulation Filter (CSF) that automatically splits the point cloud data into a relatively smooth terrain and non-terrain (may include Buildings, Trees, and other objects). Such a point cloud can be processed using the `city4cfd_pcprep` tool or `CloudCompare`[https://www.cloudcompare.org/] to split the point cloud data into terrain and building datasets. It is, however, important to note that in doing so, the building reconstruction may contain some spurious artefacts or the other way around, where buildings with large surfaces can be detected as a ground and produce artefacts in the terrain that can typically be resolved using a relatively lower building point cloud percentile in the `config.json` file.

- **Running city4CFD**

To run city4CFD we will first create a folder for the results and then run the code pointing to that location. 

`mkdir results`

` ~/city4CFDintallationFolder/build/city4cfd rotterdam.json --output_dir results_LoD2`

Then we can visualize the results with paraVIEW. 

___

**References**

1. Biljecki, F., Zhao, J., Stoter, J., & Ledoux, H. (2013). Revisiting the concept of level of detail in 3D city modelling. ISPRS Annals of The Photogrammetry, Remote Sensing and Spatial Information Sciences, 2, 63-74.
2. Main Page. (2024, September 6). OpenStreetMap Wiki, Retrieved 09:03, April 23, 2025 from https://wiki.openstreetmap.org/w/index.php?title=Main_Page&oldid=2752444. 
3. Planet Homepage. (2025, April 11). OpenStreetMap Wiki, Retrieved 09:04, April 23, 2025 from https://wiki.openstreetmap.org/wiki/Planet.osm 
4. ISO 3166-1 alpha-2. (22 April 2025). Wikipedia, Retrieved 10:00, April 23, 2025 from a https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2
5. ISO 3166-1 alpha-3. (22 April 2025). Wikipedia, Retrieved 10:00, April 23, 2025 from https://en.wikipedia.org/wiki/ISO_3166-1_alpha-3
6. J. Franke, A. Hellsten, K. H. Schlunzen, B. Carissimo, (2011). The cost 732 best practice guideline for cfd simulation of flows in the urban environment: A summary, International Journal of Environment and Pollution 44  419–427.
7. QGIS Development Team, (2025). QGIS Geographic Information System. QGIS Association. https://www.qgis.org
8. https://lastools.github.io/
9. Zhang W, Qi J, Wan P, Wang H, Xie D, Wang X, Yan G. (2016), An Easy-to-Use Airborne LiDAR Data Filtering Method Based on Cloth Simulation. Remote Sensing; 8(6):501.


