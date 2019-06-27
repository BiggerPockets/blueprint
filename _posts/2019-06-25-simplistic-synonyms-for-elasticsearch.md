---
author: David Daily
title: Simplistic Synonyms for Elasticsearch
tags: elasticsearch synonym
image: /assets/images/beverage-browsing-coffee-1157858.jpg
---

For this article, you should already have knowledge of Elasticsearch and of some relatively basic concepts like `tokens`, `analyzers`, and simple `bool` queries. If you need some background on these topics, or help setting up a local Elasticsearch instance, check out:

[https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html][1]

Now, onto the good part!

## The Problem

Here at BiggerPockets, we have all tons of different types of searchable records, most of which have some kind of location data associated with them. The issue that we have experienced is that users will search for records using state abbreviations such as `TX`, but records will be saved with the location of `Texas`, so the query will render no results. As you can imagine, this is a frustrating experience for end users.

## The Naive Solution

There are multiple ways to solve this problem, one of them is to save full name of the state AND the state abbreviation in our index, perform a match query to on these two fields, and return the matching documents.

For example, if our mapping looked something like this:

```
{
  "states" : {
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
        "_index" : "states",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "state_name" : "California",
          "state_abbrv" : "CA"
        }
      },
      {
        "_index" : "states",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "state_name" : "New Mexico",
          "state_abbrv" : "NM"
        }
      },
      {
        "_index" : "states",
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
...

  "hits": [
      {
          "_index": "states",
          "_type": "_doc",
          "_id": "1",
          "_score": 1.0925692,
          "_source": {
              "state_name": "California",
              "state_abbrv": "CA"
          }
      }
  ]

...

```

Hooray! It worked! Ship it!

Well, I don't know about you, but something about this just doesn't sit too well with me. We are storing redundant data so every time that we store a record, we will be responsible for uploading that full state name and abbreviation into Elasticsearch. Plus, the query seems a little overly complex considering this type of search is probably something that we will need to do for numerous different searchable records.

Since this is not the solution that we will be going with, let's blow our state index away and examine another solution.

```$ curl -X DELETE localhost:9200/states```

## A Better Way

To demonstrate how synonyms work, let's create a new index with only one state field that hold the full name of a state, and we will create a custom analyzer to analyze the field.  

```
$ curl -X PUT "localhost:9200/states" -H 'Content-Type: application/json' -d'
{
    "settings": {
        "index" : {
            "analysis" : {
                "filter" : {
                    "state_synonyms": {
                        "type" : "synonym",
                        "synonyms": ["ca, california", "nm, new mexico", "co, colorado"]
                    }
                },
                "analyzer" : {
                    "custom_state_analyzer" : {
                        "tokenizer" : "whitespace",
                        "filter" : ["lowercase", "state_synonyms"]
                    }
                }
            }
        }
    },
    "mappings": {
      "properties": {
        "state": { "type": "text", "analyzer": "custom_state_analyzer" }
      }
    }
}
'
```

This looks a lot more complicated than it is, but lets walk through what this is doing:

-  We create a new token filter of type `synonym` called `state_synonyms`. What this filter does is, once our tokenizer has converted our input to a stream of tokens, adds synonym tokens into our token stream.

- We define a new custom analyzer called `custom_state_analyzer` that uses our custom `state_synonyms` filter. You will notice that our analyzer also utilizes the `lowercase` filter before our `state_synonyms` filter. This is just to downcase all of the tokens in our token stream to make our analyzer case insensitive.

We can use the `analyze` API to see what this analyzer is doing on our state field:
```
$ curl -X GET localhost:9200/states/_analyze?pretty -H 'Content-Type: application/json' -d'{ "analyzer": "custom_state_analyzer", "text": "CA"}'
```

and get the response
```
{
  "tokens" : [
    {
      "token" : "ca",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "california",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "SYNONYM",
      "position" : 0
    }
  ]
}
```
So even though we only search for `CA`, our analyzer is tokenizing our query, then inserting another token that is a synonym of `CA`, namely `California`. So now documents whose state field is `CA` or `California` will match our query. Cool!

To see this in action, lets populate our new index with a few documents:

```
$ curl -XPUT 'localhost:9200/states/_doc/1?pretty' -H 'Content-Type: application/json' -d '{"state": "California"}'

$ curl -XPUT 'localhost:9200/states/_doc/2?pretty' -H 'Content-Type: application/json' -d '{"state": "New Mexico"}'

$ curl -XPUT 'localhost:9200/states/_doc/2?pretty' -H 'Content-Type: application/json' -d '{"state": "Colorado"}'
```
And issue a query to see what states match `CA`

```
$ curl -XGET 'localhost:9200/states/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "state": "CA"
        }
      }
    }
  }
}
'
```
And check out our response:

```
...

  "hits" : [
        {
          "_index" : "states",
          "_type" : "_doc",
          "_id" : "1",
          "_score" : 1.6068903,
          "_source" : {
            "state" : "California"
          }
        }
      ]
...
```

Nice!

## Conclusion

Hopefully, this small example demonstrates some of the power that synonym filters have. You can use synonyms in phrase queries, multi term queries, and basically anything else that you can think of. Now go explore what you can do with these filters and even look into making other custom filters and analyzers of your own!

Happy searching!


[1]: https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html
