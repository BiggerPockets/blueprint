---
author: Ben Stacey
title: PostGIS in Rails 6
description: Steps required to add PostGIS support to Rails 6 with use cases and pre amble
tags: product engineering
image: /assets/images/insights-comps.png
---
# PostGIS in Rails 6

Here at BiggerPockets Engineering, we have been busy building the first iteration of the new BPInsights Rent Estimator for Beta launch (You can check out the entire feature at [https://biggerpockets.com/insights/property-searches/new] [1]).
<br/>
<br/>
We have more than 100 million property records containing fields such as street address, ZIP code, property type; as well as associated records containing rental and sale price data, when the price data was recorded, valuation data and much more.
<br/>
<br/>
One of the first features to implement was to show a subset of comparable properties related to the current report page with a known longitudinal/latitudinal location.
We decided to preferentially show properties with a similar number of user selected bedrooms, that were within a fixed distance of a location/property.

## Geokit/Geocoder vs PostGIS

The Geocoder Ruby gem has by far the most downloads in the domain of street address/IP location encoding, and it possesses ActiveRecord support for a bunch of distance based database queries. The Biggerpockets app already uses the Geokit Ruby gem (in several areas including the Marketplace, Forums and Premium Listings). So why bother adding new gems, database extensions and a few workarounds to make use of PostGIS?
<br/>
<br/>
We knew BPInsights was going to be a location-heavy product from the outset. We already had a firm use case for basic spatial querying and wanted to be able to reach for as many spatial functions as possible in the future, as well as have the ability to ingest a range of spatial datatypes. It was highly likely we would have to source more data in the future to provide further useful context when a user is finding or analysing a deal.
<br/>
<br/>
Heroku serves our application and has a proven track record of working with PostGIS, and we use Postgres already - if you don't use Heroku and/or Postgres please check documentation carefully for any issues or caveats with the use of PostGIS columns in your application.
<br/>
<br/>
That's not to say we recommend PostGIS for every project that requires distance-based querying - see this excellent article by Krzysztof Zych on why you _shouldn't_ use PostGIS:

[https://blog.rebased.pl/2020/04/07/why-you-probably-dont-need-postgis.html] [2]

## The code

### Postgres extension

```ruby
class EnablePostgisExtension < ActiveRecord::Migration[6.0]
  def change
    enable_extension "postgis"
  end
end
```

### Database migration


```ruby
def change
  add_column :insights_properties, :lonlat, :geometry, geographic: true, srid: 4326
  add_index :insights_properties, :lonlat, using: :gist
end
```

### Gemfile

```ruby
gem 'rgeo' # => core component to support geospatial objects, handle geometry, parse datatypes such as WKT, WKB, Multipolygons etc
gem 'activerecord-postgis-adapter' # => Enables PostGIS database features to work with ActiveRecord - provides additional migrations, allows spatial data in queries etc.
gem 'rgeo-activerecord' # => Provides additional extensions and helpers for ActiveRecord
```

The addition of the `gist` index is recommended by the PostGIS authors to improve performance when querying large tables [https://postgis.net/docs/PostGIS_FAQ.html] [3]

### database.yml

In `production`:
<br/>
<br/>
Our databases are provisioned by Heroku so we followed the recommended procedure to alter the value of the `DATABASE_URL` env variable (https://devcenter.heroku.com/articles/rails-database-connection-behavior) in `production`:
<br/>


```diff
-    url: <%= ENV.fetch('DATABASE_URL') %>
+    url: <%= ENV.fetch('DATABASE_URL', '').sub(/^postgres/, "postgis") %>
```

For the remaining environments, in `database.yml` we replaced `postgres:///` with `postgis:///`.
<br/>
<br/>
To solve an issue with the pre-existing `geokit` methods throwing `unitialized constant` errors after this change we also added the following:

```ruby
# lib/geokit-rails/adapters/postgis.rb

require File.join("geokit-rails", "adapters", "postgresql")

module Geokit
  module Adapters
    class PostGIS < PostgreSQL
    end
  end
end
```

### CI Config
In terms of CI, you will need to amend the source database Docker image that your CI instances spin up from. We use CircleCI so we amended our `.circleci/config.yml` as follows:

```diff
-    - image: circleci/postgres:10-alpine-ram
+    - image: circleci/postgres:10-postgis-ram
```

### Database cleaner

Our database cleaning strategy was truncating the `spatial_ref_sys` table. This table stores constants and reference values required for various PostGIS calculations. If starting a Rails instance throws an `unable to find table: spatial_ref_sys` error you should alter your database cleaning configuration to ignore this table with code like the following:

```diff
-    DatabaseCleaner.clean_with(:deletion, cache_tables: false)
+    DatabaseCleaner.clean_with(:deletion, cache_tables: false, except: %w(spatial_ref_sys))
```

If you need to rebuild the `spatial_ref_sys` table after an errant deletion you can run the following `psql` command:

`psql -d [DATABASE_NAME] -f /usr/local/share/postgis/spatial_ref_sys.sql`

## Usage examples

The following example expression can be used to insert a geographical point type into our freshly created column:

```ruby
Insights::Property.update(lonlat: RGeo::Geographic.spherical_factory(srid: 4326).point("-104.9816", "39.5142"))
```

To find properties, whose coordinates are within a certain number of miles as another, we selected the PostGIS function `ST_DWithin` which has the following signature:

```sql
ST_DWithin(geometry1, geometry2, distance)
```

We also used the `ST_POINT` function to convert given coordinates to usable points for `ST_DWITHIN`.
<br/>
<br/>
The following methods were then used to implement this function as part of a `where` clause. It was necessary to wrap the clause strings in `Arel` functions so as to avoid `Dangerous query method` errors being thrown:

```ruby
def self.within_distance(long:, lat:, miles:)
  where(
    Arel::Nodes::NamedFunction.new(
      "ST_DWithin",
      [
        arel_table[:lonlat],
        st_point_node(long, lat),
        distance_conversion_node(miles),
      ]
    )
  )
end

def self.distance_conversion_node(miles)
  Arel::Nodes::Multiplication.new(
    Arel::Nodes.build_quoted(miles),
    Arel::Nodes::build_quoted(METRES_IN_MILE)
  )
end

def self.st_point_node(long, lat)
  Arel::Nodes::NamedFunction.new(
    "ST_POINT",
    [Arel::Nodes.build_quoted(long), Arel::Nodes.build_quoted(lat)]
  )
end
```

[1]: [https://biggerpockets.com/insights/property-searches/new]
[2]: [https://blog.rebased.pl/2020/04/07/why-you-probably-dont-need-postgis.html]
[3]: [https://postgis.net/docs/PostGIS_FAQ.html]
