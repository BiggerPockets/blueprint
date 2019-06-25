---
author: David Daily
title: Simplistic Synonyms for Elasticsearch

tags: elasticsearch synonym
---

You should already have knowledge and a working local instance of elastisearch. If you need help getting started, I would recommend checking out:

https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html


## The Problem

Here at BiggerPockets, we have all kinds of searchable records, most of which have some kind of location data associated with them. The issue that we have experienced is that users will search for records using state abbreviations such as `TX`, but records will be saved with the location of `Texas`, so the query will render no results, making our users sad and frustrated.

## The Naive Solution

There are multiple ways to solve this problem, one of them being is to save full name of the state AND the state abbreviation in our index, perform a match query on these two fields, and return the matching document.

For example, if our mapping looked something like this:

```
{
  "houses" : {
    "mappings" : {
      "properties" : {
        "state_abbrv" : {
          "type" : "keyword"
        },
        "state_name" : {
          "type" : "text"
        }
      }
    }
  }
}
```

and our index
```
{
  "took" : 16,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "houses",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "state_name" : "California",
          "state_abbrv" : "CA"
        }
      },
      {
        "_index" : "houses",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "state_name" : "New Mexico",
          "state_abbrv" : "NM"
        }
      },
      {
        "_index" : "houses",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "state_name" : "Colorado",
          "state_abbrv" : "CO"
        }
      }
    ]
  }
}
```

We could perform this query:

```
{
  "query": {
    "bool": {
      "minimum_should_match": 1,
      "should": [
        {
          "match": {
            "state_name": "CA"
          }
        },
        {
          "match": {
            "state_abbrv": "CA"
          }
        }
      ]
    }
  }
}
```

or this query:

```
{
  "query": {
    "bool": {
      "minimum_should_match": 1,
      "should": [
        {
          "match": {
            "state_name": "California"
          }
        },
        {
          "match": {
            "state_abbrv": "California"
          }
        }
      ]
    }
  }
}
```

and both give us the following results:

```
"hits": [
    {
        "_index": "houses",
        "_type": "_doc",
        "_id": "1",
        "_score": 1.0925692,
        "_source": {
            "state_name": "California",
            "state_abbrv": "CA"
        }
    }
]

```

Hooray! It worked! Ship it!

Well, I don't know about you, but something about this just doesn't sit too well with me. We are storing redundant data so every time that we store a record, say a house, we will be responsible for uploading that full state name and abbreviation into elasticsearch. Plus, the query seems a little overly complex considering this is probably something that we will need to do for numerous things.


## A Better Way

Enter synonyms. By defining a new analyzer for our state field, we can pass the work off to elasticsearch.  

To create the new analyzer and filter, we can do the following

```
curl -X PUT "localhost:9200/test_index" -H 'Content-Type: application/json' -d'
{
    "settings": {
        "index" : {
            "analysis" : {
                "analyzer" : {
                    "state" : {
                        "tokenizer" : "whitespace",
                        "filter" : ["standard", "lowercase", "state_synonyms", "snowball"]
                    }
                },
                "filter" : {
                    "state_synonyms": {
                        "type" : "synonym",
                        "synonyms": ["ca, california", "nm, new mexico", "co, colorado"]
                    }
                }
            }
        }
    }
}
'
```
