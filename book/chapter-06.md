Tasks in Rust
=============

One of the things that Rust is super good at is concurrency. In order to
understand Rust's strengths, you have to understand its approach to
concurrency, and then its approach to memory. The two are inter-related,
and we can't keep glossing over details like "just put `~` in front of
things" anymore.

Tasks
-----

The fundamental unit of computation in Rust is called a 'task.' Tasks are like
threads, but you can choose the low-level details of how they operate.  Rust
now supports both 1:1 scheduled and N:M scheduled threads as well, though the
work for 1:1 is still ongoing, so N:M is the default.  The details of what
_exactly_ that means are out of the scope of this tutorial, but the [Wikipedia
page](http://en.wikipedia.org/wiki/Thread_%28computing%29) has a good overview.

Here's some code that prints "Hello" 500 times:

~~~ {.rust}
    use std::io::println;

    fn main() {
        for num in range(0, 500) {
            println("Hello");
        }
    }
~~~

You may remember this from earlier. This loops 500 times, printing
"Hello." Now let's make it roflscale with tasks:

~~~ {.rust}
    use std::io::println;

    fn main() {
        for num in range(0, 500) {
            spawn(proc() {
                println("Hello");
            });
        }
    }
~~~

That's it! We spin up 500 tasks that print stuff. If you inspect your
output, you can tell it's working:

    Hello
    HelloHello

    Hello

Ha! Printing to the screen is obviously something that tasks can step
over each other with (if you're curious, it's because it is printing the
string and the newline separately. Sometimes, another task gets to print
its string before this task prints its newline). But the vast majority
of things aren't like that. Let's take a look at the type signature of
`spawn`:

~~~ {.rust}
    fn spawn(f: proc())
~~~

Spawn is a function that takes a pointer to another function (it's a
higher order function). But there's that `~` again. This means that the
pointer is an 'owned pointer.' We'll talk more about what exactly that
means in the next chapter, but you can infer from the name that this
means that we own all of the references to the data in this closure.
Therefore, Rust can move the function around at will and know it won't
break anything. The type system has helped us determine exactly how
isolated our task actually is.

Pipes, Channels, and Ports
--------------------------

If our tasks are 100% isolated, they wouldn't be that useful: we need
some kind of communication between tasks in order to get back useful
results. We can communicate between tasks with pipes. Pipes have two
ends: a channel that sends info down the pipe, and a port that receives
info. Here's an example of a task that sends us back a 10:

~~~ {.rust}
    use std::io::println;

    fn main() {
        let (chan, port) = channel();

        spawn(proc() {
            chan.send(10);
        });

        println(port.recv().to_str());
    }
~~~

The `channel` function, imported by the prelude, creates both sides of this
pipe. You can imagine that instead of sending 10, we might be doing some sort
of complex calculation. It could be doing that work in the background while we
did more important things.

What about that `chan.send` bit? Well, the task captures the `chan`
variable we set up before, so it's just matter of using it. This is
similar to Ruby's blocks:

~~~ {.ruby}
    foo = 10
    2.times do
      puts foo
    end
~~~

This is really only one-way transit, though: what if we want to
communicate back and forth? Setting up two ports and channels each time
would be pretty annoying, so we have some standard library code for
this: `DuplexStream`:

~~~ {.rust}
    extern crate sync;
    use std::io::println;

    fn plus_one(channel: &sync::DuplexStream<int, int>) {
        let mut value: int;
        loop {
            value = channel.recv();
            channel.send(value + 1);
        }
    }

    fn main() {
        let (from_child, to_child) = sync::duplex();

        spawn(proc() {
            plus_one(&to_child);
        });

        from_child.try_send(22);
        from_child.try_send(23);
        from_child.send(24);
        from_child.send(25);

        for num in range(0, 4) {
            let answer = from_child.recv();
            println(answer.to_str());
        }
    }
~~~


What's this `extern crate` madness? Well, that's how we `link` to
external libraries. If you've used C or C++ before, you know what this
means. If you haven't, it's essentially how you declare that your
program uses a certain dynamic library (`.dll` on Windows, `.dylib` on
OS X, and `.so` on other Unix systems). `sync` is part of Rust itself,
it includes extras as compared to `std` (which is automatically included
in every program), such as JSON parsing, networking, and data
structures. See all of the directories starting with 'lib'
<https://github.com/mozilla/rust/tree/master/src> for more.

We make a function that just loops forever, gets an `int` off of the
port, and sends the number plus 1 back down the channel. In the main
function, we make a `DuplexStream`, send one end to a new task, and then
send it a `22`, and print out the result. Because this task is running
in the background, we can send it bunches of values:

~~~ {.rust}
    fn main() {
        let (from_child, to_child) = DuplexStream::new();

        do spawn {
            plus_one(&to_child);
        };

        from_child.send(22);
        from_child.send(23);
        from_child.send(24);
        from_child.send(25);

        for num in range(0, 4) {
            let answer = from_child.recv();
            println(answer.to_str());
        }
    }
~~~

Pretty simple. Our task is always waiting for work. If you run this,
you'll get some weird output at the end:

    $ rustc tasks.rs && ./tasks
    23
    24
    25
    26
    task '<unnamed>' failed at 'receiving on a closed channel', /home/steveklabnik/src/rust/src/libstd/comm/mod.rs:728


`task failed at 'receiving on closed channel'`. Basically, we quit the program
without closing our child task, and so it died when our main task (the one
running `main`) died. By default, Rust tasks are bidirectionally linked, which
means if one task fails, all of its children and parents fail too. We can fix
this for now by telling our child to die:

~~~ {.rust}
    extern crate sync;
    use std::io::println;

    fn plus_one(channel: &sync::DuplexStream<int, int>) {
        let mut value: int;
        loop {
            value = channel.recv();
            channel.send(value + 1);
            if value == 0 { break; }
        }
    }

    fn main() {
        let (from_child, to_child) = sync::duplex();

        spawn(proc() {
            plus_one(&to_child);
        });

        from_child.try_send(22);
        from_child.try_send(23);
        from_child.send(24);
        from_child.send(25);

        from_child.send(0);

        for num in range(0, 4) {
            let answer = from_child.recv();
            println(answer.to_str());
        }
    }
~~~

Now when we send a zero, our child task terminates. If you run this,
you'll get no errors at the end. We can also change our failure mode.
Rust also provides unidirectional and unlinked failure modes as well,
but I don't want to talk about them right now. This would give you
patterns like "Spin up a management task that is bidirectionally linked
to main, but have it spin up children who are unlinked." Neato.

Rust tasks are so lightweight that you can conceivably spin up a ton of
tasks, maybe even one per entity in your system.
[Servo](https://github.com/mozilla/servo) is a prototype browser
rendering engine from Mozilla, and it spins up a **ton** of tasks.
Parallel rendering, parsing, downloading, everything.

I'm imagining that most production Rust programs will eventually have a
main that spins up some sort of global task setup, and all the work gets
done inside these tasks that communicate with each other. Like, for a
video game:

~~~ {.rust}
    fn main() {

        spawn(proc() {
            player_handler();
        });

        spawn(proc() {
            world_handler();
        });

        spawn(proc() {
            rendering_handler();
        });

        spawn(proc() {
            io_handler();
        });
    }
~~~

... with the associated channels, of course. This feels very Actor-y to
me. I like it. In fact, someone *is* working on an Actor
[library](http://www.reddit.com/r/rust/comments/1i3c15/experimental_actor_library_in_rust/)!
We'll see how these kinds of things develop as Rust moves forward. For more,
the [tasks and communication
tutorial](http://static.rust-lang.org/doc/0.10/guide-tasks.html) is helpful.
