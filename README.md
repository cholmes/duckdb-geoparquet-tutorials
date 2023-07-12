 # DuckDB and GeoParquet Tutorial

 This is a quick tutorial on how you can use DuckDB to easily access the [cloud-native version](https://beta.source.coop/cholmes/google-open-buildings) of the [Google Open Buildings](https://sites.research.google/open-buildings/) data set from [source.coop](https://beta.source.coop/) and transform it into your favorite GIS
 format. A big thanks to [Mark Litwintschik's post on DuckDB's Spatial Extension](https://tech.marksblogg.com/duckdb-gis-spatial-extension.html) for lots of the key information, it's highly recommended. 

## About DuckDB?

[DuckDB](https://duckdb.org/) is an awesome new tool for working with data. In some ways it's a next generation 'SQLite' (which is behind GeoPackage in the geo world) - but fundamentally designed for analysis workflows. TODO: more explanation. 

To install it just follow the instructions at: https://duckdb.org/docs/installation/index. This tutorial uses the command line version 

There are a couple of awesome extensions that make it very easy to work with parquet files on the cloud. [httpfs](https://duckdb.org/docs/extensions/httpfs.html) enables you to pull in S3
files directly within DuckDB, and [spatial](https://duckdb.org/docs/extensions/spatial) gives
you geometries with a number of operations, and lets you write out to over 50 different formats.

## Setting up DuckDB

Once you've installed it then getting started is easy. Just type `duckdb` from the command-line. If you want to persist the tables you create you can supply a name, like `duckdb buildings.db`, but it's not necessary for this tutorial. After you're in the DuckDB interface you'll need to install and load the two extensions (you just need to install once, so can skip that in the future):

```
INSTALL spatial;
LOAD spatial;
INSTALL httpfs;
LOAD httpfs;
```

The DuckDB docs say that you should set your S3 region, but it doesn't seem to be necessary. For 
this dataset it'd be:

```
SET s3_region='us-west-2';
```

## Full country into GeoParquet

We'll start with how to get a country-wide GeoParquet file from the [geoparquet-admin1](https://beta.source.coop/cholmes/google-open-buildings/browse/geoparquet-admin1) directory, which partitions the dataset into directories for each country and files for each admin level 1 region. 

For this we don't actually even need the spatial extension - we'll just use DuckDB's great S3 selection interface to easily export to Parquet and then use another tool to turn it into official GeoParquet. 

The following call selects all parquet files from the [`country=SSD`](https://beta.source.coop/cholmes/google-open-buildings/browse/geoparquet-admin1/country=SSD) (South Sudan) directory.

```
COPY (SELECT * FROM 
    's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=SSD/*.parquet') 
    TO 'south_sudan.parquet' (FORMAT PARQUET);
```

These stream directly out to a local parquet file. Below we'll explore how to save it into DuckDB and work with it. The output will not be Geoparquet, but a Parquet file with Well Known Binary. Hopefully DuckDB will support native GeoParquet output so we won't need the conversion at the end. As long as you name the geometry column 'geometry' then you can use the [gpq](https://github.com/planetlabs/gpq) `convert` function to turn it from a parquet file with WKB to a valid GeoParquet file.

To set up gpq just use the [installation docs](https://github.com/planetlabs/gpq#installation). And when it's set up you can just say:

```
gpq convert south_sudan.parquet south_sudan-geo.parquet
```


The `south_sudan-geo.parquet` file should be valid GeoParquet. You can then change it into any other format using GDAL's [`ogr2ogr`](https://gdal.org/programs/ogr2ogr.html):

```
ogr2ogr laos.fgb laos-geo.parquet
```

Note that the above file is only about 3 megabytes. Larger countries can be hundreds of megabytes or gigabytes, so be sure you have a fast connections or patience. DuckDB should give some updates on progress as it works, but it doesn't seem to be super accurate with remote files.

## Using DuckDB Spatial with GeoParquet

Now we'll get into working with DuckDB a bit more, mostly to transform it into different output formats to start. We'll start small, which should work with most connections. But once we get to the bigger requests they may take awhile if you have a slower connection. DuckDB can still be useful, but you'll probably want to save the files as tables and persist locally.

### Working with one file

We'll start with just working with a single parquet file.

#### Count one file

You can get the count of a file, just put in the S3 URL to the parquet file.

```
SELECT count(*) FROM 
   's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=LAO/Attapeu.parquet';
```

Results in:

```
┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│        98454 │
└──────────────┘
```

#### Select all one file

And you can see everything in it:

```
SELECT * FROM 
   's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=LAO/Attapeu.parquet';
```

Which should get you a response like:

```
┌────────────────┬────────────┬────────────────┬────────────────────────────────────────────────┬───────────┬─────────┬─────────┐
│ area_in_meters │ confidence │ full_plus_code │                    geometry                    │    id     │ country │ admin_1 │
│     double     │   double   │    varchar     │                      blob                      │   int64   │ varchar │ varchar │
├────────────────┼────────────┼────────────────┼────────────────────────────────────────────────┼───────────┼─────────┼─────────┤
│        86.0079 │     0.6857 │ 7P68VCWH+4962  │ \x01\x03\x00\x00\x00\x01\x00\x00\x00\x05\x00…  │ 632004767 │ LAO     │ Attapeu │
│        35.1024 │     0.6889 │ 7P68VCWH+4H65  │ \x01\x03\x00\x00\x00\x01\x00\x00\x00\x05\x00…  │ 632004768 │ LAO     │ Attapeu │
│        40.6071 │     0.6593 │ 7P68VCWH+53J5  │ \x01\x03\x00\x00\x00\x01\x00\x00\x00\x05\x00…  │ 632004769 │ LAO     │ Attapeu │
│           ·    │        ·   │       ·        │                       ·                        │     ·     │  ·      │    ·    │
│           ·    │        ·   │       ·        │                       ·                        │     ·     │  ·      │    ·    │
│           ·    │        ·   │       ·        │                       ·                        │     ·     │  ·      │    ·    │
│ 641629546 │ LAO     │ Attapeu │
│        59.2047 │     0.6885 │ 7P7976C9+J5X3  │ \x01\x03\x00\x00\x00\x01\x00\x00\x00\x05\x00…  │ 641629547 │ LAO     │ Attapeu │
│        13.8254 │     0.6183 │ 7P7976C9+M48G  │ \x01\x03\x00\x00\x00\x01\x00\x00\x00\x05\x00…  │ 641629548 │ LAO     │ Attapeu │
│       183.8289 │     0.7697 │ 7P7976H4+VHGV  │ \x01\x03\x00\x00\x00\x01\x00\x00\x00\x05\x00…  │ 641629614 │ LAO     │ Attapeu │
├────────────────┴────────────┴────────────────┴────────────────────────────────────────────────┴───────────┴─────────┴─────────┤
│ 98454 rows (40 shown)                                                                                               7 columns │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### Filter one file

From there you can easily filter, like only show the largest buildings:

```
SELECT * FROM 
  's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=LAO/Attapeu.parquet'
  WHERE area_in_meters > 1000;
```

#### Get one file in your local duckdb

If you've got a fast connection you can easily just keep doing your sql queries against the parquet files that are sitting on S3. But you can also easily pull the data into a table and then work with it locally.

```
CREATE TABLE attapeu AS SELECT * EXCLUDE geometry, ST_GEOMFROMWKB(geometry) AS geometry FROM 
's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=LAO/Attapeu.parquet';
```

This creates a true 'geometry' type from the well known binary 'geometry' field, which 
can then be used in spatial operations. Note this also shows one of Duck's [friendlier SQL](https://duckdb.org/2022/05/04/friendlier-sql.html) additions with `EXCLUDE`.

#### Write out duckdb table

You can then write this out as common geospatial formats:

```
COPY (SELECT * EXCLUDE geometry, ST_AsWKB(geometry) AS geometry from attapeu)
      TO 'attapeu-1.fgb' WITH  (FORMAT GDAL, DRIVER 'FlatGeobuf');
```

The DuckDB output does not seem to consistenly set the spatial reference system (hopefully someone will point out how to do this consistently or improve it in the future). You can clean this up with `ogr2ogr`:

```
ogr2ogr -a_srs EPSG:4326 attapeu.fgb attapeu-1.fgb
```

#### Directly streaming output

You also don't have to instantiate the table in DuckDB if your connection is fast, you can just do:

```
COPY (SELECT * EXCLUDE geometry, ST_AsWKB(ST_GEOMFROMWKB(geometry)) AS geometry FROM 
     's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=LAO/Attapeu.parquet') 
     TO 'attapeu-2.fgb' WITH (FORMAT GDAL, DRIVER 'FlatGeobuf');
```

This one also needs to cleaned up with a projection with ogr2ogr.

### Working with a whole country

Using DuckDB with Parquet on S3 starts to really shine when you want to work with lots of data. There are lots of easy options to just download a file and then transform it. But bulk downloading from S3 and then getting the data formatted as you want can be a pain - configuring your S3 client, getting the names of everything to download, etc.

With DuckDB and these GeoParquet files you can just use various * patterns to select multiple files and treat them as a single one:

```
SELECT count(*) FROM 's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=LAO/*.parquet';
```

The above query gets you all the bulidings in Laos. If your connection is quite fast you can do all these calls directly on the parquet files. But for most it's easiest to load it locally:

```
CREATE TABLE laos AS SELECT * EXCLUDE geometry, ST_GEOMFROMWKB(geometry) AS geometry FROM 
's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=LAO/*.parquet';
```

From there it's easy to add more data. Let's rename our table to `se_asia` and then pull down
the data from Thailand as well:

```
ALTER TABLE laos RENAME TO se_asia;
INSERT INTO se_asia from (SELECT * EXCLUDE geometry, ST_GEOMFROMWKB(geometry) AS geometry FROM 
's3://us-west-2.opendata.source.coop/google-research-open-buildings/geoparquet-admin1/country=THA/*.parquet');
```

This will take a bit longer, as Thailand has about ten times the number of buildings of Laos. 

You can continue to add other countries to the `se_asia` table, and then write it out as a number of gis formats just like above.
