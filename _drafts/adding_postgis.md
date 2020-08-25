---
author: Ben Stacey
title: Adding postgis support to Rails
description: Steps to add
excerpt: Start here if you want to be our next author!
tags: product engineering
image: /assets/images/blank-branding-identity-business-cards-6372.jpg
---
# PostGIS support in Rails 6

Here at BiggerPockets Engineering, we have been busy building the first iteration of the new BPInsights Rent Estimator for Beta launch.
We have more than 100 million property records containing fields such as street address, ZIP code, property type; as well as associated records containing rental and sale price data, when the price data was recorded, valuation data and much more. <br/>
<br/>
One of the first features to implement was to show a subset of comparable properties (‘Comps’) related to the current report page with a known longitudinal/latitudinal location.
We decided to preferentially show properties with a similar number of user selected bedrooms, that were within a fixed distance of a location/property. If it was not possible to extract 12 properties with a certain number of bedrooms, within a fixed distance of the starting lon/lat, the search would need to be widened to include properties which lay in a larger radius. 

INSERT COMP PAGE IMAGE

## Geokit/Geocoder vs PostGIS

The Geocoder Ruby gem has by far the most downloads in the realm of street/IP address encoding, and it possesses ActiveRecord support for a bunch of distance based database queries. The Biggerpockets app already uses the Geokit Ruby gem (in several areas including the Marketplace, Forums and Premium Listings), and has done for many years. So why bother adding new gems, database extensions and a few workarounds to make use of PostGIS right now?
<br/>
We knew BPInsights was going to be a location-heavy product from the outset. We already had a firm use case for basic spatial querying and wanted to be able to reach for as many spatial functions as possible in the future, as well as have the ability to ingest a range of spatial datatypes as it was highly likely we would have to source more data to provide as much context as possible when a user is finding or analysing a deal.
<br/>
Heroku serves our application and has a proven track record of working with PostGIS, and we use Postgres already - if you don't use Heroku and/or Postgres please check documentation carefully for any issues or caveats with the use of PostGIS columns in your application.
<br/>
That's not to say we are recommending PostGIS for every project that requires distance-based querying - see this excellent article by Krzysztof Zych on why you _shouldn't_ use PostGIS :) https://blog.rebased.pl/2020/04/07/why-you-probably-dont-need-postgis.html

## The code

### Gemfile

```ruby
gem 'rgeo' # => core component to support geospatial objects, handle geometry, parse datatypes such as WKT, WKB, Multipolygons etc
gem 'activerecord-postgis-adapter' # => Enables PostGIS database features to work with ActiveRecord - provides additional migrations, allows spatial data in queries etc.
gem 'rgeo-activerecord' # => Provides additional extensions and helpers for ActiveRecord
```

### Postgres extension

```ruby
class EnablePostgisExtension < ActiveRecord::Migration[6.0]
  def change
    enable_extension "postgis"
  end
end
```

### CI Config
In terms of CI, you will likely need to amend the source database Docker image that your CI instance(s) spin up from. We use CircleCI so we had amended our `.circleci/config.yml` as follows:

```diff
-    - image: circleci/postgres:10-alpine-ram
+    - image: circleci/postgres:10-postgis-ram
```

### database.yml

For the `production` environment:
<br/>
Our databases are provisioned by Heroku so we followed the recommended procedure to alter the value of the `DATABASE_URL` env variable (https://devcenter.heroku.com/articles/rails-database-connection-behavior) in `production`:
<br/>


```diff
-    url: <%= ENV.fetch('DATABASE_URL') %>
+    url: <%= ENV.fetch('DATABASE_URL', '').sub(/^postgres/, "postgis") %>
```

For the remaining environments, in `database.yml` we replaced `postgres:///` with `postgis:///`.
<br/>
To solve an issue with the pre-existing `geokit` methods throwing `unitialized constant` errors after this change we also added the following:







## Example querying
