---
title: External SQL Connections In Big Query 
type: article
url: https://kylie-sre.github.io/2022/03/03/external-connections-bigquery.html
image: https://kylie-sre.github.io/images/sabotabby.png
---

# External SQL Connections In Big Query

## Context

While doing more work to expose/export data for UGS one of the tools I used was an [External Data Source](https://cloud.google.com/bigquery/external-data-sources); specifically the [Cloud SQL](https://cloud.google.com/bigquery/docs/cloud-sql-federated-queries) variety. This allows the user to issue a query directly to a SQL database via the Big Query API. This is a very expedient way to get an external user access to our data but has some drawbacks

## TL; DR

They do the job well but have a few drawbacks. They will be a good tool for keeping progress moving while we develop better solutions for customers

## The Benefits 😺

### It's quick and relatively easy to set up

It's a simple process to create the connection using the Cloud Console. It's a handful of fields and the correct values are easy to find. 

### It's just a piece of configuration; this makes it pretty resilient

As far as I can tell it isn't creating any actual new infrastructure. It appears to just be an entry in a config somewhere that allows a given service account to get a response when using the [`EXTERNAL_QUERY(connection_id string, query string)`](https://cloud.google.com/bigquery/docs/cloud-sql-federated-queries) SQL function. Provided nothing happens to the database it grants access to it shouldn't require any thought once setup.

### Users write their own queries so once it's configured it's self-service

Once it's configured we aren't on the hook for updating anything should the data model in the DB change; we wouldn't have to add new fields to an export for example. The user can pull whatever data they need directly from the DB and process it further from that point without requiring support from us.

### Easy to upgrade

The consuming customer doesn't have to write any custom code to interact with this connection; it's likely that in the event we make the data accessible via another method that the same query could be used -- hopefully limiting disruption for the customer.  

## The Drawbacks 😿

### Requires giving external service accounts a very broad set of permissions

The role suggested in the documentation is `BigQuery Admin`. We might be able to find a less permissive set of permissions but since the connection doesn't appear to be an actual resource we also can't target the permissions to a specific resource. The external service account would have that broad set of permissions for all of our Big Query resources.

### There is some configuration that is hidden

The connection uses an existing SQL user and password to connect to the database. This on its own isn't an issue, but it does mean that we need to be careful about what user gets used for those connections. If a user with a lot of permissions like the `postgres` user was used then those external connections might be able to write back to our DB. In order to prevent this I used the migration: 

```postgresql
GRANT
    select
ON
    products,
    entitlements
TO
    "ugs-ro"
;
```

This ensures that the user that gets used for the connection can only perform `SELECT` statements on a subset of tables.

### No Terraform/API support

There does not appear to be support for creating these in the Google Terraform provider and Google does not expose an API to use either. They can only be created in the console. It isn't a difficult task, but it can't be automated as of time of writing.

## Conclusion 😼

As a tool external connections are a great way to expose our data to users in a pretty simple to follow process. If there is external user/team that needs access to our data this is a good way to give them access quickly so that they aren't blocked while we work with them to develop a more robust solution. This also gives us the ability to get a feel for their use cases and how use our data. It doesn't seem like there is anything deal breaking about using it for production however, it is my opinion that we should really make an effort to find a more robust solution for final products and use these external connections as a means of getting the data available to the customer. the goal being to give them the ability to begin testing their side of the process ASAP. It's possible maybe even likely that the customers asking for data will have a lot more work to do with it once they get it. We don't want to be a bottleneck for them.
