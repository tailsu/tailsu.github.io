---
published: true
title: ASP.NET MVC-style routes in Django
layout: post
---
Using regexes to describe Django routes gets really hairy as soon as you need to extract some data from the URL and map it onto the view's parameters. I really miss the route syntax from ASP.NET MVC.

Here's a sample `urlpatterns` list and the view function showing the problem:

{% highlight python %}
def thumb(request, asset_id):
	...
...

urlpatterns = [
    url(r'^assets/(?P<asset_id>[^/]+)/thumb$', api_assets.thumb),
    url(r'^assets/(?P<asset_id>[^/]+)', api_assets.content),
]

{% endhighlight %}

What I'd like to have is to just specify the mapping of url parts to parameter names and get straight into my view function. I don't care about validation within the regex - I'm much more comfortable doing that in Python, thank you very much. Shouldn't be too hard.

First, define a helper function that will take care of the conversion:

{% highlight python %}

def route(pattern, *args):
    return url('^' + re.sub(r'\{(\w+)\}', r'(?P<\1>[^/]+)', pattern) + '$', *args)

{% endhighlight %}

The function looks for patterns like `{foo}` and substitutes them with named regex match groups. The resulting regex is passed to `url()`.

Now, rewrite the `urlpatterns` list to use the new function:

{% highlight python %}
urlpatterns = [
    route('assets/{asset_id}/thumb', api_assets.thumb),
    route('assets/{asset_id}', api_assets.content),
]
{% endhighlight %}

Aaah, much better! Don't forget to validate the parameters!