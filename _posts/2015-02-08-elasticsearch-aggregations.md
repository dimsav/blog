---
layout: post
title: "Elasticsearch Aggregations"
date:   2015-02-09
author: tony
categories: [elasticsearch, aggregations,search]
---

I have already covered how you can easily [integrate Elasticsearch with your app](http://blog.madewithlove.be/post/integrating-elasticsearch-with-your-laravel-app/), but I haven't talked anything about how you can *query* your data, and I believe this is not covered (I guess it's because it depends on the context you are trying to perform your query and what you really need).

<!-- more -->

I won't cover the basics of querying or filtering here, instead I will cover a really cool feature called *aggregations*, it's a really way to perform some analysis over your data. And I'm also going to cover a still *experimental* feature called [**scripted metric**](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-aggregations-metrics-scripted-metric-aggregation.html).

To start, you need to setup Elasticsearch locally or in a VM (see here MISSING THE LINK). You will also need some schema and data. You can use the example given in the documentation:

Mapping:

{% highlight bash %}
$ curl -XPUT localhost:9200/transactions/ -d '{ 
    "mappings": {
        "sales": {
            "properties": {
                "type": { "type": "string" },
                "amount": { "type": "double" }
            }
        }
    } 
}'
{% endhighlight %}

This will create our structure, now we just need some data:

{% highlight bash %}
$ curl -XPUT localhost:9200/transactions/sales/1 -d '{
    "type": "sale",
    "amount": 80
}'

$ curl -XPUT localhost:9200/transactions/sales/2 -d '{
    "type": "cost",
    "amount": 10
}'

$ curl -XPUT localhost:9200/transactions/sales/3 -d '{
    "type": "cost",
    "amount": 30
}'

$ curl -XPUT localhost:9200/transactions/sales/4 -d '{
    "type": "sale",
    "amount": 130
}'
{% endhighlight %}

## Time to analyse

First of all, let's see how Elasticsearch stores our data, and how we can query them all:

{% highlight bash %}
$ curl -XGET localhost:9200/transactions/sales/_search -d '{
    "query": {
        "match_all": {}
    }
}'
{% endhighlight %}

This will result in something like so:

{% highlight js %}
{
   "took": 73,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 4,
      "max_score": 1,
      "hits": [
         {
            "_index": "transactions",
            "_type": "stock",
            "_id": "4",
            "_score": 1,
            "_source": {
               "type": "sale",
               "amount": 130
            }
         },
         {
            "_index": "transactions",
            "_type": "stock",
            "_id": "1",
            "_score": 1,
            "_source": {
               "type": "sale",
               "amount": 80
            }
         },
         {
            "_index": "transactions",
            "_type": "stock",
            "_id": "2",
            "_score": 1,
            "_source": {
               "type": "cost",
               "amount": 10
            }
         },
         {
            "_index": "transactions",
            "_type": "stock",
            "_id": "3",
            "_score": 1,
            "_source": {
               "type": "cost",
               "amount": 30
            }
         }
      ]
   }
}
{% endhighlight %}
 
The `_source` is our data. Let's do some analyses using some built-in *aggregations*.

### Terms Aggregation

{% highlight bash %}
$ curl -XGET localhost:9200/transactions/stock/_search -d '{
    "size": 0,
    "aggs": {
        "sales_types": {
            "terms": {
                "field": "type"
            }
        }
    }
}'
{% endhighlight %}

This will result in something like this:

{% highlight js %}
{
   "took": 40,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 4,
      "max_score": 0,
      "hits": []
   },
   "aggregations": {
      "sales_types": {
         "doc_count_error_upper_bound": 0,
         "sum_other_doc_count": 0,
         "buckets": [
            {
               "key": "cost",
               "doc_count": 2
            },
            {
               "key": "sale",
               "doc_count": 2
            }
         ]
      }
   }
}
{% endhighlight %}

We are using `size:0` because we don't care about the query result, just our agrregation. This is like `count` with `group by` clause (at least it's how I see it), for those used with SQL. Pretty bool, right?

### Stats Aggregation

{% highlight bash %}
$ curl -XGET localghost:9200/transactions/stock/_search -d '{
    "size": 0,
    "aggs": {
        "sales_stats": {
            "stats": {
                "field": "amount"
            }
        }
    }
}'
{% endhighlight %}

Response:

{% highlight js %}
{
   "took": 6,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 4,
      "max_score": 0,
      "hits": []
   },
   "aggregations": {
      "sales_stats": {
         "count": 4,
         "min": 10,
         "max": 130,
         "avg": 62.5,
         "sum": 250
      }
   }
}
{% endhighlight %}

Wow, this just gave us some analyses about our sales. It says that we have `4` items, `minimum` amount is 10, `maximum` is 130, `average` is 62.5 and the `sum` is 250. Ok, That's cool, but we can't use this data. Our sales also store some *costs*, so all of this fields are telling nothing to us in this context. Don't get me wrong, this aggregation is pretty cool, just not useful in this context. How can we perform some analyses about our `profit` then? Well, we can build our own aggregation using some Groovy scripts, it's called `scripted_metric`.

### Scripted-metric Aggregation

Let's see the scripts and then I'll try to explain what they actually are:

{% highlight bash %}
$ curl -XGET /transactions/stock/_search -d '{
    "size": 0,
    "aggs": {
        "profit": {
            "scripted_metric": {
                "init_script": "_agg[\"transactions\"] = [];",
                "map_script": "if (doc.type.value == \"sale\") { _agg.transactions.add(doc.amount.value); } else { _agg.transactions.add(-1 * doc.amount.value); }",
                "combine_script": "profit = 0; for (t in _agg.transactions) { profit += t }; return profit;",
                "reduce_script": "profit = 0; for (a in _aggs) { profit += a }; return profit;"
            }
        }
    }
}'
{% endhighlight %}

Response:

{% highlight js %}
{
   "took": 13,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 4,
      "max_score": 0,
      "hits": []
   },
   "aggregations": {
      "profit": {
         "value": 170
      }
   }
}
{% endhighlight %}

That is our profit. Let's now analyse what is going on here. We created a new aggregation called `profit` of type `scripted_metric`, this requires only the `map_script` property, but let's see all of them:

* `init_script`: it acts in the begining of the process, prior to the collection of documents and we are ust creating an array called `transactions` in the `_agg` object (our aggregation object);
* `map_script`: executed one per document. This, as I said, is the only one required. If you are not using any other script, you have to assign the resulting state to the `_agg` object in order to see what it does. Those familiar with `map/reduce` already got it, we can change the structure of our document for the aggregation result here. In fact, we are checking if our document type is `sale` or `cost` to `sum` or `subtract` its value properly;
* `combine_script`: we can use this to change our aggregation structure, right now we have an array of object with a `transactions` array containing each document amount. We are using this to tranform our array of objects into a single array containing all our `amounts`;
* `reduce_script`: the combine_script trasnformed our aggregation value into an array of `amounts`, we can *reduce*  it to a single value in this script (again, those familiar with `map/reduce` concept already knew this);

You can try performing this aggregation removing each script (except for the `map_script` that is the only one required and for the `init_script`that our `map_script` uses). It clarifies a little more. :)

### What is available for the `map_script` stage?

Well, pretty much you have `doc`, which is (of course) your document itself. But you also have a `_source` object, which corresponds to the source of your document (this one is slower than the `doc` object).

## Conclusion

Another cool thing about aggregations is that you can actually combine aggregations to your needs, but this might need it's own blog post. And you can store your scripts your scripts in elasticsearch and just reference them.

The built-in aggregations are pretty cool and you can perform a lot of analyses on your data with them, with `scripted_metric` you can build your own aggregation that fits best on your context.
