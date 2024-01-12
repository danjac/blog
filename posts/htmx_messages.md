---
title: Dynamic messages with HTMX and Alpine.js
description: Using Django messages with HTMX and Alpine.js
date: 2024-01-09
tags:
  - django
  - python
  - web development
  - htmx
  - alpinejs
layout: layouts/post.njk
---

A common requirement is returning "flash" messages from the server when the user successfully - or unsuccessfully - completes an action. The [Django messages framework](https://docs.djangoproject.com/en/5.0/ref/contrib/messages/) provides a simple interface to render messages. However, it's not so clear how to do this with HTMX.

## Getting started

To enable messages you need to ensure that [the messages framework is enabled](https://docs.djangoproject.com/en/5.0/ref/contrib/messages/#enabling-messages) in your Django project.

We'll also need the [django-htmx](https://pypi.org/project/django-htmx/) library:

```bash
  pip install django-htmx
```

[Follow the instructions](https://django-htmx.readthedocs.io/en/latest/installation.html) on adding `django-htmx` to your project.

## Our application

For our example, we have these two Django views:


```python
  from django.contrib import messages
  from django.http import HttpRequest, HttpResponse
  from django.shortcuts import render


  def index(request: HttpRequest) -> HttpResponse:
      """Front page"""
      return render(request, "index.html")

  def send_message(request: HttpRequest) -> HttpResponse:
      """Just sends an OK message back to the user."
      messages.success(request, "All OK!")
      return render(request, "_send_button.html", {"message_sent": True})
```

Our very simple index template `index.html` looks like this:

```jinja2
{% raw %}
  {% load static %}
  <!DOCTYPE html>
  <html lang="en">
      <head>
          <title>Messages demo</title>
          <meta name="description" content="HTMX django messages demo">
          <meta name="keywords" content="htmx django">
          <link rel="stylesheet" href="{% static 'index.css' %}">
          <script src="https://unpkg.com/htmx.org@1.9.10"
                  integrity="sha384-D1Kt99CQMDuVetoL1lrYwg5t+9QdHe7NLX/SoJYkXDFfX37iInKRy5xLSi8nO7UC"
                  crossorigin="anonymous"></script>
          <script defer
                  src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
      </head>
      <body>
          {% include "_messages.html" %}
          <main>
              {% include "_send_button.html" %}
          </main>
      </body>
  </html>
{% endraw %}
```

This page includes the CDNs for [HTMX](https://htmx.org) and [Alpine](https://alpinejs.dev) and a couple `include` templates.

The template `_send_button.html` looks like this:

```jinja2
{% raw %}
  <button hx-post="{% url 'send_message' %}"
          hx-swap="outerHTML"
          hx-target="this"
          hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'
          class="btn btn-lg">
      {% if message_sent %}
          Send again
      {% else %}
          Send message
      {% endif %}
  </button>
{% endraw %}
```

The button triggers an HTMX action: it will send a POST to our `send_message` view, which should then just re-render this template, swapping out the contents when a message is sent. Note that we need to include the CSRF token in the header or Django will respond with a `403 Forbidden` response.

Finally we need to render the messages in `_messages.html`:

```jinja2
{% raw %}
  <div id="messages" class="messages">
    {% if messages %}
        <ul>
            {% for message in messages %}
                <li class="message message-{{ message.tags }}"
                    role="alert">{{ message.message }}</li>
            {% endfor %}
        </ul>
    {% endif %}
</div>
{% endraw %}
```

This renders the messages. The `messages` should be available in the template context if you have included the correct middleware and context processor when setting up the Messages framework (see above), so when calling `messages.success(request, "All OK!")` in our view, it should be rendered correctly.

## Rendering messages in HTMX

So far so good! However, when we run the application and click the "Send message" button, although the button text changes, we don't see any messages. What's wrong?

The problem is that HTMX will only render content into the specified `hx-target`, in this case the button itself, and our view only re-renders the `_send_button.html` template. Somehow we need to inject the messages into our `POST` response, without re-rendering the entire page.

However HTMX has an answer for this problem: `hx-swap-oob` ("OOB" means "Out of Band"). This [handy attribute](https://htmx.org/attributes/hx-swap-oob/) allows us to "side-load" additional snippets of content into our response, allowing us to update any other part of the page in addition to whatever we want to inject into the DOM node specified by `hx-target`.

Let's add this to our `_messages.html` template:

```jinja2
{% raw %}
  <div id="messages" class="messages"
    {% if hx_oob %}hx-swap-oob="true"{% endif %}>
    {% if messages %}
        <ul>
            {% for message in messages %}
                <li class="message message-{{ message.tags }}"
                    role="alert">{{ message.message }}</li>
            {% endfor %}
        </ul>
    {% endif %}
</div>
{% endraw %}
```

Note that when using `hx-swap-oob` you should always include the unique `id` of the node you want to swap.

However, we somehow need to render this template to our final response. Here's one way to do it:

```python

  # add this to imports:
  from django.template.loader import render_to_string

  def send_message(request: HttpRequest) -> HttpResponse:
      """Just sends an OK message back to the user."
      messages.success(request, "All OK!")
      response = render(request, "_send_button.html", {"message_sent": True})
      response.write(
        template_name="_messages.html",
        context={"hx_oob": True},
        request=request,
      )
      return response
```

_Et voilà_:

![Screen showing button and 'All OK!' message](/img/django-messages.png)

When clicking the button, you should now see the 'All OK!' message at the top of the screen. We append another rendered template to our Django `HttpResponse` instance using `HttpResponse.write()`, and insert the `hx-swap-oob` attribute. HTMX will then inject the rendered messages into the `messages` node.

This works well, but in a large application writing this boilerplate each time is quite painful. We could wrap this into a function, for example:

```python
  def inject_messages(response: HttpResponse) -> HttpResponse:
      response.write(
        template_name="_messages.html",
        context={"hx_oob": True},
        request=request,
      )
      return response
```

And then in our view:

```python
  def send_message(request: HttpRequest) -> HttpResponse:
      """Just sends an OK message back to the user."
      messages.success(request, "All OK!")
      return inject_messages(render(request, "_send_button.html", {"message_sent": True}))
```

This could be done slightly better using a decorator:

```python
  import functools

  from django.contrib.messages import get_messages

  _hx_redirect_headers = frozenset(
        {
            "HX-Location",
            "HX-Redirect",
            "HX-Refresh",
        }
  )

  def inject_messages(view: Callable) -> Callable:
      """Injects HTMX messages into response. """
      @functools.wraps(view)
      def _wrapper(request: HttpRequest, *args, **kwargs) -> HttpResponse:
          response = view(request, *args, **kwargs)
          if not request.htmx:
              return response

          if set(request.headers) & _hx_redirect_headers:
              return response

          if get_messages(request)
              response.write(
                template_name="_messages.html",
                context={"hx_oob": True},
                request=request,
             )
          return response
      return _wrapper
```

And our view again:

```python

  @inject_messages
  def send_message(request: HttpRequest) -> HttpResponse:
      """Just sends an OK message back to the user."
      messages.success(request, "All OK!")
      return render(request, "_send_button.html", {"message_sent": True})
```

There is a bit more logic in our decorator. We don't want to append the messages if we are doing a full page render or redirect, as the messages will get rendered anyway. Therefore we check if the request has an `HX-Request` header (it won't if it's a non-HTMX request) or if it has an HTMX header that tells the browser to reload or re-render the entire page. See [here](https://htmx.org/reference/#request_headers) for a reference of the various HTMX request headers.

Note that the `django-htmx` middleware adds the `htmx` attribute to our `request`, and `request.htmx` will always be `False` (or false-y) if the `HX-Request` header is absent.

We call `get_messages()` to check if we do have any messages. This function allows us to check if there are any messages, without removing them from the session.

This is an improvement, but still bug-prone: we have to remember to include this decorator whenever we add a message in our view, and sooner or later we'll forget to do so (maybe when adding success or failure messages to an existing view) and wonder why our message doesn't show up.

Instead we can use middleware. Middleware executes with every request, so we can be assured this will work whether the view has messages or not. There's a bit of extra overhead compared to a decorator as it is always going to be called with every view, but in this case we don't have any expensive calls (like database operations) so it's probably worth it to avoid this particular source of bugs.

Thankfully, we've written most of the logic in our decorator, so it's not much more work to turn it into middleware:

```python
  class HtmxMessagesMiddleware:
      """Adds messages to HTMX response"""

      _hx_redirect_headers = frozenset(
          {
              "HX-Location",
              "HX-Redirect",
              "HX-Refresh",
          }
      )

      def __init__(self, get_response: Callable[[HttpRequest], HttpResponse]):
          self.get_response = get_response

      def __call__(self, request: HttpRequest) -> HttpResponse:
          """Middleware implementation"""
          response = self.get_response(request)

          if not request.htmx:
              return response

          if set(response.headers) & self._hx_redirect_headers:
              return response

          if get_messages(request):
              response.write(
                  render_to_string(
                      template_name="_messages.html",
                      context={"hx_oob": True},
                      request=request,
                  )
              )

          return response
```

You will need to add this middleware to the `MIDDLEWARE` list in your settings. It should be placed after `django_htmx.middleware.HtmxMiddleware` and `django.contrib.messages.middleware.MessageMiddleware`.

Now we can revert back to our original version of the view, as we don't need that extra logic for rendering the messages:

```python
  def send_message(request: HttpRequest) -> HttpResponse:
      """Just sends an OK message back to the user."""
      messages.success(request, "All OK!")
      return render(request, "_send_button.html", {"message_sent": True})
```



There is one annoyance remaining. When rendering the message, it just sticks around at the top of the screen (or wherever we put our messages). In a traditional server-rendered application this is less of a problem: Django messages are removed from the session when rendered, so once you reload the page (by navigating to a link, for example) the message goes away. In an HTMX-enhanced site however the whole point is to avoid reloading the page as much as possible, by just re-rendering parts of the page in response to server-side actions. But that means the messages aren't removed in between requests.

This is more of a client-side than server-side problem. If you remember earlier we included the Alpine.js CDN as well as HTMX, because we anticipated the need for doing small client-side interactions. Let's go back to our `_messages.html` template and enhance it with Alpine.js attributes:

```jinja2
{% raw %}
  <div id="messages"
       class="messages"
       {% if hx_oob %}hx-swap-oob="true"{% endif %}>
      {% if messages %}
          <ul>
              {% for message in messages %}
                  <li class="message message-{{ message.tags }}"
                      role="alert"
                      x-data="{show: true}"
                      x-show="show"
                      x-transition
                      x-init="setTimeout(() => show = false, 2000)">{{ message.message }}</li>
              {% endfor %}
          </ul>
      {% endif %}
  </div>
{% endraw %}
```

The `x-data` attribute sets a variable `show`.  The `x-show` attribute will hide the element if `show` is `false`.

We use `x-transition` to make this a little smoother (with some more CSS you can [tweak](https://alpinejs.dev/directives/transition) the transition effects).

Finally, with `x-init` we trigger the immediate behaviour when the `<li>` element is rendered: it will set a timeout that will set the `show` flag to `false` after a couple seconds.

Now, when our messages are rendered, they will disappear automatically after a little while. Note that this functionality will work whether we are rendering the messages in our HTMX response, or after a full-page refresh or redirect.

Full code for this article can be found [here](https://github.com/danjac/django_htmx_messages).

