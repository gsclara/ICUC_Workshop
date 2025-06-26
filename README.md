#### ICUC_Workshop
This repository includes all the necessary materials for our Workshop on 11th of July 2025 within the ICUC conference. 
Througout this workshop we will reconstruct Rotterdam using different level of detail around the area of the conference and far away. 

___

#### Python environment
First we will create a python environment to avoid any duplication issues with python libraries. 
This implies two steps: 

- mkdir -p reconstructions/Rotterdam
- cd reconstructions
- mkdir python-env
- cd python-env 
- cd reconstructions/Rotterdam
- source ../python-env/bin/activate
- pip install numpy, osmnx, geopy

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
Rotterdam, NL, -4,570313, 29,228890, WGS:84
Barcelona, ESP, 41.39563705715212, 2.1619087803576256, EPSG:25831
Denver, USA, 39.7392364, -104.984862, EPSG:4269
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

3. Dealing with the ocean polygon
The ocean "polygon" is downloaded as a line segment instead of a polygon when using OSM. This requires a fair bit of manual edits before it can be directly used in City4CFD. Consider the ocean line segment for the city of `Barcelona, ESP` downloaded using the `fetch_osm.py` script shown below. 

<img width="1512" alt="Coastal Line Segment for adjacent to the city of Barcelona, Spain" src="https://github.com/user-attachments/assets/20c4d88f-a11b-43ab-b1d4-2f4a2cb58705" />

Figure 2: Ocean line segment for the city of Barcelona, Spain.

As seen in the figure above, the solid-black line marks the boundary of the ocean polygon. Since there is no adjacent land on the right side of this segment OSM only downloads the line segment corresponding to the "coastline" of the city. 

Step 3.a: Open the line segment in QGIS $^7$ as shown in the figure 2.
Step 3.b: Toggle the edit mode by clicking the "Yellow pencil" icon and choose the "Digitise with segment option and create additional line segments such that it forms a closed polygon as shown in figure 3. 

<img width="1512" alt="Barcelona_Ocean_2" src="https://github.com/user-attachments/assets/5faf0d69-0f85-47cc-98fa-be7062e891b6" />

Figure 3: Editing the Ocean line segment using QGIS.

Step 3.c: Once you have generated the closed polygon, export it as a new polygon in the same EPSG as the final output.

Step 3.d: Use the `City4CFD/tools/polyprep/polyprep.py` script to generate a polygon using the line segment data obtained in Step 3.c.

The output from `polyprep.py` can be used to supply the Ocean polygon.

4. Handling large water polygons: For certain cases it is important to split large water and vegetation polygons into multiple smaller polygons using QGIS so that they are imprinted within the region of interest. Typically City4CFD handles this quite well, but if the problem persists then it is advised to split the polygon into smaller polygons such that they fit within the region of interest.

- **Fetch the Point cloud Data**

Downloading the point cloud data does not have an automated method similar to the polygon data detailed in the previous section. This is mainly a consequence of LiDAR generation and maintenance methods which are heterogenous depending on the local municipality/agency hosting the data. Consequently, we defer the point cloud acquisition to the local methods/sources for sake of brevity.

*Classified Point Clouds*

Step 4.a: For a classified point cloud, City4CFD requires the following point classes: *Vegetation, Water, Ground, and Buildings.* Depending on the point cloud classification ID's the class ID corresponding to these might change but for most cases these are the classifications that can be obtained by using a combination of the following commands using `lastools`$^8$.

```{.python .numberLines}
2 9 - Terrain
6 - Buildings
```

Extract classes 2 6 and 9 into a single point cloud.
```
lasmerge64 -i *.laz -keep_class 2 6 9 -o out.laz
```

Extract terrain point cloud after thinning it by keeping every 10th point in the point cloud
```
las2las64 -i out.laz -keep_every_nth 10 -keep_class 2 9 -o terrain.laz
```

Extract building point cloud after thinning it by keeping every 2nd point in the point cloud
```
las2las -i out.laz -keep_every_nth 2 -keep_class 6 -o buildings.laz
```

*Unclassified Point Clouds*

For unclassified point clouds, there is no easy way to separate the buildings from the terrain using lastools as the points are not classified. Hence, we need to make use of the Cloth Simulation Filter (CSF)$^9$. This can be done either via a python script or Cloud Compare.

Steps below to use Cloud Compare and apply the CSF filter.

Step 4.a: Load the point cloud in Cloud Compare
Step 4.b: Click on `Plugins/CSF Filter` - Choose the terrain follow option. Cloud compare will generate the two points clouds corresponding to ground and buildings. 
Step 4.c: Export the individual point clouds to files titled `terrain.laz` and `buildings.laz`, respectively.

NOTE on usage: When using the CSF filter there is a high chance that a lot of noise is contained within the building point cloud. Consequently, use a relatively lower percentile when reconstructing the building geometry using City4CFD.

___
Now that all data is available for City4CFD, you can follow the regular tutorial to successfully run the reconstruction process.
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


