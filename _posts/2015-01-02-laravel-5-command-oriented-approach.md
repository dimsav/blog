---
layout: post
title: "Laravel 5 Command-Oriented Approach"
date: 2015-01-02
author: tony
categories: [laravel]
---

A lot of shiny things around the Laravel world that I would like to talk, so I chose the new CommandBus that Laravel 5 brings by default. To start I'd like to say that it's pretty damn cool.

<!-- more -->

## Command-Oriented Architecture
I'm not gonna explain in details this approach, but I'm going to give a brief introduction on the topic and link a few more content at the bottom.

Basically, we should describe our application into commands to make our intentions explicit. Let's say you have a subscription system on your app, you'd have a "SubscribeUserCommand" class, or something similar, that maps to its own handler, in this case "SubscribeUserCommandHandler". Doing so you actually decouple your application from the transport layer (HTTP, cli, queue job, event handler, etc..). It means that you can *dispatch* this command from a
controller or a console command (cli) with no trouble.

## Laravel way

We used to implement this approach using some packages (see [laracasts/commander](https://github.com/laracasts/Commander)) which is actually pretty neat and works like a charm. However, Laravel 5 brings it's own CommandBus with a plus: it can handle commands (AND EVENTS!!!) in background (queues).

A typical command looks like this:

{% highlight php %}
// file: app/Commands/SubscribeUserCommand.php
<?php namespace App\Commands;

use App\Subscriptions\MembershipType;

class SubscribeUserCommand extends Command
{
    public $userId;
    public $membershipType;
    
    public function __construct($userId, MembershipType $membershipType)
    {
        $this->userId = $userId;
        $this->membershipType = $membershipType;
    }
}
{% endhighlight %}

Then you should have a handler like this:

{% highlight php %}
// file: app/Handlers/Commands/SubscribeUserCommandHandler.php
<?php namespace App\Handlers\Commands;

use App\Commands\SubscribeUserCommand;
use Illuminate\Contracts\Events\Dispatcher;
use App\Payment\PaymentInterface;

class SubscribeUserCommandHandler
{
    private $userRepository;
    private $events;
    private $payment;

    public function __construct(UserRepository $userRepository, Dispatcher $events, PaymentInterface $payment)
    {
        $this->userRepository = $userRepository;
        $this->events = $events;
        $this->payment = $payment;
    }

    public function handle(SubscribeUserCommand $command)
    {
        $user = $this->userRepository->find($command->userId);

        $user->subscribe($command->membershipType, $this->payment);

        $this->dispatchEvents($user->releaseEvents());
    }


    /**
     * @param array $events
     * @return void
     */
    private function dispatchEvents(array $events)
    {
        foreach ($events as $event)
            $this->events->fire($event);
    }
}
{% endhighlight %}

Which you can dispatch, let's say, from your controller like so:

{% highlight php %}
// file: app/Http/Controllers/SubscriptionsController.php
<?php namespace App\Http\Controllers;

use Illuminate\Contracts\Auth\Guard;
use App\Commands\SubscribeUserCommand;

class SubscriptionControllers extends Controller
{
    private $auth;

    public function __construct(Guard $auth)
    {
        $this->middleware('auth');
        $this->auth = $auth;
    }

    public function subscribe(SubscribeUserRequest $request)
    {
        $command = new SubscribeUserCommand(
            $this->auth->user()->id,
            MembershipType::build($request->get("membership_type"))
        );
        
        $this->dispatch($command);
                
        return redirect()->route("home");
    }
}

{% endhighlight %}

The *dispatch* method is inherited from the Controller class (which uses the <code>Illuminate\Foundation\Bus\DispatchesCommands</code>) and it maps commands to handlers. Cool stuff. This example works synchronously. If you need to handle the command in background (queue jobs) you just have to implement the <code>Illuminate\Contracts\Queue\ShouldBeQueued</code> interface on your command, like so:

{% highlight php %}
// file: app/Commands/SubscribeUserCommand.php
<?php namespace App\Commands;

use App\Subscriptions\MembershipType;
use Illuminate\Contracts\Queue\ShouldBeQueued;

class SubscribeUserCommand extends Command implements ShouldBeQueued
{
    public $userId;
    public $membershipType;

    public function __construct($userId, MembershipType $membershipType)
    {
        $this->userId = $userId;
        $this->membershipType = $membershipType;
    }
}
{% endhighlight %}

That is it! Well, actually you have to setup the queue config on <code>config/queue.php</code>, but I'm making a point here.

## Handling Events in background
As I said, it is also possible to handle events in background, let's see an example. Let's assume your User you have a *subscribe* named constructor on your model that builds the user instance and saves it (Eloquent/ActiveRecord). Your model should look like:

{% highlight php %}
// folder: app/User.php
<?php namespace App;

use Illuminate\Database\Eloquent\Model;
use App\Events\UserSubscribedEvent;
use App\Subscriptions\MembershipType;
use App\Payment\PaymentInterface;

class User extends Model
{
    // ...
    public function subscribe(MembershipType $membershipType, PaymentInterface $payment)
    {
        $payment->purchaseSubscription($this, $membershipType);

        $this->subscription()->create($membershipType->toArray());
        
        $this->raise( new UserSubscribedEvent($this->id, $membershipType) );

        return $this;
    }
    // ...
}
{% endhighlight %}

Your event class is just a DTO and looks like this:
{% highlight php %}
// file: app/Events/UserSubscribedEvent.php
<?php namespace App\Events;

use Illuminate\Queue\SerializesModels;
use App\Subscriptions\MembershipType;

class UserSubscribedEvent extends Event
{
    use SerializesModels;
        
    public $userId;
    public $membershipType;
            
    public function __construct($userId, MembershipType $membershipType)
    {
        $this->userId = $userId;
        $this->membershipType = $membershipType;
    }
}
{% endhighlight %}

Then you have a handler like so:

{% highlight php %}
// file: app/Handlers/Events/UserSubscribedEventHandler;
<?php namespace App\Handlers\Events;

use App\Events\UserSubscribedEvent;
use App\Mailers\UserMailer;

class UserSubscribedEventHandler
{
    public function __construct(UserMailer $mailer, UserRepository $userRepository)
    {
        $this->mailer = $mailer;
        $this->userRepository = $userRepository;
    }
    
    public function handle(UserSubscribedEvent $event)
    {
        $user = $this->userRepository->find($event->userId);
        $this->mailer->sendTo($user, $this->buildMessage($event->membershipType));
    }
                            
    // ... the buildMessage should be private or protected
}
{% endhighlight %}

To register your handler just go to <code>app/Providers/EventServiceProvider.php</code> and add your listener to the <code>$listen</code> property, like so:

{% highlight php %}
<?php namespace App\Providers;

use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider {

    /**
     * The event handler mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        \App\Events\UserSubscribedEvent::class => [ 
            \App\Handlers\Events\UserSubscribedEventHandler::class
        ]
    ];

}

{% endhighlight %}

This action is executed synchronously, it means that your user is waiting for the event handler to act before being redirected to the application.

To handle the event in background you just have to implement the same <code>Illuminate\Contracts\Queue\ShouldBeQueued</code> interface on your event handler class, like so:

{% highlight php %}
// file: app/Handlers/Events/UserSubscribedEventHandler;
<?php namespace App\Handlers\Events;

use App\Events\UserSubscribedEvent;
use App\Mailers\UserMailer;
use Illuminate\Contracts\Queue\ShouldBeQueued;

class UserSubscribedEventHandler implements ShouldBeQueued
{
    public function __construct(UserMailer $mailer, UserRepository $userRepository)
    {
        $this->mailer = $mailer;
        $this->userRepository = $userRepository;
    }
        
    public function handle(UserSubscribedEvent $event)
    {
        $user = $this->userRepository->find($event->userId);
        $this->mailer->sendTo($user, $this->buildMessage($event->membershipType));
    }
        
    // ... the buildMessage should be private or protected
}
{% endhighlight %}

Oh, by the way, you just give the event class to the event dispatcher, like so (using the Facade):

{% highlight php %}
<?php

// somewhere in your application
use App\Events\UserSubscribedEvent;

Event::fire(new UserSubscribedEvent($userId, $membershipType));
{% endhighlight %}

## Conclusion

That is it. To sum up, I like to think that Commands can change state, while Events just react to these state changes and if an Event handler has to change anything it MUST do it through Commands.

This command bus looks pretty cool. Fun fact: you can have multiple event listeners/handlers where some of them executes synchronously and others execute asynchronously. I loved it, to make it work before we had to have an event listener that add a job to the queue and then handle the event on the job handler. Now it's pretty damn simple.

## Useful resources

* [DevDiscussions - The Command Bus](https://www.youtube.com/watch?v=fbSYZFZCFS0)
* [Laracast about the Laravel 5 Command bus](https://laracasts.com/lessons/laravel-5-commands)
* [Laracast series about Commands and Domain Events](https://laracasts.com/series/commands-and-domain-events)
* [Task-based UI](https://cqrs.wordpress.com/documents/task-based-ui/)
* [Command Bus by Shawn McCool](http://shawnmc.cool/command-bus)
* [CRUD is an antipattern](http://verraes.net/2013/04/crud-is-an-anti-pattern/)