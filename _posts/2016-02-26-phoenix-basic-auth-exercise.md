---
layout: post
date: 2016-02-26 16:00:00 CET
title: Phoenix basic auth exercise
tags: [Elixir, Phoenix, tutorial]
comments: true
---

I started learning [Phoenix Framework](http://www.phoenixframework.org/) recently. It's simplicity is striking! I love it. I think it is a great framework to learn and to play with. Today I would like to show you how to implement basic authorization in Phoenix. It is not part of the Phoenix framework and this is one of the features you would like to use from time to time. It's almost never the case that you want to keep basic auth for a longer period in production, but it's definitely good to begin with.

It is quite easy to integrate Phoenix with libraries that already exist like [basic_auth](https://github.com/CultivateHQ/basic_auth) or simply search for what might [hex.pm](https://hex.pm/) offer you. Still - I believe implementing basic auth on your own might be actually a good exercise and potentially you could end up with a bit more knowledge on how to build [plugs](http://www.phoenixframework.org/docs/understanding-plug).

Let's start with building a dummy app to play with...
{% highlight bash %}
$ mix phoenix.new basic_auth_exercise
{% endhighlight %}

If you don't have phoenix installed you might want to read the [Phoenix installation](http://www.phoenixframework.org/docs/installation) article.

## Specification

Take a look at Wikipedia and [protocol section](https://en.wikipedia.org/wiki/Basic_access_authentication#Protocol) of Basic access authentication article. It expects us to do the following on the server side:

1. throw HTTP 401 Unauthorized status when the request is unauthenticated
2. send `WWW-Authenticate: Basic realm="Thou Shalt not pass"` header in the response

And from the client side there should be done only one thing:

1. Set the `Authorization: Basic QWxhZGRpbjpPcGVuU2VzYW1l` header in order to authenticate the user

The last part of the authorization header string is Base64 encoded username and password joined by the semicolon.

## Implementation

Let's open the `test/controllers/page_controllers_test.exs` - I think it's perfect to put our tests there.
<script src="https://gist.github.com/RadekMolenda/5ff181cfcb1d9d93206b.js"></script>

Nothing really interesting - we just have changed the description and change the expectations to match 401 unauthorized status we should get from the request. We also don't expect 401 will be HTML response any more.

Now let's fix the test. Before doing it let's stop and think for a while how we would like to do - it. Maybe it would be better if we actually take a look at some article that says something [about authorization](http://www.phoenixframework.org/v0.11.0/docs/understanding-plug). In there we see authorization is done as a function plug which is a part of the controller pipeline, but I think we want to make basic_auth a bit more generic and instead of controller it should go to the router level. Let's put the following line to the `:browser` pipeline in `web/router.ex` file.
<script src="https://gist.github.com/RadekMolenda/2549e24da402419da74f.js"></script>
Of course the tests would crash now as `BasicAuth` module is not even defined.
You can see that we will implement a plug module. It will accept two params: `:username` and `:password`.

Let's fix the `mix test` command by adding `web/basic_auth.ex` file (the tests would still be red thou).
<script src="https://gist.github.com/RadekMolenda/5dd6148f74889eda5b33.js"></script>
You can see that plug is a very simple module it only expects to define two functions: `init/1` and `call/2`. The `init/1` can be used for some heavy lifting jobs at the compile time while `call/2` should be fast and simple as it is used in a runtime. We will use `init/1` function to validate username and password existence. Authorization will be implemented in `call/2`.

We are ready to make the tests green again.
<script src="https://gist.github.com/RadekMolenda/b734c577e2598a825566.js"></script>
Cool we used `send_resp(conn, 401, "unauthorized")` in order to set the 401 status and body to "unauthorized". `halt(conn)` was used to set the halt connection flag to true (this "informs" other functions the connection has been stopped).

Let's write another test.
<script src="https://gist.github.com/RadekMolenda/69856bc270cc78c58386.js"></script>
This one gives `(RuntimeError) expected response with status 200, got: 401`.

How do we know `dXNlcjpzZWNyZXQ=` is a valid authorization string? The answer is simple, let's open the `iex` session and try the following:
{% highlight elixir %}
iex(1)> Base.encode64("user:secret")
"dXNlcjpzZWNyZXQ="
{% endhighlight %}
This string needs to be valid!

It's time to fix the tests again.
<script src="https://gist.github.com/RadekMolenda/71ba06c0da8f93dd103a.js"></script>
Quite a few changes here:

1. we pattern match on opts argument to get `username` and `password` quickly
2. we are fetching the "authorization" header using `get_req_header(conn, "authorization")`. If the header is set and matches to `["Basic " <> auth]` we let the connection through otherwise we send 401 status

Lets write another failing test scenario. For example we shouldn't be authorized if authorization string is set to "Basic I like turtles".
<script src="https://gist.github.com/RadekMolenda/cb0c3fd5e2dc02afb991.js"></script>
This test fails unfortunately, but it's not difficult to fix it.
<script src="https://gist.github.com/RadekMolenda/377c8a42c333579ed035.js"></script>
The tests are green again! The most important is the `auth == encode(username, password)` line where we are comparing the auth token with our secretly encoded string. Connection is being returned if there is a match, otherwise we return the `unauthorized(conn)` function which has been introduced to comply with DRY principle and we are still keeping pattern matching on "authorization" header without any change.

Lovely - we are now letting the authenticated users through and not authenticated users are getting 401 response status. We can test it by starting the server and navigating to `http://localhost:4000/` in the web browser. Unfortunately even though we are getting the "unauthorized" message the browser doesn't prompt with dialog to fill username and password.

There is one thing missing - a server response header `WWW-Authenticate` which should be sent when the user is not authenticated. Lets fix this by adding the relevant assertions to test cases.
<script src="https://gist.github.com/RadekMolenda/e70659a7617f70bbaae3.js"></script>

The following implementation makes the tests green again
<script src="https://gist.github.com/RadekMolenda/3c49046ea97b97cfde97.js"></script>
The code speaks for itself here I think `put_resp_header(conn, "www-authenticate", @realm)` sets the correct response header.

Navigating to `http://localhost:4000` after starting the server should result in following:

![We are safe and secure](/img/authenticate.png =400x "We are safe and secure")

## Testing with Elixir is fun
This step is completely optional. We would like to test and implement one more thing. The code should fail when we don't pass `:username` or `:password` to our `BasicAuth.init/1` function.
Here is how the tests look like:
<script src="https://gist.github.com/RadekMolenda/63aa17cf615d796e2f6b.js"></script>

and the implementation is very simple
<script src="https://gist.github.com/RadekMolenda/549099e826ded3108128.js"></script>

That's it! Really the final BasicAuth module looks like that:
<script src="https://gist.github.com/RadekMolenda/2b10746b1e0b96905a0c.js"></script>
And tests:
<script src="https://gist.github.com/RadekMolenda/169981d4954cefcd0b6c.js"></script>

## Conclusion
Just as rack in Rails the plug concept in Phoenix is simple but very powerful. Today we have learnt not only the basics of the plug module, but also how to implement quite useful basic authentication feature from scratch. I hope you enjoyed it.

The code could be found on github [https://github.com/RadekMolenda/basic_auth_exercise](https://github.com/RadekMolenda/basic_auth_exercise)

Happy Coding!
