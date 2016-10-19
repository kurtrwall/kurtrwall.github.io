---
layout: post
title:  "How Wagtail changes Django: Part one"
description: >
    Django projects using Wagtail are still just Django, but there are a few
    key differences that should be understood when using Wagtail. The is the
    first post of a series.
permalink: /blog/wagtail-changes-django-pt1.html
categories: ['wagtail', 'django', 'python', 'cms']
author: Kurt Wall
disqus: True
---

[Wagtail is a Django CMS](https://wagtail.io/) that an increasing amount Python web developers are using for their content management needs. Wagtail operates more like a CMS _framework_ instead of a fully featured CMS because it gives you the tools and features one might expect without making many decisions for you about how the content is structured. This provides a delicate balance between opinion and freedom that developers are always searching for.

The opinions and structure that Wagtail is built with are few but important to understand. In this series, we'll be stepping through some of them and how they make a difference to a Django project.

# The view layer

Let's first follow how a request normally flows once it hits the Django application at a high level:

1. The WSGIRequest object hits Django middleware.
2. The request is used to find the appropriate view function to call (URL resolution).
3. The view function is called with the request and it returns a response.

Again, this is a high level flow and it varies based on how you've configured your project, like the middleware.

## Generic view function
Our area of concern is step three, where the view function is called. Traditionally, this this where a Django developer would implement their own view function or a fancy [class-based view](https://docs.djangoproject.com/en/1.10/topics/class-based-views/). However, Wagtail has a different approach.

{% highlight python %}
from wagtail.wagtailcore import views

serve_pattern = r'^((?:[\w\-]+/)*)$'
urlpatterns = [
	...,
	url(serve_pattern, views.serve, name='wagtail_serve')
]
{% endhighlight %}

This code snippet was taken from [wagtailcore/urls.py](https://github.com/torchbox/wagtail/blob/v1.6.3/wagtail/wagtailcore/urls.py), and it illustrates how Wagtail is using a regex with a non-named pattern match for any request path it comes across with the same `views.serve` method. There is already somewhat of a difference here because instead of handling different paths with different views, we're using a generic pattern with a generic view function. Moving on, we have some code from `wagtailcore/views.py`, the view function that's used above:

{% highlight python %}
def serve(request, path):
    if not request.site:
        raise Http404

    path_components = [component for component in path.split('/') if component]
    page, args, kwargs = request.site.root_page.specific.route(request, path_components)

    for fn in hooks.get_hooks('before_serve_page'):
        result = fn(page, request, args, kwargs)
        if isinstance(result, HttpResponse):
            return result

    return page.serve(request, *args, **kwargs)
{% endhighlight %}

There are few things going on there so let's break it down:

### Check for `request.site` and if it's falsey, then `raise Http404`

{% highlight python %}
if not request.site:
	raise Http404
{% endhighlight %}


> What? I've never heard of `request.site` before. Where did this come from?


That's where the wagtailcore app's [middleware](https://github.com/torchbox/wagtail/blob/v1.6.3/wagtail/wagtailcore/middleware.py) kicked in. In the beginning, we had 3 steps for a "normal" flow and mentioned that it varies based on how you set up the project. Well, Wagtail adds the `process_view` middleware that gets executed before the view function is called. Without getting too far into it, the `site` object is how Wagtail finds the correct page to use for the response.

### Break down the path and feed it into the root page's `route` method

{% highlight python %}
path_components = [component for component in path.split('/') if component]
page, args, kwargs = request.site.root_page.specific.route(request, path_components)
{% endhighlight %}

After getting a list of the parts to the path, it uses the `site` object's `root_page.specific` ([see here for details on `specific`](http://docs.wagtail.io/en/v1.6.3/reference/pages/model_reference.html#wagtail.wagtailcore.models.Page.specific)) to call the `route` method. The `route` method recursively searches for the requested page and returns a [RouteResult](https://github.com/torchbox/wagtail/blob/v1.6.3/wagtail/wagtailcore/url_routing.py) if it's found and the page is published. [The code for `route` lives here.](https://github.com/torchbox/wagtail/blob/v1.6.3/wagtail/wagtailcore/models.py#L657)

### Call all registered function hooks

{% highlight python %}
for fn in hooks.get_hooks('before_serve_page'):
	result = fn(page, request, args, kwargs)
	if isinstance(result, HttpResponse):
		return result
{% endhighlight %}

Wagtail has a [hook system](http://docs.wagtail.io/en/v1.6.3/reference/hooks.html) in which a developer may add a function that will be called when loops like this exist in the code. This will call all functions registered with that hook and it returns the result if it's an HttpResponse object.

### Call the `page` object's `serve` method and return the result

{% highlight python %}
return page.serve(request, *args, **kwargs)
{% endhighlight %}

This is where Wagtail hands off the request to the Page class to handle the rest. [The `serve` method](https://github.com/torchbox/wagtail/blob/v1.6.3/wagtail/wagtailcore/models.py#L758) does a handful of things:

1. Checks to see if we're previewing the page and sets a context variable if so.
2. Calls the Page method [`get_template`](https://github.com/torchbox/wagtail/blob/v1.6.3/wagtail/wagtailcore/models.py#L752) to get the appropriate template for rendering.
3. Calls the Page method [`get_context`](https://github.com/torchbox/wagtail/blob/v1.6.3/wagtail/wagtailcore/models.py#L745) to get the context for the template for rendering.
4. Returns a [TemplateResponse](https://docs.djangoproject.com/en/1.10/ref/template-response/#templateresponse-objects), like so:
{% highlight python %}
return TemplateResponse(
    request,
    self.get_template(request, *args, **kwargs),
    self.get_context(request, *args, **kwargs)
)
{% endhighlight %}


---

## What do we gain from this?

That's an important question. This seems like we've added **a lot** of complication, so we should know why.

1. The catch-all URL pattern allows us to create a page URL structure without having to hardcode the URL's we use, while allowing us to keep other URLs unaffected.
2. Adding a `hooks.get_hooks()` before serving the page gives us an extra layer of functionality (akin to another middleware) that could be useful depending on the situation.
3. The generic view function utilizes class methods which allows developers to extend the behavior in an OO fashion. 

There are probably many other benefits that I haven't thought of, but these are a few big points.

Thanks for checking this out. If you're looking for more Wagtail articles, I'd recommend [Joss Ingram's blog](https://jossingram.wordpress.com/category/wagtail-2/) where he has loads of great stuff. As for me, I'll be continuing this series with data migrations.
