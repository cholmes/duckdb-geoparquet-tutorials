## About

This is some quick notes on how to get proper GeoParquet files (and other formats) from the new [Overture Maps Release](https://github.com/OvertureMaps/data)

I'm finding DuckDB to be a great tool for this, and love that the Overture team put up a couple sample queries at https://github.com/OvertureMaps/data/tree/main/duckdb_queries

I tried using GPQ and ogr2ogr directly. GPQ had a problem with byte array, hoping it's an easy fix at https://github.com/planetlabs/gpq/issues/57 ogr2ogr
seems to have trouble with nested data structures perhaps? I get

```
Warning 1: Field names of unhandled type map<string, list<array_element: map<string, string ('array_element')>> ('names')> ignored
Warning 1: Field sources of unhandled type list<array_element: map<string, string ('array_element')>> ignored
ERROR 1: ICreateFeature: NULL geometry not supported with spatial index
ERROR 1: Unable to write feature 0 from layer 20230725_211237_00132_5p54t_25816df1-b864-49c0-a9a3-a13da4f37a90.
ERROR 1: Terminating translation prematurely after failed
translation of layer 20230725_211237_00132_5p54t_25816df1-b864-49c0-a9a3-a13da4f37a90 (use -skipfailures to skip errors)
```

So DuckDB with their samples seems to be ideal. But the readme just shows GeoJSON output. What about GeoParquet?

#### Countries to GeoParquet

That's pretty easy, can adapt their example to do Parquet output:

```
load spatial;
load httpfs;
COPY (
    SELECT
           type,
           subType,
           localityType,
           adminLevel,
           isoCountryCodeAlpha2,
           JSON(names) AS names,
           JSON(sources) AS sources,
           geometry
      FROM read_parquet('s3://overturemaps-us-west-2/release/2023-07-26-alpha.0/theme=admins/type=*/*', filename=true, hive_partitioning=1)
     WHERE adminLevel = 2
       AND ST_GeometryType(ST_GeomFromWKB(geometry)) IN ('POLYGON','MULTIPOLYGON')
) TO 'countries-tmp.parquet'
WITH (FORMAT Parquet);
```

(be sure to have spatial and httpfs installed)

And then once you get your parquet output then use GPQ or ogr2ogr

```
gpq convert countries-tmp.parquet countries.parquet
```

#### Countries to GeoPackage

What about output to GeoPackage, Shapefile or Flatgeobuf?

```
load spatial;
load httpfs;
COPY (
    SELECT
           type,
           subType,
           localityType,
           adminLevel,
           isoCountryCodeAlpha2,
           JSON(names) AS names,
           JSON(sources) AS sources,
           ST_GeomFromWkb(geometry) AS geometry
      FROM read_parquet('s3://overturemaps-us-west-2/release/2023-07-26-alpha.0/theme=admins/type=*/*', filename=true, hive_partitioning=1)
     WHERE adminLevel = 2
       AND ST_GeometryType(ST_GeomFromWkb(geometry)) IN ('POLYGON','MULTIPOLYGON')
) TO 'countries.gpkg'
WITH (FORMAT GDAL, DRIVER 'GPKG');
```
##### correct srs

Note that DuckDB's spatial support does not yet write out the projection information. I've heard this is coming soon, but until then you need to 
convert the output using ogr2ogr:

```
ogr2ogr -a_srs EPSG:4326 countries-tmp.gpkg countries.gpkg
mv countries-tmp.gpkg countries.gpkg
```

##### Shapefile and FlatGeobuf

Note that GeoPackage output from DuckDB is particulary slow (like 3-4x slower than fgb or shapefile, 10x slower than parquet). To use another format
just change the file suffix and the driver. Shapefile is `DRIVER 'ESRI Shapefile'` and FlatGeobuf is `DRIVER 'FlatGebuf').

Note that ESRI conversion will raise errors like:
```
Warning 6: Normalized/laundered field name: 'localitytype' to 'localityty'
Warning 6: Normalized/laundered field name: 'isocountrycodealpha2' to 'isocountry'
Warning 1: One or several characters couldn't be converted correctly from UTF-8 to ISO-8859-1.  This warning will not be emitted anymore.
```
So if you have QGIS just use FlatGeobuf or GeoParquet, and if you have ESRI GeoPackage.

### Saving Data to DuckDB

Note that all the above are pulling the Parquet files from the cloud for every command. If you're going to put it in different formats or want to 
explore it locally then I recommend putting it into DuckDB. You can do this in a temporary table by just typing `duckdb` and doing the following
commands. That will keep the data local while you're in DuckDB, but it'll go away when you close it. Most convenient is to just make a DuckDB
database, you just pick any file name, but ending in .db or .duckdb is convenient to remember what it is:

```
duckdb overture.duckdb
```

Then when you start DuckDB you create a table instead of writing it out immediately. If you want the exact fields you'll write out then say:

```
load spatial;
load httpfs;
CREATE TABLE countries AS (
    SELECT
           type,
           subType,
           localityType,
           adminLevel,
           isoCountryCodeAlpha2,
           JSON(names) AS names,
           JSON(sources) AS sources,
           ST_GeomFromWKB(geometry) AS geometry
      FROM read_parquet('s3://overturemaps-us-west-2/release/2023-07-26-alpha.0/theme=admins/type=*/*', filename=true, hive_partitioning=1)
     WHERE adminLevel = 2
       AND ST_GeometryType(ST_GeomFromWKB(geometry)) IN ('POLYGON','MULTIPOLYGON')
);
```

(note you don't need to load spatial and httpfs for every call - just every time you open a duckdb. So I just put them in for copy and paste ease,
as it doesn't hurt to have them and the error messages when they aren't there are often a bit tricky to figure out).

Then you can do normal sql commands:

```
select * from countries;
```

```
select count(*) from countries;
```

Then if you want to write this out to a format  you can do:

```
COPY (SELECT * from countries) TO 'countries.fgb'
WITH (FORMAT GDAL, DRIVER 'FlatGeobuf');
```

(Remember you'll need to convert the output with ogr2ogr to get the srs, as [above](#correct-srs)

### Extracting JSON

So the JSON blobs aren't super useful, and it can be a pain to do the JSON extraction in a geospatial program. So we can also use DuckDB to 
extract out key values:

To see the ISO code and then the name in the local language you can do:

```
select isocountrycodealpha2, names->>'$.common[0].value', geometry from countries;
```

(I'm assuming that 'local' is always first, but that may be a bad assumption. It'd be better to look for the 'language' value to be 'local', but I'm not sure how to do that - suggestions welcome).

### Buildings

(just notes for myself here for now).

After download
`create table buildings as (select * from read_parquet('202*'));` for directory after aws s3 cp or sync call (that has some other stuff in it), when duckdb is started in the directory.

```
COPY (SELECT * from b) TO 'buildings-tmp.fgb'
WITH (FORMAT GDAL, DRIVER 'FlatGeobuf');
```

Write out just values that aren't complicated (to parse these out better later);

```
COPY (SELECT id, updatetime, geometry from b) TO 'buildings-t.parquet' WITH (FORMAT PARQUET);
```

Then `gpq convert buildings-t.parquet buildings.parquet`

Buildings data structure:

```
CREATE TABLE buildings(
    id VARCHAR,
    updatetime VARCHAR,
    "version" INTEGER,
    "names" MAP(VARCHAR, MAP(VARCHAR, VARCHAR)[]),
    "level" INTEGER,
    height DOUBLE,
    numfloors INTEGER,
    "class" VARCHAR,
    sources MAP(VARCHAR, VARCHAR)[],
    bbox STRUCT(minx DOUBLE, maxx DOUBLE, miny DOUBLE, maxy DOUBLE),
    geometry BLOB);;
```

Join to get country:

```
ALTER TABLE buildings
ADD COLUMN country_iso VARCHAR;

UPDATE buildings
SET country_iso = countries.isocountrycodealpha2
FROM countries
WHERE ST_Intersects(countries.geometry, ST_GeomFromWKB(buildings.geometry));
```

Initial order by attempt:

```
COPY (SELECT * from b WHERE country = 'US' ORDER BY ST_X(ST_Centroid(ST_GeomFromWKB(geometry))),
    ST_Y(ST_Centroid(ST_GeomFromWKB(geometry)))) TO 'buildings-us.parquet'
```

Code to add quadkeys is at https://github.com/opengeos/open-buildings/blob/e0825d25423b50c89754bd5b0975118db2cf3c68/open_buildings/overture-buildings-parquet-add-columns.py#L46

Statement to make a view for a smaller quadkey:

```
CREATE VIEW buildings_l5 AS SELECT id, updatetime, "version", "names", "level", height, numfloors, "class", sources,
    bbox, geometry, quadkey, SUBSTR(quadkey, 1, 5) AS quadkey5, country_iso FROM buildings;
```

The copy command I used is (used in a script, but just swap out for a specific quadkey):
```
    copy_cmd = f"COPY (SELECT * FROM buildings_l5 WHERE country_iso = 'US' AND quadkey5 = '{quad5}' ORDER BY quadkey) TO '{quad5}_temp.parquet' WITH (FORMAT PARQUET); "
```

### Getting data from source.coop

```
CREATE TABLE gg_park AS (SELECT ST_GeomFromWKB(geometry) AS geometry, JSON(names) AS names, JSON(height) as height from
read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country/country_iso=US/*.parquet', hive_partitioning=1), WHERE 
    bbox.minX > -122.5103 AND
    bbox.maxX < -122.4543 AND
    bbox.minY > 37.7658 AND
    bbox.maxY < 37.7715);
```

```
CREATE TABLE gg_park AS (SELECT * from 
read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country/country_iso=US/20321.parquet', hive_partitioning=1), WHERE 
    bbox.minX > -122.5103 AND
    bbox.maxX < -122.4543 AND
    bbox.minY > 37.7658 AND
    bbox.maxY < 37.7715);
```

```
SELECT count(*) from 
read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country/country_iso=US/20321.parquet', hive_partitioning=1), WHERE 
    bbox.minX > -122.5103 AND
    bbox.maxX < -122.4543 AND
    bbox.minY > 37.7658 AND
    bbox.maxY < 37.7715;
```

```
SELECT ST_GeomFromWKB(geometry) AS geometry, JSON(names) AS names, JSON(height) as height from '*.parquet' WHERE 
    bbox.minX > -122.5103 AND
    bbox.maxX < -122.4543 AND
    bbox.minY > 37.7658 AND
    bbox.maxY < 37.7715
```

```
SELECT * from 
read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country-2/country_iso=US/20321_20000.parquet', hive_partitioning=1), WHERE 
    bbox.minX > -122.5103 AND
    bbox.maxX < -122.4543 AND
    bbox.minY > 37.7658 AND
    bbox.maxY < 37.7715;
```

```
SELECT * from 
read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country-2/20321_5k.parquet'), WHERE 
```

```
select * from read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country-2/country_iso=US/*.parquet'), WHERE quadkey LIKE '2032120231%';
```

```
select * from read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country-2/country_iso=US/*.parquet'),
    WHERE quadkey LIKE '2032120231%' AND
    ST_Within(ST_GeomFromWKB(geometry), ST_GeomFromText('POLYGON((-122.5103 37.7658, -122.4543 37.7658, -122.4543 37.7715, -122.5103 37.7715, -122.5103 37.7658))'));
```

```
load spatial;
select * from read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country-2/*.parquet'),
    WHERE quadkey LIKE '030230%' AND
    ST_Within(ST_GeomFromWKB(geometry), ST_GeomFromText('POLYGON ((-76.272307 45.616715, -76.110399 45.462413, -76.053333 45.643628, -75.965744 45.403742, -75.848958 45.456828, -75.943183 45.294619, -76.378476 45.321687, -76.124997 45.375782, -76.342644 45.494982, -76.320083 45.552629, -76.320083 45.552629, -76.272307 45.616715))'));
```

```
load spatial;
select * from read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-country-2/*.parquet'),
    WHERE quadkey LIKE '12002332%' AND
    ST_Within(ST_GeomFromWKB(geometry), ST_GeomFromText('POLYGON ((9.446443 56.207488, 9.432342 56.178091, 9.535293 56.16731, 9.467509 56.188492, 9.543958 56.190004, 9.446443 56.207488))'));
```

```
load spatial;
select * from read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-3/*/*.parquet'),
WHERE quadkey LIKE '0213231%' AND
ST_Within(ST_GeomFromWKB(geometry), ST_GeomFromText('POLYGON ((-103.440312 44.12233, -103.319886 44.12233, -103.319886 44.045527, -103.440312 44.045527, -103.440312 44.12233))'));
```

```
load spatial;
COPY (select id, updatetime, ST_AsWKB(ST_GeomFromWKB(geometry)) AS geometry, country_iso from read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-3/*.parquet'),
WHERE quadkey LIKE '0213231%' AND
ST_Within(ST_GeomFromWKB(geometry), ST_GeomFromText('POLYGON ((-103.440312 44.12233, -103.319886 44.12233, -103.319886 44.045527, -103.440312 44.045527, -103.440312 44.12233))'))) TO 'buildings.geojson'
WITH (FORMAT GDAL, DRIVER 'GeoJSON');
```

```
COPY (select id, country_iso, ST_AsWKB(ST_GeomFromWKB(geometry)) AS geometry from read_parquet('s3://us-west-2.opendata.source.coop/cholmes/overture/geoparquet-3/*.parquet'),
WHERE quadkey LIKE '2103213003%' AND
ST_Within(ST_GeomFromWKB(geometry), ST_GeomFromText('POLYGON ((-58.53445 -34.64447, -58.570879 -34.660286, -58.597863 -34.615051, -58.514887 -34.599226, -58.542546 -34.627541, -58.542546 -34.627541, -58.53445 -34.64447))'))) TO 'buildings.fgb' WITH (FORMAT GDAL, DRIVER 'FlatGeobuf');
```
