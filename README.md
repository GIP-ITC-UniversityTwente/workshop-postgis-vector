# **C122 Workshop Data acquisition and PostGIS Vector**

### This workshop aims to initialize the student in acquiring data from elsewhere and in using PostGIS vector data.


----------
## Install software tools for using PostgreSQL

PostgreSQL is a client/server database management system.  It is open source, and can be rapidly installed on most operatingt systems.  In the course, we already have a set up server.  [You are welcome to set up your own server on your own machine, but this is not necessary for this module ... and also we will not support such insrtallation from our team.]

Because PostgreSQL operates in client-server mode, there are client-side software components needed to connect to the server.  Those you should install on your laptop.  Please find a <a href="https://github.com/GIP-ITC-UniversityTwente/workshop-postgis-vector/blob/master/Instructions_Installing_PG_ComandLineTools.docx">support document</a> on these pages that explains how this is done. So,  you will need to install some software tools on your own machine to be able to connect to the server.

Specifically, you will need to be able to run pgAdmin (a GUI), psql (a commandline tool).  Also, you will need ogr2ogr (a commandline tool), but in normal cirucmstances you already have this tool on your system, as it came with your installation of gdal.

Now, PostGIS is a spatial database library for PostgreSQL.  It brings spatial data types and spatial functions, much like you hacve seen for ogr, into the databse platform.  This allows the storage of vector and raster data in postgresql tables, and spatial function calls over such data.

If you explore the database you will notice that the schema public is populated with tables, views and functions.

More information about PostGIS installation can be found at the official documentation: [https://postgis.net/docs/postgis_installation.html](https://postgis.net/docs/postgis_installation.html)

We also recommend, in case of problems, to have a look at the PostGIS FAQ:
[https://postgis.net/docs/PostGIS_FAQ.html](https://postgis.net/docs/PostGIS_FAQ.html)

----------
## Your database account

We have organized a user account for all students on the postgresql server.  Yoiu will need its coordinates to work on that server.  Here are the details:

**server IP address**: gip.itc.utwente.nl

**port**: 5434

**user**: *your-s-number-with-s*

**password**: your regular UT password

**database**: c122

The c122 database very typically has a data schema with the name *public*.  This is where common data tables and functions are typically held.  Every user has her own schema also.  The normal case also applies here: the personal schema has the name of your account.  You will be able to query any data held anywhere in the database, but when you create a new table, a new view or function, this must be done in your own schema.  Where this applies to examples below, we have used *YOURSCHEMA* as a template name.

Much of the specific exercise data used in the below actually sits in the *vectors* schema.

The exercises below are split in two parts.  One part is about acquiring data and uploading into the database, the other part is about computing with already uploaded vector data.

----------

## Acquisition of data and loading into the database

### The case of plain data

This exercise is about acquiring data and mechanisms to upload acquired data into a database.  We will work with two data sources from The Netherlands.  One is the Dutch National Bureau of Statistics, aka CBS, (for plain stats data), the other is the Dutch National Georegistry (for spatial data).  The two data sources connect with each other and would allow some interesting combinatorial experiments.

The first dataset that we have an interest in are statistics per municipality on household garbage amounts.  (As collected by trucks.)  The data that we will obtain indicates for different garbage types the average volume per household per year. 
A description is here, though in Dutch: <a href="https://opendata.cbs.nl/ODataApi/odata/83452NED/TableInfos">table information</a>. Use google translate for some indications.

We will take it easy here and suggest you the Python script to obtain this data.  The script actually codes for two ways to do so.  Use only one, and/or try out the other, just to understand how it is done.  Clearly you eventually want to upload into the database only one dataset.

[ Incidentally, this exercise makes use of package cbsodata.  It is documented here: https://media.readthedocs.org/pdf/cbsodata/latest/cbsodata.pdf.  Only so you have a reference; there is no immediate need to chase that link. ]

```python
import cbsodata                        # install it first!  This is a specific CBS-offered package.
import json                            # Needed only for the first option

import csv                             # Needed only for the second option

mydata = cbsodata.get_data('83452NED')

#################
# Into JSON
#################
with open('83452NED.json', 'w') as jsonfile:
    json.dump(mydata, jsonfile)

#################
# Or into a CSV
#################
keys = mydata[0].keys()
with open('83452NED.csv', 'w', newline='') as output_file:
    dict_writer = csv.DictWriter(output_file, keys)
    dict_writer.writeheader()
    dict_writer.writerows(mydata)
```
You should now have a data set.  If it is JSON, ogr2ogr as discussed below can help you to load it into your table.  Understand that for such to work you need to indicate names of the database server, database, schema, and table.  Because csv files are another very common data container, we explain how these must be handled for ingestion into a database.  This is probably what you should try out for this first dataset. Here are the steps:

1. think up a proper name for the table that will hold your dataset, and look at the data first to understand which attributes (column) the table has, and which data type is associated with each attribute.
2. create the table in *YOURSCHEMA*.  This could be done with a sql CREATE TABLE statement, but we suggest here that you use the pgAdmin GUI to create the table and one by one also its attributes.  We advice that you use the same sequence of attributes as the dataset uses.  Decide whether your dataset already has an attribute (or a combination of these) that can function as primary key.  If so, define this while creating the table.  If not so, create an extra attribute, named *dsid* for instance of type *serial*.  This type hands out unique integer numbers to newly created tuples automatically.  Define the primary now when you are still in table creation mode.  If you forgot, open the table properties and define it still.
3. By now, you should have a data table that has attributes and a primary key defined, but that has no data yet.  For sake of discussion, we will cal it *mytable* here, but understand that is a hopelessly stupid name for a real table.  You ewant to study the contents of the csv file briefly.  Open it, but not in EXCEL!  This software changes your data unknowingly, so better iuse a plain text editor like Notepad or so, to look inside.

You want to know a few things now: is there a header line or does the first line already have real data values.  What is the sequence of attributes provided?  Which character (if any) separates different values?  Which character (if any) is used to delineate string values?  Take a note because you will need this info.  If your data holds non-asciss characters, you shoudl also find out which character encoding is used.  (We do not think this is a problem in this exercise, but in general it can be.)
4. Time to upload the data.  We assume it is in the file *mydataset.csv*.  There is succinct need to understand that the data is on your machine and not on the server machine.  So, we need help to bring it across.  

The commandline tool **psql** does just that: if you call it, you will be prompted to type commands which the tool will execute against a remote server.  For this exercise, you should start up psql in the directory where your data file is, as that is the easiest operation. The typical incantation to start your session can be this:
```os
psql -h hostname -p portnumber -d databasename -U username
```
You will now be prompted by psql for a password, and then next see a pure prompt for further commands to be sent to that database on that server.  We will issue only one command which indicates "read my data file and store the data into mytable."  It is done with the psql \COPY statement.  One needs to get used to it, and it is common that your first try will give erors.  Do not give up, read the error message and continue repairing your command until it works.  The arrow-up brings you back to the previous typed out command, so you can quickly repair.

[ A somewhat common problem occurs when your data file holds characters that your computer, or the receiving database server does not know how to handle.  This is especially common when data sits on a server with operating system 1, is received by a laptop such as yours with operating system 2, and is then sent to a databse server with operating system 3.  The error reported upon an upload attempt will complain about byte sequence or character encoding. In such cases, you need to teel psql which character encoding you are using.  You will do this as first command in your psql session. Typically by stating:
```os
\encoding myEncoding
```
In here, the myEncoding is a stub and typically is one of : UTF8, LATIN1 or WIN1250. It is typed without quotes around it. ]

We will not otherwise spoil the fun of your trying, and want to point out that the full syntax of postgresql's COPY and psql's \COPY command are discussed on the <a href="https://www.postgresql.org/docs/9.6/sql-copy.html">COPY manual page</a>.  The options are shared between COPY and \COPY.  Do understand them and reflect on your study of the contenst of the csv file that was suggested above.  We advice that you prepare the \COPY statement in a plain text editor, and that you copy the header line from the csv file.

If all has worked, psql will report the number of records created into your table, and prompt for the next thing.  And that next thing can be a simple **\quit**.

### The case of spatial data

Uploading spatial data is a bit special.  We wont work on raster data in this context, so will not explain that part.  Generally, ogr2ogr is quite capable, and if you run it with "--help" option, it will explain how it works.  Asking for a manual page with a web browser will also do.

So, **ogr2ogr** is also a command line tool. It converts simple feature data between file formats. Because of the multiple format capability, it is frequently used to load vector files into databases like PostgreSQL or sqlite or, vice versa, from databases into files like shapefiles or geojson.

The tool is part of the GDAL package. Although it also comes with the GDAL Python module, it is in fact a commandline tool and not a Python method.  It should be automatically found by the windows command prompt, so try that. For most of the students it can be found in C:\Program Files\Python37\Lib\site-packages\osgeo\ogr2ogr.exe if it is not found.

Ogr2ogr has many options for its use; some of the most important ones are:

**-f** : output file format name, some possible values are -f "PostgreSQL" or -f "ESRI Shapefile" or -f "GeoJSON"

**-append** : If used, the command appends to existing layer instead of creating a new layer

**-update** : If used, the command opens existing output datasource in update mode rather than trying to create a new one

**-select** field_list : Comma-delimited list of fields from input layer to copy to the new layer. Note that this setting cannot be used together with -append. To control the selection of fields when appending to a layer, use -fieldmap or -sql.

**-sql** sql_statement: SQL statement to execute. The resulting table/layer will be saved to the output.

Here are some examples of ogr2ogr usage.

**Upload a geojson file into a postgis database**
```os
ogr2ogr -f "PostgreSQL" PG:"host=gip.itc.utwente.nl dbname=c122 user=yourusername password=yourpassword port=5434" provincial.json -nln destination_table –append
```

**Download from a postgis database into a shapefile**
```os
ogr2ogr -f "ESRI Shapefile" myshapefile.shp PG:"host=gip.itc.utwente.nl dbname=c122 user=yourusername password=yourpassword port=5434" -sql "SELECT sp_count, geom FROM table WHERE province = 'blablabla'"
```
Besides the conversion between data/file formats, ogr2ogr can be used for many other operations such as vector projection, clip, extract, filter, and dimension conversion.  Such as 3d to 2d.  Also attribute conversion and many more is possible. For more information, please check the official documentation: <a href="https://www.gdal.org/ogr2ogr.html">ogr2ogr manual page</a>.



Finally, to upload shapefiles into a postgis-enabled postgresql database, the commandline tool **shape2pgsql** is also available.  We mention it for ist common use and historic perspective.  It is specific to shapefiels in context to PostGIS.  It creates a .sql file that you can subsequently load into your database with a call to psql.  This process is well-documented in the online PostGIS manual, and elsewhere.  Pay attention to the various command options.  The reverse operator is available with pgsql2shape.

## Experimentation with the data

If your time allows, and you now have two tables in your own database schema, it will be of interest to create a join between them so that the geometric data is connected with the statistical data.  Think about how to do that.  You should aim for a query that brings five pieces of information per municipality: a unique number, the municipal name, the year, and some statistic of your choice from the first data set. And also the geometry, of course! Test and execute the query until you believe it performs well.  Now assume that the query definition is some SFW code.  Then embed it as follows to define it as a stored view:
```sql
create or replace view YOURSCHEMA.YOURVIEW as
SFW;
alter table YOURSCHEMA.YOURVIEW owner to YOURNAME;
```
And finally, load it into a QGIS view for map scrutiny.  Play with the visual variables until you get some decent result.

----------

## Working with vector data in the vectors schema

For experimentation with vector data, w ehave already created anorther database with data.  It is discussed below.

----------

## Requirements

This workshop assumes a basic knowledge of SQL or *Structured Query Language* which is the language you use to interact with a PostgreSQL database (or any other relational database). If you are new to SQL we recommend you to follow the W3 Schools course on SQL: https://www.w3schools.com/sql/

## Explore the database with QGIS DB Manager

You can view spatial data held in a PostGIS database with QGIS.  Start it up and create a new data layer over a Postgis connection. Use this to explore the vectors inside the schema vectors. It is important to know your data before we start analyzing it. Make sure that first you create a connection to the database as explained here: https://www.youtube.com/watch?v=r4pOyVZ3QVs 

From now on we assume QGIS is your client application.

If you are a non-Portuguese speaker, here is a quick explanation of the attributes and table names you may see as you follow the workshop:

Freguesia = Parish

Concelho = Municipality

Distrito = District/region

Ferrovia = Railroad

Lugares = Places

Therefore the table ```vectors.porto_freguesias``` has all the parishes that exist in the district of Porto, Portugal.

----------

## First queries - standard functionality

**Example 1 - Select all**

This simplest type of query returns all the records and all the attributes of a given table:

```sql 
SELECT * 
FROM  vectors.porto_freguesias;
```
**Example 2 - Condition**

Most of the time you don't want the whole table, you just want the rows that meet certain criteria. The next query will return all the attributes but only for the parishes that belong to the municipality of Matosinhos:

```sql 
SELECT * 
FROM  vectors.porto_freguesias
WHERE concelho = 'MATOSINHOS';
```

**Example 3 - Ilike**

The following query will return the same result as the previous one, but by using  ```ilike``` instead of the ```=``` operator the string matching becomes case INsensitive:

```sql 
SELECT * 
FROM  vectors.porto_freguesias
WHERE concelho ILIKE 'mAtOsInHoS';
```
**Example 4 - Like**
Like on the other hand is case sensitive therefore the following query will give zero results.
```sql
SELECT * 
FROM  vectors.porto_freguesias
WHERE concelho LIKE 'Matosinhos';
```
**Example 5 - Selecting attributes in a query**

You may want to have your query returning specific attributes instead of all of them  (the ```*``` sign) as we have been doing. In the example below, we are still getting all the parishes that belong to the municipality of Matosinhos, but we are only interested in knowing a few facts (attributes) about them:

```sql
SELECT freguesia, area_ha 
FROM  vectors.porto_freguesias
WHERE concelho LIKE 'MATOSINHOS';
```
**Example 6 - Using alias**

Aliases are a very common technique in expressing a query. An alias provides another name for a table on the fly as in the example below:

```sql
SELECT a.freguesia, a.area_ha 
FROM  vectors.porto_freguesias AS a
WHERE a.concelho LIKE 'MATOSINHOS';
```
Note that under the ``` FROM ``` clause we add the ```AS a ```, which is saying that on what this particular query concerns, the table we are calling will be known as 'a' and that is why you see an ```a. ``` prefix whenever the query is referring to rows or attributes that belong to table a. Aliases are very useful if your query is calling more than one table as you will soon see. 

----------

## Calling PostGIS spatial functions

**Example 7 - ST_Area**

Apart from the geometry columns, so far we have only been doing plain PostgreSQL. We will now start to explore some of the spatial functions offered by PostGIS. A PostGIS function usually takes the form ``` NameOfTheFunction(arguments/inputs) ``` Usually a spatial function of PostGIS starts with **ST_**. To demonstrate this principle, we will do a simple area calculation. In this example, the area will be in meters because the SRID uses meter as its unit.:

```sql
SELECT a.freguesia, a.area_ha, ST_Area(a.geom)--/10000)::int
FROM  vectors.porto_freguesias AS a
WHERE a.concelho = 'MATOSINHOS';
```
As you can see, the function **ST_Area** takes one argument - the geometry .

**Example 8 - ST_Buffer**

Here is another example. This time we call a function that outputs a geometry.

```'sql
SELECT id, ST_Buffer(geom, 1000)
FROM vectors.ferrovia;
```


**Example 9 - ST_Intersects**

A common spatial problem in GIS is to know if two features share space. There are some variants to this problem but we can use the following as a starting point:

```sql
SELECT b.geom, b.id
FROM vectors.porto_freguesias as a, vectors.ferrovia as b
WHERE a.concelho ilike 'MATOSINHOS' AND ST_Intersects(a.geom,b.geom);
```
If you load the results in QGIS, the result might not be exactly what you were expecting - you will get the railroad that intersects the parish of Muro but it is not clipped to the boundaries of the parish because this query only applies a logical test, it does not construct a new geometry. In other words, it returns the features that intersect with the parish of Muro without changing them.

**Example 10 - ST_Intersection**

To retrieve the actual geometry that represents the space shared by two geometries (like a Clip operation), we have to use the **ST_Intersection** function. In this example, we will get the railroads that intersect Matosinhos parish.

```sql
SELECT b.id, ST_Intersection(b.geom, a.geom) as geom
FROM vectors.porto_freguesias as a, vectors.ferrovia as b
WHERE a.concelho ilike 'MATOSINHOS' --AND ST_Intersects(a.geom,b.geom); 
```

Run the above query again, but this time uncomment the ``` AND ST_intersects(a.geom,b.geom); ```  by deleting the ```--``` characters and check the consumed time. 
When we run the  **ST_Intersection** we should always add the **ST_Intersects** in the were clause, this will make sure we are only computing the intersection were in fact the geometries intersect.

Robustness can be a big concern when computing intersections and other forms of new geometry.  This is especially so when (multi-)polygons are involved as is the case here.  The reason is that resulting geometries may hold any combination of polygons, lines and points, and thus become by type a heterogeneous collection, also known as GeometryCollection.  When you subsequently aim to store the obtained geometry in a table that expects only multipolygons, your database will complain and not allow.  Most of the time, you want to extract only the polygonic part of the result.  This is done by an extra function that is known by the name **ST_CollectionExtract(_, _)**.  Its first parameter is the geometry in question, the second parameter is a geometry type indication: numbers are 1 == POINT, 2 == LINESTRING, 3 == POLYGON.

----------


## Integration challenge

Time to put together what you have learned so far to solve a spatial problem. 

**Find all the places that are at a distance less than 300 m from a railroad**
*Hint*: you will have to nest the function ST_Intersects and ST_Buffer under the WHERE clause.

If you manged to solve it, try it with a small variation:
**Find all the places that are distanced less than 300m from a railroad AND are located within the municipality of Matosinhos**.

----------

## Working with dynamic data: views and triggers

**Example 11 - Create a view**

Views are essentially a stored query, which means you can visualize the result of a query at anytime you want without having to type the query again. This is especially useful for complex queries that have to be run frequently over data that is very dynamic (i.e. changes frequently).

To explore this concept  we will create a view from the solution to the second integration challenge:

```sql
CREATE VIEW YOURSCHEMA.close2railroad AS
SELECT a.name, a.geom
FROM vectors.lugares as a, vectors.ferrovia as b, vectors.porto_freguesias as c
WHERE st_intersects(a.geom, ST_Buffer(b.geom, 300)) and c.concelho ilike 'MATOSINHOS' and ST_Intersects(a.geom, c.geom);
```
You can now load your view into QGIS just like you would any other table. The interesting part however is that if the data behind the query changes, the view will also change accordingly.

Example: lets assume the table lugares is changing frequently. If new points are created closer than 300m to railroad in the municipality of Matosinhos, the view will show those new points without you having to re-run the query!


### Dealing with invalid geometries

Although not entirely, for the most part PostGIS complies with the OGC Simple Feature Access standard (http://www.opengeospatial.org/standards/sfa), and it is according to this technical recommendation that PostGIS expects to have the geometries. That is not to say that you cannot have invalid geometries in your database, but if you do some of the spatial functions, PostGIS might not work as expected and yield wrong results simply because PostGIS functions assume that your geometries are valid. Therefore you should ALWAYS make sure that your geometries have no errors. The importance of this check is even bigger when multiple users are using/editing the same table.

Since PostGIS version 2.0 there are functions to detect and repair invalid geometries.

**Example 12 - ST_IsValid**

 In order to find invalid geometries we can use ```ST_IsValid``` function.

```sql
SELECT a.freguesia, a.area_ha, a.geom
FROM  vectors.porto_freguesias AS a
WHERE NOT ST_IsValid(a.geom);
```

**Example 13 -  Simple approach to fix invalid geometries**

```ST_makevalid``` is the function that returns a corrected geometry.   One needs to be slightly careful with this function: in rare cases it does not resolve the problem and returns a null geometry instead.  That is a valid geometry but not a very useful geometry.  You need to prepare your tests for this.

```sql
CREATE TABLE YOURSCHEMA.my_freguesias AS
SELECT id, name, ST_buffer((ST_makevalid(geom)),0)) as geom
FROM vectors.porto_freguesias
WHERE NOT ST_IsValid(geom);
```

**Example 14 - A better, more complete, approach to fix invalid polygons**.

Although the previous example works well most of the times, in some cases polygons or multipolygons ```ST_makeValid``` might return points or lines. A solution for this is to use a buffer of 0 meters: 

```sql
DROP TABLE IF EXISTS YOURSCHEMA.my_freguesias;
CREATE TABLE YOURSCHEMA.my_freguesias AS
SELECT id, name, ST_buffer((ST_makevalid(geom)),0)) as geom
FROM vectors.porto_freguesias
WHERE NOT ST_IsValid(geom);
```

**Example 15 - And in case of multipolygons...**

Because ST_buffer returns a single polygon geometry, if we have a table of multipolygons we need to apply **ST_multi** function. This function transforms any single geometry into a MULTI* geometry. In this example, instead of creating a new table we will replace the invalid polygons by valid ones, using an UPDATE TABLE.

```sql
UPDATE vectors.porto_freguesias
SET geom=(ST_multi(ST_buffer((ST_makevalid(geom)),0)))
WHERE NOT ST_IsValid(geom);
```

----------

## Solutions for the integration challenges

```sql
SELECT a.name, a.geom
FROM vectors.lugares AS a, vectors.ferrovia AS b
WHERE st_intersects(a.geom, ST_Buffer(b.geom, 300));
```
```sql
SELECT a.name, a.geom
FROM vectors.lugares AS a, vectors.ferrovia AS b, vectors.porto_freguesias AS c
WHERE st_intersects(a.geom, ST_Buffer(b.geom, 300)) AND c.concelho LIKE 'MATOSINHOS' AND ST_Intersects(a.geom, c.geom);
```
