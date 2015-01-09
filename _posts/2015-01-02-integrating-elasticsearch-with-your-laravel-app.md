---
layout: post
title: "Integrating Elasticsearch with Your Laravel app"
date: 2015-01-02
author: tony
categories: [laravel]
---

Searching is an important part of many applications, and it is most of the time treated as a simple task. "Just query using LIKE and you're good to go". Well, while the LIKE clause can be handy sometimes we have to do it in a better way. After researching for a while I found a few good resources on the subject. The most attractive one is Elasticsearch. Yes, you can go far with full-text search and other searching techniques, however Elasticsearch is very handy and comes with a variety of
useful functionalities. I'm going to cover the basics here and link more resources at the bottom, so you can dig further.

<!-- more -->

## What is Elasticsearch?

From the [website](http://www.elasticsearch.org/overview/):

> Elasticsearch is a flexible and powerful open source, distributed, real-time search and analytics engine. Architected from the ground up for use in distributed environments where reliability and scalability are must haves, Elasticsearch gives you the ability to move easily beyond simple full-text search. Through its robust set of APIs and query DSLs, plus clients for the most popular programming languages, Elasticsearch delivers on the near limitless promises of search technology.

In other words: you can use Elasticsearch for logging (see the [ELK stack](http://www.elasticsearch.org/webinars/introduction-elk-stack/)) and for searching. This article aims to explain the usage for searching, maybe I'll cover the logging and analytics in another article.

## Basics about Elasticsearch (with a SQL comparison)

So, in SQL we have a database with tables, which is like the structure of the data, and rows, which are the data itself (basically the values for the table structure). Translating this knowledge to Elasticsearch we have: indexes (like the database itself or schemas in some DBMS) and inside the indexes, we have types (like a database table) and we also have documents (like the database rows), which is the data itself.

Elasticsearch is schema free. However, it is not schema-less, 'cause in order to have better query results, we have to use schemas to make the searches relevant.

## Integration with Laravel

Well, the concepts shown here I took from a Laracon Talk linked at the bottom. It is using Laravel, but the concepts apply to any language/framework because Elasticsearch works as a RESTful API, it means that you consume it using HTTP requests. Don't worry, Elasticsearch is pretty fast and easily scalable.

First thing to know is that you have to have DATA to use elasticsearch, so in my example I have a seed command that populates the database and, while it does that, it indexes all of the data on Elasticsearch. I'll show it in a while, first let's see how we can integrate it with our Eloquent usage.

The way I'll show you is by using [Model Observers](http://laravel.com/docs/4.2/eloquent#model-observers), so you have a refular Eloquent Model, let's say <code>Article</code>. Then you have a Observer like so:

{% highlight php %}
// file: app/Observers/ElasticsearchArticleObserver.php
<?php namespace App\Observers;

use App\Article;
use Elasticsearch\Client;

class ElasticsearchArticleObserver
{
    private $elasticsearch;

    public function __construct(Client $elasticsearch)
    {
        $this->elasticsearch = $elasticsearch;
    }

    public function created(Article $article)
    {
        $this->elasticsearch->index([
            'index' => 'acme',
            'type' => 'articles',
            'id' => $article->id,
            'body' => $article->toArray()
        ]);
    }

    public function updated(Article $article)
    {
        $this->elasticsearch->index([
            'index' => 'acme',
            'type' => 'articles',
            'id' => $article->id,
            'body' => $article->toArray()
        ]);
    }

    public function deleted(Article $article)
    {
        $this->elasticsearch->delete([
            'index' => 'acme',
            'type' => 'articles',
            'id' => $article->id
        ]);
    }
}
{% endhighlight %}

We can register our Observer using a ServiceProvider, like so:

{% highlight php %}
<?php

namespace App\Providers;

use App\Observers\ElasticsearchArticleObserver;
use App\Article;
use Elasticsearch\Client;
use Illuminate\Support\ServiceProvider;

class ObserversServiceProvider extends ServiceProvider
{
    public function boot()
    {
        Article::observe($this->app->make(ElasticsearchArticleObserver::class));
    }

    public function register()
    {
        $this->app->bindShared(ElasticsearchArticleObserver::class, function()
        {
            return new ElasticsearchArticleObserver(new Client());
        });
    }
}
{% endhighlight %}

Remember to register the Service Provider on your <code>config/app.php</code> file.

Now, whenever we create, update or delete an entity using our Eloquent Article Model, we trigger the Elasticsearch Observer to update its data.
Worth noting that this happens synchronously during the Request. A better way is to use Domain Events and have a Elasticsearch handler that updates it in background to speed up the user request.

## Searching with Repositories

Now that you have your elasticsearch fed with your application data, you can perform a better search experience. Let's assume you already have a repository that makes the search using LIKE clause or some full-text search functions. Well, you can still have that as a backup in case your elasticsearch servers crash, in order to do so, you just *decorate* your Repository. Let's see how we could do that, first you need to extract an interface of your repository, in case you don't already have
one:


{% highlight php %}
// file: app/Articles/ArticlesRepository.php
<?php namespace App\Articles;

use Illuminate\Support\Collection;

interface ArticlesRepository
{
    /**
     * @param string $query = ""
     * @return Collection
     */
    public function search($query = "");

    /**
     * @return Collection
     */
    public function all();
}
{% endhighlight %}

Then your Eloquent repository should implement it like so:

{% highlight php %}
// file: app/Articles/EloquentArticlesRepository.php
<?php namespace App\Articles;

use App\Article;

class EloquentArticlesRepository implements ArticlesRepository
{
    /**
     * {@inheritdoc}
     */
    public function search($query = "")
    {
        return Article::where('body', 'like', "%{$query}%")
            ->orWhere('title', 'like', "%{$query}%")
            ->get();
    }

    /**
     * {@inheritdoc}
     */
    public function all()
    {
        return Article::all();
    }
}
{% endhighlight %}

Now, you can write the ElasticseachArticleRepository as a decorator, like so:

{% highlight php %}
// file: app/Articles/ElasticsearchArticlesRepository
<?php namespace App\Articles;

use Illuminate\Support\Collection;
use App\Article;
use Elasticsearch\Client;

class ElasticsearchArticlesRepository implements ArticlesRepository
{
    private $elasticsearch;
    private $innerRepository;

    public function __construct(Client $client, ArticlesRepository $innerRepository)
    {
        $this->elasticsearch = $client;
        $this->innerRepository = $innerRepository;
    }

    /**
     * @param string $query = ""
     * @return Collection
     */
    public function search($query = "")
    {
        $items = $this->searchOnElasticsearch($query);

        return $this->buildCollection($items);
    }

    /**
     * @return Collection
     */
    public function all()
    {
        return $this->innerRepository->all();
    }

    /**
     * @param string $query
     * @result array
     */
    private function searchOnElasticsearch($query)
    {
        $items = $this->elasticsearch->search([
            'index' => 'acme',
            'type' => 'articles',
            'body' => [
                'query' => [
                    'query_string' => [
                        'query' => $query
                    ]
                ]
            ]
        ]);

        return $items;
    }

    /**
     * @param array $items the elasticsearch result
     * @return Collection of Eloquent models
     */
    private function buildCollection($items)
    {
        $result = $items['hits']['hits'];

        return Collection::make(array_map(function($r) {
            $article = new Article();
            $article->newInstance($r['_source'], true);
            $article->setRawAttributes($r['_source'], true);
            return $article;
        }, $result));
    }
}
{% endhighlight %}

Now, the trick is to decorate your repository on your Service Provider, like so:

{% highlight php %}
// file: app/Providers/RepositoriesServiceProvider.php
<?php namespace App\Providers;

use App\Articles\ElasticsearchArticlesRepository;
use App\Articles\EloquentArticlesRepository;
use App\Articles\ArticlesRepository;
use Elasticsearch\Client;
use Illuminate\Support\ServiceProvider;

class RepositoriesServiceProvider extends ServiceProvider
{
    /**
     * {@inheritdoc}
     */
    public function register()
    {
        $this->app->bindShared(ArticlesRepository::class, function($app)
        {
            return new ElasticsearchArticlesRepository(
                new Client,
                new EloquentArticlesRepository()
            );
        });
    }
}
{% endhighlight %}

Now, everywhere you depend on ArticlesRepository interface, you will actually have a ElasticsearchArticlesRepository.

## Conclusion

The post is getting too long, so maybe I will do another one about quering and filtering on Elasticsearch. Worth saying that every example class here is easily testable, just mock the Elasticsearch\Client and you are good to go. To finish up, here is the seeder, so after setting up as above, just run the <code>php artisan db:seed</code> command to populate your database and elasticsearch:

{% highlight php %}
// file: database/seeds/ArticlesTableSeeder.php
<?php

class ArticlesTableSeeder extends Seeder
{
    public function run()
    {
        Laracasts\TestDummy\Factory::times(50)->create('App\Article');
    }
}
{% endhighlight %}

I'm using TestDummy here, so you better check the package to have an understanding of what is going on here. It is also easy to do a cli command to reindex your elasticsearch, like so:

{% highlight php %}
// file: app/Console/IndexArticlesToElasticsearchCommand.php
<?php namespace App\Console;

use App\Article;
use Elasticsearch\Client;

class IndexArticlesToElasticsearchCommand
{
    /**
     * {@inheritdoc}
     */
    protected $name = "app:es-index";

    /**
     * {@inheritdoc}
     */
    protected $description = "Indexes all articles to elasticsearch";

    /**
     * @return void
     */
    public function fire()
    {
        $models = Article::all();
        $es = new Client;

        foreach ($models as $model)
        {
            $es->index([
                'index' => 'acme',
                'type' => 'articles',
                'id' => $model->id,
                'body' => $model->toArray()
            ]);
        }
    }
}
{% endhighlight %}

After registering your command, you can run <code>php artisan app:es-index</code> to index existing articles to Elasticsearch.

## Useful Resources

* [LaraconEU talk about Elasticsearch by Ben Corlett](https://www.youtube.com/watch?v=waTWeJeFp4A)
* [Getting Down and Dirty with ElasticSearch by Clinton Gormley](https://www.youtube.com/watch?v=7FLXjgB0PQI)
* [Laravel Model Observers](http://laravel.com/docs/4.2/eloquent#model-observers)
* [My demo project on Github](https://github.com/tonysm/laravel-elasticsearch-test)
* [Introduction and Demo to the Elasticsearch, Logstash and Kibana](https://www.youtube.com/watch?v=GrdzX9BNfkg)