---
layout: post
date: 2016-02-21 22:00:00 CET
title: Playing Swarm Simulator with a bot in Elixir (part 2 - some OTP)
tags: [Elixir, Swarm Simulator, tutorial]
comments: true
---

Hello! Welcome to the next part of [Swarm Simulator](https://swarmsim.github.io/#/) tutorial in Elixir. This time I would like to focus on stability and we will do it by:

* rewriting the bot to become an OTP Application using GenServer and Supervisor
* using Supervisor features to automatically recover the game after it crashes
* introducing Save/Load functionality which should be useful for restoring the game state

There will be also some refactoring, and test removing due to `phantomjs` session management issues.
If you are finding some of the parts confusing, maybe [the first part of this tutorial](http://radekmolenda.github.io/2016/02/02/swarm-simulator-bot-in-elixir-part-1.html) could help you.

Full source of the bot can be found on Github: [https://github.com/RadekMolenda/SwarmSimulatorBOT](https://github.com/RadekMolenda/SwarmSimulatorBOT)

## Writing the OTP application

Let's start with moving the swarm url to app config:
<script src="https://gist.github.com/RadekMolenda/5dac35c4b672cd3627fd.js"></script>

Then let's add `use GenServer` line to `lib/swarmsimulatorbot.ex` and rename the  `start/0` function to `start_link/0` to emphasize we will be starting a linked process.
<script src="https://gist.github.com/RadekMolenda/f7cc84528e5d18036886.js"></script>
A good explanation about what is a `GenServer` can be found on [http://elixir-lang.org/docs/v1.2/elixir/GenServer.html](http://elixir-lang.org/docs/v1.2/elixir/GenServer.html)
> A GenServer is a process as any other Elixir process and it can be used to keep state, execute code asynchronously and so on

Basically we will use `GenServer` to help us clarify the module API. Previously we had to use `send` and `receive` which was simply to verbose. Ideally we would like to call functions more or less like `Swarmsimulatorbot.dummy_grow` or `Swarmsimulatorbot.save` and let the module internals do the right job.

In `Swarsimulatorbot.start_link/0` we have replaced `spawn_link/3`, with `GenServer.start_link/3` function. The `GenServer.start_link/3` accept three arguments and start process linked to the current process. Linked process means the current process will exit in case of crashes. The `__MODULE__` refers to the current module. The second argument can be anything (and we won't use it). The third argument is a keyword list of options. Passing a `name: __MODULE__` is a smart way of naming a process - `__MODULE__` is just a string. We will use the name of the GenServer process later to send messages to our server.

### Manual testing with chromedriver
Note that from now on the unit tests for the project are broken. I had some issues with using phantomjs as it doesn't remove the session between tests. This is why the test were failing randomly. After struggling with it for a while I decided to remove tests and simply carry on with manual testing. The most convenient way was to change `phantomjs` driver with `chrome_driver` in `config/config.exs` directory - which allowed me to see how the bot behaves. I also had make sure the [chromedriver](https://sites.google.com/a/chromium.org/chromedriver/downloads) is running before launching the commands. After manual testing I switched back to `phantomjs` - this is why you won't see this change in the code.

### Swarmsimulator - a GenServer

We need to make sure the `GenServer` callbacks are implemented properly. Let's rewrite the `Swarmsimulatorbot.init` function to match [expected init/1 declaration](http://elixir-lang.org/docs/stable/elixir/GenServer.html#c:init/1).
<script src="https://gist.github.com/RadekMolenda/113e06d331b5fb064bb7.js"></script>
This function will be called when we are spawning the process. `GenServer` expect us to return tuple like: `{ :ok, state }` the state is not relevant in our case.

We are ready to remove the private `loop/0` function now:
<script src="https://gist.github.com/RadekMolenda/c3721bfb1635a60e0bd7.js"></script>

### `Swarmsimulatorbot.dummy_grow/0` rewrite

And now the interesting part! The `dummy_grow/0` has been changed to call the `GenServer.call/3` function while the actual "clicking" was moved to `handle_call/3` callback function.
<script src="https://gist.github.com/RadekMolenda/6cb5f0767b9973b81fd6.js"></script>
We are passing 3 arguments to `GenServer.call/3`:

* The name of the process: `__MODULE__` in our case.
* The name of the call: `:dummy_grow` which will be used to correctly identify the `handle_call/3` callback by pattern matching.
* The last argument is the timeout. Passing `:infinity` means we don't want to timeout at all. There are also some minor changes that clarify the module api.

It's worth mentioning the `handle_call/3` callback needs to return a 3 element tuple - which basically allows to change the state between the calls. The only thing we care about in our case are side effects. In fact we wouldn't need to return anything from this function to execute the `dummy_grow` properly it's just GenServer that forces us to return such a data structure from the callback. Returning `nil`, and `state` in the tuple should work just fine.

### `Swarmsimulatorbot.screenshot/1` rewrite

The screenshot rewrite follows the pattern similar to previous section.
<script src="https://gist.github.com/RadekMolenda/9065499859e2d482b953.js"></script>
Passing a `{:screenshot, path}` tuple rather than an atom to `GenServer.call/3` gives us opportunity to add some extra arguments to the callback function. I have removed `units/0` as it is not needed for swarm growing at all.

### Save/Load feature
Let's implement the Save/Load functionality. It actually is quite simple - we will introduce two new functions: `save/0` and `load_game/0` both of them will follow the same pattern I used for screenshot and dummy_grow functions. The functions will call corresponding `GenServer.call/3` and we will perform some "clicking" in `handle_call` callbacks. This will be a very basic functionality, based on the Swarm Simulator [options tab](https://swarmsim.github.io/#/options). If you navigate there you should be able to find the **"Import/export saved data"** input field quickly. The value of this input field changes when your swarm grows: the serialized app data is stored there. `save/0` function will save the input value to a file, `load_game/0` will use the file contents to populate the Import/Export input field. The idea I have is to trigger `save/0` at the beginning of every "dummy_grow" iteration and use load_game whenever something goes wrong.
<script src="https://gist.github.com/RadekMolenda/086787247f379dbcbf09.js"></script>
Most of the changes are quite self-explanatory. Some private functions have been introduced to increase the readability. We are using the `save/save.dat` file so please make sure the `save` folder is created before running the code. If you look at the `import_to_game/1` function you will see a weird `execute_script("$('#export').val('#{data}')")` and than `input_into_field(" ")` it's the performance optimization hack. Hounds `input_into_field/2` helper method was giving too many timeouts. Simple JS execution and calling `input_into_field(" ")` afterwards to trigger angularJS model change workes way faster.

And that's it for the heart of our application rewrite. Right now you should be able to start the app using `iex -S mix` and play with it for a while
{% highlight elixir %}
Interactive Elixir (1.2.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> Swarmsimulatorbot.start_link
{:ok, #PID<0.157.0>}
iex(2)> Swarmsimulatorbot.dummy_grow
nil
iex(3)> Swarmsimulatorbot.dummy_grow
nil
iex(4)> Swarmsimulatorbot.screenshot("hello-part-2.png")
nil
iex(5)> Swarmsimulatorbot.save
nil
iex(6)> Swarmsimulatorbot.load_game
nil
{% endhighlight %}

Unfortunately the `Swarmsimulatorbot.Cli` module is totally not useful now, as it expects totally different `Swarmsimulatorbot` interface (or let's call it api).

### Fixing `Cli`
We will fix the `Cli` by rewriting it to... `GenServer`. Even though `Cli` purpose differs from `Swarmsimulatorbot` purpose GenServer is generic enough to help us in resolving current `Cli` problems. Let's remind that we are using `Cli` to periodically send messages to `Swarmsimulatorbot` module.
<script src="https://gist.github.com/RadekMolenda/9c897351ca0452efa3f7.js"></script>
Basically we are reusing the patterns already introduced in `Swarmsimulatorbot` module. We are loading the game in `init/1` callback which makes sense - we want to load the game if it has been saved. then we are using `Process.send_after/3` rather than `GenServer.call/3` - basically I followed the [stackoverflow example](http://stackoverflow.com/questions/32085258/how-to-run-some-code-every-few-hours-in-phoenix-framework) on how to run the code periodically and this solution works perfectly.
The `handle_info/2` callback is used when the message has not been sent from `GenServer` and `self()` references to current PID. Please mind that all the calls are synchronous and each function call will wait for previous to finish. In the end we are using `Process.send_after(self(), :tick, @tick)` to send message to the same handle_info callback after one second - that's kinda recursive.

## Building OTP application and supervision tree
Elixir is build on top of Erlang, this gives us a great opportunity to improve the bot stability by introducing supervisors and running the app as OTP application. We don't know what might happen out there in a wild - a process might crash unexpectedly for example. But Erlang has a solution to this problem. We will use supervisors to look after our process and in case of exception we will respawn them. The GenServer callbacks will allow us to load the game on it's initialization and that is truly amazing.

The only bit that is missing is a supervisor. The supervision tree is very simple we will use one supervisor to look after two worker processes: `Swarmsimulatorbot` and `Swarmsimulatorbot.Cli`. Due to using the `GenServer` for previous rewrite implementing the supervisor should be trivial! Let's have a quick look at [elixir lang example](http://elixir-lang.org/getting-started/mix-otp/supervisor-and-application.html) and write the following new file:
<script src="https://gist.github.com/RadekMolenda/1673d06ac977494a9f8c.js"></script>
Changes in `mix.exs` helps us to wire all the things up and allow to start the application automatically when run `iex -S mix`. The rest is very straightforward `Swarmsimulatorbot.Sup` module implements the `start/2` function which is required by `use Application` directive. We are spawning two worker processes at the beginning `Swarmsimulatorbot` and `Swarmsimulatorbot.Cli` and allow our Supervisor to look after the process using `:one_to_one` strategy it means the process will be replaced by a new process in case it crashes.

And that's basically it for this part. We are ready to start our process and we can do it in three ways:
<pre>
{% highlight bash %}
# start the app with iex console
$ iex -S mix
# start the app in terminal
$ mix run --no-halt
# start detached app
$ elixir --detached -S mix run --no-halt
{% endhighlight %}
</pre>

## Here's how the swarm looks like after about three days of bot running
I think it's nice and fat:
![Swarm is fat and growing](/img/growing-fat.png "Swarm is fat and Growing")
Happy Swarming!
