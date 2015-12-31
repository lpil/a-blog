---
layout: post
title:  "When things go wrong"
date:   2015-12-30
author: lpil
categories:
  - Web
  - Ruby
---

Failure is in unavoidable in programming, and possibly even more so when
programming for the web. Sometimes it's caused by our own logic errors that we
can patch and fix, and sometimes it's as out of our control. What happens if
we get rate limited by a third party API? Or some data that should be in the
database isn't? Or perhaps the heavens open and a bolt of lightning strikes
the hypervisior in which one of your applications is currently residing?

Whatever the reason, failure happens, and we just have to deal with it.

Say we have this simple method that fetches some data from a remote service.

{% highlight ruby %}
module Item
  def self.find(slug)
    RemoteService.get "/items/#{slug}"
  end
end

Item.find('my-id')
#=> #<struct Struct::Item value=500>
{% endhighlight %}

This is likely to fail. What happens when the service is not available due
to network problems? Most likely `RemoveService.get` is going to throw an
exception, causing the app to return a 500 to the user. Or the job to fail if
this is in a worker process.

Our first option is to catch the expected exception and return a value that
represents failure, in Ruby that's often `nil`. This keeps exceptions
exceptional, rather than making them part of our regular flow control.


{% highlight ruby %}
module Item
  def self.find(slug)
    RemoteService.get "/items/#{slug}"
  rescue ConnectionError
    nil
  end
end

Item.find('my-id')
#=> value OR nil
{% endhighlight %}


If we do this we're leaving it up to the caller to decide how to handle the
failure at a higher level. If this is a controller in our web app we may
decide to render a different page, or redirect the user elsewhere.

The downside here is that the caller has to be aware of the possibility of
failure, and handle it explicitly.


{% highlight ruby %}
@item = Item.find('my-id')
if @item
  render :show
else
  render 'errors/404'
end
{% endhighlight %}


And if the caller forgets to do this we'll end up with fail values being
passed into other functions, which may result in strange behaviour when they
receive input they were not capable of handling. When this happens it can be
very difficult to track the `nil` data base to the source, especially in
larger apps.

An alternative is to return a benign value that can be safely used in place of
the desired value. For example, if you're returning a collection, return an
empty collection. If it's unclear what value to return, let the caller decide
by having them pass in a default value as an argument.


{% highlight ruby %}
module Item
  def self.find(slug, default)
    RemoteService.get "/items/#{slug}"
  rescue ConnectionError
    default
  end
end

default = Struct::Item.new(100)
Item.find('my-id', default)
#=> value OR default
{% endhighlight %}


And as an bonus, when we force the caller to supply a default value it is made
extremely clear that this method may fail, and they have to handle this
failure at a higher level. If they wish they can simply pass in `nil`, and
they've recreated the previous functionality.

One more option here is to just let it crash. After an exceptional exception
is it always possible to conjure up replacement data and continue on
regardless? With user facing web application we want to avoid showing people
unsightly error views, but when we're changing state within our application
and our database we need to prioritise correctness. Better to let a task fail
than to introduce invalid data into our system, or risk an instability in one
component introduce instability into others.

When taking this approach it becomes even more important that you have tools
in place that give you good visibility of what's happening in your
application, and are alerted when these exceptional exceptions bubble up to
the top so that you can respond accordingly. At ROLI we've been using
[Sentry][sentry] for this, and it's proven itself to be a highly valuable
tool.

[sentry]: https://getsentry.com/welcome/

I've found it useful to get into the habit of asking myself questions like
"how is this likely to fail?", "what do we lose if it fails?", and "can we
continue if this fails?". If I can answer these questions, I've found it
becomes easier to decide what approach to take.

All this is relevant to most languages you're likely to use on the web, but if
you'd like to learn more about failure in the context of Ruby, Avdi Grimm's
[Exceptional Ruby][book] is a great place to start.

[book]: http://exceptionalruby.com/
