---
title: FME is getting expensive? Stop using it
date: 2024-08-16 10:30:00
categories: [Discussion, GIS, Data]
tags: [gis, data, proprietary, price, fme, licence]
---

I have trouble finding what happenend at Safe Software at the beginning of 2024, but it seems that part of the company has been [sold to an equity firm](https://betakit.com/us-based-jmi-equity-buys-out-safe-software-co-founder-in-strategic-investment/). No comment on that, but since then licences prices seem to have taken a hit and customers are.. Well not very happy, especially regarding the communication around this increase:

* [Views on recent FME price changes](https://community.safe.com/integration-8/views-on-recent-fme-price-changes-34403)
* [Safe is an unreliable partner](https://www.reddit.com/r/fme/comments/1e1bijf/safe_is_an_unreliable_partner/)
* [Is a small services consultancy using FME still viable? Sensible?](https://www.reddit.com/r/fme/comments/13jl6dj/is_a_small_services_consultancy_using_fme_still/)
* [[...]rumors of an extreme price jump[]...](https://www.reddit.com/r/gis/comments/12trnzg/i_am_hearing_rumors_of_an_extreme_price_jump_for/)

Costs become unbearable for some companies and the look for alternatives is now a necessity. What can we do about it?

## About FME and Safe

In itself I don't have anything against FME, it does its job and if it allows more people to work with spatial data and create workflows then it's cool. But there are things that I find, if not unacceptable, at least not very wise, and it's not always Safe responsibility.

### Safe as a company

I never had to negociate any licence or contract with Safe, but people around me always seemed happy about their relation with their resellers and the flexibility that was allowed with their licences. There was no problem having a home/student licence for free if you wanted to fiddle around with FME and prices were rather stable.

The rebranding and the sell at the beginning of 2024 has changed things, and some of the trust seems to be lost. The big problem is the way the company communicated the changes, which was apparently by giving no clear information. Increasing prices (reasonnably) could hardly be considered a problem, that's the world we've chosen, but at least having some transparency about it maintains the relation between seller and buyer. If you receive a x3 bill with a "go f"ck yourself if not happy", this is at best direspectful, at worst belittling, especially if you can't act against this. And to me, **the best answer to give is to gain independance and stop buying**.

### FME as a software

As I said, I don't have anything against FME itself. It makes workflows easy to create and push to production, (spatial) data processing accessible to a broader audience and gives you time to focus on something else if you have a GIS role in your company (see [On GIS and their integration](https://www.geothings.ch/posts/on-gis/) for that part of the problem). But making yourself totally dependant on a proprietary solution, with a near monopoly on the field, **IS** a big problem, and honestly felt like a matter of time before it would happen. The situation is so problematic that some high level administrations (country level) are now negociating terms with Safe because they are so dependant on FME that they can't move easily from it, and will have no choice but to take the price hit if they don't act. That's were **digital sovereignty** enter the game, regaining control over our processes, data, and costs. Some links :

* [Digital Sovereignty in Switzerland : the laboratory of federalism](https://access.archive-ouverte.unige.ch/access/metadata/bbf62d17-891e-46e4-bde8-5666dda99237/download)
* [Framasoft](https://framasoft.org/en/)

If you work extensively with FME, I'd advise to take a step back and and have a look at what you need and how you could achieve this any other way, because most of the time, there are options, but we just don't see them anymore. FME has become such an automatism that everything else is a blur, and worst, we lose skills and control.

## What alternatives are there?

The first step is to have a look at what you need to achieve. Let's see how FME defines itself:

* Expansive Data Solutions &rarr; well it doesn't mean anything so next.
* Data Transformation &rarr; Ok noted
* Entreprise Automation &rarr; Ok noted

So we get data transformation and automation. Good. Now let's add data reading and writing capabilities because otherwise we don't have much to work with, and it's part of the ETL definition (2/3 of it actually).

### Data transformation

Let's start with the main reason for all of this. We do data transformation to answer needs and questions. Data need to be shaped into different models, linked with other data, to bring additional knowledge. But how do we do that? 

#### SQL 

If you're working with databases (PostgreSQL or even Oracle), you're probably aware of their spatial capabilities. To me, SQL is a major skill, you can do **a lot** with a single SQL query. There is even a chance that your FME workspace can be entirely replaced by a single SQL query. Who said you can't "transform" your data before even reading it? Write your query, save it as a *view*, and you have your result ready to be read, in sync with your tables.

Let's say you have *regions* (polygons), *roads* (lines) and *bus stops* (points), and you want:
* *road* portions cut by *region*
* *road* portions length
* to count *bus stops* on each *road* portion
    * if a *road* has no *bus stop* you want to have "No bus stop" instead of a count
* an attribute that says "Hopp Schwiiz !" if the *road* portion is located in a *region* where attribute *name* is equal to "Switzerland", else "Oh, no Hopp Schwiitz...". 

You could have a FME workspace chaining writers, a clipper, a length calculator, aggregators, attributes managers etc. Or a single SQL query in the form as follow (optimization not included):

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

You can combine your queries with tools such as [dbt](https://docs.getdbt.com/) (Core) to have a more complete environment, or build your own processes, with GDAL or using the programming language of your choice for example.

#### GDAL

Back at it with [GDAL](https://www.geothings.ch/posts/gdal-appreciation/), but it's so versatile that it has to be in your toolkit. Your can read and write basically any format, and apply transformations on the fly.

The series of command line tools that come with GDAL/OGR make it very complete and easy to use with raster or vector data.

 You could reuse your previous SQL query with GDAL to extract your data and save it in GeoPackage for example, I could just save my query into a file named `query_file.sql` and run the following command:

```sh
ogr2ogr -f GPKG -sql @query_file.sql output.gpkg "PG:service=input_db"
```

#### (Python) librairies

There are a bunch of libraries allowing you to read, transform and write data, using Python but not only. GDAL (omfg again) is also available in many languages through its bindings. If we focus on Python, a list (non-exhaustive of course) of libraries for working with spatial data:

* [Shapely](https://shapely.readthedocs.io/en/stable/manual.html)
* [geopandas](https://geopandas.org/en/stable/index.html)
* [Fiona](https://fiona.readthedocs.io/en/latest/index.html)
* [xarray-spatial](https://xarray-spatial.readthedocs.io/en/latest/#)
* [rasterio](https://rasterio.readthedocs.io/en/latest/index.html#)

Working with a programming language allows you to easily read multiple data sources and combine them using your custom "transformers", which will basically be functions that you would have written and tailored to your need.

### Automation

Not sure about what is so incredible about giving scheduling capabilities within FME when `cron` is available by default in many Linux distributions. If you're working on Windows you can also schedule things with the Task Scheduler.

I feel like I'm oversimplifying things but I can't see where. If your on Linux, you can schedule your data export task by adding such lines into your `crontab` (`crontab -e` to edit it):

```sh
# Everyday at 05:00
00 5 * * *  ogr2ogr -f GPKG -sql @query_file.sql output.gpkg "PG:service=input_db"
```

Run the task and send an email if it failed?

```sh
00 5 * * *  if ! ogr2ogr -f GPKG -sql @query_file.sql output.gpkg "PG:service=input_db"; then echo "your export failed" | mail -s "failing process" address@mail.me; fi
```

And that's it. I doubt that FME brings so much on the table when it comes to scheduling tasks.


##


