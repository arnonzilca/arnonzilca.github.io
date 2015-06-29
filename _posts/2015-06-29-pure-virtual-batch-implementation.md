---
layout: post
title: "Pure virtual batch implementation"
tags: [code, c++, templates]
---

Itch
----

Are you familiar with the itchy feeling caused by inheriting the same class,
and implementing the same pure virtual function for several common logics?<br>
It's somewhat annoying that you have to inherit a class over & over again,
just for this one functor (or functor-like) function, that contains the important stuff.

You can get a sense of it in this example of a subscription mechanism:

{% highlight c++ %}
#include <memory>

template <typename TMessage>
class Subscription {
public:
    virtual ~Subscription() {
    }

    // user method to overwrite
    virtual bool handle(std::shared_ptr<const TMessage> message) = 0;
};
{% endhighlight %}

Now, Imagine that a certain module we want to implement needs to subscribe to several different
messages - let's say 3. The *straight forward* way to do it would be to create 3 classes - One for
each implementation of a subscription. Then we would have to create a new class representing our
module itself, which will contain those 3 subscription classes

{% highlight c++ %}
class PeaceSubscription : public Subscription<Peace> {
public:
    virtual bool handle(std::shared_ptr<const Peace> message) {
        // handling a message of peace
    }
};

class LoveSubscription : public Subscription<Love> {
public:
    virtual bool handle(std::shared_ptr<const Love> message) {
        // handling a message of love
    }
};

class HopeSubscription : public Subscription<Hope> {
public:
    virtual bool handle(std::shared_ptr<const Hope> message) {
        // handling a message of hope
    }
};

class HippieModule {
protected:
    PeaceSubscription peaceSubscription;
    LoveSubscription loveSubscription;
    HopeSubscription hopeSubscription;
};
{% endhighlight %}

Remedy
------

You feel that itch?
It could be extremely comfortable if we could just implement those 3 (functor-like) functions inside
our module. <br>
On the one hand we know we can't refrain from inheriting those 3 classes. However, on the other hand
multiple inheritance will be dead ugly, and might not work if we want to subscribe to the same
message type several times, or if `Subscription`'s is not a pure virtual class (aka Interface
in other languages).

What we *can* do, is use a **template pointer to function**.<br>
The idea would be to 'hide' the inheritance with template instantiation and implement all 3
functions inside our module.

Consider this helper class:

{% highlight c++ %}
template <typename TMessage, typename TCaller, bool(TCaller::*TMethod)(std::shared_ptr<const TMessage>)>
class SubscriptionDelegate : public Subscription<TMessage> {
public:
    SubscriptionDelegate(TCaller *c) :
        caller(*c) {
    }

    virtual bool handle(std::shared_ptr<const TMessage> message) {
        return (caller.*TMethod)(message);
    }

protected:
    TCaller &caller;
};
{% endhighlight %}

This class inherits `Subscription` and aside from taking a template `TMessage` (like the
`Subscription` class), this class takes a template `TCaller` which will be the class containing
the implementation of `handle(...)`, and a template function of `TCaller` with the same signature
as our `handle(...)` method (taking a `shared_ptr` of a `const TMessage`).<br>
When handle of this delegate class is called, it call will call `TCaller.TMethod()`.

This allow us to implement our module the following way:

{% highlight c++ %}
class HippieModule {
public:
    HippieModule() :
        peaceSubscription(new SubscriptionDelegate<Peace, HippieModule, &HippieModule::handlePeace>),
        LoveSubscription (new SubscriptionDelegate<Love,  HippieModule, &HippieModule::handleLove>),
        HopeSubscription (new SubscriptionDelegate<Hope,  HippieModule, &HippieModule::handleHope>) {
    }

    virtual bool handlePeace(std::shared_ptr<const Peace> message) {
        // handling a message of peace
    }
    virtual bool handleLove(std::shared_ptr<const Love> message) {
        // handling a message of love
    }
    virtual bool handleHope(std::shared_ptr<const Hope> message) {
        // handling a message of hope
    }

protected:
    Subscription *peaceSubscription;
    Subscription *loveSubscription;
    Subscription *hopeSubscription;
};
{% endhighlight %}

Note that:

* we *could* create the subscriptions as non-pointer members, but we would have to paste
their long definition inside the class (instead of in the c'tor which is usually in the `cpp` file
rather than the `h`) - it's a matter of taste.

* Since a different class will be instantiated for *every* different subscription function in the
module, we could implement a subscription to the *same* message type several times - we just need to
name the handling functions differently.
