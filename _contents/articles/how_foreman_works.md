---
title: How Foreman Works?
date: '2022-08-27'
---
[Foreman](https://github.com/ddollar/foreman), introduced in [2011](http://blog.daviddollar.org/2011/05/06/introducing-foreman.html) is a simple [Procfile](https://devcenter.heroku.com/articles/procfile)-based process manager written in Ruby.

With a Procfile like this:

```sh
# Profile
web_a: bundle exec ruby app_a.rb
web_b: bundle exec ruby app_b.rb
service_a: go run service_a.go
```

And a single command: `foreman start`, you can run multiple background processes (3 in this case) with a foreground monitoring process (the one attached to the terminal) that multiplexes the outputs of those processes (with color-coded) into the `STDOUT` and monitoring for signals:

![Foreman Start](/assets/articles/how_foreman_works/start.png)

And when we finished, we can send `SIGINT`, `SIGTERM`, or `SIGHUP` to the foreground process, and it will send `SIGTERM` to terminate all processes gracefully (provided that those processes also trap `SIGTERM` and terminate themselves gracefully):

![Foreman Terminate](/assets/articles/how_foreman_works/terminate.png)

Wonderful.

Alas, there is little documentation of how it actually works. Are you curious of how a simple program with core logic with [<300 LOC](https://github.com/ddollar/foreman/blob/master/lib/foreman/engine.rb) is capable of handling multiple background processes for you?

So, here we go.

> **A SIDE NOTE**
>
> Nowadays, new folks don't hear much about Process Management, mainly because I think containers are already a norm. Nevertheless, I still think this is a fun exploration.  

-

> **TL;DR**
> 
> If you are not insterested on how the code works internally, just skip to the end where I present a diagram.

## The Core Logic

After running `foreman start`, Foreman will attempt to read `Procfile` and an optional `.env` file (for setting environment variables in all processes), parse processes in Procfile, assign them into `@processes` variable, and then start the core logic.

The core logic is simple, because Foreman is written [eloquently](https://www.amazon.com/Eloquent-Ruby-Addison-Wesley-Professional/dp/0321584104), we can easily follow its main logic in `engine.rb`:

```ruby
# Start the processes registered to this +Engine+
#
def start
  register_signal_handlers
  startup
  spawn_processes
  watch_for_output
  sleep 0.1
  wait_for_shutdown_or_child_termination
  shutdown
  exit(@exitstatus) if @exitstatus
end
```

* `register_signal_handlers` traps signals: `TERM, INT, HUP` for later all processes termination and traps `USR1, USR2` for forwarding controlling signals to all background processes. All other signals are discarded.
* `startup` assigns names and colors for outputs of the background processes
* `spawn_processes` spawns the processes, for how it does we can just look at the code:

```ruby
def spawn_processes
  @processes.each do |process|
    1.upto(formation[@names[process]]) do |n|
      reader, writer = IO.pipe
      begin
        pid = process.run(:output => writer, :env => {
          "PORT" => port_for(process, n).to_s,
          "PS" => name_for_index(process, n)
        })
        writer.puts "started with pid #{pid}"
      rescue Errno::ENOENT
        writer.puts "unknown command: #{process.command}"
      end
      @running[pid] = [process, n]
      @readers[pid] = reader
    end
  end
end
```

(For `formation`, it is just a way to tell how many instances for each defined process type to run).

Basically it uses `Foreman::Process#run()`, which in turns use the standard lib's [::Process::spawn](https://ruby-doc.org/core-3.0.2/Process.html#method-c-spawn), which in its most basic usage, given the environment variables and a command, it will start a background child process that has its output reassigned to the `writer` end of a [pipe](https://ruby-doc.org/core-3.1.2/IO.html#method-c-pipe) which connects to its `reader` end, its process group set to the main Foreman foreground process (which is also its parent process), and run that given command under specified environment variables.

Any output that the spawned background child process normally outputs to the `STDOUT` will go to its `writer` instead. And can be read by reading through the `reader` pipe.

Continue to the rest of the core logic:

* `watch_for_outputs` Foreman will start a thread that continuously reads `@readers` pipes for outputs and a self-pipe for signals received on the main thread (more on this later). If any of `@readers` or the self-pipe has a value, it will output that value into `STDOUT` with its assigned process name, color, and also will timestamp the output, or will handle signals, respectively.
* `wait_for_shutdown_or_child_termination`, here is where its child processes monitoring loop happens:

```ruby
def wait_for_shutdown_or_child_termination
  loop do
    # Stop if it is time to shut down (asked via a signal)
    break if @shutdown

    # Stop if any of the children died
    break if check_for_termination

    # Sleep for a moment and do not blow up if any signals are coming our way
    begin
      sleep(1)
    rescue Exception
      # noop
    end
  end

  # Ok, we have exited from the main loop, time to shut down gracefully
  terminate_gracefully
end
```

Any of `INT`, `TERM`, `HUP` signals received, will make `@shutdown = true` and thus will break the loop. Or if any child process has terminated (checked via `check_for_termination` -- which calls [::Process::wait2](https://ruby-doc.org/core-3.0.2/Process.html#method-c-wait2)), it will also break the loop.

Breaking the loop will make the main thread calls `terminate_gracefully`, which sends `TERM` signal to all child processes. Then if all the background child processes haven't terminated yet and a timeout (default is 5 seconds) happened, it will just send `KILL` signal to kill all children immediately.

Then Foreman will shutdown.

* `shutdown` currently does nothing.
* `exit` exits with exit status, possibly set to those of its children. The end.

## Signal handling and the Self-Pipe Tricks

Handling signals is a pain (both in UNIX and Windows). Any signal can be trapped and handled with our own custom logic, but while that signal is being handled, a new signal can be sent and the signal handling will be interrupted, possibly again and again.

The [UNIX self-pipe trick](https://cr.yp.to/docs/selfpipe.html) is a hack that can be roughly explained like so:

* The main thread creates a [pipe](https://man7.org/linux/man-pages/man2/pipe.2.html), a unidirectional data channel that usually is created for interprocess communication, but in this case will be used for its own internal process communication, thus called a **self-pipe**, which has `reader` and `writer` ends
* The main thread spawns a monitoring thread that monitors the `reader` end of the self-pipe
* The main thread creates a **single signal queue**. All interested signals will be trapped and put into this single queue, and notify the monitoring thread by writing a byte to the `writer` end of the self-pipe
* The monitoring thread will get notified via the `reader` end (perhaps with [select(2)](https://man7.org/linux/man-pages/man2/select.2.html)), and will process signals in the queue with the custom logic, then returns to its normal job, notify main thread to do something (maybe by setting a specific variable), or shutdown

If this still sounds confused, perhaps [this article](https://www.sitepoint.com/the-self-pipe-trick-explained/) may elaborate better.

## Diagram of How Foreman Works

Here we conclude how Foreman works with a diagram.

![Foreman Diagram](/assets/articles/how_foreman_works/foreman.png)

If you find any mistakes, feel free to notify [me](https://sarans.co).

Till next time!