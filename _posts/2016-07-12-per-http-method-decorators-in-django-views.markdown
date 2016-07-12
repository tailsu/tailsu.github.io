---
published: false
title: Per-HTTP method decorators in Django views
layout: post
---
Django views are specified per URL using `urlpatterns` pointing to their respective view functions (or classes). If a view function has a different behavior depending on the HTTP request method, then it is expected that the view will have a `if-elif-...` bunch of blocks to do the right thing. On an unrelated note, Django provides us with a bunch of view decorators that can modify a view's behavior, like checking for user authorization, permission, allowed HTTP methods and whatnot. Together, one such view would look like this:

{% highlighter python %}

@require_http_methods(['GET', 'DELETE'])
@csrf_protect
def content(request):
	if request.method == 'GET':
		return get_this()
	elif request.method == 'DELETE'
		return login_required(lambda r: return delete_this())(request)

{% endhighlighter %}

The problem is that these decorators are applied *per view*, i.e. per URL and not per view/method pair. Yet I want to use one set of decorators for my GET method and other decorators for my DELETE method of the same REST URL. I could do all checks manually in the view body, but that defeats the purpose of having nicely visible view decorators in the first place. Moreover, calling the decorators in-place, like in the above code is ugly beyond measure. Notice also the duplication of method names in the `require_http_methods` decorator and then the view's body. Let's see if we can do better than that...

At the core of the solution stands the fact that decorators can be applied only to a function. Essentially we need to define a separate function for each HTTP method within a view.

{% highlighter python %}

@csrf_protect
def content(request):
	def GET(request):
		return get_this()
		
	@login_required
	def DELETE(request):
		return delete_this()
		
	return ...
	
{% endhighlighter %}

This already looks much more readable! Now we need to make it work. The key is to add a dispatcher that will take the request, take all method-specific functions and then call the one pertaining to the request's method.

Here's the magical function:

{% highlighter python %}

from django.views.decorators.http import require_http_methods
def dispatch_method(request, lcls, *args):
    allowed_methods = [m for m in ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'] if m in lcls]
    return require_http_methods(allowed_methods)(lambda r, *args: lcls[r.method](r, *args))(request, *args)

With this function in hand, we can modify the final return statement in our view:

{% highlighter python %}

@csrf_protect
def content(request):
	def GET(request):
		return get_this()
		
	@login_required
	def DELETE(request):
		return delete_this()
		
	return dispatch_method(request, locals()) # <-- dispatch
	
{% endhighlighter %}
	
`dispatch_method` method takes the current request and the dictionary of locals of the calling function. It then retrieves the function from the locals that is named as the request's method and invokes passing the request and any additional arguments to it. The method-specific functions themselves can have any additional decorators, as can the view function itself. The `require_http_methods` decorators is built on-the-fly from the available functions and automatically returns HTTP 405 if the corresponding function is not defined.

