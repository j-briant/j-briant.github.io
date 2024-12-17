---
title: FME is getting expensive? Stop using it
date: 2024-08-16 10:30:00
categories: [Discussion]
tags: [gis, data, proprietary, price, fme, alternative, license, python, geospatial, postgresql, expensive, transform, digital, sovereignty]
---

I have trouble finding what happened at Safe Software at the beginning of 2024, but it seems that part of the company has been [sold to an equity firm](https://betakit.com/us-based-jmi-equity-buys-out-safe-software-co-founder-in-strategic-investment/). No comment on that, but since then licenses prices seem to have taken a hit and customers are.. Well not very happy, especially regarding the communication around this increase:

* [Views on recent FME price changes](https://community.safe.com/integration-8/views-on-recent-fme-price-changes-34403)
* [Safe is an unreliable partner](https://www.reddit.com/r/fme/comments/1e1bijf/safe_is_an_unreliable_partner/)
* [Is a small services consultancy using FME still viable? Sensible?](https://www.reddit.com/r/fme/comments/13jl6dj/is_a_small_services_consultancy_using_fme_still/)
* [[...]rumors of an extreme price jump[]...](https://www.reddit.com/r/gis/comments/12trnzg/i_am_hearing_rumors_of_an_extreme_price_jump_for/)

Costs become unbearable for some companies and the look for alternatives is now a necessity. What can we do about it?

## About FME and Safe

In itself I don't have anything against FME, it does its job and if it allows more people to work with spatial data and create workflows then it's cool. But there are things that I find, if not unacceptable, at least not very wise, and it's not always Safe responsibility.

### Safe as a company

I never had to negotiate any license or contract with Safe, but people around me always seemed happy about their relation with their resellers and the flexibility that was allowed with their licenses. There was no problem having a home/student license for free if you wanted to fiddle around with FME and prices were rather stable.

The rebranding and the sell at the beginning of 2024 has changed things, and some of the trust seems to be lost. The big problem is the way the company communicated the changes, which was apparently by giving no clear information. Increasing prices (reasonably) could hardly be considered a problem, that's the world we've chosen, but at least having some transparency about it maintains the relation between seller and buyer. If you receive a x3 bill with a "go f"ck yourself if not happy", this is at best disrespectful, at worst belittling, especially if you can't act against this. And to me, **the best answer to give is to gain independence and stop buying**.

### FME as a software

As I said, I don't have anything against FME itself. It makes workflows easy to create and push to production, (spatial) data processing accessible to a broader audience and gives you time to focus on something else if you have a GIS role in your company (see [On GIS and their integration](https://www.geothings.ch/posts/on-gis/) for that part of the problem). But making yourself totally dependant on a proprietary solution, with a near monopoly on the field, **IS** a big problem, and honestly felt like a matter of time before it would happen. The situation is so problematic that some high level administrations (country level) are now negotiating terms with Safe because they are so dependant on FME that they can't move easily from it, and will have no choice but to take the price hit if they don't act. And maybe it's common practice, I wouldn't say I have enough experience to understand those kinds of negotiations, but it feels wrong. 

That's were **digital sovereignty** enter the game, regaining control over our processes, data, and costs. Some links :

* [Digital Sovereignty in Switzerland : the laboratory of federalism](https://access.archive-ouverte.unige.ch/access/metadata/bbf62d17-891e-46e4-bde8-5666dda99237/download)
* [Framasoft](https://framasoft.org/en/)

If you work extensively with FME, I'd advise to take a step back and and have a look at what you need and how you could achieve this any other way, because most of the time, there are options, but we just don't see them anymore. FME has become such an automatism that everything else is a blur, and worst, we lose skills and control.

## What alternatives are there?

The first step is to have a look at what you need to achieve. Let's see how FME defines itself from their website:

* Expansive Data Solutions &rarr; well it doesn't mean anything so next.
* Data Transformation &rarr; Ok noted
* Entreprise Automation &rarr; Ok noted

So we get data transformation and automation. Good.

### Read, Write and Data transformation

Let's start with the main reason for all of this. We read, transform and write data to solve problems and answer questions. Data need to be shaped into different models and linked with other data to bring additional knowledge. But how do we do that? 

#### Transform with SQL 

If you're working with databases (PostgreSQL, MariaDB or even Oracle), you're probably aware of their spatial capabilities. To me, SQL is a major skill, you can do **a lot** with a single SQL query. There is even a chance that your FME workspace can be entirely replaced by a single SQL query. 

Now we want to Extract, Transform and Load data. Hum but wait, who said you can't "transform" your data before even reading it? SQL allows you to do that; write your query, save it as a *view* if you wish, and you have your result ready to be read, in sync with your tables.

Let's say you have *regions* (polygons), *roads* (lines) and *bus stops* (points), and you want:
* *road* cut into portions by *region*
* *road* portions length
* to count *bus stops* on each *road* portion
    * if a *road* has no *bus stop* you want to have "No bus stop" instead of 0
* an attribute that says "Hopp Schwiiz !" if the *road* portion is located in a *region* where attribute *name* is equal to "Switzerland", else "Oh, no Hopp Schwiitz...". 

You could have a FME workspace chaining writers, a clipper, a length calculator, aggregators, attributes managers etc. Or a single SQL query in the following manner (optimization not included):

```sql
SELECT 
    portion.name,
    CASE COUNT(b.id) 
        WHEN 0 THEN 'No bus stop'
        ELSE COUNT(b.id)::text
    END AS n_bus_stop,
    CASE portion.name
        WHEN 'Switzerland' THEN 'Hopp Schwiiz !'
        ELSE 'Oh, no Hopp Schwiitz...'
    END AS hop_suisse,
    St_Length(portion.geom) AS portion_length,
    portion.geom
FROM (SELECT r.name, St_Intersection(l.geom, r.geom) AS geom
        FROM road l
        JOIN region r ON St_Intersects(l.geom, r.geom)
        ) portion
LEFT JOIN bus_stop b ON St_Intersects(portion.geom, b.geom)
GROUP BY portion.name, portion.geom
```

And that's it, you can read this data, export it, do whatever you want with it. Everything done within the query is visible at once, and if it takes some time to get used to the syntax and be comfortable with it, it becomes much easier to understand what happens in a well written SQL query than in the tidiest FME workspace (personal opinion included).

Another huge advantage of working with SQL is that you're closer to a standard **interoperable** format (with a grain of salt since we use Postgis here, which is Postgres specific). If you ever need to change your system, the required effort to make the conversion will be minimal compared to translating a FME workspace.

You can import other data into your database to use it in your query, or use [Foreign Data Wrappers](https://www.postgresql.org/docs/current/postgres-fdw.html) (FDW) to read data from another database. You can combine your queries with tools such as [dbt](https://docs.getdbt.com/) (Core) to have a more complete environment, or build your own processes, with GDAL or using the programming language of your choice.

#### Read and write with GDAL

Back at it with [GDAL](https://www.geothings.ch/posts/gdal-appreciation/), but it's so versatile that it has to be in your toolkit. Your can read and write basically any format, and apply transformations on the fly.

The series of command line tools that come with GDAL/OGR make it very complete and easy to use with raster or vector data.

 You could reuse your previous SQL query with GDAL to extract your data and save it in GeoPackage for example. I could just save my query into a file named `query_file.sql` and run the following command:

```sh
ogr2ogr -f GPKG -sql @query_file.sql output.gpkg "PG:service=input_db"
```

Remember **interoperability**? Well that's a nice example, have one standard way of solving a problem and you can connect it with other without changing anything. **You have many possibilities**.

#### Do everything with (Python) libraries

There are a bunch of libraries allowing you to read, transform and write data, using Python but not only. GDAL (omfg again) is also available in many languages through its bindings. If we focus on Python, a list (non-exhaustive of course) of libraries for working with spatial data:

* [Shapely](https://shapely.readthedocs.io/en/stable/manual.html)
* [geopandas](https://geopandas.org/en/stable/index.html)
* [Fiona](https://fiona.readthedocs.io/en/latest/index.html)
* [xarray-spatial](https://xarray-spatial.readthedocs.io/en/latest/#)
* [rasterio](https://rasterio.readthedocs.io/en/latest/index.html#)

Let's try somethings with `geopandas`. If we consider that we are working with a single PostgreSQL database, we could reuse our earlier query to read the data and save it as a GeoPackage:

```python
import geopandas as gpd
from sqlalchemy import create_engine


def run():
    # Connection
    db_connection_url = "postgresql://username:password@localhost/dbname"
    con = create_engine(db_connection_url)

    # Open query file, apply and save
    with open("query.sql") as f:
        query = "".join(f.readlines())
        df = gpd.read_postgis(query, con)
        df.to_file("my_analysis.gpkg", layer="result", driver="GPKG")

        # Print the head for good measure
        print(df.head)


if __name__ == "__main__":
    run()

```

Will create a new file with your result saved in the `result` layer. It should also print something like:

```
0       France            2  Oh, no Hopp Schwiitz...      823.126128  LINESTRING (2537525.502 1152634.769, 2537254.4...
1       France            1  Oh, no Hopp Schwiitz...      118.753415  LINESTRING (2537290.714 1152264.433, 2537281.7...
2        Italy  No bus stop  Oh, no Hopp Schwiitz...      686.494161  LINESTRING (2538145.102 1151496.452, 2538283.1...
3  Switzerland  No bus stop           Hopp Schwiiz !      123.440260  LINESTRING (2536930.949 1152095.427, 2536884.0...
4  Switzerland            1           Hopp Schwiiz !     1102.779141  LINESTRING (2537281.727 1152146.02, 2537249.56...
5  Switzerland            1           Hopp Schwiiz !      206.632320  LINESTRING (2538019.283 1151332.542, 2538145.1...
6  Switzerland  No bus stop           Hopp Schwiiz !     1207.518986  LINESTRING (2537677.993 1151998.178, 2537874.0...
```

That was easy enough but we didn't bring much since everything is done with SQL again. Let's do the same thing but only using `geopandas` capabilities:

```python
import geopandas as gpd
from sqlalchemy import create_engine


def run():
    # Connection
    db_connection_url = "postgresql://username:password@localhost/dbname"
    con = create_engine(db_connection_url)

    # Read roads, regions and bus_stops
    roads = gpd.read_postgis("road", con)
    regions = gpd.read_postgis("region", con)
    bus_stops = gpd.read_postgis("bus_stop", con)

    # Intersection
    intersection = roads.overlay(regions, how="intersection")

    # Intersect bus stops
    intersect = intersection.sjoin(bus_stops, how="left", predicate="intersects")

    # Group by
    groupby = intersect.groupby(["name", "geometry"]).id.count()
    output = groupby.reset_index(name="n_bus_stop")

    # Hop Suisse
    output["hop_suisse"] = "Hopp Schwiiz !"
    output.loc[output["name"] != "Switzerland", "hop_suisse"] = (
        "Oh, no Hopp Schwiitz..."
    )

    # No bus stop
    output.loc[output["n_bus_stop"] == 0, "hop_suisse"] = "No bus stop"

    # Lost the Geo from GeoDataFrame at some point 
    output = gpd.GeoDataFrame(output, geometry="geometry")

    # Length
    output["portion_length"] = output["geometry"].length

    output.to_file("my_analysis.gpkg", layer="result_only_gpd", driver="GPKG")


if __name__ == "__main__":
    run()

```

Doesn't take much more. `Roads`, `regions` and `bus_stops` could have been data from anywhere: another database, a csv file, a cloud hosted file, etc. 

Working with a programming language allows you to easily read multiple data sources and combine them using your custom "transformers", which will basically be functions that you would have written and tailored to your need.

But now let's see what you can do to automate your process.

### Automation

Not sure about what is so incredible about giving scheduling capabilities within FME when [`cron`](https://en.wikipedia.org/wiki/Cron) is available by default in many Linux distributions. If you're working on Windows you can also schedule things with the `Task Scheduler`.

I feel like I'm oversimplifying things but I can't see where. If you're on Linux, you can schedule your data export task by adding such lines into your `crontab` (`crontab -e` to edit it):

```sh
# Everyday at 05:00
00 5 * * *  ogr2ogr -f GPKG -sql @query_file.sql output.gpkg "PG:service=input_db"
```

Run the task and send an email if it failed?

```sh
00 5 * * *  if ! ogr2ogr -f GPKG -sql @query_file.sql output.gpkg "PG:service=input_db"; then echo "your export failed" | mail -s "failing process" address@mail.me; fi
```

And that's it. I doubt that FME brings so much on the table when it comes to scheduling tasks.

### And that's it?

I guess not, but I don't think we will find many operations that are totally unfeasible without FME. If I'm missing something, please don't hesitate to correct me.

## Limitations

Unfortunately, there are some things that I don't know how to do without FME, well at least one that I have no idea how to solve.

### DWG and the closed formats problem

DWGs are ***** everywhere when your have people doing CAD around you. They could use DXF, which is an open source equivalent, but no, AutoCAD then DWG (which is no data btw, it's just lines ffs). And I have no clue how to tackle that, except through a long process of mentality change around the tools used.

Earlier we saw that using standard formats can start a virtuous cycle, where you keep open the door for many solution to solve your problems. Well here we have the opposite, closed formats reduce the interoperability by making a handful of solutions your only options, and keep you from trying other things. If you need to have DWGs, then to my knowledge, FME becomes the only tool you can use to produce them programmatically from other data (even GDAL can't all hope is lost). So from one license, you need "automatically" two.

And then one company decide to increase there prices, and there is nothing you can do about it.

In this case, the problem is to use a closed data format. To change that is to change deeply the habits of people using the tool creating this format, because, like with FME, there is a real addiction to the software. The skill becomes the software; not the technic, not the knowledge around the field, but just the software.

### And that's it for now

Fortunately, FME is not that essential. Maybe you won't need to pay for the license for long... But I'm curious to read about what makes it a must-have for you. **What can FME do that other tools cannot? Why is it necessary to you?**

## Conclusion

Feels good to rant a bit. To summarize:

* Learn SQL
* Don't be afraid of command lines and programming languages
* Have interoperability in mind
* Get away from closed data formats

If want ot talk about all of this or if you need help converting some processes from FME to more open solutions, hit me up on [Mastodon](https://tooting.ch/@jln_brnt) or [Linkedin](https://www.linkedin.com/in/julien-briant/)
