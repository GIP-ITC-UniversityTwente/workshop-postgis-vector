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
*server IP: gip.itc.utwente.nl
port: 5434
user: your s number
password: your regular UT password
database: c122*

----------

## Load data into the database

Assuming you are not accessing an already configured version of the database used in this workshop, you will start by creating a new empty database in your system, after which you will create the postgis extension and then import the following shapefiles: *porto_freguesias* ; *ferrovias* and *lugares*

As an alternative you can restore the *postgis_vectors.backup* database from this repository, this DB already has all the vectors imported.

Uploading spatial data is a bit special.  We wont work on raster data in this context, so will not explain that part.  Generally, ogr2ogr is quite capable, and if you run it with "--help" option, it will explain how it works.  Asking for a manual page with a web browser will also do.

To upload shapefiles into a postgis-enabled postgresql database, the cmmandline tool shape2pgsql is available.  It creates a *.sql file which you can subsequently load into your database with a call to psql.  This process is well-documented in the online PostGIS manual, and elsewhere.  Pay attention to the various command options.

----------


## Requirements

This workshop assumes a basic knowledge of SQL or *Structured Query Language* which is the language you use to interact with a PostgreSQL database (or any other relational database). If you are new to SQL we recommend you to follow the W3 Schools course on SQL: https://www.w3schools.com/sql/

## Explore the database with QGIS DB Manager

Please use your QGIS DB Manager to explore the vectors inside the schema vectors. It is important to know your data before we start analyzing it. Make sure that first you create a connection to the database as explained here: https://www.youtube.com/watch?v=r4pOyVZ3QVs 

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
CREATE VIEW close2railroad AS
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
CREATE TABLE my_freguesias AS
SELECT id, name, ST_buffer((ST_makevalid(geom)),0)) as geom
FROM vectors.porto_freguesias
WHERE NOT ST_IsValid(geom);
```

**Example 14 - A better, more complete, approach to fix invalid polygons**.

Although the previous example works well most of the times, in some cases polygons or multipolygons ```ST_makeValid``` might return points or lines. A solution for this is to use a buffer of 0 meters: 

```sql
DROP TABLE IF EXISTS my_freguesias;
CREATE TABLE my_freguesias AS
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
### Triggers

Triggers execute a given task whenever a specific event occurs in the database. This event can be anything that changes the state of your database - an insertion, a drop, an update. They are extremely useful not only to automate tasks but also to minimize the number of interactions between the users and the database (the source of many errors...). 
One useful and important example is a trigger that automatically fixes invalid geometries when a new row/feature is added to the table.

**Example 16 - Create a trigger that fixes invalid multipolygon geometries in real time.**

```sql
 -- First lets add function INVALID()

CREATE OR REPLACE FUNCTION invalid()
  RETURNS trigger AS
$BODY$
BEGIN
	if not st_isValid(NEW.geom) THEN
	NEW.geom = (ST_multi(ST_buffer((ST_makevalid(NEW.geom)),0))); 
	RETURN NEW;
 else    
      RETURN NEW;
    END IF;

END;
$BODY$
  LANGUAGE 'plpgsql' VOLATILE
  COST 100;

-- Now lets add trigger INVALID on our table 

DROP TRIGGER IF EXISTS trg_c_invalid ON vectors.porto_freguesias;
CREATE TRIGGER trg_c_invalid BEFORE INSERT OR UPDATE
ON vectors.porto_freguesias FOR EACH ROW EXECUTE PROCEDURE invalid();
```
Triggers are very important in production databases, but they can be quite complex. For more information about triggers please check the official documentation:
[https://www.postgresql.org/docs/current/static/plpgsql-trigger.html](https://www.postgresql.org/docs/current/static/plpgsql-trigger.html)

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
