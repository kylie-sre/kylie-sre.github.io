# JSON Columns and You
_Or How I Learned to Stop Worrying and Love SQL_

## Context

While working on [UBZR-715](https://jira.unity3d.com/browse/UBZR-715) I encountered some of our data stored as `JSON` columns in Postgres. This made things very complicated for that task. I am attempting to add as little custom code as possible to make our data accessible for our customers.

As we add customers and become more of a platform we can't only be reactive to their needs. We don't call up Google and ask them to add a new API for us every time we need to complete a new task. They provide a pretty nice toolbox for us to use to achieve our goals. By virtue of using their tools we get access to data for the services we use and any professional engineer using our platform will expect the same. These things have to "Just Work :tm_s:" *every time* and fewer moving parts helps us to achieve that. Ultimately I would like to be able to have a lot fo these exports and schemas generated off the same proto files we use for our APIs to reduce the amount of work we have to do.

## Disclaimer

To be totally clear up front, I don't necessarily think that every use of a `JSON` column is a "bad practice"; AFAICT there are a few niche uses for it where the downsides of the type aren't a big enough deal to be an issue. I don't think we need to stop using them all at once by any means. I think we are using them in a smart way in a few places that I saw (I'm also not intricately familiar with everything in the DB so I could be wrong.). /Disclaimer

## Overview

My experience with these columns so far is limited to attempting to export the data from the `products` table on the `metered-billing` DB instance in test and staging. I added the new config to export the data requested and during the initial import for the `products` table in encountered numerous errors (error, fix, new error, fix, new error... ad naseum) all specific to the `charges` and `child_components` columns; these are both `JSON` columns. Some of these issues were with the import in to Big Query and some were general data consistency and one interesting one regarding how Postgres evaluates a `JSON` column when it's value is used in a `WHERE` clause. Outside the niche cases mentioned below it's my opinion based on my time with SRE here at Unity and as an engineer at OfferUP (we had a lot of DB issues that I was involved in diagnosing/mediating) I don't think this column type is a safe way to store data in a production environment. I say this because while working with this data initially I noticed various ways that the data stored in these columns could potentially become inconsistent or accidentally be overwritten/deleted... then I found examples of those things happening in our test DB.

## The Niche Uses

### When the value you are storing is JSON and a black box

If there is just some user defined JSON that we need to store (e.g. user defined key/values pairs that are part of an object) and it is a black box to us that seems like a good use for the column type. The DB will ensure whatever we try to stick in the column is in fact valid JSON.

### When there is value in storing a raw event/request/response/etc with the rest of the data

I think there is value in storing these raw JSON encoded objects for auditing or data recovery purposes. Additionally, if it's possible the raw object might have fields that we don't store individually that could become important we would be able to back fill based off the stored raw event

## Issues Highlighted During Testing

### Minor: We are using the unparsed JSON type; This will be an efficiency loss eventually.

This is fairly minor but should presumably be as easy as a one line migration to remedy. There are two kinds of `JSON` columns, `JSON` and `JSONB`. We are using `JSON`. The difference being that in a `JSON` column the value that gets stored is the raw JSON string. With `JSONB` the parsed structure is stored; the result is writes are slightly slower but read are much faster.

> There are two JSON data types: json and jsonb. They accept almost identical sets of values as input. The major practical difference is one of efficiency. The json data type stores an exact copy of the input text, which processing functions must reparse on each execution; while jsonb data is stored in a decomposed binary format that makes it slightly slower to input due to added conversion overhead, but significantly faster to process, since no reparsing is needed. jsonb also supports indexing, which can be a significant advantage.

_source_: https://www.postgresql.org/docs/10/datatype-json.html

### The way Postgres evaluates the column in a query is a bit of a gotcha if you're not aware of it.

This isn't really an issue with the column as much as it that it works a bit different and that could cause confusion. From what I can tell when Postgres is evaluating these columns it loads whatever the value is into a `JSONB` object. I encountered an error during one import attempt because the field `child_components` (`JSON` column) was `null`. First I added a call to `coalesce(child_components, '[]'::json)` in my query but that failed for the same reason. When checking the DB to find rows that had a `null` where there should have been an array I tried to use the following query:

```postgresql
SELECT id FROM products WHERE child_components is null;
id
----
    (0 rows)
```

This returned 0 rows... which shouldn't be the case since there are `null` values for that column if I select all the rows and fields. After finding one of the PKs of a row that had a null I ran this query:

```postgresql
SELECT coalesce(child_components, '[]'::json) as child_components FROM products WHERE id = '8f984c8a-7c92-4655-ad93-8f6b1682f6ad';
 child_components
------------------
(0 rows)
```

Okay... that shouldn't be happening. From the Postgres documentation:

> The `COALESCE` function returns the first of its arguments that is not `null`. Null is returned only if all arguments are null.

An empty array certainly isn't null so why is this happening? From what I can tell when those rows were stored, either initially or on an update, there was a `nil` value in the Go struct for the pointer to the `json.RawMessage`. When the `nil` was marshalled to JSON something, `gorm` most likely, sent the value as a  single `null`; this is valid JSON:

````
$ python3
Python 3.8.2 (default, Dec 21 2020, 15:06:04)
[Clang 12.0.0 (clang-1200.0.32.29)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import json
>>> js = "null"
>>> x = json.loads(js)
>>> x
>>> x is None
True
````

So my assumption is that since there is valid JSON stored for the column even though the value that is stored is `null` that Postgres considers that parsed JSON as a non-null value. This is correct since it's not the business of the database to start poking around in your JSON; there is a valid value for it to store -- that value just happens to be null. Represented as Go this would look like:

```
type JSONColumn struct {
    Value *json.RawMessage
}

col := JSONColumn{Value: nil}
fmt.Println(col == nil)  // false
fmt.Println(col.Value == nil)   // true
```

The query I found that worked to replace `null` with `[]` in the query results (there are probably better ways:

```sql
select
	   row_to_json(row, false)::text  -- Reason for this is explained below
from (
	SELECT
		   CAST(id AS TEXT) AS product_id,
		   service_id,
		   name,
		   sku,
           (case when charges->0 is null then '[]'::text else charges::text end) as charges,
           (case when child_components->0 is null then '[]'::text else child_components::text end) as child_components
	FROM
		 products
	) as row
;
```

This query has its own problems but, it solved my issue which brings me to the next one...

### Due to the way column nullness is checked you lose a lot of the safety provided by a SQL database

Hopefully these rows just never had any data except a nil pointer but if they did we were able to silently hard delete a bunch of data from our DB by accident. What does Postgres consider a non-null value? According to the results of the above queries it didn't think a `JSON` column that contained a `null` was actually null. Knowing that we also know that if a column is required that the DB schema validation process could allow us to delete a lot of data by accident. For example, what happens if I select a single row but don't need that column so i exclude it from the query. When the data is hydrated to the `gorm` struct the `child_components` field wil be a `nil` pointer of the type `Json.RawMessage`. What happens when that struct has it's `Save()` method called? Will it overwrite a column potentially full of JSON (the array contained in Multiplays `child_components` column has 89 elements in it for example) silently?

A huge reason to use a SQL database is the built-in data validation it provides. If you mark a column as required you can't insert a row that is missing a value for that column. We lose that here.

### It doesn't play nice with the Cloud SQL Admin export method w/o some special considerations

```sql
row_to_json(row, false)::text  -- Reason for this is explained... here actually
```

The reason for the call to `row_to_json()` is that on my initial attempts the bigquery importer would choke on a bunch of the random wad of JSON in the middle of a CSV file. So there were A LOT of random text parsing issues around quotes and commas and other arcana. Eventually I came to the conclusion that you couldn't easily use the CSV format. Big Query can also import JSON so why not just export the data in JSON format? Well for one the Cloud SQL Admin API doesn't support that; you can ask for a CSV file or a SQL dump. But we can just turn the whole row to JSON! Sort of! As far as I could tell this was the command that Google was using to do the export:

```postgresql
\copy ($QUERY) to "gs://..." CSV
```

The issue with that is the CSV parser isn't a JSON parser so this results in some wonky JSON that looks like this:

```
"[{""key"":""value""}, ...]"
```

In order to fix that there was some tweaking to the CSV output options that needed to be done -- chanigng the escape and quote characters to be ` ` (ASCII space, %20). That worked but feels gross and could break.


### Less flexibility in what data gets returned from queries

If we are selecting those columns by default you get the whole thing. If there are fields in the JSON that you don't want to be returned by the query you get to basically figure out a way to rebuild the object with the keys you don't want in it. Postgres provides operators and functions for doing this stuff but, given the very deeply nested structure in the `child_components` column if you want to remove a key from a object 2 or 3 levels down your query just a lot more complicated than selecting related records (most ORMs make this pretty easy). Here's an example of that data:

```json
{
  // level 0: child_components
  "child_components": [
    // level 1: child_components[0]
    {
      "name": "Unity  Game  Simulation  Minutes",
      "sku": "UTY-GAME-SIM-MIN",
      "service_id": "GameSim",
      "description": "Unity  Game  Simulation:  Simulation  Minutes",
      // level 2: child_components[0].charges
      "charges": [
        // level 3: child_components[0].charges[0]
        {
          "name": "Unity  Game  Simulation  Minutes",
          "type": "Usage",
          "model": "Tiered",
          "unit_of_measure": "GameSim-simulation-minutes",
          // level: 5 child_components[0].charges[0].prices
          "prices": [
            // level 6 child_components[0].charges[0].prices[0]
            {
              "currency": "USD",
              "amount": "0",
              // level 7: child_components[0].charges[0].prices[0].tiers
              "tiers": [
                // level 8: child_components[0].charges[0].prices[0].tiers[0]
                {
                  // level 9: child_components[0].charges[0].prices[0].tiers[0].tier
                  "tier": 1,
                  "starting_unit": "0",
                  "ending_unit": "0",
                  "price": "0.01667",
                  "price_format": "PerUnit"
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

### Major issues with data consistency and integrity

This is what me decide these columns were not safe for us to use. I thought of some edge cases and bugs I have seen in the past regarding unstructured data like this (believe it or not before JSON columns people would just stick JSON in there as strings). There are lots of things that can go a bit weird or unexpected when doing Go JSON marshalling especially around scoping inside loops and closures. These usually happen when a strange set of circumstances align to do something you don't want. I figured these would be good to point out but them I saw evidence of them already happening in our DB:

*Multiplay metadata in Cloud Build rows*

```
{
	"product_id": "15b79dcc-8956-4ba8-b669-37d09fe23a31",
	"service_id": "CCD",
	"name": "Cloud  Content  Delivery",
	"child_components": [
		{
			"name": "Cloud  Content  Delivery  -  Bandwidth",
			"sku": "UTY-CLD-CNT-BAN",
			"service_id": "CCD",
			"description": "Cloud  Content  Delivery:  Download  delivery  in  gigabytes",
			"charges": [...],
			"metadata": {
				"MultiplayLocationId": "",
				"MultiplayLocationName": "",
				"MultiplayRegion": "",
				"MultiplayType": ""
			}
		}
	]
}
```

*Inconsistent structure of data*

```
{
	"product_id": "15b79dcc-8956-4ba8-b669-37d09fe23a31",
	"service_id": "CCD",
	"child_components": [
		{
		    // did they have metadata originally?
			"metadata": {
				"MultiplayLocationId": "",
				"MultiplayLocationName": "",
				"MultiplayRegion": "",
				"MultiplayType": ""
			}
		}
	]
}
{
	"product_id": "8b5300f4-4a6c-4e5f-9628-6ad227da51ef",
	"service_id": "Vivox",
	"name": "Vivox  Voice  Services  Standard",
	"sku": "VIV-VOI-SER-STD",
	"charges": [],
	// these were originally returned as null; anything processing this JSON down stream would have to guard itself from us
	"child_components": []
}
{
	"product_id": "3760894d-efe8-4820-bc2c-45edb2d603d3",
	"service_id": "Plastic",
	"child_components": [
		{
		    // no metadata and missing some keys compared to other components
			"name": "Plastic  SCM  Cloud  Edition  -  Users",
			"sku": "PLA-CLD-USR",
			"service_id": "Plastic",
			"charges": [...]
		},
	]
}
{
	"product_id": "580c2b7e-bbee-43bf-ab55-0055c8471382",
	"service_id": "MP",
	"child_components": [
		{
			"name": "Multiplay  RAM",
			"sku": "MP-RAM",
			"description": "2.0",
			"charges": [...]
		},
	]
}
```

There are a lot of other scenarios I can think of that we would have to guard ourselves and our downstream users from by storing data with this method. Keep in mind we can't edit any of this JSON via migrations. If there is a value in that JSON that needs to be removed we can't just drop a column we have to check all the JSON to make sure it's not in there anymore. Any code processing this JSON would have to employ a pretty deeply layered "look before you leap" approach adding a bunch of logic and branching; places where things can break.


### Disk performance and data fragmentation

Adding this one at the end since I would have to confirm this but, I have a concern that over time the varying size of each JSON blob could lead to excessive fragmentation of the data on the DBs disk and lead to slower queries. There are rows in our DB with nothing in that column and there are rows like Multiplay that have hundreds of elements in the components column with each one having dozens of elements. When DB allocates the disk space for each row it will try to put all the data together on disk so that it is easier to fetch all together. Usually this is pretty efficient for the DB since SQL columns are statically typed; most of the types have a fixed maximum value. The DB knows the theoretical max size of the row so it can reserve all of it ahead of time to avoid having to shuffle things around later. I assume with variable nature of a `JSON` column that the DB stores essentially a pointer to the actual data. this would allow it to use its max size method to colocate all the other columns. Since the JSON data can vary greatly in bits of space needed to store it this means there are either large blocks of allocated but empty space on the disk (if the biggest row needs 1kb of disk space for its json then the DB has to assume that all of them can potentially be at least that big) or data that is frequently shuffled around the disk (if it doesn't allocate room for the value to grow if it does grow then it has to move everything else around as well). New rows with a large JSON structure could cause everything to need to be shuffled.

All that stuff is managed by the DB of course (there are some knobs we can turn tho) but regardless of how automatic it is it's still disk i/o; one of the major bottlenecks for a database (major enough that there is in fact a whole class of databases that simply avoid the problem by doing it all in memory e.g. Redis). Over time, we will add more data and more tables and that increases the amount work of this nature it does. On its own this would be unlikely to be an issue but when combined with other issues that could crop up it could have a non-trivial impact on the performance of the disk.

### This isn't unstructured data and already has schema

The structure of these values is defined in the product proto file. We have a schema for them, we amy as well use it.

## Conclusion

Personally I don't see enough value provided by using this method of storing data in these columns to justify the dangers and annoyances around using them. I will admit I could be wrong on this... I've only been poking around on this for a few days and this is the first time I've used these columns. I am basing my opinion on my understanding of the Postgres docs for the data type and working with trying to get in to a usable format for our customers. I think this introduces a place for special processing to be needed on a per column level for these (and future) data exports and don't want to introduce that over head to that process if we can avoid it.
