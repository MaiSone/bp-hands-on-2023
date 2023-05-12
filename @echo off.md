@echo off

:   Command Line Cartography by Mike Bostock


:       See https://bl.ocks.org/mbostock/fb6c1e5ff700f9713a9dc2f0fd392c35

:       Bash Script Translated to Batch File (Tom Rutherford, 8/17)

:       As needed, install Node packages:

:call npm install -g d3
:call npm install -g shapefile
:call npm install -g d3-geo-projection
:call npm install -g ndjson-cli
:call npm install -g d3-scale-chromatic
:call npm install -g topojson

:       Download the census boundary archive (-L = follow redirects)

:c:\curl\src\curl.exe -L --output "cb_2014_55_tract_500k.zip" "http://www2.census.gov/geo/tiger/GENZ2014/shp/cb_2014_55_tract_500k.zip"

:       Retrieve the shape files:

title   unzip -o cb_2014_55_tract_500k.zip 
call unzip -o cb_2014_55_tract_500k.zip 

:       Convert the shape file (a dBase file defining feature properties) to GeoJSON:

title   shp2json cb_2014_55_tract_500k.shp -o wi.json
call shp2json cb_2014_55_tract_500k.shp -o wi.json


:       We could now display this in a browser using D3, but first we should
:       apply a geographic projection. By avoiding expensive trigonometric
:       operations at runtime, the resulting GeoJSON renders much faster,
:       especially on mobile devices. Pre-projecting also improves the
:       efficacy of simplification:

:       Perform a geographic projection.  The d3.geoConicEqualArea
:       projection is Albers, an equal-area projection, which is 
:       recommended for choropleth maps.  Other projections are
:       described at d3-stateplane (https://github.com/veltman/d3-stateplane)   
:       or spatialreference.org:
 
title   geoproject -o wi-albers.json "d3.geoConicEqualArea().parallels([44.5,45.5]).rotate([90,-42]).fitSize([960,960],d)" wi.json
call geoproject -o wi-albers.json "d3.geoConicEqualArea().parallels([44.5,45.5]).rotate([90,-42]).fitSize([960,960],d)" wi.json


:       NB: We use projection.fitSize to fit the input geometry (represented by d) to the
:       desired 960ﾃ�960 bounding box!

:       Generate an SVG image of the projection:

title   geo2svg -w 960 -h 960 < wi-albers.json > wi-albers.svg
call geo2svg -w 960 -h 960 < wi-albers.json > wi-albers.svg


:       Convert the the Geo-JSON feature collection to a newline-delimited 
:       JSON (NDJSON),  in which JSON values are separated by newlines (\n) 
:       to improve readability:

title   ndjson-split "d.features" <wi-albers.json >wi-albers.ndjs
call ndjson-split "d.features" <wi-albers.json >wi-albers.ndjs


:       The only difference between Geo_JSON and NDJSON is that there is 
:       one feature (one census tract) per line. We can now manipulate 
:       individual features using a text editory.  We are also able to 
:       use JS toolkit to modify the file.  Here we set each feature's id 
:       using ndjson-map:

title   ndjson-map "d.id = d.properties.GEOID.slice(2), d"  < wi-albers.ndjs > wi-id.ndjs
call ndjson-map "d.id = d.properties.GEOID.slice(2), d"  < wi-albers.ndjs > wi-id.ndjs


:       We use the feature IDs to join the geometry with the population 
:       estimates, which we will now download from the Census Bureau's API using  curl:

title   c:\curl\src\curl.exe "http://api.census.gov/data/2014/acs5?get=B01003_001E&for=tract:*&in=state:55" -o cb_2014_55_tract_B01003.json
:call c:\curl\src\curl.exe "http://api.census.gov/data/2014/acs5?get=B01003_001E&for=tract:*&in=state:55" -o cb_2014_55_tract_B01003.json


:       Note: please request a key when using the Census API.) The B01003_001E in the URL
:       specifies the total population estimate, while the for and in values specify that we want
:       data for each census tract in California. See the API documentation for details.
:       http://www.census.gov/data/developers/data-sets/acs-5year.html

:       The Census file is a JSON array. We convert it to an NDJSON stream in two steps
:       first using ndjson-cat (to remove the newlines):

title   ndjson-cat < cb_2014_55_tract_B01003.json > temp1.json
call ndjson-cat < cb_2014_55_tract_B01003.json > temp1.json


:       Then use ndjson-split (to separate the array into multiple lines):

title   ndjson-split "d.slice(1)" < temp1.json > temp2.json 
call ndjson-split "d.slice(1)" < temp1.json > temp2.json 


:       Use ndjson-map to reformat each line as an object:

title   ndjson-map "{id: d[2] + d[3], B01003: +d[0]}" < temp2.json  > cb_2014_55_tract_B01003.ndjs
call ndjson-map "{id: d[2] + d[3], B01003: +d[0]}" < temp2.json  > cb_2014_55_tract_B01003.ndjs


:       Now, magic! Join the population data to the geometry. Each line in the 
:       resulting NDJSON stream is a two-element array. The first
:       element (d[0]) is from wi-albers-id.ndj: a GeoJSON Feature representing 
:       a census tract polygon. The second element (d[1]) is from 
:       cb_2014_06_tract_B01003.ndjs: an object representing the population estimate 
:       for the same census tract.

title   ndjson-join "d.id" wi-id.ndjs cb_2014_55_tract_B01003.ndjs  > wi-join.ndjs
call ndjson-join "d.id" wi-id.ndjs cb_2014_55_tract_B01003.ndjs  > wi-join.ndjs


:       Compute the population density using ndjson-map, and to remove the additional 
:       properties we no longer need.   N.B.  The constant 2589975.2356 = 1609.34ﾂｲ 
:       converts the land area from square meters to square miles.

:       Note that the density value is floored rather than rounded. We don't need the extra
:       precision, so either results in a smaller output file. But since we will later apply a
:       threshold color encoding in the choropleth, rounding would be inappropriate: for example,
:       it would change an effective threshold of 4,000 to 3,999.5!

title   ndjson-map "d[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]" < wi-join.ndjs > wi-density.ndjs
call ndjson-map "d[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]" < wi-join.ndjs > wi-density.ndjs


:       To convert back to GeoJSON, use ndjson-reduce and ndjson-map:

title   ndjson-reduce "p.features.push(d), p" "{type: ""FeatureCollection"", features: []}" < wi-density.ndjs > wi-density.json
call ndjson-reduce "p.features.push(d), p" "{type: ""FeatureCollection"", features: []}" < wi-density.ndjs > wi-density.json


title   geo2svg -n -w 960 -h 960  -o wi-density.svg wi-density.json
call geo2svg -n -w 960 -h 960  -o wi-density.svg wi-density.json


:       Next use ndjson-map, requiring D3 via -r d3, and defining a fill property using a
:       sequential scale with the Viridis color scheme:

title   ndjson-map -r d3 "(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000])(d.properties.density), d)"  < wi-density.ndjs  > wi-color.ndjs
call ndjson-map -r d3 "(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000])(d.properties.density), d)"  < wi-density.ndjs  > wi-color.ndjs


:       Generate an SVG image file of the sequential scale heat map:

title   geo2svg -n --stroke none -p 1 -w 960 -h 960  < wi-color.ndjs > wi-color.svg
call geo2svg -n --stroke none -p 1 -w 960 -h 960  < wi-color.ndjs > wi-color.svg

:       Cynthia A. Brewer's ColorBrewer (http://colorbrewer2.org/) provides a fantastic
:       set of well-designed discrete color schemes. The d3-scale-chromatic package
:       provides a convenient API for ColorBrewer in JavaScript. Here we implement a
:        threshold scale with the OrRd color scheme:

title   ndjson-map -r d3 -r d3=d3-scale-chromatic "(d.properties.fill = d3.scaleThreshold(d3.interpolateViridas).domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9])(d.properties.density), d)"  <wi-density.ndjs >wi-heat.ndjs
call ndjson-map -r d3 -r d3=d3-scale-chromatic "(d.properties.fill = d3.scaleThreshold(d3.interpolateViridas).domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9])(d.properties.density), d)"  <wi-density.ndjs >wi-heat.ndjs


:       Generate an SVG so we can see whether the color scale is OK:

title   geo2svg -n --stroke none -p 1 -w 960 -h 960  < wi-heat.ndjs > wi-heat.svg
call geo2svg -n --stroke none -p 1 -w 960 -h 960  < wi-heat.ndjs > wi-heat.svg


:       Convert the density dataset into TopoJSON format:

title   geo2topo -n tracts=wi-density.ndjs  > wi.topo
call geo2topo -n tracts=wi-density.ndjs  > wi.topo


:       The Census Bureau publishes county boundaries, but we don't actually need them. 
:       TopoJSON has another powerful trick up its sleeve: since census tracts compose 
:       hierarchically into counties, we can derive county geometry using topomerge.

:       The -k argument defines a key expression that topomerge will evaluate to group features
:       from the tracts object before merging. (It's similar to nest.key in d3-collection.)  

:       The first three digits of the census tract id represent the state-specific part 
:       of the county FIPS code, so the census tracts for each county will be merged, 
:       resulting in county polygons. The result forms a new counties object on the 
:       output topology.

title   topomerge -k "d.id.slice(0, 3)" counties=tracts < wi.topo  > wi-merge.topo
call topomerge -k "d.id.slice(0, 3)" counties=tracts < wi.topo  > wi-merge.topo

:       We don't actually want the full county polygons; we want only the internal 
:       bordersﾂ葉he ones separating counties. (Stroking exterior borders tends to 
:       lose detail along coastlines.) We can also compute these with topomerge. 

:       A filter (-f) expression is evaluated for each arc, given the arc's adjacent 
:       polygons a and b. By convention, a and b are the same on exterior arcs, 
:       and thus we can overwrite the counties object with a mesh of the internal 
:       borders like so:

title   topomerge --mesh -f "a !== b" counties=counties  < wi-merge.topo > wi-mesh.topo
call topomerge --mesh -f "a !== b" counties=counties  < wi-merge.topo > wi-mesh.topo

:       Use topo2geo to extract the simplified tracts from the topology:

title   topo2geo -n counties=- < wi-mesh.topo >temp3.ndjs
call topo2geo -n counties=- < wi-mesh.topo >temp3.ndjs

:       Use ndjson-map to assign the fill property for each tract:

title   ndjson-map -r d3 "(d.properties = {""stroke"": ""#000"", ""stroke-opacity"": 0.3}, d)" <temp3.ndjs >wi-counties.ndjs
call ndjson-map -r d3 "(d.properties = {""stroke"": ""#000"", ""stroke-opacity"": 0.3}, d)" <temp3.ndjs >wi-counties.ndjs

:       Generate an SVG to assess the county outlines:

title   geo2svg -n -w 960 -h 960  < wi-counties.ndjs > wi-counties.svg
call geo2svg -n -w 960 -h 960  < wi-counties.ndjs > wi-counties.svg

:       Concatenate the newline-delimited files:

title   copy /b wi-heat.ndjs+wi-counties.ndjs wi.ndjs
call copy /b wi-heat.ndjs+wi-counties.ndjs wi.ndjs


:       Generate a the SVG visualization:

title   geo2svg -n --stroke none -p 1 -w 960 -h 960  < wi.ndjs > wi.svg
call geo2svg -n --stroke none -p 1 -w 960 -h 960  < wi.ndjs > wi.svg

:       Generate simplified (low overhead) versions.
:       The -p 1 argument tells toposimplify to use a planar area 
:       threshold of one square pixel when implementing Visvalingham's 
:       method; this is appropriate because we previously applied a conic 
:       equal-area projection. 

:       The -f says to remove small, detached ringsﾂ様ittle islands, but not 
:       contiguous tractsﾂ庸urther reducing the output size.

:       wi.topo (1,980,328) => wi-simple.topo (683,200) => wi-quantized.topo (311,948)

title   toposimplify -p 1 -f < wi.topo > wi-simple.topo
call toposimplify -p 1 -f < wi.topo > wi-simple.topo

title   topoquantize 1e5 <wi-simple.topo >wi-quantized.topo
call topoquantize 1e5 <wi-simple.topo >wi-quantized.topo

:   The remaining steps are identical to those described above:

title   topomerge -k "d.id.slice(0, 3)" counties=tracts < wi-quantized.topo  > qwi-merge.topo
call topomerge -k "d.id.slice(0, 3)" counties=tracts < wi-quantized.topo  > qwi-merge.topo
call topo2geo -n tracts=- <wi-quantized.topo >wi-quantized.ndjs

title   ndjson-map -r d3 -r d3=d3-scale-chromatic "(d.properties.fill = d3.scaleThreshold(d3.interpolateViridas).domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9])(d.properties.density), d)"  <wi-quantized.ndjs >qwi-heat.ndjs
call ndjson-map -r d3 -r d3=d3-scale-chromatic "(d.properties.fill = d3.scaleThreshold(d3.interpolateViridas).domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9])(d.properties.density), d)"  <wi-quantized.ndjs >qwi-heat.ndjs

title   topo2geo -n counties=- >temp4.ndjs < qwi-merge.topo
call topo2geo -n counties=- >temp4.ndjs < qwi-merge.topo

title   ndjson-map -r d3 "(d.properties = {""stroke"": ""#000"", ""stroke-opacity"": 0.3}, d)" <temp4.ndjs >qwi-counties.ndjs
call ndjson-map -r d3 "(d.properties = {""stroke"": ""#000"", ""stroke-opacity"": 0.3}, d)" <temp4.ndjs >qwi-counties.ndjs

title   copy /b qwi-heat.ndjs+qwi-counties.ndjs qwi.ndjs
call copy /b qwi-heat.ndjs+qwi-counties.ndjs qwi.ndjs

title   geo2svg -n --stroke none -p 1 -w 960 -h 960  < qwi.ndjs > qwi.svg
call geo2svg -n --stroke none -p 1 -w 960 -h 960  < qwi.ndjs > qwi.svg

title   Add the legend to wi.svg.
sed "$d" <wi.svg >wi_legend.svg
tail -n+4 legend.svg >>wi_legend.svg

title   Add the legend to qwi.svg.
sed "$d" <qwi.svg >qwi_legend.svg
tail -n+4 legend.svg >>qwi_legend.svg

title   All done
