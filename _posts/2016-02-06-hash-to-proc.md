---
layout: post
date: 2016-02-06 21:00:00 CET
title: How to use Hash#to_proc in Ruby?
tags: [Ruby, Ruby2.3, Hash#to_proc, Tip of a day]
comments: true
---

Hash#to_proc has been introduced in [Ruby 2.3.0 release](https://www.ruby-lang.org/en/news/2015/12/25/ruby-2-3-0-released/). I really like this method. It reminds obtaining a value from a HashMap in Clojure.

<pre>
{% highlight clojure %}
=> ({:a 1} :a)
1
{% endhighlight %}
</pre>

We simply "call" hash-map with `:a` keyword as a parameter to get the `:a`'s keyword value. Thanks to `Hash#to_proc` method, in Ruby now we can do almost the same thing.

<pre>
{% highlight ruby %}
irb(main):001:0> {a: 1}.to_proc.call(:a)
=> 1
{% endhighlight %}
</pre>

Of course for getting a value from hash it's much easier to use `#[]` but sometimes `#to_proc` is just as useful and allows you to simplify the code.
### Here's how I use it in wildlife

Let's say I'm not sure what the name of the key is, but what I really need to know is the value.
<pre>
{% highlight ruby %}
irb(main):001:0> ENV.keys.grep(/shell/i).map(&ENV.to_hash)
=> ["/usr/local/opt/bash/bin/bash", "bash"]
irb(main):002:0> ENV.keys.grep(/shell/i)
=> ["SHELL", "RBENV_SHELL"]
{% endhighlight %}
</pre>

Rails console using ActiveRecord::AttributeMethods#attributes method.
<pre>
{% highlight ruby %}
[1] (main)> attrs = User.last.attributes
[2] (main)> attrs.keys.grep(/name/).map(&attrs)
=> ["Radek", "Molenda"]
{% endhighlight %}
</pre>

Before Ruby 2.3.0 you would need to do something like that:
<pre>
{% highlight ruby %}
[3] (main)> keys = attrs.keys.grep(/name/)
=> ["first_name", "last_name"]
[4] (main)> attrs.values_at(*keys)
=> ["Radek", "Molenda"]
{% endhighlight %}
</pre>

Happy coding!
