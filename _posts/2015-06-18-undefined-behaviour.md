---
layout: post
title: "Undefined Behaviour - Foo Ex Nihilo"
tags: [code, c++, stl]
---

Backgroud
---------

We all know of the dangerous dark path named *Undefined Behaviour*.

It happens at the edges of a standard when said standatd does not define (or defines to NOT define) a
certain use case. The reason for this is mainly optimization - When some edge cases are not defined,
the library / compiler is free to ignore them in any way its developers see fit, and thus create
shorter and/or more efficient assembly.

Despite what most might think, avoiding undefined behaviour **completely** is not as easy as it
seems. If this statement sounds bogus to you, I implore you to check out this
[stackoverflow question](http://goo.gl/Bcqi3y) explaining how excruciating it could be to convert a
double (ANY double) to an integer without invoking undefined behaviour **at all**. Note that it took
more than a month for this question to be finally answered by the op himself.

Case Study
----------

Programmers are taught to fear undefined behaviour (and rightly so), and while it might be interesting
to explore ways to avoid undefined behaviour, it would be a lot more fun to see what happens if we
simply, well, don't.

This case study is the reason I decided to write this post, and I found it interesting because I
wasn't really looking for it. It didn't originate in forcing a way to reach a scenario-specific
compiler-specific undefined behaviour. It actually came from trying to understand `std::queue<>`.

In my work I got to a point where I needed to use a thread safe queue, and while wrapping STL's
`std::queue<>` with a thread safe implementation I called SafeQueue, I wondered what would happen if
a thread tries to `front()` or `pop()` an empty `std::queue<>`.

I figured that if stl's queue throws an exception for such a case, then it will be thrown through my
SafeQueue which is good enough. However, if it doesn't, I have a real problem since while a single
thread using a queue can call `empty()` and be sure that the queue is indeed empty after the call,
in multi-threaded environment the moment the `empty()` method returns with an answer, this answer
does not qualify anymore and might be wrong (imagine a context switch right after `empty()`
returns).

So instead of checking the standard I decided to test it myself - As it turns out, queue doesn't
throw an exception, and obviously doesn't bother to check if it's empty. At this point I assumed
that calling `front()` or `pop()` on an empty queue is indeed undefined. [It turns out I was correct
when I finally checked it out and stumbled upon [this question](http://goo.gl/yajFzI)]. The fact
that `front()` didn't throw an exception stimulated my thought to wondering what would `front()`
return for an object T.

So I created the following example:

{% highlight c++ %}
#include <iostream>
#include <queue>

using namespace std;

struct Foo {
    Foo() : x(3) {
        cout << "in 0-arg c'tor" << endl;
    }
    Foo(const Foo &f) : x(f.x) {
        cout << "in copy c'tor" << endl;
    }
    int x;
};

int  main() {
    queue<Foo> q;
    Foo f = q.front();

    cout << f.x << endl;

    return 0;
}
{% endhighlight %}

*For the record, I tried this on gcc 4.4.7 (Centos 6.4)*

The question that was most interesting to me was what value could `f.x` have? My first example didn't
include prints in the c'tors (and didn't even include a non-default copy c'tor). I added the copy
c'tor and the prints after this program **printed 0**!

Weird, huh?<br>
Since `Foo`'s `x` is initialized to 3 in the c'tor.
When I ran the full example (with copy c'tor and c'tors prints) I got the following result:

>in copy c'tor<br>
>0

What this meant is that the copy c'tor was called for an object `Foo` without ever calling `Foo`'s c'tor!

*Ergo, a `Foo` was present without it ever being created - [Creation ex nihilo](https://en.wikipedia.org/wiki/Ex_nihilo)*.
