---
layout: post
title: "Float like a butterfly, sting like a bee"
tags: [code, c++, precision]
---

Floats are devious
------------------

Yea, we all know that.

You [shouldn't compare doubles to floats](http://goo.gl/DWM9K0), you [shouldn't even compare a
float and a literal](http://goo.gl/gQMY08) (which is pretty much the same), and in general you
should be aware of the precision limitations leading to ["incorrect" floating point
arithmetic](http://goo.gl/q37ERI).

Is it what it is?
-----------------

Recently I stumbled upon a piece of code that ridiculed c++'s `float` like I've never seen before,
and when I say 'stumbled upon' I mean - unit tests actually failed. It was a simple arithmetic
problem that I managed to simplify to the following:

{% highlight c++ %}
#include <iostream>

using namespace std;

int main() {
    unsigned int n30 = 30;
    float num = (float)3 / 10;

    if (num <= n30 / (float)100) {
        cout << "It is what it is." << endl;
    } else {
        cout << "If you try to fail, and succeed, which have you done?" << endl;
    }

    return 0;
}
{% endhighlight %}

Compiling & running this piece of code on a 32-bit redhat using g++ 4.4.7 **actually fails**!

This example doesn't compare floats to doubles, nor does it compare floats to non-float literals (if
you're thinking `(float)100` is like comparing to non-float literals you're welcome to replace it
with `100.0f` - it will fail nevertheless).

* Note that on a 64-bit redhat (still g++ 4.4.7) it passes even though `float`'s size is the same.

On top of that you'll probably be surprised to know that if we were to put `n30 / (float)100` in a
new float variable and compare `num` with that new variable the comparison will be correct.

Well, you've been warned.
