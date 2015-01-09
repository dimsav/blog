---
layout: post
title: Laravel IoC container
date: 2015-01-02
author: hannes
categories: [laravel]
---

At work I'm currently modernising a legacy project without composer into an application with less code and thus less bugs. The first composer package I required was an application container to add Dependency Injection to the project. The container I used is called [orno/di](https://github.com/orno/di).

I bootstrapped the container in the index file. After that I registered some basic stuff to it like
an object of the Symfony Request class, a Monolog logger with rotating log files, ... Finally I was able to adapt
the existing old router to resolve the controllers and their dependencies from the IoC container.

In a couple of hours I had enabled the project to adopt and inject composer loaded classes within minutes.

## Auto resolving
But how does this container, often called application container, know what to inject? Some containers allow you to
configure a hierarchical tree of dependencies in config files or bootstrap code.

Since PHP 5 there is a Reflection extension available. This extension allows you to request meta data about extensions, classes, properties, objects, methods, functions and arguments. All of these have Reflector classes like ReflectionExtension,
ReflectionClass, etc. This Reflection interface allows containers to inspect type hinted arguments of methods and thus constructors.

## Constructor Injection
One way to inject a dependent object into a depending object is to pass it as an argument with the constructor.

You can type hint arguments of a constructor of a Controller to indicate what type of object this needs.
If the Reflection API tells the container that this constructor needs a `\GuzzleHttp\Client` as the first argument, the
container can create one by recursively resolving it. If the depending class has depending classes of its own this action of
resolving dependencies can be executed recursively.

### Interface type hinting
Some type hints can refer to interfaces. By definition Interfaces can't be instantiated. In order to allow the container
to know which type to use, that implements the interface, we manually need to hint that to the container. For example
with the Laravel container you would do this:

<script src="https://gist.github.com/hannesvdvreken/4e7d90a49c498797b73f.js"></script>

Where `\Vendor\Client` implements `\Vendor\Contracts\Client`.

## Method injection
The same action that is used to retrieve the constructor argument types can be used to get the type hints of a callable.
A callable is everything that can be passed as the first argument of `call_user_func` and `call_user_func_array`.
The `ReflectionFunctionAbstract` class allows you to inspect all of these forms of a callable:

<script src="https://gist.github.com/hannesvdvreken/ea9666c659685f7c69f9.js"></script>

## Laravel's Class@method syntax
Laravel's IoC container also allows you to call `Class@method`, this will internally not only resolve an object of
type `Class` but also resolve the `method`'s dependencies and call it with the right arguments. This effectively does
two recursive dependency lookups. It is used in several places like `Event::fire('Handler@method', [])` but it's probably
most used for the routes definitions: `Route::get('/about', 'Controller@about')`.

## What got added?
In the first version of `App::call(...)` it was only possible to use Laravel's `Class@method` syntax and
anonymous functions (`App::call(function (\Vendor\Client $client) {...})`).

After my pull request a few weeks ago it is now possible to use every type of PHP callable, including `[$object, 'method']`.
Enjoy!

## The power to inject dependencies into callables
Now that it is possible to add dependencies to every callable and not having to initialize them from where you call it,
there are a ton of new possibilities. One of them is the new type of validation that got introduced at Laracon EU last summer:

If your controller method depends on a `Request` object that also implements the `ValidatesWhenResolved` interface, the
`FormRequest`'s `validate` method will be called even before the controller method is called. This helps you to
keep your controllers clean.

## Other callables

### ServiceProvider
When an application that runs on Laravel gets booted it will make use of `ServiceProvider`s to register stuff to the
container. All (non-deferred) `ServiceProvider`'s `register` methods will be executed first to bind callables to the
container. After that all `boot` methods will be called.

The callables that get registered to the container using the `$app->bind(...)` and `$app->singleton(...)` methods
should be able to use the advantage of method dependency injection. For example this is currently a typical way to bind a callable to the container inside the `boot` method.

<script src="https://gist.github.com/hannesvdvreken/0dad01f5ce541d7514d8.js"></script>

If the callables that are registered in the `register` methods of the `ServiceProvider` are called with injected dependencies, this could be rewritten to:

<script src="https://gist.github.com/hannesvdvreken/e248393cfaacb56dfff0.js"></script>

Theoretically everything should be bound to the container using the `register` methods.
That's why theoretically the boot method could be called using the `App::call(...)`
functionality to automatically inject dependencies that need extra bootstrapping inside the `boot` method.

For example in the [boot method of the DatabaseServiceProvider](https://github.com/laravel/framework/blob/e989e173252e38eb41def4d0d85241dd28ab38bd/src/Illuminate/Database/DatabaseServiceProvider.php#L16-L18)
one could inject both the `Illuminate\Database\DatabaseManager` and the `Illuminate\Events\Dispatcher`.

**Edit** apparently the `boot` method of ServiceProviders are [already being called](https://github.com/laravel/framework/blob/master/src/Illuminate/Foundation/Application.php#L560-L566) with method dependency injection. The ServiceProviders just didn't use this yet.

### Other use cases

Following are some other use cases for method injection. These are also not enabled by Laravel. Probably because
every time you depend on `App::call` it involves adding a dependency on `illuminate/container` to another Illuminate component.
And that's generally not a good thing.

#### Cache::remember

<script src="https://gist.github.com/hannesvdvreken/0535caefc11ffa83068e.js"></script>

#### Schema::table

<script src="https://gist.github.com/hannesvdvreken/372e79a9a00b20a4f900.js"></script>

## Wrapping

Laravel's container also features a [wrap method](https://github.com/laravel/framework/blob/e5e14291ad0cb1e9a1f663d7ad255aed39f8fad2/src/Illuminate/Container/Container.php#L505-L511).
This wrap method literally wraps any passed anonymous function into a new anonymous function.
When the returned anonymous function gets called the original function is called with
`App::call()`. This enables you to add method dependency injection everywhere you want on your own.
In the case of `Cache::remember`:

<script src="https://gist.github.com/hannesvdvreken/10fa1e6d37321746d55b.js"></script>

### Other use cases

When plain PHP expects a callable, one could use App::wrap to generate a callable which in turn executes a callable
via the `App::call` functionality. This could be used by Laravel for example to register exception handlers which
instantiate Whoops in a more lazy way.

<script src="https://gist.github.com/hannesvdvreken/6c16c908547a1ee5a3c2.js"></script>

## Available containers
There are a number of Containers available in Composer packages. Here are a couple of them that come to my mind:

- [Orno DI](https://github.com/orno/di)
- [Laravel's container](https://github.com/illuminate/container)
- [SlimPHP's container](https://github.com/codeguy/Slim/blob/master/Slim/Helper/Set.php)
- [PHP-DI](https://github.com/mnapoli/PHP-DI)
- [Aura DI](https://github.com/auraphp/Aura.Di)

There are probably a ton of other containers. If you know one that is not on the list:
[@hannesvdvreken](https://twitter.com/hannesvdvreken)