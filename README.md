# Hiking Trails WebMap
An interactive web-based map that shows hiking routes in California, and displays basic information about these routes to users.

## Data Extraction:
The data has been extracted from OpenStreetMap. The following method have been used; 
- Quick OSM: A QGIS plugin to query OSM live database servers using overpass API. Write the queries by providing a key/value and choose to run the query on an area or an extent
  - For hiking trails: 
    ```
    key = route;
    value = hiking, foot;  
    layer extent = California (By providing California boundary layer)
    ```
  - For Greenery Factor: 
    ```
    key = natural; value = grassland, wood, heath, scrub, fell, water
    key = landuse; value = forest, greenfield, grass, meadow, orchard, recreational_ground, village_green, vineyard, cemetery
    key = tourism; value = camp_site
    key = amenity; value = grave_yard
    key = leisure; value = garden, golf_course, nature_reserve, park, pitch, dog_park, common
    layer extent = California (By providing California boundary layer)
    ```
   Remove extra attributes except the ones that might be useful e.g. osm_id, name, natural, landuse, tourism, amenity, leisure, geometry

## Data Processing:
- Merge all polygon layers into single layer, uning Merge Vector Layers tool in QGIS i.e. all_layers
- Select all polygons in 500 meters buffer of hike trail, using Select by Location tool in QGIS
- Import selected polygons to a postgres database using the following command
  ```
  ogr2ogr -f "PostgreSQL" "PG:host=127.0.0.1 user=postgres dbname=postgres password=postgres" "E:\Data\all_layers.shp" all_layers_500m -overwrite
  ```
- Dissolve overlapping polygons in all_layers_500m that we created in previous step, using the following SQL query:
  ```
  CREATE TABLE greenareas AS
  SELECT (ST_DUMP(ST_UNION(ST_SNAPTOGRID(geom,0.0000)))).geom as geom
  FROM all_layers_500m;
  ```
- Find intersections of hiking trails with polygons;
  ```
  SELECT h.id,ST_UNION(ST_Intersection(h.geom,p.geom)) as len_intersect
  FROM hiking_trails h INNER JOIN greenareas p
  ON ST_Intersects(h.geom,p.geom)
  GROUP BY h.id
  ```
- Calculate total and intersecting length of trails in miles;
  ```
  ALTER TABLE hiking_trails
  ADD COLUMN len_intersect float,
  ADD COLUMN len_total float,
  ADD COLUMN green_factor float;

  UPDATE hiking_trails
  SET len_total = (ST_Length(ST_Transform(geom, 2877))/5280);

  UPDATE hiking_trails
  SET len_intersect = s.len_intersect
  FROM(
  select h.id,SUM(ST_Length(ST_Transform(st_intersection(h.geom,p.geom), 2877))/5280) as len_intersect
  FROM hiking_trails h inner join greenareas p
  ON ST_Intersects(h.geom,p.geom)
  GROUP BY h.id) AS s
  WHERE hiking_trails.id = s.id;

  update hiking_trails
  SET len_intersect = 0
  WHERE len_intersect is null;
  ```
- Calculate Green Factor:
  ```
  UPDATE hiking_trails
  SET green_factor = round(((len_intersect/len_total)*1)::numeric, 2)
  ```
  
## Tippecanoe
- Instalation:
  ```
  sudo apt-get update
  sudo apt-get install software-properties-common
  sudo apt-add-repository ppa:git-core/ppa
  sudo apt-get update
  sudo apt-get install git

  ## Tippecanoe Pre-reqas:
  sudo apt-get install build-essential libsqlite3-dev zlib1g-dev
  sudo apt-get update -y
  sudo apt-get install -y g++-5
  export CXX=g++-5

  git clone https://github.com/mapbox/tippecanoe.git
  cd tippecanoe
  sudo make
  sudo make install

  tippecanoe --version
  ```
- Creating MBTiles:
  - Export hike_trails layer to a GeoJSON file
  - Run the following command to generate MBTiles
    ```
    tippecanoe -P -o hike_trails.mbtiles --coalesce-densest-as-needed --extend-zooms-if-still-dropping -z16 -f -l "hike_trails" hike_trails_v1.geojson
    
    ## Description:
    -z or --maximum-zoom: The highest zoom level for which tiles are generated
    -o file.mbtiles or --output=file.mbtiles: Name the output fil
    --coalesce-densest-as-needed: If the tiles are too big at low or medium zoom levels, merge as many features together as are necessary to allow tiles to be created with those features that are still distinguished.
    --extend-zooms-if-still-dropping: If even the tiles at high zoom levels are too big, keep adding zoom levels until one is reached that can represent all the features
    -l : Specify the layer name instead of letting it be derived from the source file names
    -f or --force: Delete the mbtiles file if it already exists instead of giving an error
    -P or --read-parallel: Use multiple threads to read different parts of each GeoJSON input file at once
    ```

## Mapbox Studio
If you are new to Mapbox Studio, it may help to take a look at the Studio Manual (https://docs.mapbox.com/studio-manual/guides/).
- After signup on Mapbox, Navigate to Studio under profile icon and then tilesets
- Click on new tileset and Upload the mbtiles file
- Publish the tileset and copy the tileset id (to be used in webmap)
- Now navigate to Styles and create a new style or use existing one. You can add add the tileset layer as datasource and then play with the styling components.

## WebMap
To create a webmap, all you need to do is
- Create an html file
- Include Mapbox GL libraries
  ```
  <script src="https://api.mapbox.com/mapbox-gl-js/v1.12.0/mapbox-gl.js"></script>
  <link href="https://api.mapbox.com/mapbox-gl-js/v1.12.0/mapbox-gl.css" rel="stylesheet" />
  ```
- Add basemap and container
  ```
  var map = new mapboxgl.Map({
		container: 'map',
		style: 'mapbox://styles/shahzadbacha/ckpa9q4md37hu17oan8dyhti1',
		center: [-122.212,37.892], // Specify the starting position [lng, lat]
		zoom: 10, // Specify the starting zoom
		minZoom: 6,
		hash: true
	});
  ```
- Add vector layer source
  ```
  map.addSource('hike_trails_source', {
      type: 'vector',
      url: 'mapbox://shahzadbacha.7ohviuhf'
    });
  ```
- Add and style the layer
  ```
  map.addLayer({
    'id': 'trails_id',
    'type': 'line',
    'source': 'hike_trails_source',
    'source-layer': 'hike_trails',
    'layout': {},
    "minzoom": 5,
    "maxzoom": 22,
    "layout": {},
    "paint": {
      "line-color": "#1a9641",
      "line-width": 1px
      "line-dasharray": [1, 1]
    }
  });
  ```
- Add user interactivity
  ```
  // When the user moves their mouse over the trails layer, we'll filter the hover layer for the feature id under the mouse.
  map.on("mousemove", "trails_id", function(e) {
    map.getCanvas().style.cursor = 'pointer';
    map.setFilter("trails_hover_id", ["==", "id", e.features[0].properties.id]);
  });

  // Reset the trails layer's filter when the mouse leaves the layer.
  map.on("mouseleave", "trails_id", function() {
    map.getCanvas().style.cursor = '';
    map.setFilter("trails_hover_id", ["==", "id", ""]);
  });
  
  // Add popup when the user click on a feature
	map.on('click', function(e) {
    var features = map.queryRenderedFeatures(e.point, {
      layers: ['trails_id'] // replace this with the name of the layer
    });

    if (!features.length) {
      return;
    }
    var feature = features[0];

    // console.log(feature.layer['id'])
    var popup = new mapboxgl.Popup()
      .setLngLat(e.lngLat)
      .setHTML('<h3>' + feature.properties.name + '</h3><h4>Length : ' + feature.properties.len_total.toFixed(1) + ' Miles</h4><h4>Green Factor : '+ (feature.properties.green_factor*100).toFixed(0) +'%</h4>')
      .addTo(map);
	});
  ```
